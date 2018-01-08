Docker使用linux iptables来控制与它创建的接口和网络之间的通信

# 初始化

docker安装好的时候，本身是没有任何添加任何iptables规则的，当docker Daemon启动的时候，nat将会增加 DOCKER链；filter表增加DOCKER和DOCKER-ISOLATION链

只是启动daemon的时候, 是没有创建overlay网络的.

```
root@computer-node4 /root >iptables -L -t nat
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination
DOCKER     all  --  anywhere             anywhere             ADDRTYPE match dst-type LOCAL

Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
DOCKER     all  --  anywhere            !loopback/8           ADDRTYPE match dst-type LOCAL

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
MASQUERADE  all  --  172.17.0.0/16        anywhere

Chain DOCKER (2 references)
target     prot opt source               destination
RETURN     all  --  anywhere             anywhere


root@computer-node4 /home/czs >iptables -L -t filter
Chain INPUT (policy ACCEPT)
target     prot opt source               destination
ACCEPT     udp  --  anywhere             anywhere             udp dpt:domain

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination
DOCKER-ISOLATION  all  --  anywhere             anywhere
DOCKER     all  --  anywhere             anywhere

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
ACCEPT     udp  --  anywhere             anywhere             udp dpt:bootpc

Chain DOCKER (1 references)
target     prot opt source               destination

Chain DOCKER-ISOLATION (1 references)
target     prot opt source               destination
RETURN     all  --  anywhere             anywhere

```

如果这个时候新建一个swarm网络，docker会默认新建一个名称是ingress的overlay网络，**但是这个时候对应的iptables规则并没有建立好**

在swarm下启动一个容器(使用端口映射)，即使用该overlay网络启动一个容器，这个时候会在filter和nat表里面新建一条DOCKER-INGRESS链

```
...... 其他省略,可以看到在filter表里面, FORWARD链里面增加一条规则,并新增了DOCKER-INGRESS链 ......
Chain FORWARD (policy ACCEPT)
target     prot opt source               destination
DOCKER-INGRESS  all  --  anywhere             anywhere
DOCKER-ISOLATION  all  --  anywhere             anywhere

Chain DOCKER-INGRESS (1 references)
target     prot opt source               destination
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:radan-http
ACCEPT     tcp  --  anywhere             anywhere             state RELATED,ESTABLISHED tcp spt:radan-http
RETURN     all  --  anywhere             anywhere

...... 其他省略,可以看到在nat表里面, OUTPUT链里面增加一条规则,并新增了DOCKER-INGRESS链 ......
Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
DOCKER-INGRESS  all  --  anywhere             anywhere             ADDRTYPE match dst-type LOCAL
DOCKER     all  --  anywhere            !loopback/8           ADDRTYPE match dst-type LOCAL

Chain DOCKER-INGRESS (2 references)
target     prot opt source               destination
DNAT       tcp  --  anywhere             anywhere             tcp dpt:radan-http to:172.18.0.2:8088
RETURN     all  --  anywhere             anywhere
```

# filter表

filter表是网络或接口的流量的安全规则表

过滤器表具有3个链：用于处理到达主机并且去往同一主机的分组的输入链，用于发送到外部目的地的主机的分组的输出链，以及用于进入主机但具有目的地外部主机

```
Chain DOCKER (2 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 ACCEPT     tcp  --  !docker_gwbridge docker_gwbridge  anywhere             172.18.0.5           tcp dpt:4534
    0     0 ACCEPT     tcp  --  !docker_gwbridge docker_gwbridge  anywhere             172.18.0.5           tcp dpt:4533
    0     0 ACCEPT     tcp  --  !docker_gwbridge docker_gwbridge  anywhere             172.18.0.5           tcp dpt:4532
    0     0 ACCEPT     tcp  --  !docker_gwbridge docker_gwbridge  anywhere             172.18.0.6           tcp dpt:4531

Chain DOCKER-INGRESS (1 references)
 pkts bytes target     prot opt in     out     source               destination         
   10  1044 ACCEPT     tcp  --  any    any     anywhere             anywhere             tcp dpt:4511
    6  2030 ACCEPT     tcp  --  any    any     anywhere             anywhere             state RELATED,ESTABLISHED tcp spt:4511
 1215  228K ACCEPT     tcp  --  any    any     anywhere             anywhere             tcp dpt:4510
  799 9728K ACCEPT     tcp  --  any    any     anywhere             anywhere             state RELATED,ESTABLISHED tcp spt:4510
  282 23069 RETURN     all  --  any    any     anywhere             anywhere            

Chain DOCKER-ISOLATION (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 DROP       all  --  docker_gwbridge docker0  anywhere             anywhere            
    0     0 DROP       all  --  docker0 docker_gwbridge  anywhere             anywhere            
  282 23069 RETURN     all  --  any    any     anywhere             anywhere            
```

`DOCKER-ISOLATION`包含限制不同容器网络之间的访问的规则

# Nat表

Docker使用nat允许桥接网络上的容器与docker主机之外的目的地进行通信（否则指向容器网络的路由必须在docker主机的网络中添加）

```
Chain DOCKER (2 references)
target     prot opt source               destination         
RETURN     all  --  anywhere             anywhere            
RETURN     all  --  anywhere             anywhere            
DNAT       tcp  --  anywhere             anywhere             tcp dpt:4534 to:172.18.0.5:4534
DNAT       tcp  --  anywhere             anywhere             tcp dpt:4533 to:172.18.0.5:4533
DNAT       tcp  --  anywhere             anywhere             tcp dpt:4532 to:172.18.0.5:4532
DNAT       tcp  --  anywhere             anywhere             tcp dpt:4531 to:172.18.0.6:4531

Chain DOCKER-INGRESS (2 references)
target     prot opt source               destination         
DNAT       tcp  --  anywhere             anywhere             tcp dpt:4511 to:172.18.0.2:4511
DNAT       tcp  --  anywhere             anywhere             tcp dpt:4510 to:172.18.0.2:4510
RETURN     all  --  anywhere             anywhere            

```

# 代码逻辑

从上面的叙述可以知道, docker的iptables添加可以分成两部分, 一部分是daemon在启动的时候,一部分是容器在启动的时候.

## 

在createNetwork()[bridge.go]中,遍历循环一个结构体{Condition bool ; Fn setupStep}, 里面保存了创建网络的步骤, 其中一步为 network.setupIPTables()

在setupIPTables()里面,判断config.Internal,如果为真,则调用setupInternalNetworkRules()(?看含义为真是内部网络,还设置网络规则?); 如果为假就调用一系列设置: 先调用setupIPTablesInternal(),再getDriverChains()获取了需要设置的规则,最后用ProgramChain()函数设置了Filter表和Nat表的链;


removeIPChains() 删除规则链:
调用了 chain.Remove() 函数删除了Nat表的DOCKER链，Filter表的DOCKER、DOCKER-ISOLATION两个链。

```
-> containerStart() [start.go]
-> daemon.initializeNetworking()
   1. getNetworkedContainer()
   2. allocateNetwork()
      (1). controller.SandboxDestroy()  #清理旧的沙盒
      (2). 判断容器网络设置如果为空,更新一把
      (3). connectToNetwork()
           a). updateNetworkConfig()
           b). getNetworkSandbox()
           c). updateEndpointNetworkSettings()
           d). BuildJoinOptions()
           e). ep.Join()
               -> 判断网络和endpoint是否存在,存在则退出
               -> 各种获取,赋值,更新
               -> populateNetworkResources() #ep填到sb中
               -> addToCluster()
               -> ProgramExternalConnectivity()
                  -> allocatePorts()
                  -> allocatePort()
                     -> MapRange() [mapper.go]  #容器地址端口映射到主机
                        -> pm.forward  #设置Iptables
                        -> Forward() [iptables.go]
```
