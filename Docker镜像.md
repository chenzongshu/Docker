
# 镜像原理

Linux 操作系统由内核空间和用户空间组成, 如下:

```
+------------------+
| rootfs(/dev,/bin,|
| /proc,/etc.....) |
+------------------+----------+
|                             |
| bootfs(kernel)              |
|                             |
|                             |
+-----------------------------+
```

**rootfs:**

内核空间是 kernel，Linux 刚启动时会加载 bootfs 文件系统，之后 bootfs 会被卸载掉。

用户空间的文件系统是 rootfs，包含我们熟悉的 /dev, /proc, /bin 等目录。

对于 base 镜像来说，底层直接用 Host 的 kernel，自己只需要提供 rootfs 就行了。

而对于一个精简的 OS，rootfs 可以很小，只需要包括最基本的命令、工具和程序库就可以了。相比其他 Linux 发行版，CentOS 的 rootfs 已经算臃肿的了，alpine 还不到 10MB。

我们平时安装的 CentOS 除了 rootfs 还会选装很多软件、服务、图形桌面等，需要好几个 GB 就不足为奇了。

不同 Linux 发行版的区别主要就是 rootfs。

比如 Ubuntu 14.04 使用 upstart 管理服务，apt 管理软件包；而 CentOS 7 使用 systemd 和 yum。这些都是用户空间上的区别，Linux kernel 差别不大。

所以 Docker 可以同时支持多种 Linux 镜像，模拟出多种操作系统环境

注意, **base 镜像只是在用户空间与发行版一致，kernel 版本与发行版是不同的**

