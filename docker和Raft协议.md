docker swarmkit中需要多节点,每个节点之间需要同步数据,其使用了raft协议来完成数据一致性

# Raft协议介绍

Raft协议为了解决数据一致性而诞生,诞生比Paxos协议易于理解.

## 基本概念

### 角色

Raft算法将Server划分为3种角色：

- Leader : 负责Client交互和log复制，同一时刻系统中最多存在1个
- Follower : 被动响应请求RPC，从不主动发起请求RPC
- Candidate : 又称为候选者,由Follower向Leader转换的中间状态,只有候选者能发起选举

### Terms

terms简单来说,就是一个时间戳,表明选举的轮数

1. 每个Term至多存在1个Leader
2. 某些Term由于选举失败，不存在Leader
3. 每个Server本地维护currentTerm

### Heartbeats and Timeouts

选出 Leader 后，Leader 通过定期向所有 Follower 发送心跳信息维持其统治。若 Follower 一段时间未收到 Leader 的心跳则认为 Leader 可能已经挂了再次发起选主过程。

1. 所有的Server均以Follower角色启动，并启动选举定时器
2. Follower期望从Leader或者Candidate接收RPC
3. Leader必须广播Heartbeat重置Follower的选举定时器
4. 如果Follower选举定时器超时，则假定Leader已经crash，发起选举

## 选举过程

要实现一致性，**管理节点的数量必须时刻为奇数（1、3、5、7等）**。如果你只有两个管理节点，万一其中一个崩溃了，就会导致无法取得一致的状况出现。理由是——超过 50% 的管理节点在实际运用 Raft 一致性算法前，都需要得到另一方的“同意”。

自增currentTerm，由Follower转换为Candidate，设置votedFor为自身，并行发起RequestVote RPC，不断重试，直至满足以下任一条件：
1. 获得超过半数Server的投票，转换为Leader，广播Heartbeat
2. 接收到合法Leader的AppendEntries RPC，转换为Follower
3. 选举超时，没有Server选举成功，自增currentTerm，重新选举

### 错误处理

1. Candidate在等待投票结果的过程中，可能会接收到来自其它Leader的AppendEntries RPC。如果该Leader的Term不小于本地的currentTerm，则认可该Leader身份的合法性，主动降级为Follower；反之，则维持Candidate身份，继续等待投票结果
2. Candidate既没有选举成功，也没有收到其它Leader的RPC，这种情况一般出现在多个节点同时发起选举（如图Split Vote），最终每个Candidate都将超时。为了减少冲突，这里采取“随机退让”策略，每个Candidate重启选举定时器（随机值），大大降低了冲突概率


# Swarmkit和Raft

每个manager 创建两个 grpc server 以及对应的两个listener(
"Listening for connections" addr="[::]:2377" proto=tcp : 外部server
"Listening for local connections" addr="/var/lib/docker/swarm/control.sock" proto=unix  : localserver
)

针对外部使用server 注册多个服务（service），gRPC里有两种通讯方式，一个是Unary方式（单次请求方式），另外一个就是Stream（流式请求方式）

定义的MethodDesc就在Server.go 文件里，然而StreamDesc的定义是在stream.go里

## 初始化

在Swarm的Manager结构体中,有一个成员变量是 ` RaftNode  *raft.Node`

在Manager的New()函数中(主要分析和raft相关的)

通过调用raft.NewNode()函数新建一个`RaftNode`变量,该变量会塞入`Manager`结构体的`caserver`,`Dispatcher`,`RaftNode`成员变量中, 调用链如下:

```
NewNode() [\swarmkit\manager\state\raft\raft.go]
-> NewMemoryStorage()新建一个存储,里面就是新建一个MemoryStorage结构体,该结构体定义在etcd中,含有Snapshot,Term,Vote,Commit等信息
-> 新建一个raft中的node节点,里面的关键数据如下:
   1. cluster保存集群信息,含有两个map,一个是保存Member,里面还含了grpc.ClientConn,另一个map保存该节点是否removed标记
   2. raftStore就是上面分配的存储
   3. 还有一个memoryStore,这个新建了memDB并建了watch的queue
   4. 设置默认超时时间为2秒
   5. 设置了leadership的广播器Broadcaster
   
```

## run

在Manager run的时候，完成raft相关操作

```
Manager.Run()
-> m.RaftNode.SubscribeLeadership()返回leadershipCh的channel,关系的通知信息就从这来
-> 起一个协程循环leadershipCh
-> 先给manager挂了互斥锁
-> 根据是Leader还是follower状态分别动作,这里略过,详细见swarm集群的文章
-> raftpicker.NewConnSelector()返回一个ConnSelector对象,该对象是到Leader的连接(含GRPC),并且定时更新
-> 初始化了一些raft相关的GRPC Server Service
-> m.RaftNode.JoinAndStart()
   -> n.loadAndStart()
   -> 获取了n.raftStore的snapshot
   -> 如果没有wal文件则创建新的
      1. 如果n.joinAddr不为空,表明不是第一个节点,
         -> n.ConnectToMember() 返回membership的Member,含有grpc的连接
         -> NewRaftMembershipClient()
         -> client.Join() 加入raft cluster
         -> n.createWAL()
         -> raft.StartNode() [函数实现在etcd中]
            -> newRaft()
            -> r.becomeFollower(1, None) 变成follower,term从1开始
            -> 写log
            -> newNode() 建了raft node
            -> 起了一个协程 n.run() [github.com\coreos\etcd\raft\node.go]
               -> 判断leader是否存在,如果没有leader把propc置为nil
               -> 持续监听node管道,【这个是一个很关键的地方！！！】
                  当propc和n.recvc收到信息时,调用r.Step()函数
                  -> 【判断收到消息的Term和本地的Term的大小】
                     如果收到的Term大于自身Term,则自身降级为follower,becomeFollower()
                     如果收到的Term小于自身Term,则忽略
                  -> 调用r.step,这是一个函数指针,当节点变为候选者和跟随者是赋不同函数
         -> n.registerNodes()
      2. 如果n.joinAddr为空,是集群第一个节点
         -> n.createWAL() 创建一个wal文件
         -> raft.StartNode() [函数实现在etcd中]
         -> n.Campaign() [函数实现在etcd中]
            -> 调用step()函数  [\github.com\coreos\etcd\raft\node.go]
-> 起一个协程跑m.RaftNode.Run() [\swarmkit\manager\state\raft\raft.go]
    -> 当n.Ready()这个channel收到消息时,调用了n.send()
       -> 循环入参messages
          -> 如果是本地raft,直接调用n.Step()
          -> 否则起一个协程调用n.sendToMember()
             -> 先判断发送的目的节点是否被removed,如果是则退出
             -> 判断目的节点是否在members的map中,是则直接取出,不是则走下面流程连接
                -> n.ConnectToMember()
                   -> dial() 得到一个gRPC client connection
                   -> 返回一个membership.Member结构体,里面new了一个gRPC client
             -> gRPC ProcessRaftMessage() 调用目的节点 [\swarmkit\api\raft.pb.go]
                通过grpc.Invoke("/docker.swarmkit.v1.Raft/ProcessRaftMessage"),调用了ProcessRaftMessage()
                -> 主要就是调用 node.Step() [\github.com\coreos\etcd\raft\node.go]
             
-> raft.WaitForLeader() 等待集群选出leader,如果ctx错误停止所有GRPC Server
-> raft.WaitForCluster() 等待信息提交保存,否则停止所有GRPC Server
```



`raft`结构体中,定义了一个成员变量 `msgs []pb.Message`, 是一个数组,使用`send()`发消息是,是把所有需要发的消息都塞入该数组中

在swarm的manager中,raft使用了`\vendor\src\github.com\coreos\etcd\raft\raft.go`的代码流程,但是gRPC部分位于swarm代码里面

在Manager的Run()函数中,前面如上面所说起了两个gRPC server,然后下面起了一个协程来运行了
`m.RaftNode.Run()`函数

## 选主

raft选主这块的逻辑主要是swarm使用了etcd插件中的代码来实现的,代码位于`\vendor\src\github.com\coreos\etcd\raft\raft.go`中

当swarm mananger中,调用到`JoinAndStart()`函数时,会调用`raft.StartNode()`和`raft.RestartNode()`函数,里面起了一个协程来 `n.run()`函数,上面部分也有提到

每个节点,先把自身初始化为follower,调用`becomeFollower()`函数

```
becomeFollower()
-> 把stepFollower()赋值到r.step上,后续抽象调用
   根据消息类型做不同处理
-> 重置term这个值
-> tickElection()赋值给r.tick
   -> 如果超时了,调用Step()函数,使用pb.MsgHup消息
      -> 如果是pb.MsgHup消息,则判断自身,如果是leader忽略,不是就用campaign()发起新选举
         -> r.becomeCandidate() 成为候选者
            -> 把stepCandidate()赋值到r.step上,后续抽象调用
            -> tickElection()赋值给r.tick
            -> Term自身加1
         -> 判断得票数是否过半,如果过半则变为becomeLeader()变为leader并退出
         -> 否则发出投票消息 r.send()
      -> 如果收到消息的term大于自身,则调用becomeFollower()变成follower
      -> 如果收到消息的term小于自身,则忽略
      -> 调用r.step这个函数指针
```

### raft通信

swarm中的`Node`结构体中,镶嵌了etcd里面的`raft.Node`,该接口里面,通过一个函数返回了etcd里面的Ready类型的的channel,定义在`\github.com\coreos\etcd\raft\node.go`

```
type Node interface {
    ......
	Ready() <-chan Ready
```

这仅仅是一个接口,具体实现为

```
func (n *node) Ready() <-chan Ready { return n.readyc }
```

该函数的意思是返回一个只读的Ready类型的channel,该值为`node`结构体的`readyc`

意味着,**发送消息时候,swarm判断该`readyc`是否有数据,有则发送**

#### 发送

在`\swarmkit\manager\state\raft\raft.go`里面,`*Node.Run()`函数里面,一直在循环监听

当一个`Ready`结构体类型的通道收到数据之后,开始做raft的发送动作

```
-> 从内存数据库中读取集群信息
-> n.saveToStorage()先把当前信息保存
-> n.send()发送消息到集群其他节点
   send()函数调用栈在上面【run】相关的章节里面有解析
```


#### 接收

etcd中起的协程中,会循环监控`recvc`管道,首先判断一把,如果发消息的节点是自身`prs`里面的节点,而且然后调用`r.Step()`, 然后就会进入`stepFollower()`或者`stepCandidate()`,里面就判断了消息的类型来处理

下面有多处会操作这个管道

1 `ProcessRaftMessage()`函数中,有一个`n.Step`的调用,该函数调用了`etcd`里面的[/etcd/raft/node.go]里面, `node`结构体的`step`函数,这个函数关键点如下:

```
	ch := n.recvc    # 把recvc赋值给ch
	if m.Type == pb.MsgProp {
		ch = n.propc   # 如果是MsgProp消息,把propc赋值给ch
	}

	select {
	case ch <- m:  # 这里就把message塞到了recv或者prop里面了
		return nil
```

2 在swarm中,调用`sendToMember()`里面,调用完ProcessRaftMessage()之后,会判断,然后调用`ReportUnreachable()`或者`ReportSnapshot()`,这两个函数会写数据到node的`recvc`管道中



## gRPC
