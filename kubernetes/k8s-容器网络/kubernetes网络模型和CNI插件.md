#### CNI网桥

>   Kubernetes 是通过一个叫作 CNI 的接口，维护了一个单独的网桥来代替 docker0。
>
>   这个网桥的名字就叫作：CNI 网桥，它在宿主机上的设备名称默认是：cni0。
>
>   <img src="https://static001.geekbang.org/resource/image/9f/8c/9f11d8716f6d895ff6d1c813d460488c.jpg" alt="img" style="zoom: 33%;" />
>
>   注意：
>
>   ​	CNI 网桥只是接管所有 CNI 插件负责的、即 Kubernetes 创建的容器（Pod），而此时，如果你用 docker run 单独启动一个容器，那么 Docker 项目还是会把这个容器连接到 docker0 网桥上。所以这个容器的 IP 地址，一定是属于 docker0 网桥的 172.17.0.0/16 网段。
>
>   
>
>   **为什么kubernetes要设置一个和docker0网桥一模一样功能的CNI网桥？？**
>
>   -   Kubernetes 项目并没有使用 Docker 的网络模型（CNM），所以它并不希望、也不具备配置 docker0 网桥的能力
>   -   这还与 Kubernetes 如何配置 Pod，也就是 Infra 容器的 Network Namespace 密切相关（Kubernetes 在启动 Infra 容器之后，就可以直接调用 CNI 网络插件，为这个 Infra 容器的 Network Namespace，配置符合预期的网络栈）
>   -   

#### CNI网络插件

>   CNI 包：
>
>   ```shell
>   
>   $ ls -al /opt/cni/bin/
>   total 73088
>   -rwxr-xr-x 1 root root  3890407 Aug 17  2017 bridge
>   -rwxr-xr-x 1 root root  9921982 Aug 17  2017 dhcp
>   -rwxr-xr-x 1 root root  2814104 Aug 17  2017 flannel
>   -rwxr-xr-x 1 root root  2991965 Aug 17  2017 host-local
>   -rwxr-xr-x 1 root root  3475802 Aug 17  2017 ipvlan
>   -rwxr-xr-x 1 root root  3026388 Aug 17  2017 loopback
>   -rwxr-xr-x 1 root root  3520724 Aug 17  2017 macvlan
>   -rwxr-xr-x 1 root root  3470464 Aug 17  2017 portmap
>   -rwxr-xr-x 1 root root  3877986 Aug 17  2017 ptp
>   -rwxr-xr-x 1 root root  2605279 Aug 17  2017 sample
>   -rwxr-xr-x 1 root root  2808402 Aug 17  2017 tuning
>   -rwxr-xr-x 1 root root  3475750 Aug 17  2017 vlan
>   ```
>
>   这些 CNI 的基础可执行文件，按照功能可以分为三类：
>
>   -   第一类，叫作 Main 插件，它是用来创建具体网络设备的二进制文件。比如，bridge（网桥设备）、ipvlan、loopback（lo 设备）、macvlan、ptp（Veth Pair 设备），以及 vlan。
>   -   第二类，叫作 IPAM（IP Address Management）插件，它是负责分配 IP 地址的二进制文件。比如，dhcp，这个文件会向 DHCP 服务器发起请求；host-local，则会使用预先配置的 IP 地址段来进行分配。
>   -   第三类，是由 CNI 社区维护的内置 CNI 插件。比如：flannel，就是专门为 Flannel 项目提供的 CNI 插件；tuning，是一个通过 sysctl 调整网络设备参数的二进制文件；portmap，是一个通过 iptables 配置端口映射的二进制文件；bandwidth，是一个使用 Token Bucket Filter (TBF) 来进行限流的二进制文件。

#### 实现kubernetes的网络方案

>-   首先，实现网络方案本身。这一部分需要编写的，其实就是 flanneld 进程里的主要逻辑。比如，创建和配置 flannel.1 设备、配置宿主机路由、配置 ARP 和 FDB 表里的信息等等。
>
>-   实现该网络方案对应的 CNI 插件。这一部分主要需要做的，就是配置 Infra 容器里面的网络栈，并把它连接在 CNI 网桥上。
>
>    

#### CNI网络插件工作原理

>   