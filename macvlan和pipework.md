
# 基础概念

首先来回顾一下网络的基础概念, 网桥和网关

## 网桥

网桥（Bridge）是一个局域网与另一个局域网之间建立连接的桥梁. 是一种二层网络设备, 

## 网关

网关实质上是一个网络通向其他网络的IP地址。

比如有网络A和网络B，网络A的IP地址范围为“192.168.1.1~192. 168.1.254”，子网掩码为255.255.255.0；网络B的IP地址范围为“192.168.2.1~192.168.2.254”，子网掩码为255.255.255.0。

在没有路由器的情况下，两个网络之间是不能进行TCP/IP通信的，即使是两个网络连接在同一台交换机(或集线器)上，TCP/IP协议也会根据子网掩码(255.255.255.0)判定两个网络中的主机处在不同的网络里。

而要实现这两个网络之间的通信，则必须通过网关。如果网络A中的主机发现数据包的目的主机不在本地网络中，就把数据包转发给它自己的网关，再由网关转发给网络B的网关，网络B的网关再转发给网络B的某个主机

# macvlan

macvlan 是linux内核在3.9版本以上提供较新的一种特性,

允许在同一个物理网卡上配置多个 MAC 地址，即多个 interface，每个 interface 可以配置自己的 IP。

macvlan 本质上是一种网卡虚拟化技术，Docker 用 macvlan 实现容器网络就不奇怪了。

macvlan 的最大优点是性能极好，相比其他实现，macvlan 不需要创建 Linux bridge，而是直接通过以太 interface 连接到物理网络

# 简单macvlan环境配置

环境是两台VM, 172.16.104.145/172.16.104.146

需要在两台VM上都打开网卡混杂模式和创建docker的macvlan网络

这种模式下的容器, 和宿主机不能互通, 和其他主机以及上面的容器都可以通信

## 网卡混杂模式设置

```
[root@centos1 ~]# ip link set ens33 promisc on
[root@centos1 ~]# ifconfig
ens33: flags=4419<UP,BROADCAST,RUNNING,PROMISC,MULTICAST>  mtu 1500
        inet 172.16.104.145  netmask 255.255.255.0  broadcast 172.16.104.255
        inet6 fe80::9abe:3a2e:cbd9:3192  prefixlen 64  scopeid 0x20<link>
```

`ifconfig`可以看到有`PROMISC`字样表示混杂模式打开

## 新建macvlan网络

```
docker network create -d macvlan --subnet=172.16.104.0/24 --gateway=172.16.104.1 -o parent=ens33 mynet
```

> 参数说明：
`-d macvlan`:  创建macvlan网络，使用macvlan网络驱动
`--subnet`: 指定宿主机所在网段
`--gateway`: 指定宿主机所在网段网关, 必须是真实存在的网关,否则容器无法路由
`-o parent`: 继承指定网段的网卡

注意:

- 1、macvlan是一种本地网络, docker不会为它创建网关, 
- 2、用户也需要自己管理 IP subnet, 而且不提供DNS服务
- 3、macvlan是一种独占网络, 即一个网卡只能创建一个 macvlan 网络, 如果要支持多个macvlan网络, 必须使用 sub-interface , 具体方法略.

## 创建容器

两台VM上分别创建容器(IP为10/11)之后

```
docker run --net=mynet --ip=172.16.104.10 -itd --name=test1 docker.io/centos bash
```

```
docker run --net=mynet --ip=172.16.104.11 -itd --name=test2 docker.io/centos bash
```

进入容器, 能互相ping通

```
[root@634408c6c399 /]# ping 172.16.104.10
PING 172.16.104.10 (172.16.104.10) 56(84) bytes of data.
64 bytes from 172.16.104.10: icmp_seq=1 ttl=64 time=0.595 ms
```

**使用macVLAN模式的容器，无法ping通宿主机，宿主机也无法ping通该容器，对其他同网段的服务器和容器都可以联通**

```
# 容器ping宿主机IP
[root@634408c6c399 /]# ping 172.16.104.146
PING 172.16.104.146 (172.16.104.146) 56(84) bytes of data.
^C
--- 172.16.104.146 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 2001ms
```

```
# 主机ping另外主机的容器
[root@centos2 ~]# ping 172.16.104.10
PING 172.16.104.10 (172.16.104.10) 56(84) bytes of data.
64 bytes from 172.16.104.10: icmp_seq=1 ttl=64 time=0.762 ms

# 容器内ping另外的主机
[root@634408c6c399 /]# ping 172.16.104.145
PING 172.16.104.145 (172.16.104.145) 56(84) bytes of data.
64 bytes from 172.16.104.145: icmp_seq=1 ttl=64 time=0.513 ms
```

# pipework

pipework是一个400多行的shell程序,封装Linux上的ip、brctl等命令, 简化了在复杂场景下对容器连接的操作命令，为我们配置复杂的网络拓扑提供了一个强有力的工具.

使用pipework命令为容器设置固定IP

格式为: `pipework <网卡> <容器> <IP/掩码>@<网关>`

## 方法一

这种方法直接绑定到网卡上, 和上面方法类似, 容器内ping不同宿主机ip, 其他可以

```
docker run --net=none -itd --name=test1 -h test1 busybox /bin/sh
```

```
pipework ens33 test1 172.16.104.10/24@172.16.104.1
```

```
pipework ens33 test2 172.16.104.11/24@172.16.104.1
```

执行过程大概包括：

查看主机是否包含ens33，如果不存在就创建，向容器实例test1添加一块网卡，并配置固定IP：172.16.104.10，若test1已经有默认的路由，则删除掉，将@后面的172.16.104.1设置为默认路由的网关，将test1容器实例连接到创建的ens33上。

```
[root@centos1 network-scripts]# docker exec test1 ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
18: eth1@if2: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue qlen 1000
    link/ether f6:c5:fe:92:08:c0 brd ff:ff:ff:ff:ff:ff
    inet 172.16.104.10/24 brd 172.16.104.255 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::f4c5:feff:fe92:8c0/64 scope link
       valid_lft forever preferred_lft forever
```

容器可以ping通另外的主机了

```
[root@centos1 network-scripts]# docker exec test1 ping 172.16.104.147
PING 172.16.104.147 (172.16.104.147): 56 data bytes
64 bytes from 172.16.104.147: seq=0 ttl=64 time=1.343 ms
64 bytes from 172.16.104.147: seq=1 ttl=64 time=0.553 ms
```

## 方法二

该方法效果较好, 容器和宿主机互通, 但是配置的时候需要通过VNC或者串口连接(单网卡), 要断网

启动容器, pipework绑定过程和方法一相同, 这里省略

```
$ ip addr add 172.16.104.148/24 dev br0
$ ip addr del 172.16.104.148/24 dev ens33
$ brctl addif br0 ens33
$ ip route del default
$ ip route add default via 172.16.104.1 dev br0
```

过程中会断网, 可以写到一条命令中, 但实测过程中并不好用 (因为可能中间哪条抛错下面的就执行不了,  实测中第一次执行时候第四条抛错)

```
ip addr add 172.16.104.148/24 dev br0;\
ip addr del 172.16.104.148/24 dev ens33;\
brctl addif br0 ens33;\
ip route del default;\
ip route add default via 172.16.104.1 dev br0;\
```
