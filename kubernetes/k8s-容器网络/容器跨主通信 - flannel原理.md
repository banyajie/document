**Flannel后端实现的三种方式**

-   VXLAN（Virtual Extensible Lan）虚拟可扩展网络

    >   VXLAN，即 Virtual Extensible LAN（虚拟可扩展局域网），是 Linux 内核本身就支持的一种网络虚似化技术。所以说，VXLAN 可以完全在内核态实现上述封装和解封装的工作，从而通过与前面相似的“隧道”机制，构建出覆盖网络（Overlay Network）
    >
    >   
    >
    >   VXLAN 的覆盖网络的设计思想是：在现有的三层网络之上，“覆盖”一层虚拟的、由内核 VXLAN 模块负责维护的二层网络，使得连接在这个 VXLAN 二层网络上的“主机”（虚拟机或者容器都可以）之间，可以像在同一个局域网（LAN）里那样自由通信。当然，实际上，这些“主机”可能分布在不同的宿主机上，甚至是分布在不同的物理机房里.
    >
    >   
    >
    >   而为了能够在二层网络上打通“隧道”，VXLAN 会在宿主机上设置一个特殊的网络设备作为“隧道”的两端。这个设备就叫作 VTEP，即：VXLAN Tunnel End Point（虚拟隧道端点）。
    >
    >   
    >
    >   VTEP 设备的作用，其实跟前面的 flanneld 进程非常相似。只不过，它进行封装和解封装的对象，是二层数据帧（Ethernet frame）；而且这个工作的执行流程，全部是在内核里完成的（因为 VXLAN 本身就是 Linux 内核中的一个模块）
    >
    >   <img src="https://static001.geekbang.org/resource/image/03/f5/03185fab251a833fef7ed6665d5049f5.jpg" alt="img" style="zoom:50%;" />
    >
    >   flannel设备，就是 VXLAN 所需的 VTEP 设备，它既有 IP 地址，也有 MAC 地址。

    

-   Host-gw

​		> host-gw 模式的工作原理，其实就是将每个 Flannel 子网（Flannel Subnet，比如：10.244.1.0/24）的“下一跳”，设置成了该子网对应的宿主机的 IP 地址。

-   UDP

    >   flanneld 在收到 container-1 发给 container-2 的 IP 包之后，就会把这个 IP 包直接封装在一个 UDP 包里，然后发送给 Node 2。不难理解，这个 UDP 包的源地址，就是 flanneld 所在的 Node 1 的地址，而目的地址，则是 container-2 所在的宿主机 Node 2 的地址。
    >
    >   这个请求得以完成的原因是，每台宿主机上的 flanneld，都监听着一个 8285 端口，所以 flanneld 只要把 UDP 包发往 Node 2 的 8285 端口即可
    >
    >   <img src="https://static001.geekbang.org/resource/image/83/6c/8332564c0547bf46d1fbba2a1e0e166c.jpg" alt="img" style="zoom: 50%;" />
    >
    >   Flannel UDP 模式提供的其实是一个三层的 Overlay 网络，即：它首先对发出端的 IP 包进行 UDP 封装，然后在接收端进行解封装拿到原始的 IP 包，进而把这个 IP 包转发给目标容器。这就好比，Flannel 在不同宿主机上的两个容器之间打通了一条“隧道”，使得这两个容器可以直接使用 IP 地址进行通信，而无需关心容器和宿主机的分布情况
    >
    >   

**工作原理**

>   Flannel 设备类型：Tunnel 
>
>   Tun设备工作在三层（Network Layer）的虚拟网络设备，功能：在操作系统内核和用户应用程序之间传递ip包
>
>   
>
>   $ ip route
>   default via 10.168.0.1 dev eth0
>   100.96.0.0/16 dev flannel0  proto kernel  scope link  src 100.96.1.0
>   100.96.1.0/24 dev docker0  proto kernel  scope link  src 100.96.1.1
>   10.168.0.0/24 dev eth0  proto kernel  scope link  src 10.168.0.2
>
>   
>
>   ip包流动方向
>
>   1：当操作系统将一个 IP 包发送给 flannel0 设备之后，flannel0 就会把这个 IP 包，交给创建这个设备的应用程序，也就是 Flannel 进程。这是一个从内核态（Linux 操作系统）向用户态（Flannel 进程）的流动方向。
>
>   2：反之，如果 Flannel 进程向 flannel0 设备发送了一个 IP 包，那么这个 IP 包就会出现在宿主机网络栈中，然后根据宿主机的路由表进行下一步处理。这是一个从用户态向内核态的流动方向
>
>   3：所以，当 IP 包从容器经过 docker0 出现在宿主机，然后又根据路由表进入 flannel0 设备后，宿主机上的 flanneld 进程（Flannel 项目在每个宿主机上的主进程），就会收到这个 IP 包。然后，flanneld 看到了这个 IP 包的目的地址，是 100.96.2.3，就把它发送给了 Node 2 宿主机
>
>   
>
>   **flanneld 又是如何知道这个 IP 地址对应的容器，是运行在 Node 2 上的呢？**
>
>   子网（Subnet）
>
>   一台宿主机上的所有容器，都属于宿主机被分配的一个子网
>
>   比如：
>
>   Node1 子网100.96.1.0/24     Container1 ip地址：100.96.1.2
>
>   Node1 子网100.96.2.0/24     Container2 ip地址：100.96.2.3
>
>   子网和宿主机的对应关系保存在ETCD中：
>
>   $ etcdctl ls /coreos.com/network/subnets
>   /coreos.com/network/subnets/100.96.1.0-24
>   /coreos.com/network/subnets/100.96.2.0-24
>   /coreos.com/network/subnets/100.96.3.0-24
>
>   $ etcdctl get /coreos.com/network/subnets/100.96.2.0-24
>   {"PublicIP":"10.168.0.3"}
>
>   所以，flanneld 进程在处理由 flannel0 传入的 IP 包时，就可以根据目的 IP 的地址（比如 100.96.2.3），匹配到对应的子网（比如 100.96.2.0/24）
>
>   
>
>   
>
>   

