
# 介绍

Namespace解决的问题主要是环境隔离的问题，这只是虚拟化中最最基础的一步，我们还需要解决对计算机资源使用上的隔离。

也就是说，虽然你通过Namespace把我Jail到一个特定的环境中去了，但是我在其中的进程使用用CPU、内存、磁盘等这些计算资源其实还是可以随心所欲的。所以，我们希望对进程进行资源利用上的限制或控制。这就是Linux CGroup出来了的原因。

Linux CGroup全称Linux Control Group， 是Linux内核的一个功能，用来限制，控制与分离一个进程组群的资源（如CPU、内存、磁盘输入输出等）。

Linux CGroup 可让用户为系统中所运行任务(进程)的用户定义组群分配资源,比如CPU 时间、系统内存、网络带宽或者这些资源的组合。您可以监控您配置的 cgroup，拒绝cgroup 访问某些资源，甚至在运行的系统中动态配置您的cgroup。


主要提供了如下功能：

- Resource limitation: 限制资源使用，比如内存使用上限以及文件系统的缓存限制。
- Prioritization: 优先级控制，比如：CPU利用和磁盘IO吞吐。
- Accounting: 一些审计或一些统计，主要目的是为了计费。
- Control: 挂起进程，恢复执行进程。

在实践中，系统管理员一般会利用CGroup做下面这些事（有点像为某个虚拟机分配资源似的）：

- 隔离一个进程集合（比如：nginx的所有进程），并限制他们所消费的资源，比如绑定CPU的核。
- 为这组进程 分配其足够使用的内存
- 为这组进程分配相应的网络带宽和磁盘存储限制
- 限制访问某些设备（通过设置设备的白名单）

那么CGroup是怎么干的呢？

# Cgroup核心概念

cgroups 是用来对进程进行资源管理的，因此 cgroup 需要考虑如何抽象这两种概念：进程和资源，同时如何组织自己的结构。cgroups 中有几个非常重要的概念：

- **task**：任务，对应于系统中运行的一个实体，一般是指进程
- **subsystem**：子系统，具体的资源控制器（resource class 或者 resource controller），控制某个特定的资源使用。比如 CPU 子系统可以控制 CPU 时间，memory 子系统可以控制内存使用量
- **cgroup**：控制组，一组任务和子系统的关联关系，表示对这些任务进行怎样的资源管理策略
- **hierarchy**：层级树，一系列 cgroup 组成的树形结构。每个节点都是一个 cgroup，cgroup 可以有多个子节点，子节点默认会继承父节点的属性。系统中可以有多个 hierarchy

虽然 cgroups 支持 hierarchy，允许不同的子资源挂到不同的目录，但是多个树之间有各种限制，增加了理解和维护的复杂性。在实际使用中，所有的子资源都会统一放到某个路径下（比如 ubuntu16.04 的 /sys/fs/cgroup/）

# 子资源系统（Resource Classes or SubSystem）

目前有下面这些资源子系统：

- Block IO（blkio)：限制块设备（磁盘、SSD、USB 等）的 IO 速率
- CPU Set(cpuset)：限制任务能运行在哪些 CPU 核上
- CPU Accounting(cpuacct)：生成 cgroup 中任务使用 CPU 的报告
- CPU (CPU)：限制调度器分配的 CPU 时间
- Devices (devices)：允许或者拒绝 cgroup 中任务对设备的访问
- Freezer (freezer)：挂起或者重启 cgroup 中的任务
- Memory (memory)：限制 cgroup 中任务使用内存的量，并生成任务当前内存的使用情况报告
- Network Classifier(net_cls)：为 cgroup 中的报文设置上特定的 classid 标志，这样 tc 等工具就能根据标记对网络进行配置
- Network Priority (net_prio)：对每个网络接口设置报文的优先级
- perf_event：识别任务的 cgroup 成员，可以用来做性能分析

# 使用 cgroups

cgroup 内核功能比较有趣的地方是它没有提供任何的系统调用接口，而是对 linux vfs 的一个实现，因此可以用类似文件系统的方式进行操作。

使用 cgroups 的方式有几种：

- 使用 cgroups 提供的虚拟文件系统，直接通过创建、读写和删除目录、文件来控制 cgroups
- 使用命令行工具，比如 libcgroup 包提供的 cgcreate、cgexec、cgclassify 命令
- 使用 rules engine daemon 提供的配置文件
- 当然，systemd、lxc、docker 这些封装了 cgroups 的软件也能让你通过它们定义的接口控制 cgroups 的内容

## 直接操作 cgroup 文件系统

首先，Linux把CGroup这个事实现成了一个file system，你可以mount

```
hchen@ubuntu:~$ mount -t cgroup
cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,relatime,cpuset)
cgroup on /sys/fs/cgroup/cpu type cgroup (rw,relatime,cpu)
cgroup on /sys/fs/cgroup/cpuacct type cgroup (rw,relatime,cpuacct)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,relatime,memory)
cgroup on /sys/fs/cgroup/devices type cgroup (rw,relatime,devices)
cgroup on /sys/fs/cgroup/freezer type cgroup (rw,relatime,freezer)
cgroup on /sys/fs/cgroup/blkio type cgroup (rw,relatime,blkio)
cgroup on /sys/fs/cgroup/net_prio type cgroup (rw,net_prio)
cgroup on /sys/fs/cgroup/net_cls type cgroup (rw,net_cls)
cgroup on /sys/fs/cgroup/perf_event type cgroup (rw,relatime,perf_event)
cgroup on /sys/fs/cgroup/hugetlb type cgroup (rw,relatime,hugetlb)
```

如果没有的话，也可以通过以下命令来把想要的 subsystem mount 到系统中：
``` $ mount -t cgroup -o cpu,cpuset,memory cpu_and_mem /cgroup/cpu_and_mem ```

上述命令表示把 cpu、cpuset、memory 三个子资源 mount 到 /cgroup/cpu_and_mem 目录下。


可以看到，在/sys/fs下有一个cgroup的目录，这个目录下还有很多子目录，比如： cpu，cpuset，memory，blkio……这些，这些都是cgroup的子系统。分别用于干不同的事的。

里面会有很多文件

```
hchen@ubuntu:~$ ls /sys/fs/cgroup/cpu /sys/fs/cgroup/cpuset/
/sys/fs/cgroup/cpu:
cgroup.clone_children  cgroup.sane_behavior  cpu.shares         release_agent
cgroup.event_control   cpu.cfs_period_us     cpu.stat           tasks
cgroup.procs           cpu.cfs_quota_us      notify_on_release  user
```

可以到/sys/fs/cgroup的各个子目录下去make个dir，你会发现，一旦你创建了一个子目录，这个子目录里又有很多文件了。

```
hchen@ubuntu:/sys/fs/cgroup/cpu$ sudo mkdir haoel
[sudo] password for hchen: 
hchen@ubuntu:/sys/fs/cgroup/cpu$ ls ./haoel
cgroup.clone_children  cgroup.procs       cpu.cfs_quota_us  cpu.stat           tasks
cgroup.event_control   cpu.cfs_period_us  cpu.shares        notify_on_release
```

## 简单使用

### CPU限制

假设，我们有一个非常吃CPU的程序，叫deadloop(死循环,一般占100%)

之前,不是在/sys/fs/cgroup/cpu下创建了一个haoel的group。我们先设置一下这个group的cpu利用的限制：

```
hchen@ubuntu:~# cat /sys/fs/cgroup/cpu/haoel/cpu.cfs_quota_us 
-1
root@ubuntu:~# echo 20000 > /sys/fs/cgroup/cpu/haoel/cpu.cfs_quota_us
```

deadloop进程的PID是3529，我们把这个进程加到这个cgroup中：

```
echo 3529 >> /sys/fs/cgroup/cpu/haoel/tasks
```

然后，就会在top中看到CPU的利用立马下降成20%了。（前面我们设置的20000就是20%的意思）


### 内存使用限制

写一个死循环,不断分配内存的例子(略)

然后限制其内存

```
# 创建memory cgroup
$ mkdir /sys/fs/cgroup/memory/haoel
$ echo 64k > /sys/fs/cgroup/memory/haoel/memory.limit_in_bytes
 
# 把上面的进程的pid加入这个cgroup
$ echo [pid] > /sys/fs/cgroup/memory/haoel/tasks
```

你会看到，一会上面的进程就会因为内存问题被kill掉了。

### 磁盘I/O限制

我们先看一下我们的硬盘IO，我们的模拟命令如下：（从/dev/sda1上读入数据，输出到/dev/null上）

```
sudo dd if=/dev/sda1 of=/dev/null
```

我们通过iotop命令我们可以看到相关的IO速度是55MB/s（虚拟机内）：

```
TID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN     IO>    COMMAND          
8128 be/4 root       55.74 M/s    0.00 B/s  0.00 % 85.65 % dd if=/de~=/dev/null...
```

然后，我们先创建一个blkio（块设备IO）的cgroup

```
mkdir /sys/fs/cgroup/blkio/haoel
```

并把读IO限制到1MB/s，并把前面那个dd命令的pid放进去（注：8:0 是设备号，你可以通过ls -l /dev/sda1获得）：

```
root@ubuntu:~# echo '8:0 1048576'  > /sys/fs/cgroup/blkio/haoel/blkio.throttle.read_bps_device 
root@ubuntu:~# echo 8128 > /sys/fs/cgroup/blkio/haoel/tasks
```

再用iotop命令，你马上就能看到读速度被限制到了1MB/s左右。

```
TID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN     IO>    COMMAND          
8128 be/4 root      973.20 K/s    0.00 B/s  0.00 % 94.41 % dd if=/de~=/dev/null...
```

# cgroup-tools

cgroup-tools 软件包提供了一系列命令可以操作和管理 cgroup

## 列出 cgroup mount 信息

最简单的，`lssubsys` 可以查看系统中存在的 subsystems：

```
lssubsys -am
cpuset /sys/fs/cgroup/cpuset
cpu,cpuacct /sys/fs/cgroup/cpu,cpuacct
blkio /sys/fs/cgroup/blkio
memory /sys/fs/cgroup/memory
devices /sys/fs/cgroup/devices
freezer /sys/fs/cgroup/freezer
net_cls,net_prio /sys/fs/cgroup/net_cls,net_prio
perf_event /sys/fs/cgroup/perf_event
hugetlb /sys/fs/cgroup/hugetlb
pids /sys/fs/cgroup/pids
```

## 创建 cgroup

`cgcreate` 可以用来为用户创建指定的 cgroups：

```
➜  sudo cgcreate -a cizixs -t cizixs -g cpu,memory:test1 
➜  ls cpu/test1 
cgroup.clone_children  cpuacct.stat   cpuacct.usage_all     cpuacct.usage_percpu_sys   cpuacct.usage_sys   cpu.cfs_period_us  cpu.shares  notify_on_release
cgroup.procs           cpuacct.usage  cpuacct.usage_percpu  cpuacct.usage_percpu_user  cpuacct.usage_user  cpu.cfs_quota_us   cpu.stat    tasks
```

上面的命令表示在 /sys/fs/cgroup/cpu 和 /sys/fs/cgroup/memory 目录下面分别创建 test1 目录，也就是为 cpu 和 memory 子资源创建对应的 cgroup

> - 选项 -t 指定 tasks 文件的用户和组，也就是指定哪些人可以把任务添加到 cgroup 中，默认是从父 cgroup 继承
- -a 指定除了 tasks 之外所有文件（资源控制文件）的用户和组，也就是哪些人可以管理资源参数
- -g 指定要添加的 cgroup，冒号前是逗号分割的子资源类型，冒号后面是 cgroup 的路径（这个路径会添加到对应资源 mount 到的目录后面）。也就是说在特定目录下面添加指定的子资源

## 删除 cgroup

`cgdelete` 可以删除对应的 cgroups，它和 cgcreate 命令类似，可以用 `-g` 指定要删除的 cgroup

```cgroup sudo cgdelete -g cpu,memory:test1```

cgdelete 也提供了 `-r` 参数可以递归地删除某个 cgroup 以及它所有的子 cgroup。

如果被删除的 cgroup 中有任务，这些任务会自动移到父 cgroup 中。

## 设置 cgroup 的参数

`cgset` 命令可以设置某个子资源的参数，比如如果要限制某个 cgroup 中任务能使用的 CPU 核数：

```cgset -r cpuset.cpus=0-1 /mycgroup```

`-r` 后面跟着参数的键值对

`cgset` 还能够把一个 cgroup 的参数拷贝到另外一个 cgroup 中：

``` cgset --copy-from group1/ group2/ ```

**cgset 如果设置没有成功也不会报错，请一定要注意。**

## 在某个 cgroup 中运行进程

`cgexec` 执行某个程序，并把程序添加到对应的 cgroups 中：

``` cgroup cgexec -g memory,cpu:cizixs bash ```


## 把已经运行的进程移动到某个 cgroup

要把某个已经存在的程序（能够知道它的 pid）移到某个 cgroup，可以使用 `cgclassify` 命令:

比如把当前 bash shell 移入到特定的 cgroup 中

``` $ cgclassify -g memory,cpu:/mycgroup $$ ```

$$ 表示当前进程的 pid 号，上面命令可以方便地测试一些耗费内存或者 CPU 的进程，如果 /mycgroup 对 CPU 和 memory 做了限制。

# cgroup 子资源参数详解
