
# 通过镜像来搭建

## 1.配置yum源
详细见其他文档，内容略

## 2.通过yum下载docker

```
yum install -y docker
```

#  3.设置docker为开机自启动

```
systemctl enable docker-latest
```

## 4.修改docker配置文件：（如果docker-registry服务已启动，需要重启该服务）

```
vim /etc/sysconfig/docker     #如果使用的是docker-latest，在对应位置修改该配置增加下面这行
OPTIONS='--insecure-registry 10.74.29.152:5000'
```
	修改一下配置，让你的私有仓库支持 http，因为从 docker1.3.2 开始，docker registry 默认都是使用 https 协议而不使用 http，内网能访问的只是 http，所以，就需要解决启用 http 的问题，因为要在内网里搭建，外网无法访问，何必要加密，只会拖慢速度

## 5.下载Registry镜像

## 6.关闭防火墙和selinux 

```
[root@localhost ~]# systemctl stop firewalld.service
[root@localhost ~]# systemctl disable firewalld.service
vim /etc/sysconfig/selinux
SELINUX=disabled
```
## 7.启动docker服务

```
[root@localhost ~]# systemctl start docker-latest
```

## 8.运行docker Registry镜像

```
docker run -d -p 5000:5000 -v /home/my_registry:/tmp/registry 32926a550834  #32926a550834为容器ID
```
如果没有load该镜像，请先load

## 9.确认仓库是否建立
通过网页或者在其他机器上访问，可以看到docker仓库已经建立
在10.74.120.190上，输入：

```
curl "http://10.74.29.152:5000" 
"\"docker-registry server\""
```
 
# 通过rpm包来搭建

## 1.配置yum源

## 2.通过yum下载docker 

```
yum install -y docker
```

## 3.通过yum下载docker-registry

```
yum install -y docker-registry
```

## 4.设置docker和docker-registry为开机自启动

```
systemctl enable docker
systemctl enable docker-registry
```

## 5.关闭防火墙和selinux 

```
[root@localhost ~]# systemctl stop firewalld.service
[root@localhost ~]# systemctl disable firewalld.service
vim /etc/sysconfig/selinux
SELINUX=disabled
```


## 6.在Client设置Registry IP

```
Vim /etc/sysconfig/docker-registry
REGISTRY_ADDRESS=10.74.120.190          #增加这个服务器IP
```


## 7.启动docker和docker-registry服务

```
[root@localhost ~]# systemctl start docker-latest
[root@localhost ~]# systemctl start docker-registry
```


# Registry的使用
在10.74.120.190  Push镜像

## 1.在使用Registry的客户端，需要做如下配置
在/etc/docker/目录下，创建daemon.json文件。在文件中写入：

```
{ "insecure-registries":["10.74.29.152:5000"] }
```


```
vim /etc/sysconfig/docker     #如果使用的是docker-latest，在对应位置修改该配置增加下面这行
OPTIONS='--insecure-registry 10.74.29.152:5000'
```

保存后重启docker

## 2.先load一个镜像

```
[root@desktop-dev-190 docker_images_min]# docker load < ubuntu_latest.tar.gz 
c8305edc6321: Loading layer [==================================================>] 132.5 MB/132.5 MB
5c42a1311e7e: Loading layer [==================================================>] 15.87 kB/15.87 kB
c3c1fd75be70: Loading layer [==================================================>] 9.728 kB/9.728 kB
cc8df16ebbaf: Loading layer [==================================================>] 4.608 kB/4.608 kB
4f9d527ff23f: Loading layer [==================================================>] 3.072 kB/3.072 kB
Loaded image ID: sha256:45bc58500fa3d3c0d67233d4a7798134b46b486af1389ca87000c543f46c3d24 B/3.072 kB
[root@desktop-dev-190 ~]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
<none>              <none>              45bc58500fa3        4 months ago        126.8 MB
```

## 3.打个标签

```
[root@desktop-dev-190 ~]# docker tag 45bc58500fa3 10.74.29.152:5000/ubuntu
[root@desktop-dev-190 ~]# docker images
REPOSITORY                 TAG                 IMAGE ID            CREATED             SIZE
10.74.29.152:5000/ubuntu   latest              45bc58500fa3        4 months ago        126.8 MB
```

## 4.Push镜像

```
docker push 10.74.29.152:5000/ubuntu
```


## 5.在10.74.120.50下载镜像成功


##6.通过http API查询仓库


# 其他问题

镜像存放在容器仓库里的什么地方？
通过Registry容器提供仓库服务：
{F1468}
通过rpm包安装：
/var/lib/docker-registry/repositories

---

# Docker Registry V2搭建及使用

现在docker Registry V2已经更名为docker-distribution

# 1 服务器搭建

## 1.1 安装rpm包

```
yum install docker-distribution 
```
## 1.2 启动服务

```
service docker-distribution start
```

# 2 客户端配置

```
vim /etc/sysconfig/docker     #如果使用的是docker-latest，在对应位置修改该配置增加下面这行
OPTIONS='--insecure-registry 10.74.29.152:5000'
```
打标签之后就可以正常上传了

# 3 Registry V2使用方法
    同V1
