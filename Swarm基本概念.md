
# 概念模型

SwarmKit基于Raft算法在内存中维护集群的状态一致性，在一组Manager选出一个Leader。只有Leader处理所有的请求，其它的Manager只是把请求传给leader,起到了反向代理的作用。

swarm 底层是基于 overlay vxlan 模型

SwarmKit提供了两个可执行程序：swarmd和swarmctl。swarmd部署在cluster中的每一个node上，彼此间互相通信，组成cluster；而swarmctl则用来控制整个集群.

## Service(服务)

一个Service包含完成同一项工作的一组Task，它分为

Global(全局服务模式)， 需要每个node上部署一个task实例，有点像kubernetes中的daemon set，用来部署类似gluster等分布式存储和fluented日志搜集模块这种类型的基础服务

Replicated(重复服务模式)， 需要按照最终用户指定的数量尽可能在不同的节点上部署task的实例

## Task(任务)

作为SwarmKit中的基本调度单元， Task承担了创建docker容器，并且运行指定命令的责任（docker run)。Task一旦被分配到目标机器，它就是不可修改的（immutable），它的结果只能是running或者failed。而且上在未来，Task的工作可以更灵活和插件化。

task定义为

```
type Task struct {
	ID string
	Meta
	Spec                TaskSpec            `json:",omitempty"`
	ServiceID           string              `json:",omitempty"`
	Slot                int                 `json:",omitempty"`
	NodeID              string              `json:",omitempty"`
	Status              TaskStatus          `json:",omitempty"`
	DesiredState        TaskState           `json:",omitempty"`
	NetworksAttachments []NetworkAttachment `json:",omitempty"`
}
```

有多种状态

```
	TaskStateNew TaskState = "new"
	TaskStateAllocated TaskState = "allocated"
	TaskStatePending TaskState = "pending"
	TaskStateAssigned TaskState = "assigned"
	TaskStateAccepted TaskState = "accepted"
	TaskStatePreparing TaskState = "preparing"
	TaskStateReady TaskState = "ready"
	TaskStateStarting TaskState = "starting"
	TaskStateRunning TaskState = "running"
	TaskStateComplete TaskState = "complete"
	TaskStateShutdown TaskState = "shutdown"
	TaskStateFailed TaskState = "failed"
	TaskStateRejected TaskState = "rejected"
```

task 的 DesiredState 在 task 创建之后是 TaskStateRunning，如果 manager 想关闭 task，把 DesiredState 设置成 TaskStateShutdown(DesiredState 只能被 manager 修改)。


## Cluster(集群)

一个 cluster 由一组统一配置的的装有docker引擎的节点连接起来完成计算工作

## Node(节点）

Node 是集群的基本组成单元，其身份分为Manager和Worker

## Manager(管理器)

Manager 负责接收用户创建的 _Service_, 并且根据 service的定义创建一组task，根据task所需分配计算资源和选择运行节点，并且将task调度到指定的节点。而manager含有以下子模块：

```
type Manager struct {
	config    *Config
	listeners map[string]net.Listener
	caserver               *ca.Server
	Dispatcher             *dispatcher.Dispatcher
	replicatedOrchestrator *orchestrator.ReplicatedOrchestrator
	globalOrchestrator     *orchestrator.GlobalOrchestrator
	taskReaper             *orchestrator.TaskReaper
	scheduler              *scheduler.Scheduler
	allocator              *allocator.Allocator
	keyManager             *keymanager.KeyManager
	server                 *grpc.Server
	localserver            *grpc.Server
	RaftNode               *raft.Node
	connSelector           *raftpicker.ConnSelector
	mu sync.Mutex
	started chan struct{}
	stopped chan struct{}
}
```

集群信息是保存到go-memdb一个内存数据库中,通过raft+etcd控管.

### Orchestrator(编排器)

Orchestrator负责确保每个service中的task按照service定义正确的运行

swarmkit 支持两种 service 类型，一种是 global service，一种是 replicated service，对于 global service 它的 task 会运行在所有的 node 中，除非 node 被 drain 掉；对于 replicated service，需要一个 num，即运行 task 的数量，比如 task 数量是 10，那么 service 的 Slot 从 1 到 10。

####  global service

```
type GlobalOrchestrator struct {
	store *store.MemoryStore
	// nodes contains nodeID of all valid nodes in the cluster
	nodes map[string]struct{}
	// globalServices have all the global services in the cluster, indexed by ServiceID
	globalServices map[string]*api.Service

	// stopChan signals to the state machine to stop running.
	stopChan chan struct{}
	// doneChan is closed when the state machine terminates.
	doneChan chan struct{}

	updater  *UpdateSupervisor
	restarts *RestartSupervisor

	cluster *api.Cluster // local instance of the cluster
}

```

#### replicated service

```
type ReplicatedOrchestrator struct {
	store *store.MemoryStore

	reconcileServices map[string]*api.Service
	restartTasks      map[string]struct{}

	// stopChan signals to the state machine to stop running.
	stopChan chan struct{}
	// doneChan is closed when the state machine terminates.
	doneChan chan struct{}

	updater  *UpdateSupervisor
	restarts *RestartSupervisor

	cluster *api.Cluster // local cluster instance
}
```

**总结一下，Orchestrator 负责从「数据库」拿到service的变更并执行，至于资源分配和调度就不是它的工作了。**


### Allocator(资源分配器)

Allocator主要负责分配资源，而这种资源通常为全局的，比如overlay网络的ip地址和分布式存储，目前只是实现是vip地址分配。未来一些自定义资源也可以通过Allocator来分配。

定义在`vendor\src\github.com\docker\swarmkit\manager\allocator\allocator.go`中

```
type Allocator struct {
	store *store.MemoryStore

    # 用于在所有分配器之间进行同步的选票，以确保所有分配器都已完成其各自的分配，以便任务可以转移到ALLOCATED状态。
	taskBallot *taskBallot

	netCtx *networkContext
	stopChan chan struct{}
	doneChan chan struct{}
}
```



可以看到,Allocator数据存储在内存中

### Scheduler(调度器)

Scheduler负责将Service中定义的task调度到可用的Node上

```
type Scheduler struct {
	store           *store.MemoryStore
	unassignedTasks *list.List  #保存没有 attach node 的 task
	preassignedTasks map[string]*api.Task #保存已经 attach node 但是需要确认资源的 task
	nodeHeap         nodeHeap #以 node id 保存 node 上的 task 信息，同时会保存 node 自身信息和 node 可用资源信息(包括 CPU 和 MEMORY)
	allTasks         map[string]*api.Task #allTasks 记录所有 task(必须是 ALLOCATED 状态，也就是必须先收集网络资源才会被调度)；
	pipeline         *Pipeline  #pipeline 记录 Filter 信息，用于判断过滤 node。
	stopChan chan struct{}
	doneChan chan struct{}
	scanAllNodes bool
}
```

### Dispatcher(分发器）

Dispatcher直接处理与所有agent的连接， 这里包含agent的注册，session的管理以及每个task的部署和状态追踪。

Dispatcher 只运行在 manager 中的 leader 上面

定义为:

```
type Dispatcher struct {
	mu                   sync.Mutex
	addr                 string
	nodes                *nodeStore
	store                *store.MemoryStore
	mgrQueue             *watch.Queue
	lastSeenManagers     []*api.WeightedPeer
	networkBootstrapKeys []*api.EncryptionKey
	keyMgrQueue          *watch.Queue
	config               *Config
	cluster              Cluster
	ctx                  context.Context
	cancel               context.CancelFunc

	taskUpdates     map[string]*api.TaskStatus // indexed by task ID
	taskUpdatesLock sync.Mutex

	processTaskUpdatesTrigger chan struct{}
}
```


### RaftNode


### Manager初始化

```
func New(config *Config) (*Manager, error) 
-> 
```


## Agent

Agent中有 Config, task, Storage使用了boltdb


gent 负责管理Node，部署Task以及追踪Task的状态，并且将Task的状态汇报给Manager。Agent包含以下子模块:

```
type Agent struct {
    ...
    node *api.Node
    worker   Worker
    ...
}
```

### api.Node(节点状态)

api.Node 负责向Manager定期汇报所在节点实际状态. 当一个节点的这实际状态和期望状态不一致时，Manager会自动将服务中的任务调度到其他节点，以保证服务的正常运行。

### Worker(任务处理器）

Worker 处理以下工作：

- 部署和启动Task
- Task状态追踪和汇报
- 对于部署在本机上的task内容及状态的持久化

```
type worker struct {
	db           *bolt.DB
	executor     exec.Executor
	// listeners 负责监听要任务上报的信息并上报给 Dispatcher
	listeners    map[*statusReporterKey]struct{} 
	// taskManagers 用来管理任务的执行
	taskManagers map[string]*taskManager
	mu           sync.RWMutex
}
```

```
type taskManager struct {
	task     *api.Task
	ctlr     exec.Controller
	// reporter 用来报告任务的状态，包括本地 DB 和 Manger，reporter 由 worker 声明
	reporter StatusReporter
	updateq  chan *api.Task
	shutdown chan struct{}
	closed   chan struct{}
}
```
