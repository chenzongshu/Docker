
# 进程

在unix/linux系统中，正常情况下，子进程是通过父进程fork创建的。子进程的结束和父进程的运行是一个异步过程,即父进程永远无法预测子进程到底什么时候结束。 当一个进程完成它的工作终止之后，它的父进程需要调用wait()或者waitpid()系统调用取得子进程的终止状态。

## 僵尸进程

子进程退出，而父进程并没有调用wait或waitpid获取子进程的状态信息，那么子进程的进程描述符仍然保存在系统中，这种进程称之为僵尸进程

> 在每个进程退出的时候,内核释放该进程所有的资源,包括打开的文件,占用的内存等。 但是仍然为其保留一定的信息(包括进程号、退出状态、运行时间等)。直到父进程通过wait / waitpid来取时才释放。 如果父进程不调用wait / waitpid的话， 那么保留的那段信息就不会释放，其进程号就会一直被占用，系统所能使用的进程号是有限的，如果大量的产生僵尸进程，可能导致系统不能产生新的进程.

## 孤儿进程

父进程先于子进程退出，那么子进程将成为孤儿进程。孤儿进程将被init进程(进程号为1)接管，并由init进程对它完成状态收集(wait/waitpid)工作。



# Docker进程树

![](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/5e0ea99909d34a2df356ba6c6c426f8c)

- containerd主要职责是镜像管理（镜像、元信息等）、容器执行（调用最终运行时组件执行）
> containerd向上为Docker Daemon提供了gRPC接口，使得Docker Daemon屏蔽下面的结构变化，确保原有接口向下兼容。向下通过containerd-shim结合runC，使得引擎可以独立升级，避免之前Docker Daemon升级会导致所有容器不可用的问题。

- containerd-shim是容器应用在主机上的父进程
- runC是从Docker的libcontainer中迁移而来的，实现了容器启停、资源隔离等功能。Docker默认提供了docker-runc实现

dockerd拿到image后，就会在本地创建相应的容器，然后再做一些初始化工作后，最后通过grpc的方式通知docker-containerd进程启动指定的容器

当docker-containerd收到dockerd的启动容器请求之后，会做一些初始化工作，然后启动docker-containerd-shim进程，并将相关配置所在的目录作为参数传给它。

可以简单的理解成docker-containerd管理所有本机正在运行的容器，而docker-containerd-shim只负责管理一个运行的容器，相当于是对runc的一个包装，充当containerd和runc之间的桥梁，runc能干的就交给runc来做，runc做不了的就放到这里来做。

docker-containerd-shim进程启动后，就会按照runtime的标准准备好相关运行时环境，然后启动docker-runc进程。

runc进程打开容器的配置文件，找到rootfs的位置，并启动配置文件中指定的相应进程，在hello-world的这个例子中，runc会启动容器中的hello程序。


```
docker run -it --net=host --memory 1G bda69f7e8bef /bin/bash

[root@controller home]# ps xf -o pid,ppid,stat,args
   PID   PPID STAT COMMAND
     1      0 Ss   /bin/bash
  1831      1 R+   ps xf -o pid,ppid,stat,args
```

运行了一个容器,在容器里面可以看到,启动时候运行的程序作为了容器的父进程,PID为1

- **容器里面的孤儿进程被docker-containerd-shim进程接收**
- **杀死容器里面的子进程,容器不会退出,如果强行杀死PID为1的进程,容器会退出**



# containerd-shim

```
main()
-> 在/run/docker/libcontainerd/containerd/{容器ID}/init/shim-log.json创建一个文件
-> start()
   -> 创建信号管道signals
   -> 设置child_subreaper(设置shim接收以他为父节点的进程树下所有的孤儿进程)
   -> 打开exit和control的pipe
   -> newProcess()
      -> 去process.json读取process信息
      -> openIO() 
   -> process.create()
   -> 创建管道msgC来接收控制信息
   -> 监控管道signals的SIGCHLD信号
      如果收到SIGCHLD信号,判断该进程是否存在,存在则释放进程资源
   -> 监控管道msgC,关闭标准输入
```
