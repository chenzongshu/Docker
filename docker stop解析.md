
`docker stop` 命令可以停止正在运行的容器

```
runStop() [\docker-engine\api\client\container\stop.go]
-> 循环容器列表(可以同时停止多个容器),调用ContainerStop()
   -> 根据容器名或者容器
   -> 判断容器是否在运行
   -> daemon.containerStop()
      -> 停止健康检查
      -> killPossiblyDeadProcess()
         -> killWithSignal()
            -> container.Lock()锁定容器
            -> 关闭restartManager
            -> daemon.kill()
               -> Signal() [\libcontainerd\client_linux.go]调用gRPC给容器发信号
      -> WaitStop()
         -> 如果出错,则再调用daemon.kill(),时间入参设为-1
```
