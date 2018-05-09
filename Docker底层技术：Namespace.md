
# Namespace简介

**Linux Namespace是Linux提供的一种内核级别资源隔离的方法**

## Namespace API

Linux内核中就提供了六种namespace隔离的系统调用

|Namespace | 系统调用参数 | 隔离内容 |
|:--|:--|:--|
|UTS|CLONE_NEWUTS|主机名与域名|
|IPC|CLONE_NEWIPC|信号量、消息队列和共享内存|
|PID|CLONE_NEWPID|进程编号|
|Network|CLONE_NEWNET|网络设备、网络栈、端口等等|
|Mount|CLONE_NEWNS|挂载点（文件系统）|
|User|CLONE_NEWUSER| 用户和用户组|

主要是三个系统调用:

- `clone()` – 实现线程的系统调用，用来创建一个新的进程，并可以通过设计上述参数达到隔离。
- `unshare()` – 使某进程脱离某个namespace
- `setns()` – 把某进程加入到某个namespace
- 还有`/proc`下的部分文件

## 查看Namespace

3.8版本内核之后, 可以在/proc/[pid]/ns文件下看到指向不同namespace号的文件，效果如下所示，形如[4026531839]者即为namespace号

```
$ ls -l /proc/$$/ns         <<-- $$ 表示应用的PID
total 0
lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 ipc -> ipc:[4026531839]
lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 mnt -> mnt:[4026531840]
lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 net -> net:[4026531956]
lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 pid -> pid:[4026531836]
lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 user->user:[4026531837]
lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 uts -> uts:[4026531838]
```

如果两个进程指向的namespace编号相同，就说明他们在同一个namespace下，否则则在不同namespace里面

## 通过clone()创建新进程的同时创建namespace

```
int clone(int (*child_func)(void *), void *child_stack, int flags, void *arg);
```

- 参数child_func传入子进程运行的程序主函数。
- 参数child_stack传入子进程使用的栈空间
- 参数flags表示使用哪些CLONE_*标志位
- 参数args则可用于传入用户参数

```
int container_pid = clone(container_main, container_stack+STACK_SIZE, 
            CLONE_NEWUTS | SIGCHLD, NULL); /*启用CLONE_NEWUTS Namespace隔离 */
```

```
#define _GNU_SOURCE
#include <sys/types.h>
#include <sys/wait.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>

#define STACK_SIZE (1024 * 1024)

static char child_stack[STACK_SIZE];
char* const child_args[] = {
  "/bin/bash",
  NULL
};

int child_main(void* args) {
  printf("在子进程中!\n");
  execv(child_args[0], child_args);
  return 1;
}

int main() {
  printf("程序开始: \n");
  int child_pid = clone(child_main, child_stack + STACK_SIZE, SIGCHLD, NULL);
  waitpid(child_pid, NULL, 0);
  printf("已退出\n");
  return 0;
}
```

编译并执行,可以看到,已经切换到子进程,需要退出才切换出来

```
root@local:~# gcc -Wall uts.c -o uts.o && ./uts.o
程序开始:
在子进程中!
root@local:~# exit
exit
已退出
root@local:~#
```


## 通过setns()加入一个已经存在的namespace

namespace是进程创建的,是不是意味着进程结束之后,Namespace关闭了呢?

只要打开的文件描述符（fd）存在，那么就算PID所属的所有进程都已经结束，创建的namespace就会一直存在。那如何打开文件描述符呢？把/proc/[pid]/ns目录挂载起来就可以达到这个效果，命令如下.

```
# touch ~/uts
# mount --bind /proc/27514/ns/uts ~/uts
```

通过setns()系统调用，可以把进程从原先的namespace加入准备好的新namespace

```
int setns(int fd, int nstype);
```

- 参数fd表示我们要加入的namespace的文件描述符。上文已经提到，它是一个指向/proc/[pid]/ns目录的文件描述符，可以通过直接打开该目录下的链接或者打开一个挂载了该目录下链接的文件得到。
- 参数nstype让调用者可以去检查fd指向的namespace类型是否符合我们实际的要求。如果填0表示不检查。

为了把我们创建的namespace利用起来，我们需要引入execve()系列函数，这个函数可以执行用户命令，最常用的就是调用/bin/bash并接受参数，运行起一个shell，用法如下。

```
fd = open(argv[1], O_RDONLY);   /* 获取namespace文件描述符 */
setns(fd, 0);                   /* 加入新的namespace */
execvp(argv[2], &argv[2]);      /* 执行程序 */
```

假设编译后的程序名称为setns。

```
# ./setns ~/uts /bin/bash   # ~/uts 是绑定的/proc/27514/ns/uts
```

至此，你就可以在新的命名空间中执行shell命令了

## 通过unshare()在原先进程上进行namespace隔离

unshare()跟clone()很像，不同的是，unshare()运行在原先的进程上，不需要启动一个新进程

```
int unshare(int flags);
```

调用unshare()的主要作用就是不启动一个新进程就可以起到隔离的效果，相当于跳出原先的namespace进行操作。这样，你就可以在原进程进行一些需要隔离的操作

## UTS Namespace

对上面clone的代码进行改造,试验就可以指定hostname已经变成子进程的

```
#define _GNU_SOURCE
#include <sys/types.h>
#include <sys/wait.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>

#define STACK_SIZE (1024 * 1024)

static char child_stack[STACK_SIZE];
char* const child_args[] = {
  "/bin/bash",
  NULL
};

int child_main(void* args) {
  printf("在子进程中!\n");
  sethostname("Changed Namespace", 12);
  execv(child_args[0], child_args);
  return 1;
}

int main() {
  printf("程序开始: \n");
  // int child_pid = clone(child_main, child_stack + STACK_SIZE, SIGCHLD, NULL);
  int child_pid = clone(child_main, child_stack+STACK_SIZE, CLONE_NEWUTS | SIGCHLD, NULL);
  waitpid(child_pid, NULL, 0);
  printf("已退出\n");
  return 0;
}
```

当exit之后,你会发现主机名已经更换回来

如果不加CLONE_NEWUTS参数进行隔离而使用sethostname已经把宿主机的主机名改掉了

## IPC Namespace

IPC全称 Inter-Process Communication，是Unix/Linux下进程间通信的一种方式，IPC有共享内存、信号量、消息队列等方法

容器内部进程间通信对宿主机来说，实际上是具有相同PID namespace中的进程间通信， 因此需要一个唯一的标识符来进行区别。申请IPC资源就申请了这样一个全局唯一的32位ID，所以IPC namespace中实际上包含了系统IPC标识符以及实现POSIX消息队列的文件系统。在同一个IPC namespace下的进程彼此可见，而与其他的IPC namespace下的进程则互相不可见。

目前使用IPC namespace机制的系统不多，其中比较有名的有PostgreSQL。Docker本身通过socket或tcp进行通信。

## PID namespace

PID namespace隔离非常实用，它对进程PID重新标号，即两个不同namespace下的进程可以有同一个PID。每个PID namespace都有自己的计数程序。

内核为所有的PID namespace维护了一个树状结构，最顶层的是系统初始时创建的，我们称之为root namespace。 他创建的新PID namespace就称之为child namespace （树的子节点），而原先的PID namespace 就是新创建的 PID namespace的parent namespace （树的父节点）。通过这种方式，不同的PID namespaces  会形成一个等级体系。 所属的父节点可以看到子节点中的进程，并可以通过信号等方式对子节点中的进程产生影响。反过来，子节点不能看到父节点PID namespace中的任何内容

- 每个PID namespace中的第一个进程“PID 1“， 都会像传统Linux中的init进程一样拥有特权， 起特殊作用。
- 一个namespace中的进程，不可能通过kill或ptrace影响父节点或者兄弟节点中的进程，因为其他节点的PID在这个namespace中没有任何意义。
- 如果你在新的PID namespace中重新挂载/proc文件系统，会发现其下只显示同属一个PID namespace中的其他进程。
- 在root namespace中可以看到所有的进程，并且递归包含所有子节点中的进程。

改造下上面的程序

```
//[...]
int child_pid = clone(child_main, child_stack+STACK_SIZE,
           CLONE_NEWPID | CLONE_NEWIPC | CLONE_NEWUTS 
           | SIGCHLD, NULL);
//[...]
```

编译运行,可以看到

```
root@local:~# gcc -Wall pid.c -o pid.o && ./pid.o
程序开始:
在子进程中!
root@NewNamespace:~# echo $$
1                      <<--注意此处看到shell的PID变成了1
root@NewNamespace:~# exit
exit
已退出
```

打印$$可以看到shell的PID，退出后如果再次执行可以看到效果如下。

```
root@local:~# echo $$
17542
```

子进程的shell中执行ps aux/top之类的命令,发现还是可以看到所有父进程的PID,那是因为我们还没有对文件系统进行隔离,ps/top之类的命令调用的是真实系统下的/proc文件内容，看到的自然是所有的进程。

如果需要解决这个问题,需要重新挂载/proc,但是需要mount Namespace配合

注意,**创建其他namespace时unshare()和setns()会直接进入新的namespace而唯独PID namespace不是**,

因为调用getpid()函数得到的PID是根据调用者所在的PID namespace而决定返回哪个PID， 进入新的PID namespace会导致PID产生变化。而对用户态的程序和库函数来说，他们都认为进程的PID是一个常量，PID的变化会引起这些进程奔溃。

换句话说，一旦程序进程创建以后，那么它的PID namespace的关系就确定下来了， 进程不会变更他们对应的PID namespace。 

## Mount namespaces

Mount namespace通过隔离文件系统挂载点对隔离文件系统提供支持，它是历史上第一个Linux namespace，所以它的标识位比较特殊，就是CLONE_NEWNS。

隔离后，不同mount namespace中的文件结构发生变化也互不影响。你可以通过`/proc/[pid]/mounts`查看到所有挂载在当前namespace中的文件系统，还可以通过`/proc/[pid]/mountstats`看到mount namespace中文件设备的统计信息，包括挂载文件的名字、文件系统类型、挂载位置等等

进程在创建mount namespace时，会把当前的文件结构复制给新的namespace。新namespace中的所有mount操作都只影响自身的文件系统，而对外界不会产生任何影响

挂载传播定义了挂载对象（mount object）之间的关系， 系统用这些关系决定任何挂载对象中的挂载事件如何传播到其他挂载对象

- 共享关系（share relationship）。如果两个挂载对象具有共享关系，那么一个挂载对象中的挂载事件会传播到另一个挂载对象，反之亦然。
- 从属关系（slave relationship）。如果两个挂载对象形成从属关系，那么一个挂载对象中的挂载事件会传播到另一个挂载对象，但是反过来不行；在这种关系中，从属对象是事件的接收者。

一个挂载状态可能为如下的其中一种：

- 共享挂载（shared）
- 从属挂载（slave）
- 共享/从属挂载（shared and slave）
- 私有挂载（private）
- 不可绑定挂载（unbindable）

> 传播事件的挂载对象称为共享挂载（shared mount）；接收传播事件的挂载对象称为从属挂载（slave mount）。既不传播也不接收传播事件的挂载对象称为私有挂载（private mount）。另一种特殊的挂载对象称为不可绑定的挂载（unbindable mount），它们与私有挂载相似，但是不允许执行绑定挂载，即创建mount namespace时这块文件对象不可被复制。

## Network namespace

Network namespace主要提供了关于网络资源的隔离，包括网络设备、IPv4和IPv6协议栈、IP路由表、防火墙、/proc/net目录、/sys/class/net目录、端口（socket）等等。

一个物理的网络设备最多存在在一个network namespace中，你可以通过创建veth pair（虚拟网络设备对：有两端，类似管道，如果数据从一端传入另一端也能接收到，反之亦然）在不同的network namespace间创建通道，以此达到通信的目的。

一般情况下，物理网络设备都分配在最初的root namespace(表示系统默认的namespace)

## User namespaces

User namespace主要隔离了安全相关的标识符（identifiers）和属性（attributes），包括用户ID、用户组ID、root目录、key（指密钥）以及特殊权限。说得通俗一点，一个普通用户的进程通过clone()创建的新进程在新user namespace中可以拥有不同的用户和用户组。
