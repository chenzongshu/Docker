
编辑/etc/rsyslog.conf，添加如下内容

```
:programname,isequal,"dockerd"                                 /var/log/dockerd
:programname,isequal,"dockerd"                                  ~
```

> 说明：
1、/var/log/dockerd是docker日志信息将要写入的文件名，此文件名可按照实际需求进行设置，但必须写绝对路径。
2、添加内容最好放在RULES之前，否则docker的日志信息会重复写入到文件/var/log/messages中。

重启rsyslog和docker

```
systemctl restart rsyslog
systemctl restart docker
```
