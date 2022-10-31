##### 前言



> 应用部署方式演变
>
> - 传统部署
>   - 优点：部署简单，不需要其他技术支持
>   - 缺点：不能为应用定义资源使用边界，很难合理的分配资源，造成资源浪费，服务之间可能互相影响
> - 虚拟化部署
>   - 优点：解决了传统部署方式中服务之间互相影响的问题，一定程度提高了安全性
>   - 缺点：引入了操作系统，造成资源浪费
> - 容器化部署
>   - 优点
>     - 共享操作系统，严格定义了容器资源（每个容器拥有自己的文件系统、进程空间、cpu、内存）
>     - 容器化的服务可以跨服务商、跨操作系统

![image-20220708171018280](/Users/banyajie/Library/Application Support/typora-user-images/image-20220708171018280.png)





##### 容器基础 - 进程

###### 进程：程序运行起来后的计算机执行环境的总和



###### 容器

> 容器其实是一种沙盒技术。顾名思义，沙盒就是能够像一个集装箱一样，把你的应用“装”起来的技术。



> 容器技术的核心功能，就是通过约束和修改进程的动态表现，从而为其创造出一个“边界”。对于 Docker 等大多数 Linux 容器来说，Cgroups 技术是用来制造约束的主要手段，而 Namespace 技术则是用来修改进程视图的主要方法。



例子：

```shell
# 启动一个容器
$ docker run -it busybox /bin/sh

# 容器中执行
/ # ps
PID  USER   TIME COMMAND
  1 root   0:00 /bin/sh
  10 root   0:00 ps
```

可以看到，我们在 Docker 里最开始执行的 /bin/sh，就是这个容器内部的第 1 号进程（PID=1），而这个容器里一共只有两个进程在运行。这就意味着，前面执行的 /bin/sh，以及我们刚刚执行的 ps，已经被 Docker 隔离在了一个跟宿主机完全不同的世界当中。



###### 隔离

这种机制，其实就是对被隔离应用的进程空间做了手脚，使得这些进程只能看到重新计算过的进程编号，比如 PID=1。可实际上，他们在宿主机的操作系统里，还是原来的第 100 号进程。其实就是linux中的namespace机制



###### 结论

> 容器只是一个特殊的进程



###### 传统虚拟机和容器的比较

> - 资源问题
>   - 传统虚拟机需要运行内核和操作系统，造成资源浪费
>   - 容器化后的进程实际上还是一个普通进程，共享的操作系统内核
> - 安全问题
>   - 容器 - 隔离不彻底，首先，既然容器只是运行在宿主机上的一种特殊的进程，那么多个容器之间使用的就还是同一个宿主机的操作系统内核。





##### 容器基础 - 隔离与限制



###### 隔离（namespace）

**namespace 是 Linux 内核用来隔离内核资源的方式。**通过 namespace 可以让一些进程只能看到与自己相关的一部分资源，而另外一些进程也只能看到与它们自己相关的资源，这两拨进程根本就感觉不到对方的存在。具体的实现方式是把一个或多个进程的相关资源指定在同一个 namespace 中。



###### Linux 中的namespace

> - PID - 进程空间
> - Mount - 让隔离进程只看到自己当前namespace的挂载点
> - UTS - 主机名
> - IPC - System V IPC（信号量、消息队列、共享内存）
> - Network - 网络设备和配置
> - User - 用户和用户组



###### Linux 环境下容器的基本实现原理

```c
// 正常创建进程的clone系统调用就会为我们创建一个新的进程，并且返回它的进程号 pid。
int pid = clone(main_function, stack_size, SIGCHLD, NULL); 

// 在参数中指定 CLONE_NEWPID 参数，这时，新创建的这个进程将会“看到”一个全新的进程空间，在这个进程空间里，它的 PID 是 1。之所以说“看到”，是因为这只是一个“障眼法”，在宿主机真实的进程空间里，这个进程的 PID 还是真实的数值，比如 100。
int pid = clone(main_function, stack_size, CLONE_NEWPID | SIGCHLD, NULL); 

```





```c
// 创建namespace
clone() 函数
我们可以通过 clone() 在创建新进程的同时创建 namespace。clone() 在 C 语言库中的声明如下：

/* Prototype for the glibc wrapper function */
#define _GNU_SOURCE
#include <sched.h>
int clone(int (*fn)(void *), void *child_stack, int flags, void *arg);

flags：表示使用哪些 CLONE_ 开头的标志位，与 namespace 相关的有CLONE_NEWIPC、CLONE_NEWNET、CLONE_NEWNS、CLONE_NEWPID、CLONE_NEWUSER、CLONE_NEWUTS 和 CLONE_NEWCGROUP。


// 加入namespace
setns() 函数
通过 setns() 函数可以将当前进程加入到已有的 namespace 中。setns() 在 C 语言库中的声明如下：

#define _GNU_SOURCE #include <sched.h> int setns(int fd, int nstype);

和 clone() 函数一样，C 语言库中的 setns() 函数也是对 setns() 系统调用的封装：

fd：表示要加入 namespace 的文件描述符。它是一个指向 /proc/[pid]/ns 目录中文件的文件描述符，可以通过直接打开该目录下的链接文件或者打开一个挂载了该目录下链接文件的文件得到。
nstype：参数 nstype 让调用者可以检查 fd 指向的 namespace 类型是否符合实际要求。若把该参数设置为 0 表示不检查。

```







###### 限制（cgroup）

> 为什么有了namespace后还需要对容器进行限制？
>
> 虽然容器内部的第 1 号进程在“障眼法”的干扰下只能看到容器里的情况，但是宿主机上，它作为第 100 号进程与其他所有进程之间依然是平等的竞争关系。这就意味着，虽然第 100 号进程表面上被隔离了起来，但是它所能够使用到的资源（比如 CPU、内存），却是可以随时被宿主机上的其他进程（或者其他容器）占用的。当然，这个 100 号进程自己也可能把所有资源吃光。



Linux Cgroups（Linux Control Group） 就是 Linux 内核中用来为进程设置资源限制的一个重要功能。Linux Cgroups 的全称是 Linux Control Group。它最主要的作用，就是限制一个进程组能够使用的资源上限，包括 CPU、内存、磁盘、网络带宽等等。此外，Cgroups 还能够对进程进行优先级设置、审计，以及将进程挂起和恢复等操作。



###### CGroup实现

> 在 Linux 中，Cgroups 给用户暴露出来的操作接口是文件系统，即它以文件和目录的方式组织在操作系统的 /sys/fs/cgroup 路径下。在 Ubuntu 16.04 机器里，我可以用 mount 指令把它们展示出来

```shell

# 如果没有cgroup，需要手动挂载
arallels@ubuntu-linux-20-04-desktop:~$ mount -t cgroup
cgroup on /sys/fs/cgroup/systemd type cgroup (rw,nosuid,nodev,noexec,relatime,xattr,name=systemd)
cgroup on /sys/fs/cgroup/net_cls,net_prio type cgroup (rw,nosuid,nodev,noexec,relatime,net_cls,net_prio)
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,cpu,cpuacct)
cgroup on /sys/fs/cgroup/freezer type cgroup (rw,nosuid,nodev,noexec,relatime,freezer)
cgroup on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,blkio)
cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,cpuset)
cgroup on /sys/fs/cgroup/rdma type cgroup (rw,nosuid,nodev,noexec,relatime,rdma)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
cgroup on /sys/fs/cgroup/pids type cgroup (rw,nosuid,nodev,noexec,relatime,pids)
cgroup on /sys/fs/cgroup/perf_event type cgroup (rw,nosuid,nodev,noexec,relatime,perf_event)
cgroup on /sys/fs/cgroup/devices type cgroup (rw,nosuid,nodev,noexec,relatime,devices)
cgroup on /sys/fs/cgroup/misc type cgroup (rw,nosuid,nodev,noexec,relatime,misc)
cgroup on /sys/fs/cgroup/hugetlb type cgroup (rw,nosuid,nodev,noexec,relatime,hugetlb)

# 在 /sys/fs/cgroup 下面有很多诸如 cpuset、cpu、 memory 这样的子目录，也叫子系统。这些都是我这台机器当前可以被 Cgroups 进行限制的资源种类。而在子系统对应的资源种类下，你就可以看到该类资源具体可以被限制的方法。
```



###### 如何通过CGroup对进程资源进行限制

> 在 /sys/fs/cgroup 下面有很多诸如 cpuset、cpu、 memory 这样的子目录，也叫子系统。这些都是我这台机器当前可以被 Cgroups 进行限制的资源种类。而在子系统对应的资源种类下，你就可以看到该类资源具体可以被限制的方法。
>
> ```shell
> $ ls /sys/fs/cgroup/cpu
> cgroup.clone_children cpu.cfs_period_us cpu.rt_period_us  cpu.shares notify_on_release
> cgroup.procs      cpu.cfs_quota_us  cpu.rt_runtime_us cpu.stat  tasks
> 
> # 比如：cfs_period 和 cfs_quota 可以用来限制进程在长度为 cfs_period 的一段时间内，只能被分配到总量为 cfs_quota 的 CPU 时间。
> ```
>
> 使用步骤
>
> - 需要在对应的子系统下面创建一个目录，比如，我们现在进入 /sys/fs/cgroup/cpu 目录下
>
> - ```shell
>   root@ubuntu:/sys/fs/cgroup/cpu$ mkdir container
>   root@ubuntu:/sys/fs/cgroup/cpu$ ls container/
>   cgroup.clone_children cpu.cfs_period_us cpu.rt_period_us  cpu.shares notify_on_release
>   cgroup.procs      cpu.cfs_quota_us  cpu.rt_runtime_us cpu.stat  tasks
>   
>   # 这个目录就称为一个“控制组”。你会发现，操作系统会在你新创建的 container 目录下，自动生成该子系统对应的资源限制文件。
>   ```
>
> - 例子
>
> - ```shell
>   # 运行
>   arallels@ubuntu-linux-20-04-desktop:/sys/fs/cgroup/cpu/container$ while : ; do : ; done & 
>   [1] 3705
>   
>   
>   169160
>   
>   
>   PID：3705
>   
>   # top 查看
>   top - 10:18:26 up 5 min,  1 user,  load average: 0.67, 0.30, 0.14
>   Tasks: 191 total,   2 running, 189 sleeping,   0 stopped,   0 zombie
>   %Cpu0  : 99.7 us,  0.0 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.3 si,  0.0 st
>   %Cpu1  :  3.3 us,  2.0 sy,  0.0 ni, 93.4 id,  0.3 wa,  0.0 hi,  1.0 si,  0.0 st
>   MiB Mem :   3914.3 total,   2160.2 free,    721.4 used,   1032.7 buff/cache
>   MiB Swap:   2048.0 total,   2048.0 free,      0.0 used.   2909.1 avail Mem 
>   
>       PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND                                                                                                                         
>      3705 paralle+  20   0    7776   2360    892 R 100.0   0.1   0:42.06 bash           
>      
>      
>   # 设置限制
>   parallels@ubuntu-linux-20-04-desktop:/sys/fs/cgroup/cpu/container$ cat cpu.cfs_quota_us  
>   -1
>   parallels@ubuntu-linux-20-04-desktop:/sys/fs/cgroup/cpu/container$ cat cpu.cfs_period_us 
>   100000
>   
>   # 默认没有限制
>   echo 20000 >> cpu.cfs_quota_us 
>   # 每100毫秒中可以使用20毫秒
>   
>   
>   # 开启限制
>   echo 169160 >> /sys/fs/cgroup/cpu/container/task
>   
>   %Cpu0  : 20.7 us,  2.7 sy,  0.0 ni, 76.6 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
>   %Cpu1  :  7.8 us,  5.2 sy,  0.0 ni, 86.0 id,  0.0 wa,  0.0 hi,  1.0 si,  0.0 st
>   MiB Mem :   3914.3 total,   2023.2 free,    725.0 used,   1166.1 buff/cache
>   MiB Swap:   2048.0 total,   2048.0 free,      0.0 used.   2910.7 avail Mem 
>   
>       PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND                                                                                                                         
>      3705 paralle+  20   0    7776   2360    892 R  20.3   0.1   6:41.33 bash  
>   ```
>
> - 





##### 容器基础 - 镜像



> 容器中的文件系统
>
> ```shell
>  ✘ $  ~  docker run -it busybox /bin/sh
> / #
> / # ls
> bin   dev   etc   home  proc  root  sys   tmp   usr   var
> ```



linux 容器中看到的文件系统是通过linux mount namespace 以及 choot 来实现的。

- 为什么容器打包一个完整的操作系统文件目录？
  - 一致性
  - 容器打包



###### Docker 镜像设计

> 引入分层概念



###### 联合文件系统（Union File System）

> ```shell
> 
> # 假如有两个目录A、B
> $ tree
> .
> ├── A
> │  ├── a
> │  └── x
> └── B
>   ├── b
>   └── x
>   
> # 用联合挂载的方式，将这两个目录挂载到一个公共的目录 C 上：
> $ mkdir C
> $ mount -t aufs -o dirs=./A:./B none ./C
> 
> # 
> 
> $ tree ./C
> ./C
> ├── a
> ├── b
> └── x
> # 可以看到，在这个合并后的目录 C 里，有 a、b、x 三个文件，并且 x 文件只有一份。这，就是“合并”的含义。此外，如果你在目录 C 里对 a、b、x 文件做修改，这些修改也会在对应的目录 A、B 中生效。
> ```



###### Docker 镜像原理

> ```shell
> # 启动一个容器
> 
> $ docker run -d ubuntu:latest sleep 3600
> 
> # “镜像”，实际上就是一个 Ubuntu 操作系统的 rootfs，它的内容是 Ubuntu 操作系统的所有文件和目录。不过，与之前我们讲述的 rootfs 稍微不同的是，Docker 镜像使用的 rootfs，往往由多个“层”组成：
> 
> $ docker image inspect ubuntu:latest
> ...
>      "RootFS": {
>       "Type": "layers",
>       "Layers": [
>         "sha256:f49017d4d5ce9c0f544c...",
>         "sha256:8f2b771487e9d6354080...",
>         "sha256:ccd4d61916aaa2159429...",
>         "sha256:c01d74f99de40e097c73...",
>         "sha256:268a067217b5fe78e000..."
>       ]
>     }
>     
> # 可以看到，这个 Ubuntu 镜像，实际上由五个层组成。这五个层就是五个增量 rootfs，每一层都是 Ubuntu 操作系统文件与目录的一部分；而在使用镜像时，Docker 会把这些增量联合挂载在一个统一的挂载点上
> 
> ```
>
> 



##### 容器基础 - 容器

Docker 项目来说，它最核心的原理实际上就是为待创建的用户进程：

- 启用 Linux Namespace 配置
- 设置指定的 Cgroups 参数；
- 切换进程的根目录（Change Root）。



##### 容器到容器云 - k8s本质





###### k8s解决了什么问题？

> 容器编排？调度？容器云？还是集群管理？
>
> 核心需求：对于开发人员来讲，有了应用的容器化镜像后，希望可以帮助开发人员在给定集群上运行起来，并提供路由网关、水平扩展、监控、备份、灾难回复等一系列功能 （Pass能力）



###### k8s基础架构

![image-20220719112627290](/Users/banyajie/Library/Application Support/typora-user-images/image-20220719112627290.png)

- Master节点 （控制节点）
  - kube-apiserver - 负责API服务
  - kube-scheduler - 负责调度
  - kube-controller-manager - 负责编排
  - etcd - 保存集群的持久化数据
- Node节点
  - kubelet 组件 - 负责同容器运行时打交道，通过CRI（Container RunTime Interface）的远程调用接口，实现容器的操作，比如如何启动容器、修改容器操作等。并且负责调用网络插件以及存储插件等工作



###### k8s核心解决的问题

> 运行在大规模集群中的各种任务之间，实际上存在着各种各样的关系。这些关系的处理，才是作业编排和管理系统最困难的地方。



这种任务与任务之间的关系，在我们平常的各种技术场景中随处可见。比如，一个 Web 应用与数据库之间的访问关系，一个负载均衡器和它的后端服务之间的代理关系，一个门户应用与授权组件之间的调用关系。更进一步地说，同属于一个服务单位的不同功能之间，也完全可能存在这样的关系。比如，一个 Web 应用与日志搜集组件之间的文件交换关系。



而在容器技术普及之前，传统虚拟机环境对这种关系的处理方法都是比较“粗粒度”的。你会经常发现很多功能并不相关的应用被一股脑儿地部署在同一台虚拟机中，只是因为它们之间偶尔会互相发起几个 HTTP 请求。



更常见的情况则是，一个应用被部署在虚拟机里之后，你还得手动维护很多跟它协作的守护进程（Daemon），用来处理它的日志搜集、灾难恢复、数据备份等辅助工作。



容器技术出现以后，在“功能单位”的划分上，容器有着独一无二的“细粒度”优势：毕竟容器的本质，只是一个进程而已。也就是说，只要你愿意，那些原先拥挤在同一个虚拟机里的各个应用、组件、守护进程，都可以被分别做成镜像，然后运行在一个个专属的容器中。它们之间互不干涉，拥有各自的资源配额，可以被调度在整个集群里的任何一台机器上。而这，正是一个 PaaS 系统最理想的工作状态，也是所谓“微服务”思想得以落地的先决条件。



但是，如果只做到“封装微服务、调度单容器”这一层次，Docker Swarm 项目就已经绰绰有余了。如果再加上 Compose 项目，你甚至还具备了处理一些简单依赖关系的能力，比如：一个“Web 容器”和它要访问的数据库“DB 容器”。

如果我们现在的需求是，要求这个项目能够处理前面提到的所有类型的关系，甚至还要能够支持未来可能出现的更多种类的关系呢？



>  Kubernetes 项目最主要的设计思想是，从更宏观的角度，以统一的方式来定义任务之间的各种关系，并且为将来支持更多种类的关系留有余地。



比如，Kubernetes 项目对容器间的“访问”进行了分类，首先总结出了一类非常常见的“紧密交互”的关系，即：这些应用之间需要非常频繁的交互和访问；又或者，它们会直接通过本地文件进行信息交换。

在常规环境下，这些应用往往会被直接部署在同一台机器上，通过 Localhost 通信，通过本地磁盘目录交换文件。而在 Kubernetes 项目中，这些容器则会被划分为一个“Pod”，Pod 里的容器共享同一个 Network Namespace、同一组数据卷，从而达到高效率交换信息的目的。



而对于另外一种更为常见的需求，比如 Web 应用与数据库之间的访问关系，Kubernetes 项目则提供了一种叫作“Service”的服务。像这样的两个应用，往往故意不部署在同一台机器上，这样即使 Web 应用所在的机器宕机了，数据库也完全不受影响。可是，我们知道，对于一个容器来说，它的 IP 地址等信息不是固定的，那么 Web 应用又怎么找到数据库容器的 Pod 呢？所以，Kubernetes 项目的做法是给 Pod 绑定一个 Service 服务，而 Service 服务声明的 IP 地址等信息是“终生不变”的。这个 Service 服务的主要作用，就是作为 Pod 的代理入口（Portal），从而代替 Pod 对外暴露一个固定的网络地址。

这样，对于 Web 应用的 Pod 来说，它需要关心的就是数据库 Pod 的 Service 信息。不难想象，Service 后端真正代理的 Pod 的 IP 地址、端口等信息的自动更新、维护，则是 Kubernetes 项目的职责。



像这样，围绕着容器和 Pod 不断向真实的技术场景扩展，我们就能够摸索出一幅如下所示的 Kubernetes 项目核心功能的“全景图”。

![image-20220719150405392](/Users/banyajie/Library/Application Support/typora-user-images/image-20220719150405392.png)



我们从容器这个最基础的概念出发，首先遇到了容器间“紧密协作”关系的难题，于是就扩展到了 Pod；有了 Pod 之后，我们希望能一次启动多个应用的实例，这样就需要 Deployment 这个 Pod 的多实例管理器；而有了这样一组相同的 Pod 后，我们又需要通过一个固定的 IP 地址和端口以负载均衡的方式访问它，于是就有了 Service。可是，如果现在两个不同 Pod 之间不仅有“访问关系”，还要求在发起时加上授权信息。最典型的例子就是 Web 应用对数据库访问时需要 Credential（数据库的用户名和密码）信息。那么，在 Kubernetes 中这样的关系又如何处理呢？Kubernetes 项目提供了一种叫作 Secret 的对象，它其实是一个保存在 Etcd 里的键值对数据。这样，你把 Credential 信息以 Secret 的方式存在 Etcd 里，Kubernetes 就会在你指定的 Pod（比如，Web 应用的 Pod）启动时，自动把 Secret 里的数据以 Volume 的方式挂载到容器里。这样，这个 Web 应用就可以访问数据库了。



> 除了应用与应用之间的关系外，应用运行的形态是影响“如何容器化这个应用”的第二个重要因素。

为此，Kubernetes 定义了新的、基于 Pod 改进后的对象。比如 Job，用来描述一次性运行的 Pod（比如，大数据任务）；再比如 DaemonSet，用来描述每个宿主机上必须且只能运行一个副本的守护进程服务；又比如 CronJob，则用于描述定时任务等等。如此种种，正是 Kubernetes 项目定义容器间关系和形态的主要方法。



>  kubernetes项目核心
>
> - 首先，通过一个“编排对象”，比如 Pod、Job、CronJob 等，来描述你试图管理的应用；
> - 然后，再为它定义一些“服务对象”，比如 Service、Secret、Horizontal Pod Autoscaler（自动水平扩展器）等。这些对象，会负责具体的平台级功能。
>
> 这种使用方法，就是所谓的“声明式 API”。这种 API 对应的“编排对象”和“服务对象”，都是 Kubernetes 项目中的 API 对象（API Object）。



> Kubernetes 项目如何启动一个容器化任务呢？



一个deployment的yaml

```yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80

```

```shell

# 执行
$ kubectl create -f nginx-deployment.yaml
```





##### 容器编排与k8s作业管理

###### pod

###### 控制器模型

###### 作业副本与水平扩展

###### 存储状态

###### 有状态应用

###### job

###### 自定义控制器

###### 角色控制RBAC

###### Operator



##### 容器持久化存储

##### 容器网络

##### 容器运行时

