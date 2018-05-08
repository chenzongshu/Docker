
golang语言自带性能调优工具，pprof，可以方便的监控内存、cpu等使用情况

# 两种收集方式

做 Profiling 第一步就是怎么获取应用程序的运行情况数据。go 语言提供了**`runtime/pprof`** 和 **`net/http/pprof`** 两个库，这部分我们讲讲它们的用法以及使用场景。

## 工具型应用

如果你的应用是一次性的，运行一段时间就结束。那么最好的办法，就是在应用退出的时候把 profiling 的报告保存到文件中，进行分析。对于这种情况，可以使用 **`runtime/pprof`** 库

pprof 封装了很好的接口供我们使用，比如要想进行 CPU Profiling，可以调用 pprof.StartCPUProfile() 方法，它会对当前应用程序进行 CPU profiling，并写入到提供的参数中（w io.Writer），要停止调用 `StopCPUProfile()` 即可。

去除错误处理只需要三行内容，一般把部分内容写在 main.go 文件中，应用程序启动之后就开始执行：

```
// 定义 flag cpuprofile
var cpuprofile = flag.String("cpuprofile", "", "write cpu profile to file")
func main() {
	flag.Parse()
	// 如果命令行设置了 cpuprofile
	if *cpuprofile != "" {
		// 根据命令行指定文件名创建 profile 文件
		f, err := os.Create(*cpuprofile)
		if err != nil {
			log.Fatal(err)
		}
		// 开启 CPU profiling
		pprof.StartCPUProfile(f)
		defer pprof.StopCPUProfile()
	}
	...
```

应用执行结束后，就会生成一个文件，保存了我们的 CPU profiling 数据。假定我们编写的一个程序 mytest 中加入了上述代码则可以执行并生成 profile 文件：

```
./mytest -cpuprofile=mytest.prof
```

这里，我们生成了 mytest.prof profile 文件。有了 profile 文件就可以使用 go tool pprof 程序来解析此文件

```
go tool pprof mytest mytest.prof
```

想要获得内存的数据，直接使用 WriteHeapProfile 就行，不用 start 和 stop 这两个步骤了：

```
f, err := os.Create(*memprofile)
pprof.WriteHeapProfile(f)
f.Close()
```

## 服务型应用

如果你的应用是一直运行的，比如 web 应用，那么可以使用 **`net/http/pprof`** 库，它能够在提供 HTTP 服务进行分析。

如果使用了默认的 `http.DefaultServeMux`（通常是代码直接使用 `http.ListenAndServe("0.0.0.0:8000", nil)`），只需要添加一行：

```
import _ "net/http/pprof"
```

如果你使用自定义的 Mux，则需要手动注册一些路由规则：

```
r.HandleFunc("/debug/pprof/", pprof.Index)
r.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline)
r.HandleFunc("/debug/pprof/profile", pprof.Profile)
r.HandleFunc("/debug/pprof/symbol", pprof.Symbol)
r.HandleFunc("/debug/pprof/trace", pprof.Trace)
```

## go tool pprof使用

go tool pprof 最简单的使用方式为 

```go tool pprof [binary] [source]```

binary 是应用的二进制文件，用来解析各种符号；
source 表示 profile 数据的来源，可以是本地的文件，也可以是 http 地址。比如：

``` go tool pprof ./hyperkube http://172.16.3.232:10251/debug/pprof/profile```

目前有下面几个收集项:

- /debug/pprof/profile：访问这个链接会自动进行 CPU profiling，持续 30s，并生成一个文件供下载
- /debug/pprof/heap： Memory Profiling 的路径，访问这个链接会得到一个内存 Profiling 结果的文件
- /debug/pprof/block：block Profiling 的路径
- /debug/pprof/goroutines：运行的 goroutines 列表，以及调用关系


# pprof和docker

Docker代码已经内置了pprof的参数收集,可以直接使用

需要修改配置

```
[root@paas]$cat /etc/docker/daemon.json 
{
    "hosts": ["unix:///var/run/docker.sock", "tcp://0.0.0.0:2375"],
    "insecure-registries": ["0.0.0.0/0"],
    "log-opts": {"max-size": "10M", "max-file": "5"},
    "max-concurrent-downloads": 100,
    "max-concurrent-uploads": 100,
    "graph": "/root/docker",
    "storage-driver": "overlay2",
    "storage-opts": ["overlay2.override_kernel_check=true"],
    "debug" : true
}
```


```
首先需要在定位环境中安装一个工具 socat：
$ yum install socat

其次执行如下命令：
$ socat -d -d TCP-LISTEN:8080,fork,bind=localhost UNIX:/var/run/docker.sock &

最后执行如下命令，可以进去pprof的交互命令环境
$ go tool pprof -inuse_space  localhost:8080/debug/pprof/heap

```

- 用--inuse_space来分析程序常驻内存的占用情况;
- 用--alloc_objects来分析内存的临时分配情况，可以提高程序的运行速度。
