#### Service

>   Service会对提供同一个服务的多个pod进行聚合，并且提供一个统一的入口地址。通过访问Service的入口地址就能访问到后面的pod服务。
>
>   

<img src="https://gitee.com/yooome/golang/raw/main/k8s%E8%AF%A6%E7%BB%86%E6%95%99%E7%A8%8B/Kubenetes.assets/image-20200408194716912-1626783758946.png" alt="img" style="zoom:70%;" />

>   Service在很多情况下只是一个概念，真正起作用的其实是kube-proxy服务进程，每个Node节点上都运行着一个kube-proxy服务进程。当创建Service的时候会通过api-server向etcd写入创建的service的信息，而kube-proxy会基于监听的机制发现这种Service的变动，然后**它会将最新的Service信息转换成对应的访问规则**。

![img](https://gitee.com/yooome/golang/raw/main/k8s%E8%AF%A6%E7%BB%86%E6%95%99%E7%A8%8B/Kubenetes.assets/image-20200509121254425.png)



service 三种工作模式：

- userspace模式

    >   userspace模式下，kube-proxy会为每一个Service创建一个监听端口，发向Cluster IP的请求被Iptables规则重定向到kube-proxy监听的端口上，kube-proxy根据LB算法选择一个提供服务的Pod并和其建立链接，以将请求转发到Pod上。 该模式下，kube-proxy充当了一个四层负责均衡器的角色。由于kube-proxy运行在userspace中，在进行转发处理时会增加内核和用户空间之间的数据拷贝，虽然比较稳定，但是效率比较低。
    >
    >   ![img](https://gitee.com/yooome/golang/raw/main/k8s%E8%AF%A6%E7%BB%86%E6%95%99%E7%A8%8B/Kubenetes.assets/image-20200509151424280.png)

- iptables模式 

    >   ​	iptables模式下，kube-proxy为service后端的每个Pod创建对应的iptables规则，直接将发向Cluster IP的请求重定向到一个Pod IP。 该模式下kube-proxy不承担四层负责均衡器的角色，只负责创建iptables规则。该模式的优点是较userspace模式效率更高，但不能提供灵活的LB策略，当后端Pod不可用时也无法进行重试。
    >
    >   ![img](https://gitee.com/yooome/golang/raw/main/k8s%E8%AF%A6%E7%BB%86%E6%95%99%E7%A8%8B/Kubenetes.assets/image-20200509152947714.png)

- ipvs模式

    >      ipvs模式和iptables类似，kube-proxy监控Pod的变化并创建相应的ipvs规则。ipvs相对iptables转发效率更高。除此以外，ipvs支持更多的LB算法。
    >
    >   ![img](https://gitee.com/yooome/golang/raw/main/k8s%E8%AF%A6%E7%BB%86%E6%95%99%E7%A8%8B/Kubenetes.assets/image-20200509153731363.png)

资源清单：





