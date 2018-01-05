# 命令入口

```
\SOURCES\docker-engine\cli\cobraadaptor\adaptor.go
```

# Docker Daemon

Daemon为Docker守护进程，提供Server功能，负责处理Docker相关请求

## 程序入口

~/docker/SOURCES/docker-engine/cmd/dockerd/docker.go
cmd下面有2个文件夹,docker是Client入口,dockerd是Daemon入口

调用链:

```
-> main()  #入口
-> daemonCli.start() [daemon.go]
   1. apiserver.New() 启动一个apiserver监听端口,
   2. new了service和libcontainer,然后调用          
   3. daemon.NewDaemon() 新建Daemon
   4. initRouter() 新建请求路由,路由来选择docker命令的执行
-> NewDaemon()  新建一个Daemon,各种初始化
-> d.restore():
   1. 初始化了本地的docker容器;
   2. daemon.initNetworkController() 初始化网络控制器
      -> initBridgeDriver()
      -> NewNetwork()
      -> addNetwork()
      -> CreateNetwork()
      -> createNetwork()
      -> setupIPTables()
   3. 初始化挂载点
-> 
```

Daemon中保存了所有容器的列表,在新建Daemon的时候内存中读取
`d.containers = container.NewMemoryStore()`

Docker给每一个容器保存了配置信息,位于
`/var/lib/docker/containers/$ID/config.v2.json`

## 请求路由

通过包gorilla/mux，创建了一个mux.Router，提供请求的路由功能。在Golang中，gorilla/mux是一个强大的URL路由器以及调度分发器，该mux.Router中添加了众多的路由项，每一个路由项由HTTP请求方法（PUT、POST、GET或DELETE）、URL、Handler三部分组成

`docker-engine\api\server\router\container\container.go`

```
-> initRouter()  
   1. 里面对container,volume等分别NewRouter()
   2. s.InitRouter()  -> s.createMux() 创建路由分发器
-> NewRouter()   #每个类型都有自己的NewRouter函数,下同
-> r.initRoutes()  # 里面把对应restful转换为操作执行
