###### k8s工作原理

###### k8s内建资源类型

> Pod
>
> deployment
>
> service

###### Operator 

工作原理

> Operator 的工作原理，实际上是利用了 Kubernetes 的自定义 API 资源（CRD），来描述我们想要部署的“有状态应用”；然后在自定义控制器里，根据自定义 API 对象的变化，来完成具体的部署和运维工作。
>
> 
>
> Operator 是描述、部署和管理 kubernetes 应用的一套机制，从实现上说，可以将其理解为 CRD 配合可选的webhook 与 controller来实现用户业务逻辑
>
> 即 Operator = CRD + webhook + controller

工作流程

> ![image-20220525092601385](/Users/banyajie/Library/Application Support/typora-user-images/image-20220525092601385.png)
>
> - 用户创建一个自定义资源（CRD）
> - apiserver 根据自己注册的一个pass列表，把该CRD的请求转发给 webhook
> - webhook一般会完成该CRD的缺省值设定和参数校验。webhook处理完成后，相应的CR会被写入数据库，返回给用户
> - 与此同时，contorller会在后台监控该自定义资源，按照业务逻辑，处理与该自定义资源相关联的操作
> - 上述处理一般会引起集群内的状态变化，controller 会监测这些关联的变化，把这些变化记录到 CRD 的状态中

开发流程

> - CRD
> - RBAC
> - 实现自定义控制器