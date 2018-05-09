
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

### Init

收到swarm的init命令的时候开始初始化,**这个流程潜藏的含义就是,这个节点是一个Manager Leader节点**

**注意,daemon的cluster中的node和cluster,与raft中node,cluster定义名字相同,但是内容不同**

```
type MemoryStorage struct {
	sync.Mutex   //MemoryStorage主要跑在raft的goroutine上,也有部分在应用的协程上
	hardState pb.HardState
	snapshot  pb.Snapshot
	ents []pb.Entry  // ents[i] has raft log position i+snapshot.Metadata.Index
}

type HardState struct {
	Term             uint64 `protobuf:"varint,1,opt,name=term" json:"term"`
	Vote             uint64 `protobuf:"varint,2,opt,name=vote" json:"vote"`
	Commit           uint64 `protobuf:"varint,3,opt,name=commit" json:"commit"`
	XXX_unrecognized []byte `json:"-"`
}

type Snapshot struct {
	Data   []byte      `protobuf:"bytes,1,opt,name=data" json:"data,omitempty"`
	Metadata   SnapshotMetadata `protobuf:"bytes,2,opt,name=metadata" json:"metadata"`
	XXX_unrecognized []byte  json:"-"`
}

type Entry struct {
	Type             EntryType `protobuf:"varint,1,opt,name=Type,enum=raftpb.EntryType" json:"Type"`
	Term             uint64    `protobuf:"varint,2,opt,name=Term" json:"Term"`
	Index            uint64    `protobuf:"varint,3,opt,name=Index" json:"Index"`
	Data             []byte    `protobuf:"bytes,4,opt,name=Data" json:"Data,omitempty"`
	XXX_unrecognized []byte    `json:"-"`
}
```

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
	 -> 起一个协程监控角色变化，并广播
	 -> 起一个协程监控更新TLS和CA，并广播
	 -> 起一个协程监控跑runManager(),把Manager拉起来
	    -> manager.New 新建一个Manager对象
	       -> 构建IP地址
	       -> 初始化raft配置: 心跳tick=1,选举tick=10,
	       -> NewNode()新建raft节点
	          -> raft.NewMemoryStorage 新建存储raftStore,在raft的wal和snapshot里面
	          -> 新建一个raft节点n,里面会新建cluster,新建一个Broadcaster,设置Timeout为2,并把上面新建的raftStore赋为config里面的存储
	          -> store.NewMemoryStore新建存储赋值给n.memoryStore,这里存储在memDB里面
	          -> 最后New了一个Manager对象
	        -> 起一个goroutine调用run()把Manager跑起来
	           -> 订阅一个leadership通知的channel,赋给leadershipCh
	           -> 循环判断leadershipCh中的事件状态
	              1. 如果是leader
	                 -> 取出前面在memDB中分配的内存赋值给s
	                 -> 创建一个"cluster"表,"node"表并更新的memDB中
	                 -> 新建一个CA认证
	                 -> 新建一个replicated类型的Orchestrator
	                 -> 新建一个global类型的Orchestrator
	                 -> 新建一个taskReaper
	                 -> 新建一个scheduler
	                 -> 新建一个keyManager
	                 -> 新建一个allocator
	                 -> 分别起一个goroutine把上面的对象run起来
	              2. 如果是follower,把所有的分配器,调度器什么的全部停掉
	           -> raftpicker.NewConnSelector() 返回带有cluster和grpc.DialOpts的新的ConnSelector用于连接创建
	           -> 创建了一堆server服务
	           -> m.RaftNode.JoinAndStart新建或者读取raft信息
	              -> loadAndStart()
	                 -> new了snapshotter,赋值到node里面n.snapshotter
	                 -> 如果wal不存在则报错
	                 -> 读取snapshot,
	                 -> 如果上面读取的snapshot不为空,restoreFromSnapshot()写到memDB中并更新内存中的数据
	                 -> readWAL()读取日志,同步操作日志,以便更新信息
	              -> 如果没有wal文件
	                 1. 如果join地址不为空
	                    -> ConnectToMember()连接join地址 // 使用grpc
	                    -> NewRaftMembershipClient()
	                    -> createWAL()
	                    -> raft.StartNode()
	                    -> n.registerNodes()
	                 2. 如果join地址为空,表明是集群第一个成员
	                    -> createWAL()
	                    -> raft.StartNode()
	              -> raft.RestartNode()
	                 -> newRaft()
	                 -> newNode()
	                 -> 起协程run起来
	              -> atomic.StoreUint32设置 n.member为1
	           -> 起一个协程m.RaftNode.Run()
	           -> raft.WaitForLeader()
	           -> raft.WaitForCluster()
	        -> initManagerConnection()起一个goroutine建立连接
	 -> 起一个协程监控跑runAgent(),把Agent拉起来,这个时候把db传了进去
            -> new一个agent对象
            -> agent.Start
	        -> 响应agent的stop,close,和完成事件
               -> agent.run()
                  -> worker.Init() 初始化
                     -> 读取tasks.db中的task,如果没有被指派则删除,否则获取状态,启动新的taskManager,并更新到数据
                  -> newStatusReporter() 新建状态上报器
                     -> statusReporter.run()
                        -> 做一个死循环,直到closed标致位被置为ture才退出
                        -> 循环statusReporter中的statuses这个map
                        -> 删除任务,再把UpdateTaskStatus把status发出去
                        -> 起一个协程调用sendTaskStatus发送状态
                  -> worker.Listen
                  -> for循环做多个channel的监听处理,重要的有:
                     1. session.closed通道有值时,起一个新的session
                     2. session.errs通道有值时,关闭session
                     3. session.tasks通道有值: worker.Assign()处理任务
                     4. session.messages通道有值: 处理消息
      -> 把节点信息保存在Cluster结构体中,并调用saveState()并把state写入docker-state.json
      -> 起三个协程,分别处理 清理cluster的node和err信息; 处理事件; 建立grpc的连接
   -> 节点通道收到事件分别处理(节点第一次初始化完成之后会关闭通道?)

```

