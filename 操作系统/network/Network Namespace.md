-   Linux network namespace

    >   network interface
    >
    >   iptables
    >
    >   route table

-   Linux network namesapce

    -   view
    -   add
    -   enter
    -   del
    -   oper

    >   ip netns add name
    >
    >   ip netns delete name
    >
    >   ip netns exec name bash
    >
    >   ip netns exec name bash -rffile <(echo "PS1=\\"name> \\"")
    >
    >   ip link set lo up （启动）

-   Communicate between two net namespace

    >   Veth Peer（虚拟设备）
    >
    >   1.   创建 veth peer：
    >        1.   ip link add network1 type veth peer name network2  
    >   2.   为每一个网络空间建立网络
    >        1.   ip link set network1 netns network1DevName
    >        2.   ip link set network2 netns network2DevName
    >   3.   联通
    >        1.   ip netns exec network1 ip addr add dev devName ipAddr/24
    >        2.   ip netns exec network2 ip addr add dev devName  ipAddr/24
    >   4.   开启
    >        1.   ip netns exec network1 ip link set devName up
    >        2.   ip netns exec network2 ip link set devName up

-   Communicate between two multi namespace

    >   Bridge（网桥）
    >
    >   bridge link
    >
    >   ip link set bridgeName up
    >
    >   
    >
    >   通过veth peer和bridge实现多个网络空间的通信
    >
    >   1.   ip link add bridgeName type bridge （创建bridge）
    >   2.   ip link add  network1  type veth peer name networkBg （创建veth peer） 
    >   3.   ip link set network1 netns network1DevName （network1 开启veth peer）
    >   4.   ip link set xxx master bridgeName （bridge 连接veth peer）
    >   5.   ip netns exec network1 ip addr add dev devName ipAddr/24（设置network1的地址）
    >   6.   ip netns exec network1 ip link set devName up （network1 启动）
    >   7.   ip link set bridgeName up（bridge启动）
    >   8.   重复2、3、4、5、6、7开启第二个veth peer....
    >   9.   

    

