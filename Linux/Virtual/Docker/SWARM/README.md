#### 环境
```txt
	Host-A	--->	Master
	Host-B 	--->	Node-1
	Host-C 	--->	Node-2
```
#### 说明
```txt
Swarm支持设置一组Manager Node，通过支持多Manager Node实现HA
Docker 1.12中Swarm已内置了服务发现工具，不再需要像以前使用 Etcd 或 Consul 这些工具来配置服务发现
对于容器来说若没有外部通信但又是运行中的状态会被服务发现工具认为是 Preparing 状态，但若映射了端口则会是 Running 状态。
docker service [ls/ps/rm/scale/inspect/update]

Swarm使用Raft协议保证多Manager间状态的一致性。基于Raft协议，Manager Node具有一定容错功能（可容忍最多有(N-1)/2个节点失效）
每个Node的配置可能不同，比如有的适合CPU密集型应用，有的适合运行IO密集型应用
Swarm支持给每个Node添加标签元数据，这样可根据Node标签来选择性地调度某个服务部署到期望的一组Node上
```
#### 各节点加入swarm集群
```bash
#master节点创建集群并将其他节点加入swarm集群中
[root@host-a ~]# docker swarm init --advertise-addr 192.168.0.3     #IP地址为本节点在集群中对外的地址
Swarm initialized: current node (c4aa18akyid7wctl4e0hpbqmr) is now a manager.
To add a worker to this swarm, run the following command:           #下列输出说明如何将Worker Node加入到集群

    docker swarm join \
    --token SWMTKN-1-17d18kwcn6mef2usiz7p7d38txo6az4rrdxqzxtwdi9qvmrxwx-48hhk1wzm51sjcktwnbnm7qgl \
    192.168.0.3:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.

[root@host-b ~]# docker swarm join \
> --token SWMTKN-1-17d18kwcn6mef2usiz7p7d38txo6az4rrdxqzxtwdi9qvmrxwx-48hhk1wzm51sjcktwnbnm7qgl \
> 192.168.0.3:2377
This node joined a swarm as a worker.

[root@host-c ~]# docker swarm join \
> --token SWMTKN-1-17d18kwcn6mef2usiz7p7d38txo6az4rrdxqzxtwdi9qvmrxwx-48hhk1wzm51sjcktwnbnm7qgl \
> 192.168.0.3:2377
This node joined a swarm as a worker.

[root@host-a ~]# docker node ls   #查看swarm集群节点信息
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
3fcyk6uf3l161uk6p3x1xwpsv    host-b    Ready   Active        
ags2l01cijtxux8kq0ft4mz8l    host-c    Ready   Active        
c4aa18akyid7wctl4e0hpbqmr *  host-a    Ready   Active        Leader
#Active：该Node可被指派Task
#Pause： 该Node不可被指派新Task，但其他已存在的Task保持运行（暂停一个Node后该其不再接收新的Task）
#Drain： 该Node不可被指派新Task，Swarm Scheduler停掉已存在的Task并将它们调度到可用Node上（进行停机维护时可修改\
         AVAILABILITY为Drain状态）

#查看Node状态
[root@host-a ~]# docker node inspect self
[
    {
        "ID": "c4aa18akyid7wctl4e0hpbqmr",
        "Version": {
            "Index": 10
        },
        "CreatedAt": "2017-12-29T16:04:28.575482252Z",
        "UpdatedAt": "2017-12-29T16:04:28.590768171Z",
        "Spec": {
            "Role": "manager",
            "Availability": "active"
        },
        "Description": {
            "Hostname": "host-a",
            "Platform": {
                "Architecture": "x86_64",
                "OS": "linux"
            },
            "Resources": {
                "NanoCPUs": 4000000000,
                "MemoryBytes": 2082357248
            },
            "Engine": {
                "EngineVersion": "1.12.6",
                "Plugins": [
                    {
                        "Type": "Network",
                        "Name": "bridge"
                    },
                    {
                        "Type": "Network",
                        "Name": "host"
                    },
                    {
                        "Type": "Network",
                        "Name": "null"
                    },
                    {
                        "Type": "Network",
                        "Name": "overlay"
                    },
                    {
                        "Type": "Volume",
                        "Name": "local"
                    }
                ]
            }
        },
        "Status": {
            "State": "ready"
        },
        "ManagerStatus": {
            "Leader": true,
            "Reachability": "reachable",
            "Addr": "192.168.0.3:2377"
        }
    }
]
#在Mater上修改其他node状态：（将Node的AVAILABILITY值改为Drain状态，使其只具备管理功能）
[root@host-a ~]# docker node update --availability drain <Node*>

#在Mater上设置各个节点的lable标签元数据
[root@host-a ~]#  docker node update --label-add cpu_num=2 host-b
host-b
[root@host-a ~]#  docker node update --label-add disk_num=2 host-c   
host-c

#改变Node角色：（dorker Node可以变为Manager Node，这样实际Worker Node由工作Node变成了管理Node）
[root@host-a ~]# docker node promote  <node*>		#提权
[root@host-a ~]# docker node demote  <node*>		#降权
[root@host-a ~]#  docker swarm node leave [--force]	#退出所在集群
```
#### 流程
```bash
#创建Overlay网络（创建Overlay网络my-network后集群中所有的Manager都可访问。以后在创建服务时只要指定使用的网络即可）
[root@host-a ~]# docker network create -d overlay --subnet=10.0.9.0/24  my-network
a88ql269d3t1ev1yq4n828yut

[root@host-a ~]# docker pull docker.io/bashell/alpine-bash     #swarm集群内各节点下载好镜像先...
[root@host-b ~]# docker pull docker.io/bashell/alpine-bash     #
[root@host-c ~]# docker pull docker.io/bashell/alpine-bash     #

#创建bash容器的服务，服务名为t1，使其同时能运行在2个节点上（必须在Manager操作）
#若Swarm集群中其他Node的容器也使用my-network这个网络，那么处于该网络中的所有容器间均可连通！
[root@host-a ~]# docker service create --replicas 2 --network my-network  --name t1  docker.io/bashell/alpine-bash      
9ye311ixoipa4zt372c5smx5i
[root@host-a ~]# docker service ps t1   	#查看t1服务状态
ID                         NAME      IMAGE                          NODE    DESIRED STATE  CURRENT STATE             ERROR
0f0eiadl2npvvfm3qi8jincvn  t1.1      docker.io/bashell/alpine-bash  host-c  Running        Preparing 16 seconds ago  
brz7bfagxyzkmqlcte3h7i6r3  t1.2      docker.io/bashell/alpine-bash  host-b  Running        Preparing 8 seconds ago   
dfj51c5jfbbl25rb59e8zts3d   \_ t1.2  docker.io/bashell/alpine-bash  host-a  Shutdown       Complete 8 seconds ago 
[root@host-a ~]# docker service rm t1   	#删除t1服务

[root@host-a ~]# docker service ls      	#查看集群服务列表
ID            NAME       REPLICAS  IMAGE           COMMAND
436wxwxfb7je  test_bash  0/2       docker.io/bash  
9cziqd3bxk96  bash_test  0/1       docker.io/bash 

[root@host-a ~]# docker service ps test_bash    #查看集群服务的信息
ID                         NAME             IMAGE           NODE    DESIRED STATE  CURRENT STATE             ERROR
7hooklz0thqordkdbdiypyiih  test_bash.1      docker.io/bash  host-a  Running        Preparing 27 minutes ago  
eem9x7ktt4ome0rbgf05e6f45   \_ test_bash.1  docker.io/bash  host-a  Shutdown       Complete 27 minutes ago   
elm590j5t2jmmvfewqgr7g78z  test_bash.2      docker.io/bash  host-b  Running        Preparing 27 minutes ago 

#查看集群服务的详细信息
[root@host-a ~]# docker service inspect test_bash
[
    {
        "ID": "436wxwxfb7jefm10h0fdsp2ro",
        "Version": {
            "Index": 45
        },
        "CreatedAt": "2017-12-29T16:20:14.854181266Z",
        "UpdatedAt": "2017-12-29T16:20:14.856143634Z",
        "Spec": {
            "Name": "test_bash",
            "TaskTemplate": {
                "ContainerSpec": {
                    "Image": "docker.io/bash"
                },
                "Resources": {
                    "Limits": {},
                    "Reservations": {}
                },
                "RestartPolicy": {
                    "Condition": "any",
                    "MaxAttempts": 0
                },
                "Placement": {}
            },
            "Mode": {
                "Replicated": {
                    "Replicas": 2
                }
            },
            "UpdateConfig": {
                "Parallelism": 1,
                "FailureAction": "pause"
            },
            "Networks": [
                {
                    "Target": "a88ql269d3t1ev1yq4n828yut"
                }
            ],
            "EndpointSpec": {
                "Mode": "vip"
            }
        },
        "Endpoint": {
            "Spec": {
                "Mode": "vip"
            },
            "VirtualIPs": [
                {
                    "NetworkID": "a88ql269d3t1ev1yq4n828yut",
                    "Addr": "10.0.9.4/24"
                }
            ]
        },
        "UpdateStatus": {
            "StartedAt": "0001-01-01T00:00:00Z",
            "CompletedAt": "0001-01-01T00:00:00Z"
        }
    }
]
```
#### 扩容缩容及滚动更新
```bash
#Swarm支持服务扩容缩容，通过--mode设置服务类型，提供了两种模式
#replicated：	指定服务Task的个数（需要创建几个冗余副本），这也是Swarm默认使用的服务类型
#global：	在Swarm集群的每个Node上都创建一个服务！
		
#服务扩容缩容：在Manager Node上执行（将前面部署的2个副本的myredis服务，扩容到3个副本）
#docker service scale <ID>=<Task>	
[root@host-a ~]# docker service scale myredis=3		
	#查看服务信息：docker service ls
	#ID            NAME    MODE        	REPLICAS  IMAGE
	#kilpacb9uy4q  myapp   replicated  	1/1       alpine:latest
	#vf1kcgtd5byc  myredis replicated  	3/3       redis

	#查看指定服务在各副本的状态：docker service ps myredis
	#ID            NAME       IMAGE  	NODE     	DESIRED     STATE  	CURRENT STATE     ERROR  PORTS
	#0p3r9zm2uxpl  myredis.1  redis  	manager  	Running     Running 	14 minutes ago                
	#ty3undmoielo  myredis.2  redis  	worker1  	Running     Running 	14 minutes ago                
	#zxsvynsgqmpk  myredis.3  redis  	worker2  	Running     Running 	less than a second ago
	#可以看到目前3个Node的Swarm集群，每个Node上都有个myredis服务的副本，可见也实现了很好的负载均衡
	
	#缩容时只需将副本数小于当前应用服务拥有的副本数即可实现！大于指定缩容副本数的副本会被删除
	#若需删除所有服务，只需在Manager Node上执行：
	#docker service rm <服务ID>
		
#服务滚动更新：
[root@host-a ~]# docker service create  --replicas 3  --name redis  --update-delay 10s  redis:3.0.6
#通过--update-delay表示需更新的服务每成功部署1个延迟10秒后再更新下1个。若更新失败则调度器会暂停本次服务的部署更新
	
#更新已部署的服务所在容器中使用的Image的版本：
[root@host-a ~]# docker service update --image redis:3.0.7 redis:3.0.6
#将Redis服务对应的Image版本由3.0.6更新为3.0.7，同样，若更新失败则暂停本次更新。
```