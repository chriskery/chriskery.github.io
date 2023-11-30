---
title: linux虚拟网络设备
date: 2023-11-29 22:45:32
tags:
- linux
- network
categories:
- linux
- network
---
## Linux 虚拟网络设备

### Linux Veth

#### veth-pair

顾名思义，veth-pair 就是一对的虚拟设备接口，和 tap/tun 设备不同的是，它都是成对出现的。一端连着协议栈，一端彼此相连着。

一般用于跨 namespace 通信。

> Linux 中默认不同 net namespace 设备无法通信。



#### 网络命名空间

为了支持网络协议栈的多个实例，Linux在网络栈中引入了网络命名空间。这些独立的协议栈被隔离到不同的命名空间中。


#### Demo

本次演示中，先创建一个网络命名空间，然后创建一个 veth 设备对，将设备对分别切换到不同的命名空间中，实现不同命名空间的互相通信。



准备一个 net namespace

```sh
# 1创建 namespace
$ ip netns add netns1
# 查询
$ ip netns list
netns1 (id: 0)
```





创建两个Veth设备对

```sh
# ip link add <veth name> type veth peer name <peer name>
$ ip link add veth0 type veth peer name veth1
# 查询
$ ip link show
```

此时两个 veth 都在默认 net namespace 中，为了测试，先将其中一个切换到 netns1

```sh
$ ip link set veth1 netns netns1
# 此时再看就只能看到一个了
$ ip link show
# 去 netns 中查询
$ ip netns exec netns1 ip link show
```

至此，两个不同的命名空间各自有一个Veth，不过还不能通信，因为我们还没给它们分配IP

```sh
$ ip netns exec netns1 ip addr add 10.1.1.1/24 dev veth1
$ ip addr add 10.1.1.2/24 dev veth0
```

再启动它们

```sh
$ ip netns exec netns1 ip link set dev veth1 up
$ ip link set dev veth0 up
```

现在两个网络命名空间可以互相通信了

```sh
$ ping 10.1.1.1
```



至此，Veth设备对的基本原理和用法演示结束。

> Docker 内部，Veth设备对也是连通容器与宿主机的主要网络设备。

### Linux Bridge

Bridge 虚拟设备是用来桥接的网络设备，它相当于现实世界中的交换机，可以连接不同的网络设备，当请求到达 Bridge 设备时，可以通过报文中的 Mac 地址进行广播或转发。

例如，我们可以创建一个 Bridge 设备来连接 Namespace 中的网络设备和宿主机上的网络。

```shell
#创建veth设备并将一端移动到指定Namespace中
$ sudo ip netns add ns1
$ sudo ip link add veth0 type veth peer name veth1
$ sudo ip link set veth1 netns ns1
# 先安装bridge-utils
$ sudo apt-get install bridge-utils
#创建网桥,
$ sudo brctl addbr br0
#挂载网络设备
$ sudo brctl addif br0 eth0
$ sudo brctl addif br0 veth0
```



[linux bridge实践](https://zhuanlan.zhihu.com/p/339070912)





## 3. Linux 路由表

路由表是 Linux 内核的一个模块，通过定义路由表来决定在某个网络Namespace 中包的流向，从而定义请求会到哪个网络设备上。

继续以上面的例子演示

```shell
#启动宿主机上的虚拟网络设备，
$ sudo ip link set veth0 up
$ sudo ip link set br0 up
#给ns1中的虚拟网络设备设置IP并启动它
$ sudo ip netns exec ns1 ip addr add 172.18.0.2/24 dev veth1
$ sudo ip netns exec ns1 ip link set dev veth1 up
#分别设置ns1网络空间的路由和宿主机上的路由
#default代表0.0.0.0/0,即在Net Namespace中的所有流量都经过veth1的网络设备流出
$ sudo apt-get install net-tools
$ sudo ip netns exec ns1  route add default dev veth1
#在宿主机上将172.18.0.0/24网段的请求路由到br0网桥
$ sudo route add -net 172.18.0.0/24 dev br0
```

通过设置路由，对IP地址的请求就能正确被路由到对应的网络设备上，从而实现通信，如下：

```shell
#查看宿主机的IP地址
$ sudo ip addr show eth0
6: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq master br0 state UP group default qlen 1000
    link/ether 00:15:5d:ff:cd:9a brd ff:ff:ff:ff:ff:ff
    inet 172.20.160.67/20 brd 172.20.175.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::215:5dff:feff:cd9a/64 scope link
       valid_lft forever preferred_lft forever
#从Namespace中访问宿主机的地址
$ sudo ip netns exec ns1 ping -c 1 172.20.160.67

#从宿主机访问Namespace中的网络地址
$ ping -c 1 172.18.0.2
```

到此就实现了通过网桥连接不同Namespace网络了。





## 4. Linux iptables

iptables 是对 Linux 内核的 netfilter 模块进行操作和展示的工具，用来管理包的流动和转送。iptables 定义了一套链式处理的结构，在网络包传输的各个阶段可以使用不同的策略对包进行加工、传送或丢弃。

在容器虚拟化的技术中，经常会用到两种策略 MASQUERADE 和 DNAT，用于容器和宿主机外部的网络通信。



### MASQUERADE 

iptables 中的 MASQUERADE 策略可以将请求包中的源地址转换成一个网络设备的地址。

比如上述例子中的那个 Namespace 中网络设备的地址是 172.18.0.2, 这个地址虽然在宿主机上可以路由到 br0 网桥，但是到达宿主机外部之后，是不知道如何路由到这个 IP 地址的，所以如果要请求外部地址的话，需要先通过MASQUERADE 策略将这个 IP 转换成宿主机出口网卡的 IP。

```shell
#打开IP转发
$ sudo sysctl -w net.ipv4.conf.all.forwarding=1
net.ipv4.conf.all.forwarding = 1
#对Namespace中发出的包添加网络地址转换
$ sudo iptables -t nat -A POSTROUTING -s 172.18.0.0/24 -o eth0 -j MASQUERADE
```

在 Namespace 中请求宿主机外部地址时，将 Namespace 中的源地址转换成宿主机的地址作为源地址，就可以在 Namespace 中访问宿主机外的网络了。



### DNAT

iptables 中的 DNAT 策略也是做网络地址的转换，不过它是要更换目标地址，经常用于将内部网络地址的端口映射到外部去。

比如，上面那个例子中的 Namespace 如果需要提供服务给宿主机之外的应用去请求要怎么办呢?

外部应用没办法直接路由到 172.18.0.2 这个地址，这时候就可以用到 DNAT 策略。

```shell
#将宿主机上80端口的请求转发到Namespace里的IP上
$ sudo iptables -t nat -A PREROUTING -p tcp -m tcp --dport 80 -j DNAT --to-destination 172.18.0.2:80
```

这样就可以把宿主机上 80 端口的 TCP 请求转发到 Namespace 中的地址172.18.0.2:80,从而实现外部的应用调用。





## 5. Go语言网络库

容器网络用到的技术就是上面提到的几个点，不过用 Go 语言该怎么配置呢？

Go 中需要用到的几个网络库介绍。



### net 库

net 库是 Go 语言内置的库，提供了跨平台支持的网络地址处理，以及各种常见协议的IO支持，比如TCP、UDP、DNS、Unix Socket等。



### netlink库

[netlink库](https://github.com/vishvananda/netlink) 是Go 语言的操作网络接口、路由表等配置的库 ，使用它的调用相当于我们通过 IP 命令去管理网络接口。



### netns库

[netns库](https://github.com/vishvananda/netns) 就是 Go 语言版的`ip netns exec` 命令实现。通过这个库可以让 netlink 库中配置网络接口的代码在某个容器的 Net amespace 中执行。


### 单节点容器相互访问

![](https://pic2.zhimg.com/v2-b3b33ed53fdfd86fff8ace576528f671_b.jpg)

![](https://pic2.zhimg.com/80/v2-b3b33ed53fdfd86fff8ace576528f671_1440w.webp)

veth pair的一端在network namespace中，另一端接在了linux bridge上，所以图中的veth00/veth01就变为了从设备（linux bridge），即他们就变为了网线，没有了处理数据包的能力，只能转发数据包。

以192.268.3.101（container1） → 192.268.3.106（container2）为例：

1、 container1发送APR请求得到container2的mac地址

2、 container1的这个请求出现在了docker0网桥

3、 docker0网桥将ARP广播转发到其他接在docker0上的设备

4、 container2收到这个ARP请求，响应给container1

5、 container1拿到mac地址封装ICMP请求

6、 同样这个请求也会经由docker0出现在宿主机上

7、 docker0查到container2属于自己就转发给了container2

### 容器访问外部网络

![](https://pic3.zhimg.com/v2-f7891b35584bd9a960fb042e1a651206_b.jpg)

![](https://pic3.zhimg.com/80/v2-f7891b35584bd9a960fb042e1a651206_1440w.webp)

以图中的container2要访问百度为例：

1、 container2 ping 220.181.38.148

2、 数据包同样会到达docker0网桥，即到达了宿主机

3、宿主机发现目的目的网段要交给eth0处理

```text
~# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.28.143.253  0.0.0.0         UG    100    0        0 eth0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
```

4、 由于源ip是内网ip还要进行一个NAT，通过eth0发到外网

转载自: https://github.com/lixd/daily-notes/blob/master/Golang/mydocker/06-1-%E7%BD%91%E7%BB%9C%E8%99%9A%E6%8B%9F%E5%8C%96%E6%8A%80%E6%9C%AF.md