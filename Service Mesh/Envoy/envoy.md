#### Envoy

##### Envoy概念，什么是Envoy？

>   Envoy is an L7 proxy and communication bus designed for large modern service oriented architectures.
>
>   L7 service proxy、通信总线
>
>   
>
>   高级特性：
>
>   -   L3、L4层 network fileter
>   -   支持各种高级协议
>   -   支持服务发现、路由检查
>   -   负载均衡
>   -   动态、监控、trace
>   -   

##### Envoy 拓扑结构

##### Envoy xDs API

>   v1：json/rest
>
>   V2：proto，grpc
>
>   通用数据平面API

##### Envoy 部署类型

>   Ingress
>
>   Engress

##### Envoy 核心配置组件

>   -   Host：
>   -   Downstream——下游（连接envoy，发送请求，接收响应）
>   -   Upstream——上游（接收来自envoy的请求返回响应）
>   -   Listerner
>   -   Filter
>   -   Cluster
>   -   Mesher
>   -   runtime config——运行时配置

##### 线程模型和连接处理机制

>   单进程多线程架构：
>
>   一个主线程控制各种零星的协调任务
>
>   工作线程负责监听、过滤、转发。一旦一个链接被一个Listner接收，该链接剩余的声明周期内一直绑定改工作线程
>
>   