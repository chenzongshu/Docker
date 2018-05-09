
# containerd介绍

## 什么是containerd

containerd是从Docker Engine中分离的开源项目,提供一个更加开放、稳定的容器运行基础设施。和原先包含在Docker Engine里containerd相比，独立的containerd将具有更多的功能，可以涵盖整个容器运行时管理的所有需求

containerd并不是直接面向最终用户的，而是主要用于集成到更上层的系统里，比如Swarm, Kubernetes, Mesos等容器编排系统。containerd以Daemon的形式运行在系统上，通过unix domain docket暴露很低层的gRPC API，上层系统可以通过这些API管理机器上的容器。每个containerd只负责一台机器，Pull镜像，对容器的操作（启动、停止等），网络，存储都是由containerd完成。具体运行容器由runC负责，实际上只要是符合OCI规范的容器都可以支持。

对于容器编排服务来说，运行时只需要使用containerd+runC，更加轻量，容易管理。

实际上containerd就是将docker daemon传过来的GRPC消息转化成实际要操作runc要执行的命令。


## 架构

### 架构层次

![containerd架构](http://djanus-test.oss-cn-hangzhou.aliyuncs.com/architecture.png)

中间这一层里包含了三个子系统，从这里可以看出containerd支持哪些能力

- Distribution: 和Docker Registry打交道，拉取镜像
- Bundle: 管理本地磁盘上面镜像的子系统。
- Runtime：创建容器、管理容器的子系统。

### 交互

![](https://github.com/JinhuaWei/docker-notes/blob/master/docker-%E9%80%9A%E4%BF%A1.png)

1、docker daemon 模块通过 grpc 和 containerd模块通信：dockerd 由libcontainerd负责和containerd模块进行交换， dockerd 和 containerd 通信socket文件：docker-containerd.sock
2、containerd 在dockerd 启动时被启动，启动时，启动grpc请求监听。containerd处理grpc请求，根据请求做相应动作；
若是start或是exec 容器，containerd 拉起一个container-shim , 并通过exit 、control 文件（每个容器独有）通信；
3、container-shim别拉起后，start/exec/create拉起runC进程，通过exit、control文件和containerd通信，通过父子进程关系和SIGCHLD监控容器中进程状态；
4、若是top等命令，containerd通过runC二级制组件直接和容器交换；
5、在整个容器生命周期中，containerd通过 epoll 监控容器文件，监控容器的OOM等事件；

## 特性

- 支持OCI镜像
- 支持OCI运行时（runC）
- 支持镜像的pull/push操作
- 容器运行时和生命周期管理
- 网络原语：创建/修改/删除接口
让容器加入已有的Network Namespace
使用“内容可寻址”存储支持全局镜像多租户共享

## containerd和Docker Engine之间的关系

Docker Engine包含containerd，containerd专注于运行时的容器管理，而Docker除了容器管理之外还可以完成镜像构建之类的功能。

containerd提供的API偏底层，不是给普通用户直接用的。对于普通用户来说，可以继续使用Docker。容器编排系统的开发者才需要containerd，比如阿里云容器服务团队。

## containerd，OCI和runC之间的关系

OCI是一个标准化的容器规范，包括运行时规范和镜像规范。runC是基于这个规范的一个参考实现，Docker贡献了runC的主要的代码。

从技术栈上，containerd比runC的层次更高，containerd可以使用runC启动容器，还可以下载镜像，管理网络。

## containerd和容器编排系统的关系

目前像阿里云CS,微软ACS,AWS ECS都是直接使用containerd, 而Kubernetes现在直接使用Docker，将来的版本可以转而使用containerd

# 源码分析

最关键的代码在package supervisor中(supervisor.go , worker.go)，由它负责任务的监控和处理。

而supervisor中最重要的是两个channel和三个协程

- Task
- startTask
- api server协程，主要负责将Task放入Task chan
- supervisor协程，主要将Task从Task chan中取出放入startTask chan
- spuervisor worker协程（十个）主要负责从startTask chan中取出startTask，做相应的操作。

## 基本概念

### epoll

首先,poll是一种监控文件是否可读的机制,与select类似,但是epoll是一种高并发的I/O多路复用技术.

I/O多路复用就是通过一种机制，一个进程可以监视多个描述符，一旦某个描述符就绪（一般是读就绪或者写就绪），能够通知程序进行相应的读写操作。但select，poll，epoll本质上都是同步I/O，因为他们都需要在读写事件就绪后自己负责进行读写，也就是说这个读写过程是阻塞的，而异步I/O则无需自己负责进行读写，异步I/O的实现会负责把数据从内核拷贝到用户空间

### 文件描述符fd

文件描述符（File descriptor）是一个用于表述指向文件的引用的抽象化概念。

文件描述符在形式上是一个非负整数。实际上，它是一个索引值，指向内核为每一个进程所维护的该进程打开文件的记录表。当程序打开一个现有文件或者创建一个新文件时，内核向进程返回一个文件描述符。在程序设计中，一些涉及底层的程序编写往往会围绕着文件描述符展开。但是文件描述符这一概念往往只适用于UNIX、Linux这样的操作系统。

## 组件通信

### daemon -> libcontainerd

LIBCONTAINERD 部分主要作用： 赋值; 和CONTAINERD进程通信; 监控CONTAINERD进程状态

LIBCONTAINERD 随DOCKERD启动，启动过程中启动协程监控：
1、和CONTAINERD间的grpc链接情况 
2、监听由CONTAINERD发送过来的消息 启动监控消息协程

```
//libcontainerd/remote_linux.go
//starEventMonitor() ---> handleEventStream()

func (r *remote) handleEventStream(events containerd.API_EventsClient) {
	for {
		e, err := events.Recv()   // ---> 此为阻塞方法：等待CONTAINERD发送 gRPC消息
...
		if err := container.handleEvent(e); err != nil {
			logrus.Errorf("libcontainerd: error processing state change for %s: %v", e.Id, err)
		}
		...
}
```



## 入口函数

在`docker\containerd\containerd\main.go`

containerd提供一组GRPC接口,位于 `\containerd\api` 中

`Docker`本身封装了一层称为`libcontainerd`,有一个单独的目录,里面通过GRPC对`containerd`进行调用

Docker的`Daemon`结构体中,封装了2个成员变量,分别是`libcontainerd.Client`和`libcontainerd.Remote`


```
mian()
-> daemon()
   -> supervisor.New(): 新建一个容器管理器
   -> supervisor.NewWorker()开了10个worker,worker是容器工作队列,含有Supervisor结构体
   -> 对于每个work,调用work.Start()
      -> 循环work中Supervisor里面的startTasks
         -> 对于每个task,调用runtime.Container的Start()方法启动一个进程process
         -> 调用Supervisor.monitor.MonitorOOM()把容器加入OOM监控列表
            所有的监控都是通过runC的epoll来实现的
         -> 用Monitor()把上面得到的容器进程process加入监控列表
         -> 如果CheckpointPath为空则启动一个新进程, process.start
   -> sv.Start()  开启内部消息处理loop
      -> 监控Supervisor的管道task,调用handleTask()函数
         【handleTask()里面分别对各种事件进行监控,是监控的消息中心!!】
   -> startServer() 启动GRPC服务,
      GRPC启动一个服务端的三要素:
      1. 监听对象listener
         sockets, err := listeners.Init()
         l := sockets[0]
      2. gRPC服务对象server
         s := grpc.NewServer()
      3. 封装的服务函数对象&server{}
         types.RegisterAPIServer(s, server.NewServer(sv))
      最后起了一个goroutine来监听gRPC连接
   -> 通过channel响应收到Ctr+C等中断信息,其他都不反应
```

## containerd-shim

CONTAINERD-SHIM的代码和业务逻辑都很简单；其主要实现目的：

1、通过runC命令可以启动、执行容器、进程；
2、监控容器进程状态，当容器执行完成后，通过exit fifo文件报告容器进程结束状态；
3、当此容器SHIM的第一个实例进程被杀死后，reaper掉所有其子进程；

一个容器可以由多个container-shim进程；container-shim进程由containerd进程拉起，并持续存在到容器实例进程退出为止；

CONTAINERD-SHIM的代码在containerd/container-shim/目录下；主要包含main.go和process.go； 主要启动业务逻辑

```
main()
------>start()
------------>(p *process) create() //主要功能启动runC进程
------>start函数中，监控runC进程状态
```
