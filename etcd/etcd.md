##### 分布式数据存储中间件

##### 概念

> 一个用于配置共享和服务发现的键值存储系统

##### 知识点

> - Raft算法，强一致性的核心
> - Follower： Raft中的从属节点
> - Leader：Raft中的领导协调节点，用于处理数据提交
> - Candidate：候选节点 - 当follower接收Leader节点消息超时，follower会转变成Candidate
> - Node：Raft 状态机实例/节点
> - Member：etcd实例。管理对应的node节点。可处理客户端请求
> - Peer：同一个集群中的另一个Member
> - Cluster：集群 - 拥有多个etcd member
> - Lease：租期 - 关键设置的租期，过期删除
> - Watch：检测机制 - 
> - Term：任期
> - WAL：预写式日志，用于持久化存储的日志格式
> - Client：

##### 应用场景

> - 键值对存储
> - 服务注册和发现
> - 消息发布与订阅
> - 分布式锁

##### 核心架构

> 

##### etcdctl

##### etcd网关和grpc-gateway

> - Etcd网关模式 - 构建etcd集群的门户
>   - etcd网关是一个简单的TCP代理，可将网络数据转发到etcd集群
>   - 网关是无状态透明的。不会检查客户端请求、也不会干扰etcd集群
>   - 
> - grpc-gateway
>   - 针对etcd的grpc通信协议的补充
>   - 

##### grpc代理模式，实现可伸缩的etcd API

> GRPC proxy 是在gRPC层（L7）运行的无状态etcd反向代理。旨在减少核心 etcd集群上的总处理负载
>
> 

##### 共识算法

> 解决多个节点数据一致性问题
>
> paxos
>
> raft