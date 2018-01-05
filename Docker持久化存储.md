所谓docker的数据,有2部分,

一部分是docker运行过程中,本来的管理数据,比如说容器的各个layer,镜像信息,网络信息,swarm集群信息等,用于保证当docker Daemon服务重启之后,能够恢复数据.
这部分数据会保存在磁盘文件中和数据库里面

另外一部分是容器内部业务运行的数据,这部分可以通过mount挂载和flocker等软件挂载到远端磁阵上.

这里所说的持久化存储,并不是docker容器内部的业务数据的持久化存储,而且

# Docker数据库

## NoSQL数据库与关系型数据

Docker里面使用的数据库都是key-value数据库,属于NoSQL数据库,和传统的关系型数据库(MySQL,Oracle,SQL Server,DB2等)不一致.

关系型数据库,是建立在关系模型基础上的数据库,结构化存储,完整性约束.
优点是可以保持较好的数据一致性(事务),支持复杂的查询(SQL).
缺点是语句要经过SQL解析,对大数据,高并发性能表现不足.

NoSQL数据库随着互联网的增长而兴起,数据逻辑关系比较松散,优缺点正好和关系型数据库相反.
优点是对大数据,高并发程序性能好,支持分布式,易扩展.
缺点是不支持复杂查询,没有事务的支持.

现在更多是结合来使用,关系型比如mysql来保存不经常变更的管理数据,k-v数据库来保存对性能要求更高的业务数据

就Docker数据库来说,目前主要使用了libkv和go-memdb两种.

## libkv

libkv是一个go语言的k-v数据库,支持分布式特性,可以用来选举,注册服务发现,存储数据等.
目前libkv支持Consul，Etcd，Zookeeper（分布式存储）和BoltDB（本地存储）

Docker 里面有本地和全局驱动的概念。本地驱动（比如 “bridge”）固定于一台机器而不能做跨机器的远程协调。全局驱动（比如 “overlay”）依赖于 libkv 库去做跨机器间的协调。

libnetwork的数据主要就是由libkv保存,并且主机的服务发现也是由libkv完成

本地落盘的主要保存在 `./network/files/local-kv.db`



## boltDB

### 基本知识

Bolt是一个纯粹的Go语言版的嵌入式key/value的数据库，而且在Go的应用中很方便地去用作持久化。

BoltDB支持完全可序列化的ACID事务，也不同于SQLlite，BoltDB没有查询语句，对于用户而言，更加易用。

BoltDB将数据保存在一个单独的内存映射的文件里。它没有wal、线程压缩和垃圾回收；它仅仅安全地处理一个文件。

下面是一些常用操作

```
import  "github.com/boltdb/bolt"

db, err := bolt.Open("test.db", 0600, nil) #打开数据库文件
db.Close()  #关闭数据库文件
```

更新事务

```
err := db.Update(func(tx *bolt.Tx) error {
    ...
    return nil
})

# 通过该接口可以实现数据更新操作
# 该操作会被当做一个事务来处理，如果Update()内的操作返回nil，则事务会被提交，否则事务会回滚
```

只读事务

```
# 通过该接口可以且只能进行数据查询操作
err := db.View(func(tx *bolt.Tx) error {
    ...
    return nil
})
```

批量更新事务

```
err := db.Batch(func(tx *bolt.Tx) error {
    ...
    return nil
})

# 通过该接口可以实现多次数据更新操作
# 所有的更新会被当做一个事务来处理，如果Update()内的操作返回nil，则事务会被提交，否则事务会回滚
```

### docker中使用

- libkv的本地存储使用的就是boltdb
- volume的元数据 `./volumes/metadata.db`
- swarm agent中的task和worker数据 `./swarm/worker/tasks.db`

## go memdb

实现了一个基于不变基数树(radix tree)的内存数据库。该数据库提供了来自ACID的原子性，一致性和隔离性。因为它是在内存中，所以不提供持久存储。特性如下：

- 多版本控制
- 事务级支持
- 丰富的索引(高级复合字段索引)
- 支持watch

go-memdb在docker里面主要应用两个地方

### container

把容器运行时候的状态等一些动态信息保存在memdb中,代码主要在
`\docker\SOURCES\docker-engine\container\view.go`

### swarm

集群有关的信息,由Manager控制的信息,如cluster,node,service,task等,都会保存在memdb中
