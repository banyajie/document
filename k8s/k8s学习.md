## kubernetes是什么？

#### 概念

>   Kubernetes 是一个可移植的、可扩展的开源平台，用于管理容器化的工作负载和服务，可促进声明式配置和自动化。 Kubernetes 拥有一个庞大且快速增长的生态系统。Kubernetes 的服务、支持和工具广泛可用。

#### kubernetes提供了什么功能

>   -   **服务发现和负载均衡**
>       -   Kubernetes 可以使用 DNS 名称或自己的 IP 地址公开容器，如果进入容器的流量很大， Kubernetes 可以负载均衡并分配网络流量，从而使部署稳定。
>   -   **自动部署和回滚**
>       -   Kubernetes 允许你自动挂载你选择的存储系统，例如本地存储、公共云提供商等。
>   -   **自动部署和回滚**
>   -   **自动完成装箱计算**
>       -   Kubernetes 允许你指定每个容器所需 CPU 和内存（RAM）。 当容器指定了资源请求时，Kubernetes 可以做出更好的决策来管理容器的资源。
>   -   **自我修复**
>       -   Kubernetes 重新启动失败的容器、替换容器、杀死不响应用户定义的 运行状况检查的容器，并且在准备好服务之前不将其通告给客户端。
>   -   **密钥与配置管理**
>       -   Kubernetes 允许你存储和管理敏感信息，例如密码、OAuth 令牌和 ssh 密钥。 你可以在不重建容器镜像的情况下部署和更新密钥和应用程序配置，也无需在堆栈配置中暴露密钥。



#### Kubernetes 组件

##### 架构图

![Kubernetes 组件](https://d33wubrfki0l68.cloudfront.net/2475489eaf20163ec0f54ddc1d92aa8d4c87c96b/e7c81/images/docs/components-of-kubernetes.svg)

##### Control Plane（控制平面）

###### kube-apiserver

###### etcd

###### Kube-scheduler

>   控制平面组件，负责监视新创建的、未指定运行[节点（node）](https://kubernetes.io/zh/docs/concepts/architecture/nodes/)的 [Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/)，选择节点让 Pod 在上面运行。
>
>   调度决策考虑的因素包括单个 Pod 和 Pod 集合的资源需求、硬件/软件/策略约束、亲和性和反亲和性规范、数据位置、工作负载间的干扰和最后时限。

###### Kube-controller-manager

>   **运行[控制器](https://kubernetes.io/zh/docs/concepts/architecture/controller/)进程的控制平面组件。**
>
>   -   节点控制器（Node Controller）: 负责在节点出现故障时进行通知和响应
>   -   任务控制器（Job controller）: 监测代表一次性任务的 Job 对象，然后创建 Pods 来运行这些任务直至完成
>   -   端点控制器（Endpoints Controller）: 填充端点(Endpoints)对象(即加入 Service 与 Pod)
>   -   服务帐户和令牌控制器（Service Account & Token Controllers）: 为新的命名空间创建默认帐户和 API 访问令牌

###### Cloud-controller-manager

##### Data Plane（数据平面）

###### Kubelet

>   一个在集群中每个[节点（node）](https://kubernetes.io/zh/docs/concepts/architecture/nodes/)上运行的代理。 它保证[容器（containers）](https://kubernetes.io/zh/docs/concepts/overview/what-is-kubernetes/#why-containers)都 运行在 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) 中。
>
>   kubelet 接收一组通过各类机制提供给它的 PodSpecs，确保这些 PodSpecs 中描述的容器处于运行状态且健康。 kubelet 不会管理不是由 Kubernetes 创建的容器。

###### kube-proxy

>   [kube-proxy](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kube-proxy/) 是集群中每个节点上运行的网络代理， 实现 Kubernetes [服务（Service）](https://kubernetes.io/zh/docs/concepts/services-networking/service/) 概念的一部分。
>
>   kube-proxy 维护节点上的网络规则。这些网络规则允许从集群内部或外部的网络会话与 Pod 进行网络通信。
>
>   如果操作系统提供了数据包过滤层并可用的话，kube-proxy 会通过它来实现网络规则。否则， kube-proxy 仅转发流量本身。

##### 容器运行时

>   容器运行环境是负责运行容器的软件。
>
>   Kubernetes 支持多个容器运行环境: [Docker](https://kubernetes.io/zh/docs/reference/kubectl/docker-cli-to-kubectl/)、 [containerd](https://containerd.io/docs/)、[CRI-O](https://cri-o.io/#what-is-cri-o) 以及任何实现 [Kubernetes CRI (容器运行环境接口)](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-node/container-runtime-interface.md)。

##### 插件（Addons）

>   容器运行环境是负责运行容器的软件。
>
>   Kubernetes 支持多个容器运行环境: [Docker](https://kubernetes.io/zh/docs/reference/kubectl/docker-cli-to-kubectl/)、 [containerd](https://containerd.io/docs/)、[CRI-O](https://cri-o.io/#what-is-cri-o) 以及任何实现 [Kubernetes CRI (容器运行环境接口)](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-node/container-runtime-interface.md)。

###### DNS插件

>   coreDNS





#### k8s 集群搭建

>   https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
>
>   https://kubernetes.io/zh/docs/setup/



#### k8s 资源清单

##### 资源类型

>   一切皆资源（pod、控制器、service、ingress、volume、pv、pvc、configmap、secret、.....）
>
>   -   命名空间资源
>   -   集群资源
>       -   Namespace、Node、Role、ClusterRole、RoleBinding、ClusterRoleBinding
>       -   
>   -   元数据
>       -   HPA、PodTemplate、LimitRange

##### 资源清单yaml文件

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 # tells deployment to run 2 pods matching the template
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80

```

###### 语法

>   

###### 字段说明

>   ```shell
>   kubectl explain pod、xx、
>   ```





##### 容器生命周期

###### Pause

>   网络和数据卷的初始化。



###### InitContainer

>   具有container配置的所有字段，除了 readinessProbe。因为 InitContainer没有就绪状态



测试案例

```yaml
apiVersin: v1
kind: Pod
metadata: 
    name: myapp-pod
    labels:
        app: myapp

spec:
    containers:
    - name: myapp-container
      images: busybox
      command: ['sh', '-c', 'echo The app is running! && sleep 3000']
    
    initContainers:
    - name: init-myservice
      images: busybox
      command: ['sh', '-c', 'until nslookup myservice; do echo waiting for myservice; sleep 2; done;']
    - name: init-mydb
      images: busybox
      commend: ['sh', '-c', 'until nslookup mydb; do echo waiting for mydb; sleep 2; done;']
      
```

```yaml
apiVersion: v1
kind: Service
metadata:
    name: myservice
    
spec:
    ports:
      - protocol: TCP
        port: 80
        targetPort: 9376
        
        
---

apiVersion: v1
kind: Service
metadata:
    name: mydb
spec:
    ports:
      - protocol: TCP
        port: 80
        targetPort: 9377
```

###### 探针

>   探针是 kubelet 对容器的定期诊断。kubelet 调用由容器实现的 handler 。有三种类型的处理程序
>
>   -   ExecAction：在容器内执行指定命令。如果命令返回时退出码是0。则任务诊断成功
>   -   TCPSocketAction：对指定端口上的容器的IP地址进行 TCP 检查。如果端口可用。则诊断成功
>   -   HTTPGetAction：对指定端口和路径的容器的ip地址进行 HTTP Get 请求。响应状态码 200 或者 < 400 。则诊断成功

-   livenessProbe

>   存活探测，指示容器是否正在运行。
>
>   如果存活探测失败，则kubelet会杀死容器，并且容器会根据自己容器 重启策略 判断是否重启。如果没有配置存活探测。则默认成功
>
>   ```yaml
>   # livenessProbe - exec
>   
>   apiVersion: v1
>   kind: Pod
>   metadata:
>   	name: liveness-exec-pod
>   	namespace: dev
>   spec:
>   	containers:
>   	- name: liveness-exec-container
>   	  images: busybox:lastest
>   	  imagePullPolicy: IfNotPresent
>   	  command: ['/bin/sh', '-c', "touch /tmp/live; sleep 60; rm -rf /tmp/live; sleep 3600"]
>   	  livenessProbe:
>   	  	exec:
>   	  		command: ["test", "-e", "/tmp/live"]
>   	  	initialDelaySeconds: 1
>   	  	periodSeconds: 3
>   
>   ---
>   
>   # livenessProbe - TCPSocketAction
>   apiVersion: v1
>   kind: Pod
>   metadata:
>   	name: liveness-exec-pod
>   	namespace: dev
>   spec:
>   	containers:
>   	- name: liveness-exec-container
>   	  images: busybox:lastest
>   	  imagePullPolicy: IfNotPresent
>   	  command: ['/bin/sh', '-c', "touch /tmp/live; sleep 60; rm -rf /tmp/live; sleep 3600"]
>   	  livenessProbe:
>   	  	tcpSocket:
>   	  		host:
>   	  		post: 80
>   	  	initialDelaySeconds: 1
>   	  	periodSeconds: 3
>   	  	
>   ---
>   
>   # livenessProbe - HTTPGetAction
>   apiVersion: v1
>   kind: Pod
>   metadata:
>   	name: liveness-exec-pod
>   	namespace: dev
>   spec:
>   	containers:
>   	- name: liveness-exec-container
>   	  images: busybox:lastest
>   	  imagePullPolicy: IfNotPresent
>   	  command: ['/bin/sh', '-c', "touch /tmp/live; sleep 60; rm -rf /tmp/live; sleep 3600"]
>   	  livenessProbe:
>   	  	httpGet:
>   	  		port: 999
>   	  		path: /tmp/index.html
>   	  	initialDelaySeconds: 1
>   	  	periodSeconds: 3
>   	  	timeoutSeconds: 10
>   	  	
>   	  	
>   # 手动测试
>   # kubectl exec liveness-pod -it -- command
>   
>   
>   ```
>
>   

-   readinessProbe

>   就绪探测，指示容器是否准备好提供服务。如果就绪探针失败，端点控制器将从与 Pod 匹配的 所有 service 端点中删除该 Pod 的地址。
>
>   探测失败后 - 容器虽然 running 状态。但是不是就绪状态
>
>   ```yaml
>   apiVersion: v1
>   kind: Pod
>   metadata:
>       name: readiness-httpget-pod
>       namespace: dev
>   spec:
>       containers:
>       - name: readines-httpget-container
>         image: bosybox
>         imagePullPolicy: IfNotPresent
>         readinessProbe:
>             httpGet:
>                 port: 999
>                 path: /index/probe
>             initialDelaySeconds: 1	# 延时，容器启动后多长时间开始
>             periodSeconds: 3          # 执行间隔
>   
>   
>   ```
>
>   

###### postStart、preStop

>   lifecycle：主容器 - 启动、退出时动作
>
>   ```yaml
>   apiVersion: v1
>   kind: Pod
>   metadata:
>   	name: lifeCycle-pod
>   	namespace: dev
>   spec:
>   	containers:
>   	- name: lifeCycle-container
>   	  image: bosybox
>   	  imagePullPolicy: IfNotPresent
>   	  lifecycle:
>   	  	postStart:
>   	  		exec:
>   	  			
>   	  	preStop:
>   	  		httpGet:
>   	  		
>   
>   ```
>
>   



#### Pod

##### Pod 的状态

-   Pending：调度期间、镜像下载期间
-   Running：已和 node 绑定，至少一个容器在运行
-   Succeeded：Pod中所有容器都被成功终止，并且不会重启。
-   Failed：至少一个容器因为失败而终止
-   Unknown：



>   **为什么需要Pod？**
>
>   **Pause容器 ？**
>
>   **Pod实现原理？**
>
>   



#### Pod 控制器

##### 什么是控制器？



##### 控制器模型



##### ReplicationController 和 ReplicaSet

>   作业副本、水平扩展

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
	name: frontend
spec:
	replicas: 3
	selector:
		matchLabels:
			tier: frontend
	template: 					# pod 模板
		metadata:
			labels:
				tier: frontend
		spec:
			containers:
			- name: pip-redis
			  images: redis
			  env:
			  - name: GET_HOST_FROM
			  	value: dns
			  ports:
			  - containerPort: 80

```



##### Deployment

>   支持滚动更新、回滚
>
>   ![image-20211202161400255](/Users/banyajie/Library/Application%20Support/typora-user-images/image-20211202161400255.png)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
	name: my-deploy
spec:
	replicas: 3
	revisionHistoryLimit: 10  # 保留历史版本个数，如果为0表示不记录历史版本，不支持回滚
	tempate:
		matadata:
			labels:
				app: nginx
		spec:
			containers:
			- name: nginx
			  images: nginx:1.2
			  ports:
			  -	containerPort: 999
			  	

```

```shell
# 操作
# 扩缩容
kubectl scale deployment deployName --replicas num

# HPA扩缩容条件
kubectl autoscale deployment deployName --min=10 --max=15 --cpu-percent=80

# 更新镜像
kubectl set image deployment/deployName nginx=nginx:1.91 

# 回滚
kubectl rollout undo deployment/nginx-deployment 

kubectl rollout undo deployment/ngx-deployment --to-version=2   # 回滚到指定版本

# 查看更新状态
kubectl rollout status deployments ngx-deployment

# 查看历史 
kubectl rollout history deployment/ngx-deployment

# 暂停
kubectl rollout pause deployment/ngx-deployment


banyajie@bogon ~/kube_test %> kubectl rollout                                                                                                                                                                                                                                      [0]
Manage the rollout of a resource.
  
 Valid resource types include:

  *  deployments
  *  daemonsets
  *  statefulsets

Examples:
  # Rollback to the previous deployment
  kubectl rollout undo deployment/abc
  
  # Check the rollout status of a daemonset
  kubectl rollout status daemonset/foo

Available Commands:
  history     显示 rollout 历史
  pause       标记提供的 resource 为中止状态
  restart     Restart a resource
  resume      继续一个停止的 resource
  status      显示 rollout 的状态
  undo        撤销上一次的 rollout
  
```



##### HPA（Horizontal Pod Autoscaling）

>   基于 Deployment 和 RS 。在 V1版本中仅支持根据 Pod 的 CPU 利用率扩缩容。在 vlalpha 版本中，支持根据用户自定义的 metric 扩缩song



##### DaemonSet

>   作用：确保全部或者一些 Node 上运行一个 Pod 的副本。当有 Pod 加入集群时。也会为他们新增一个 Pod 。当有 Node 从集群移除时，这些Pod 也会被回收。删除 DaemonSet 将会删除它所创建的所有Pod
>
>   使用场景：
>
>   -   日志搜集组件
>   -   网络插件组件
>   -   监控组件

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
	name: deamonset-example
	namespace: dev
	labels:
		app: daemonset
spec:
	selector:							# 选择器
		matchLabels:					# 匹配的标签
			name: deamonset-example	
	template:
		metadata:
			labels:
				name: deamonset-example	# pod 标签
		spec:
			containers:
			- name: dae-xx
			  images: xxxxxx

```



##### Job

>   作用：负责批处理任务，即仅执行一次的任务。它保证批处理任务的一个或者多个Pod执行结束

```yaml
apiVersion: batch/v1
kind: Job
metadata:
	name: pi
spec:
	completions: 10 	# job 结束需要成功运行的pod个数
	parallelism： 1		# 并行运行的pod个数，默认是1 
	activeDeadlineSeconds： 10 # pod失败重试的最长时间
	template:
		metadata:
			name: pi
		spec:
			containers:
			- name: pi
			  images: xxxx
			  command: []
			 restartPolicy: Never

```



##### CronJob

>   管理基于时间的 Job
>
>   RestartPolicy 只支持：Never 和 OnFailed

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
	name: hello
spec:
	schedule: "*/1 * * * * *"	# 时间规则
	startingDeadlineSeconds: 10 	# 启动job的期限
	concurrencyPolicy: Allow        # 是否运行job并发 Allow、Forbid、Replace
	jobTemplate:
		spec:
			tmplate:
				spec:
					containers:
					- name: hello
					  images: xxx
					  args:
					  - /bin/sh
					  - -c 
					  - date; echo Hello 
					 restartPolicy: OnFilure
					 
```





##### StatefulSet

>   为了解决有状态服务的问题，应用场景包括
>
>   -   稳定的持久化存储。即 Pod 重新调度后还是能访问到相同的持久化数据。基于 PVC 实现
>   -   拓扑状态 - 稳定的网络标志。即 Pod 重新调度后其 PodName 和 HostName 不变。基于 HeadLess Service（即没有 Cluster IP）的 Service 实现
>       -   
>
>   -   有序部署、有序扩展。即Pod中容器的启动有顺序的、有依赖关系。在下一个 Pod 运行之前之前启动的 Pod 必须是 Running 或者 Ready的。基于 Init Container 实现
>   -   有序收缩、有序删除
>
>   



#### 服务发现

##### 网络通讯

>   -   同一个 Pod 内部通讯？
>
>   -   不同 Pod 之间通讯？
>
>   -   Pod 与 Service 通讯？
>       -   各个Node 上的 IPtables 规则
>       -   LVS





##### service 

>   Kubernetes Service 定义了这样一种抽象：逻辑上的一组 Pod，一种可以访问它们的策略 —— 通常称为微服务
>
>   Service 所针对的 Pods 集合通常是通过 selector 来确定的

```yaml
# service

apiVersion: v1
kind: Service
metadata:
  name: hostnames
spec:
  selector:
    app: hostnames
  ports:
  - name: default
    protocol: TCP
    port: 80
    targetPort: 9376
      
# 使用了 selector 字段来声明这个 Service 只代理携带了 app=hostnames 标签的 Pod。并且，这个 Service 的 80 端口，代理的是 Pod 的 9376 端口。

--- 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hostnames
spec:
  selector:
    matchLabels:
      app: hostnames
  replicas: 3
  template:
    metadata:
      labels:
        app: hostnames
    spec:
      containers:
      - name: hostnames
        image: k8s.gcr.io/serve_hostname
        ports:
        - containerPort: 9376
          protocol: TCP

# 每次访问 9376 端口时，返回它自己的 hostname

```

```shell
# 而被 selector 选中的 Pod，就称为 Service 的 Endpoints，你可以使用 kubectl get ep 命令看到它们，如下所示：
$ kubectl get endpoints hostnames
NAME        ENDPOINTS
hostnames   10.244.0.5:9376,10.244.0.6:9376,10.244.0.7:9376

# 只有处于 Running 状态，且 readinessProbe 检查通过的 Pod，才会出现在 Service 的 Endpoints 列表里。并且，当某一个 Pod 出现问题时，Kubernetes 会自动把它从 Service 里摘除掉。


```

##### Service 实现原理

>   service 是通过 kube-proxy 组件，加上 iptables 共同实现的



##### Service 负载均衡

>   RR
>
>   限制：只提供四层负载均衡的能力，无法提供七层功能



##### service 类型

-   ClusterIP：自动分配一个集群内部可以访问的 cluster ip

    -   clusterIp 主要是在每个 node 上使用 iptables，将发向 clusterIp 对应端口的数据，转发到 kube-proxy 中。然后 kube-proxy 自己内部实现的有负载均衡的方法，并可以查询到这个 service 下对应pod的地址和端口。进而转发数据给pod
    -   

-   NodePort：在 ClusterIP 的基础上为service在每台机器上绑定一个端口。这样就可以通过 NodeIP:port 来访问service

    -   ```yaml
        apiVersion: v1
        kind: Service
        metadata:
        	name: myapp-headless
        	namespace: dev
        spec:
        	type: NodePort
        	selector:
        		app: myapp
        	ports:
        	- name: http
        	  port: 80
        	  targetPort: 80
        
        # 查询生成的地址规则
        # iptables -t nat -nvL
        
        ```

    -   

-   LoadBalancer：在NodePort的基础上，借助 cloud provider 创建一个外部负载均衡器。并将请求转发到 NodeIP:port

    -   ```yaml
        ```

    -   

-   ExternalName：把集群外部的服务引入到集群内部来，在集群内部直接使用。没有任何代理被创建

    -   ```yaml
        apiVersion: v1
        kind: Service
        metadata:
        	name: my-service-1
        	namespace: dev
        spec:
        	type: ExternalName
        	externalName: www.baidu.com
        	
        
        # 当查询主机 my-service-1.default.svc.cluster.local 。 集群的 DNS 服务会返回一个值：www.baidu.com 的 CNAME 记录
        ```

    -   

-   HeadLessService：有时候不需要或者不想要负载均衡，以及单独的ClusterIP。这个时候可以通过指定 ClusterIP（spec.clusterIP）的值为”None“来创建 HeadLessService。这类service 不会分配 ClusterIP。kube-proxy不会处理他们

    -   ```yaml
        apiVersion: v1
        kind: Service
        metadata:
        	name: myapp-headless
        	namespace: dev
        spec:
        	selector:
        		app: myapp
        	clusterIP: "None"
        	ports:
        	- port: 80
        	  targetPort: 80
        
        # 没有分配Ip地址。可以早coreDNS节点中查看域名配置
        
        # myapp-headless.dev.default.svc.cluster.local
        
        
        # dig 命令。解析域名
        
        # dig -t A myqpp-headless.dev.default.svc.cluster.local. @dns解析地址
        ```

    -   

![image-20211202181557139](/Users/banyajie/Library/Application%20Support/typora-user-images/image-20211202181557139.png)



##### Service 代理模式

-   userspace
-   iptables
-   ipvs

##### Ingress

Ingress-nginx：https://kubernetes.github.io/ingress-nginx/examples/



###### Ingress-nginx架构图

![img](https://gimg2.baidu.com/image_search/src=http%3A%2F%2Fupload-images.jianshu.io%2Fupload_images%2F18968739-da680d4998109947.png&refer=http%3A%2F%2Fupload-images.jianshu.io&app=2002&size=f9999,10000&q=a80&n=0&g=0n&fmt=jpeg?sec=1641093921&t=51e5c529047afc2521e37078c11c72e1)





![img](https://gimg2.baidu.com/image_search/src=http%3A%2F%2Fcdn.bianchengquan.com%2Fffeabd223de0d4eacb9a3e6e53e5448d%2Fblog%2F5ffe6caff0509.png&refer=http%3A%2F%2Fcdn.bianchengquan.com&app=2002&size=f9999,10000&q=a80&n=0&g=0n&fmt=jpeg?sec=1641093921&t=8c03996f41f850aa7c8f30db08ddeeed)



###### Ingress http 代理访问

```yaml
# deployment
apiVersion: v1
kind: Deployment
metadata:
	name: nginx-dm
spec:
	replicas: 2
	template:
		metadata:
			labels:
				name: nginx
		spec:
			containers:
			- name: nginx
			  images: nginx:v1
			  imagePullPolicy: IfNotPresent
			  ports:
			  - port: 80
			  
--- 
# Service
apiVersion: v1
kind: Service
metadata:
	name: nginx-svc
spec:
	selectors:
		name: nginx
	ports:
	- port: 80
	  targetPort: 80
	  protocol: TCP
	  
--- 
# Ingress
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
	name: nginx-test
spec:
	rules:
	  - host: foo.bar.com
	    http:
	    	paths:
	    	- path: /
	    	  backends:
	    	  	serviceName: ngingx-svc
	    	  	servicePort: 80
	    	  	
# 
```

###### Ingress https 代理访问

```shell
# 1：生成证书
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=nginxsvc/O=nigxsvc"


# 创建一个secret 对象保存证书
kubectl create secret tls tls-secret --key tls.key --cert tls.crt
```

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
	name: nginx-test
spec:
	tls:			# 增加tls证书访问
	  - host:
	  	- foo.bar.com
	  	secretName: tls-secret
	  	
	rules:
	  - host: foo.bar.com
	    http:
	    	paths:
	    	- path: /
	    	  backends:
	    	  	serviceName: ngingx-svc
	    	  	servicePort: 80
	    	  

```



#### 存储

##### ConfigMap

>   配置文件、热更新
>
>   -   使用目录、文件创建
>
>       -   ```shell
>           # 目录： /conf/game.properties
>           # 文件1内容
>           enemies=alines
>           lives=3
>           enemies.cheat=true
>           enemies.cheat.level=1
>           ..
>           
>           
>           # 文件2 /conf/ui.properties
>           color.good= purpe
>           ...
>           
>           # 创建
>           kubectl create configmap game-config --from-file=/conf
>           
>           # 目录下的所有文件都会在 configMap 中创建出来，key名称是文件名。 value 是文件内容
>           
>           
>           kubectl describe configmap game-config
>           
>           Name:         game-config
>           Namespace:    default
>           Labels:       <none>
>           Annotations:  <none>
>           
>           Data
>           ====
>           game.properties:
>           ----
>           lives=3
>           enemies=aliens
>           enemies.cheat=true
>           enemies.cheat.level=noGoodRotten
>           enemies.code.passphrass=UUUUUUU
>           enemies.code.allowed=true
>           enemies.code.lives=30
>           
>           
>           ui.properties:
>           ----
>           color.good=purple
>           color.bad=yellow
>           allow.textmode=true
>           how.nice.to.look=Nice
>           
>           
>           
>           BinaryData
>           ====
>           
>           Events:  <none>
>           
>           
>           
>           ```
>
>   -   使用字面值创建
>
>       -   ```shell
>           
>           kubectl create configmap config2 --from-literal=special.how=very --from-literal=special.type=charm
>           
>           
>           
>           banyajie@bogon ~/kube_test/configMap %> kubectl describe cm config2                                                                                                                                                                                                                [0]
>           Name:         config2
>           Namespace:    default
>           Labels:       <none>
>           Annotations:  <none>
>           
>           Data
>           ====
>           special.how:
>           ----
>           very
>           special.type:
>           ----
>           charm
>           
>           BinaryData
>           ====
>           
>           Events:  <none>
>           
>           
>           
>           
>           ```
>
>       -   
>
>   使用场景：
>
>   -   配置文件
>
>       -   ```yaml
>           apiVersion: v1
>           kind: ConfigMap
>           metadata:
>           	name: spacial-config
>           	namespace: dev
>           data:
>           	special.how: very
>           	special.type: charm
>           	
>           	
>           ---
>           apiVersion: v1
>           kind: ConfigMap
>           metadata:
>           	name: env-config
>           	namespace: default
>           data:
>           	log_level: INFO
>           	
>           ---
>           # 在pod中使用configMap
>           apiVersion: v1
>           kind: Pod
>           metadata:
>           	name: configmap-test
>           	namespace: dev
>           spec:
>           	containers:
>           	- name: test-container
>           	  image: nginx
>           	  command: ["bin/bash", "-c", "env"]
>           	  env:
>           	  	- name: SPECIAL_LEVLE_KEY
>           	  	  valueFrom:
>           	  	  	configMapKeyRef:
>           	  	  		name: special-config
>           	  	  		key: special.how
>           	  	- name: SPECIAL_TYPE_KEY
>           	  	  valueFrom:
>           	  	  	configMapKeyRef:
>           	  	  		name: special-config
>           	  	  		key: special.type
>           	  envFrom:
>                 	- configMapRef:
>                 		name: env-config
>                 restartPolicy: Never
>                 
>           ---
>           # 在数据卷中使用
>           apiVersin: v1
>           kind: Pod
>           metadata:
>           	name: test
>           spec:
>           	containers:
>           		- name: test
>           	  		images: xx
>           	  		command: []
>           	  		volumeMounts:
>           	  			- name: config-volume
>           	  	  		mountPath: /etc/config
>           	volumes:
>               	- name: config-volume
>               	  configMap:
>               	  	name: spacial-config
>               	  	
>               restartPolicy: Nerver
>           ```
>
>       -   

##### Secret

>   敏感信息配置，有三种类型
>
>   -   Service Account：api-server account
>   -   Opaque：base64编码的secret。存储密码、秘钥等
>   -   Dockerconfigjson：docker 仓库认证



##### Volume

>   Kubernetes 中的卷有明确的寿命 - 与封装的Pod相同。所以，卷的生命周期比pod中的容器生命周期长。当这个容器重启后数据得以保存。当然 Pod 不存在时，卷会被删除

-   EmptyDir

    -   暂存空间，例如用于磁盘的合并排序

    -   用作长时间计算崩溃恢复时的检查点

    -   Web服务器容器提供数据时，保存内容管理器提取的文件

    -   ```yaml
        apiVersion: v1
        kind: Pod
        metadata:
        	name: pod-emptydir
        	namesapce: dev
        spec:
        	containers:
        	- name: container1
        	  images: xxxx
        	  volumeMounts:
        	  - mountPath: /cache
        	    name: cache-volume
        	- name: container2
        	  images: busybox
        	  volumeMounts:
        	  - mountPath: /conf
        	  	name: cache-volume
        	volumes:
        	- name: cache-volume
        	  emptyDir: {}
        	  
        # 多个容器共享数据
        ```

    -   

-   HostPath

    -   将主机节点中的文件系统中的文件或者目录挂载到集群呢

    -   运行需要访问 Docker  内部的容器； 使用 /var/lib/docker 的hostPath

    -   在容器中运行 cAdvisor；使用 /dev/cgroups 的hostPath

    -   ```yaml
        apiVersion: v1
        kind: Pod
        metadata:
        	name: pod-hostPath
        	namespace: dev
        spec:
        	containers:
        	- name: test
        	  iamges: busybox:v1
        	  volumeMounts:
        	  - mountPath: /test
        	    name: test-volume
        	volumes:
        	- name: test-volume
        	  hostPath:
        	    path: /data        # 主机目录
        	    type: Directory    # 类型：
        
        
        ```

    -   



##### PV（Persistent Volume）

>   PV：是由管理员设置的存储，是集群的一部分，就像节点中其他资源一样，PV也是集群中的资源。PV是Volume之类的卷插件。但是具有独立于使用PV的POD的生命周期。包含存储的实现细节。即NFS、。。。
>
>   -   静态PV
>       -   集群管理员创建一些PV。它们带有可供集群用户使用实际存储的细节。存在于 Kubernetes Api中。用于消费
>   -   动态
>       -   当管理员创建的PV都不匹配用户的 PersistentVolumeClaim 时，集群可能会尝试动态的为 PVC 创建卷，此配置依赖 StarageClass ：PVC 必须请求【存储类】，并且管理员必须创建并配置改类才能动态的创建。
>   -   绑定
>       -   master 节点中的控制循环监视新的PVC。寻找可用的PV。并把他们绑定再一起。
>       -   PVC 和 PV的绑定是一一映射的
>
>   PVC：用户存储的请求。与POD类似。pod消耗资源，PVC消耗PV资源。POD可以请求特定级别的资源（CPU和内存）。声明可以请求特定大小和访问模式（例如：可以读写一次或者只读多次模式挂载）
>
>   

###### PVC、PV

```yaml
# PV 创建
apiVersion: v1
kind: PersistentVolume
metadata:
	name: pv001
spec:
	capacity:
		storage: 5Gi
	volumeMode: Filesystem      # 文件类型
    accessModes:				# 访问模式
      - ReadWriteOnce 
    persistentVolumeReclaimPolicy: Recycle		# 回收策略
    storageClassName: slow 						# 存储类名称
    mountOptions:
      - hard
      - nfsvers=4.1
    nfs:
      path: /tmp
      server: 172.1.1.1
	
# PV 访问模式
# ReadWriteOnce  该卷可以被单个节点以读、写模式挂载    RWO
# ReadOnlyMany   该卷可以被多个节点以只读模式挂载	   ROX
# ReadWriteMany  该卷可以被多个节点以读写模式挂载      RWX

# PV 回收策略
# Retain - 保留 - 手动回收
# Recycle - 回收 -- （rm -rf ）
# Delete - 删除 删除关联的存储资产

# PVC 声明
--- 
apiVersion: v1
kind: 

# statefulset

```



###### 原理



#### 调度器

##### 调度过程

>   Scheduler 是kubernetes的调度器，主要的任务是把定义的pod分配到集群的节点上。调度原则
>
>   -   公平：保证每个节点都分配资源
>   -   资源高效利用：
>   -   效率：调度效率
>   -   灵活：用户自定义....
>
>   Scheduler作为单独的程序运行，启动之后一直监听 api-server，获取 PodSpec.NodeName为空的pod，为pod创建一个binding。
>
>   调度过程
>
>   -   首先过滤掉不满足条件的节点，这个过程叫做预选（predicate）
>       -   PodFitsResource：节点上剩余的资源是否满足 pod 申请的资源
>       -   PodFitsHost：如果pod指定了NodName，检查节点名
>       -   PodFitsHostPort：端口冲突
>       -   PodSelectorMatches：
>       -   NoDiskConfict：
>   -   然后通过对节点按照优先级排序，这个叫做（priority），最后选择优先级最高的节点
>
>   

##### 节点亲和性

>   
>
>   -   Node 亲和性
>       -   Pod.Spec.nodeAffinity
>       -   preferredDuringSchedulingIgnoredDuringExecution：软策略
>       -   requiredDuringSchedulingIgnoredDuringExecution：硬策略
>
>   ```yaml
>   # 硬侧率
>   apiVersion: v1
>   kind: Pod
>   metadata:
>   	name: test-affinity
>   spec:
>   	containers:
>   	- name: test-affinity
>   	  images: xxx
>   	affinity:
>   	  nodeAffinity:    										# 节点亲和性
>   		requiredDuringSchedulingIgnoredDuringExecution:  	# 硬策略，不满足不会运行
>   		  nodeSelectorTerms:								# 
>   		  - matchExpressions:
>   			- key: kubernetes.io/hostname			# node 节点的默认标签
>   		      operator: NotIn						# In/NotIn/Exists/DoesNotExist/Gt label的值大于某个值/It label的值小于
>   			  values:
>   			  - k8s-node02
>   				    
>   --- 
>   # 软策略
>   apiVersion: v1
>   kind: Pod
>   metadata:
>   	name: test-affinity
>   	labels: 
>   	  apps: node-affinity-pod
>   spec:
>   	containers:
>   	- name: test-affinity
>   	  images: xxx
>   	affinity:
>   	  nodeAffinity:    										# 节点亲和性
>   		preferredDuringSchedulingIgnoredDuringExecution:  	# 
>   		- weight: 1			# 权重
>   		  preference:		
>   		    matchExpressions:
>   		    - key: kubernetes.io/hostname
>   		      operator: In	# 存在
>   		      values:
>   		      - k8s-node01
>   
>   ```
>
>   
>
>   -   Pod 亲和性和反亲和性
>       -   pod.spec.affinity.podAffinity   /      pod.spec.affinity.podAntiAffinity 
>       -   preferredDuringSchedulingIgnoredDuringExecution：软策略
>       -   requiredDuringSchedulingIgnoredDuringExecution：硬策略
>
>   ```yaml
>   apiVersion: v1
>   kind: Pod
>   metadata:
>     name: pod-test
>     labels:
>       app: pod-test
>   spec:
>     containers:
>     - name: pod-test
>       images: xxx
>     affinity:
>       podAffinity: 										# 亲和性
>         requiredDuringSchedulingIgnoredDuringExecution: # 硬策略
>         - labelSelector: 								# 标签选择器
>             matchExpresstions:							# 匹配规则
>             - key: app									# 标签 app In pod-1
>               operator: In
>               values:
>               - pod-1
>           topologyKey: kubernetes.io/hostname		# 匹配成功后再同一个节点运行
>       podAntiAffinity:                                	# 反亲和性
>         preferredDuringSchedulingIgnoredDuringExecution:
>         - weight: 1
>           podAffinityTerm:
>             labelSelector:
>               matchExpresstions:
>               - key: app
>                 operator: In
>                 values:
>                 - pod-2
>   	
>   ```
>
>   

##### 污点和容忍

>   Taint、Toleration
>
>   -   Taint - 污点
>
>   ```shell
>   # kubectl taint 给Node增加污点
>   kubectl taint nodes nodeName key=value:effect
>   
>   # 删除污点
>   kubectl taint nodes nodeName key:effect-
>   
>   # 查看
>   kubectl describe pod podName
>   
>   # value 可以为空，effect描述污点的作用
>   # effect
>   # NoSchedule：标识k8s将不会把Pod调度到此节点上
>   # PreferNoSchedule：k8s 尽量避免将 Pod 调度到此节点上
>   # NoExecute：k8s不会把 pod 调度到此节点上，并且会把此节点上已有的 pod 驱逐出去
>   
>   ```
>
>   -   Toleration (容忍)
>
>   ```yaml
>   # pod.spec.tolerations
>   # key、value、effect 必须和taint保持一致
>   # operator = Exists 时将会忽略 value的值
>   # 当不指定key时，标识容忍所有的key
>   # 当不指定effect时，表示容忍所有的污点作用
>   
>   
>   tolerations:
>   - key: "key1"
>     operator: "Equal"
>     value: "value"
>     effect: "NoSchedule"
>     tolerationSecond: 3600	    # 当pod需要驱逐时可以在node上保留的时间
>   - key: "key2"
>     operator: "Equal"
>     value: "value"
>     effect: "NoSchedule"
>     
>     
>   ```
>
>   

##### 固定节点调度

>   通过 Pod.Spec.nodeName  或者 Pod.Spec.nodeSelector
>
>   ```yaml
>   apiVersion: v1
>   kind: Deployment
>   metadata:
>     name: node-name
>   spec:
>     replicas: 7
>     template:
>       metadata:
>         labels:
>           app: myweb
>       spec:
>         nodeName: node-1
>         nodeSelector:
>           type: xxx
>         containers:
>         - name: xxx
>           images: xxx/x:v1
>           imagePullPolicy: IfNotPresent
>           prot:
>           - containerPort: 80
>           
>   ---
>   
>   
>   
>   ```



#### 安全机制

>   

##### 认证

>   -   HTTP Token 认证：通过token识别合法用户
>       -   每一个token对应一个用户名存储在apiServer能访问的文件中，client 在 请求时 header 中设置token
>   -   HTTP Base 认证：通过 用户名 + 密码 认证
>       -   用户名和密码经过 base64编码后放在 HTTP Request 中的 Heather Authorization 域中发送给服务端
>   -   HTTPS 证书

##### 鉴权

###### RBAC

>   -   Role
>   -   ClusterRole
>   -   RoleBinding
>   -   ClusterRooleBinding

##### 访问控制（准入控制）



#### HELM



#### kubernetes高可用集群搭建

