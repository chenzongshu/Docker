
Agent中有 Config, task, Storage使用了boltdb


agent 负责管理Node，部署Task以及追踪Task的状态，并且将Task的状态汇报给Manager。Agent包含以下子模块:

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

```
Agent.run()
-> newSession() 创建一个新的session
-> worker.Init()  #Init方法事实上是确保已经在本地DB中的 task 正确运行，对于新的 task 在后面看如何处理。
   -> WalkTasks()遍历 DB(本地存储，task 会存在本地) 中的 task，
   -> TaskAssigned()判断如果 task 没有 assigned 给一个 node，则DeleteTask删除 task，
   -> 否则从 DB 中获取 task 的状态
   -> 覆盖掉当前状态，
   -> startTask()然后启动 task
      -> taskManager()
         -> 如果task 在 taskManagers，直接返回
         -> 否则newTaskManager()
            -> exec.Resolve() 方法获取 Controller 和 状态，其中状态会被改成 TaskStateAccepted，
            -> updateTaskStatus()保存状态
               -> PutTaskStatus()
               -> 循环广播到所有listeners
            -> 最后调用另一个 newTaskManager 来创建 taskManager 结构并运行
-> newStatusReporter()

-> worker.Listen()

```

处理逻辑:

> 1). 如果状态为 TaskStatePreparing，则调用 Controller 的 Prepare 方法，并设置状态为 TaskStateReady；
2). 如果状态为 TaskStateStarting，则调用 Controller 的 Start 方法，并设置状态为 TaskStateRunning；
3). 如果状态为 TaskStateRunning，则调用 Controller 的 Wait 方法，如果 task 执行完了，状态改为 TaskStateCompleted；
4). 如果状态为 TaskStateNew、TaskStateAllocated、TaskStateAssigned 则改为 TaskStateAccepted；
5). 如果状态为 TaskStateAccepted，则改成 TaskStatePreparing；
6). 如果状态为 TaskStateReady，则改成 TaskStateStarting。

实际上是新建一个 task 的情况，状态是 TaskStateAccepted，套用状态变更流程，先会改成 TaskStatePreparing，然后改成 TaskStateReady，然后改成 TaskStateStarting，再改成 TaskStateRunning。

事实上，taskManager 结构在运行开始就会调用一次处理流程，以保证 TaskStateAccepted 状态的 task 立即被执行，而且 taskManager 还接收 task 的 update，如果和当前维护 task 的配置(CPU、MEMORY 等)不同，则进入处理流程

### Agent归纳

1. agent 基于自己的 UpdateTaskStatus 包装出满足 StatusReporter 接口的 statusReporter，并把 statusReporter 注册到到 worker.listeners，当有新状态时 statusReporter 后台自动调用 agent 的 UpdateTaskStatus 上报(到Dispatcher)

> 调用到 Dispatcher RPC 请求，保存到 store 中，Dispatcher 的 UpdateTaskStatus 实现此功能

2. worker 会为每一个 task 分配 taskManager，taskManager 会包含上报函数，当有状态变更时，会执行 statusReporter 的 UpdateTaskStatus 实现向 statusReporter 的状态列表中增加状态。

3. agent 会监听 Dispatcher 来的信息，包括 session.assignments 和 session.messages，agent 会分别处理这两个信息

- 对于 session.assignments，是任务信息，分为全量信息和增量信息，增量信息包括该更新的任务和该删除的任务。

对于更新的任务：

> 1). 先把任务保存到本地 DB 并标记为 assigned；
2). 如果任务已经在 taskManagers，则更新，任务的配置不一样时才会真正更新，状态会被忽略，比如 CPU、MEMORY 变化，会更新；
3). 如果任务不在 taskManagers，看本地 DB 里是否存在，如果存在，更新任务的状态(本地 DB 的状态更准)，如果不在本地 DB，则保存状态，然后启动任务。

对于删除的任务:

> 1). 删除本地 DB 的 assigned 状态；
2). 从 taskManagers 中删除；
3). 关闭 taskManager。

全量信息先全部当做更新的任务处理，但是当前运行的任务如果不在全量列表里则会被删除。

-  对于 session.messages，包括的功能有：

> 1). 更新 Manager 列表，保持最新(agent 来负责更新 Manager 列表)；
2). worker 的角色是否有变(NotifyRoleChange)；
3). 保持 NetworkBootstrapKeys 最新。

启动任务，会调用 worker.taskManager，创建新的 taskManager 并加到 taskManagers

node节点信息使用boltdb存在`worker/tasks.db`
