Docker Daemon本身是一个单机版本的容器,后面加入swarm,才产生了集群的概念.代码位于 \daemon\cluster 文件夹和 \vendor\...\swarmkit\文件夹

# 原理


# 初始化

## 数据结构

Cluster结构体位于daemon中,里面包含了集群信息

```
type Cluster struct {
	sync.RWMutex
	*node
	root            string
	config          Config
	configEvent     chan struct{} // todo: make this array and goroutine safe
	localAddr       string
	actualLocalAddr string // after resolution, not persisted
	remoteAddr      string
	listenAddr      string
	advertiseAddr   string
	stop            bool
	err             error
	cancelDelay     func()
}

type node struct {
	*swarmagent.Node
	done           chan struct{}
	ready          bool
	conn           *grpc.ClientConn
	client         swarmapi.ControlClient
	reconnectDelay time.Duration
}

type Config struct {
	Root                   string
	Name                   string
	Backend                executorpkg.Backend
	NetworkSubnetsProvider NetworkSubnetsProvider
	DefaultAdvertiseAddr string   //默认的主机,IP,网络端口
}
```
其中,`swarmagent.Node`是swarmkit中定义的Node相关结构体,代码位于`\vendor\src\github.com\docker\swarmkit\agent\node.go`

```
type Node struct {
	sync.RWMutex
	config               *NodeConfig
	remotes              *persistentRemotes
	role                 string
	roleCond             *sync.Cond
	conn                 *grpc.ClientConn
	connCond             *sync.Cond
	nodeID               string
	nodeMembership       api.NodeSpec_Membership
	started              chan struct{}
	stopped              chan struct{}
	ready                chan struct{} // closed when agent has completed registration and manager(if enabled) is ready to receive control requests
	certificateRequested chan struct{} // closed when certificate issue request has been sent by node
	closed               chan struct{}
	err                  error
	agent                *Agent
	manager              *manager.Manager
	roleChangeReq        chan api.NodeRole // used to send role updates from the dispatcher api on promotion/demotion
}
```


## 流程

收到swarm的init命令的时候开始初始化,**这个流程潜藏的含义就是,这个节点是一个Manager Leader节点**

```
-> swarmRouter.initCluster()
-> Cluster.Init()
   -> 判断集群是否存在,判断是不是需要强行启动新集群,如果不满足则返回错误
   -> 解析设置了一些默认值:raft的心跳间隔10s,心跳tick为1,选举tick为10,Dispatcher心跳周期5s,Orchestration的task历史租约限制为10
   -> startNewNode()
      -> 如果没输入IP,则单独处理
      -> swarmagent.NewNode() [\swarmkit\agent\node.go]
         -> 读取了state.json文件
	 -> 读取CA
      -> n.Start() 启动节点
         -> 读取CA
	 -> 使用boltdb从 /work/task.db中读取db
	 -> 起一个协程监控角色变化
	 -> 
      -> 把节点信息保存在Cluster结构体中,并调用saveState()并把state写入docker-state.json
      -> 起三个协程,分别处理 清理cluster的node和err信息; 处理事件; 建立grpc的连接
   -> 节点通道收到事件分别处理(节点第一次初始化完成之后会关闭通道?)
   -> 

```
