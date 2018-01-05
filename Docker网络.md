Libnetwork作为Docker网络部分的依赖库，实现了CNM（Container Network Model）。

# 概念

- IPAM： CNM通过IPAM(IP address management)驱动管理IP地址的分配

# 驱动

docker自带4个网络驱动,还支持第三方驱动

- Bridge: 这是启动容器的默认网络。通过docker主机上的网桥接口实现连接。使用相同网桥的容器有自己的子网，并且可以相互通信（默认情况下）。
- Host:这个驱动程序允许容器访问docker主机自己的网络空间（容器将看到和使用与docker主机相同的接口）。
- Macvlan:此驱动程序允许容器直接访问主机的接口或子接口（vlan）。 它还允许中继链接。
- Overlay：此驱动程序允许在运行docker的多个主机（通常是docker群集群）上构建网络。 容器还具有自己的子网和网络地址，并且可以直接相互通信，即使它们在不同的物理主机上运行。


# 组件

CNM主要有3个组件：

- 沙盒（Sandbox）：一个沙盒包含了一个容器网络栈的信息。沙盒可以对容器的接口，路由和DNS设置等进行管理。沙盒的实现可以是Linux Network Namespace, FreeBSD Jail或者类似的机制。一个沙盒可以有多个端点（Endpoint）和多个网络（Network）。

```
type sandbox struct {
	id                 string
	containerID        string
	config             containerConfig
	extDNS             []string
	osSbox             osl.Sandbox
	controller         *controller
	resolver           Resolver
	resolverOnce       sync.Once
	refCnt             int
	endpoints          epHeap
	epPriority         map[string]int
	populatedEndpoints map[string]struct{}
	joinLeaveDone      chan struct{}
	dbIndex            uint64
	dbExists           bool
	isStub             bool
	inDelete           bool
	ingress            bool
	sync.Mutex
}
```

- 端点（Endpoint）：一个端点可以加入一个沙盒和一个网络。端点的实现可以是veth pair, Open vSwitch内部端口或者相似的设备。一个端点只可以属于一个网络并且只属于一个沙盒。

```
type endpoint struct {
	name              string
	id                string
	network           *network
	iface             *endpointInterface
	joinInfo          *endpointJoinInfo
	sandboxID         string
	locator           string
	exposedPorts      []types.TransportPort
	anonymous         bool
	disableResolution bool
	generic           map[string]interface{}
	joinLeaveDone     chan struct{}
	prefAddress       net.IP
	prefAddressV6     net.IP
	ipamOptions       map[string]string
	aliases           map[string]string
	myAliases         []string
	svcID             string
	svcName           string
	virtualIP         net.IP
	svcAliases        []string
	ingressPorts      []*PortConfig
	dbIndex           uint64
	dbExists          bool
	sync.Mutex
}
```

- 网络（Network）：一个网络是一组可以直接互相联通的端点。网络的实现可以是Linux bridge，VLAN等等。一个网络可以包含多个端点。

# 对象

- NetworkController：NetworkController通过暴露给用户创建和管理网络的API，以便用户对libnetwork进行调用。Libnetwork支持多种驱动，其中包括内置的bridge，host，container和overlay，也对远程驱动进行了支持（即支持用户使用自定义的网络驱动）。

- Driver：Driver不直接暴露给用户接口，但是driver是真正实现了网络功能的对象。NetworkController可以对特定driver提供特定的配置选项，其选项对libnetwork透明。为了满足用户使用需求和开发需求，driver既可以是内置的类型，也可以是用户指定的远程插件。每一种驱动可以创建自己独特类型的network，并对其进行管理。在未来可以会对不同插件的管理功能进行整合，以求做到方便管理不同类型的网络。

- Network：Network对象是CNM的一种实现。NetworkController通过提供API对Network对象进行创建和管理。NetworkController当需要创建或者更新Network的时候，它所对应的Driver会得到通知。Libnetwork通过抽象层面对同属于一个网络的各个endpoint提供通信支持，同时将其与其他endpoint进行隔离。这种通信支持可以是同一主机的，也可以是跨主机的。

- Endpoint：Endpoint代表了服务端点。它可以提供不同网络中容器的互联。Endpoint由创建对应Sandbox的驱动进行创建。

- Sandbox：Sandbox代表了一个容器中诸如ip地址，mac地址，routes，DNS等信息的配置。Sandbox在用户要求在一个Network创建Endpoint的时候进行创建并分配对应的网络资源。Libnetwork会用特定的操作系统机制（例如：linux的netns）去填充sandbox对应的容器中的网络配置。一个sandbox中可以有多个连接到不同网络的endpoint。

# 代码逻辑

总的来说，Endpoint是由Network创建的，隶属于这个创建它的Network，当Endpoint加入Sandbox的时候，就相当于这个Sandbox加入到了这个Network中来。

```
-> initNetworkController() [daemon_unix.go]
-> libnetwork.New() [controller.go] # 创建NetWorkController实例。这个实例提供了各种接口，Docker可以通过它创建新的NetWork和Sandbox等
   1. c.initStores() #初始化kv存储,保存网络信息
   2. drvregistry.New() #注册driver
   3. getInitializers() #循环初始化了各种网络模式
   4. 初始化IPMA driver
   5. 清理资源 #sandbox,endpoint等
-> controller.NewNetwork() #在initNetworkController()函数中
   1. allocate一个ipma
   2. 保存endpoint到数据库
```

创建endpoint流程

```
-> restore() [daemon.go] / ContainerStart() <-postContainersStart() / Start()
-> containerStart() [start.go]
-> initializeNetworking() [container_operations.go]
-> allocateNetwork()
-> connectToNetwork()
    1. daemon.updateNetworkConfig() #更新网络配置
    2. getNetworkSandbox() #获取sandbox
    3. n.CreateEndpoint() #创建endpoint
    4. daemon.updateEndpointNetworkSettings() #更新endpoint设置
    5. 没有sandbox则新建一个
    6. ep.Join() #将Endpoint加入指定的Sandbox中，则这个Sandbox也会加入创建Endpoint对应的Network中
```
