分析docker ps命令

`docker ps` 命令可以查看正在运行的容器

```
runPs() [docker-engine\api\client\container\ps.go]
-> ContainerList()  下发restfulAPI调用"/containers/json",映射到下面的接口
-> getContainersJSON [\api\server\router\container\container_routes.go]
-> 构建config
-> Containers() [\daemon\list.go]
-> reduceContainers()
   -> 根据nameIndex,从daemon.containersReplica.Snapshot()读取view
      -> Snapshot()属于memDB的一个方法,提供保证一致性的只读数据,containersReplica是一个ViewDB类型的数据结构,提供了是一个在内存中的ACID事务型的数据结构体
   -> foldFilter() 传入view,生成一个过滤上下文
   -> 生成一个容器的Snapshot的数组
   -> 循环这个数组,还原每个容器,加入到一个容器数组中,返回回去
```
