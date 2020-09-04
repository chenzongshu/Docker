

因为在看docker源代码，必须需要了解Go语言，所以做了一些学习和记录，主要记录两者不同的地方。根据实际代码阅读中的问题而来，省略了和C语言相同的部分,干货满满。

> Go语言定义类型和变量名，方向和一般语言是反的，这点我觉得简直是反人类

# GO官方资料

进入到官网后，会看到很多资源。

- 文档：[golang.org/doc](https://golang.org/doc/)，官方文档，仔细读下文档首页并分类，了解下自己要学哪些内容；
- 一览：[tour.golang.org](https://tour.golang.org/)，交互式运行环境，不安装golang便可体验学习它的语法与使用；
- 指南：[golang.org/ref/spec](https://golang.org/ref/spec)，golang学习指导手册，从基础语法到高级特性全部都有介绍；
- 标准库：[golang.org/pkg/](https://golang.org/pkg/)，可以查看所有的官方库的接口、源码以及使用介绍；
- 博客：[blog.golang.org/](https://blog.golang.org/)，不定期分享go的最佳实践，有些公司也会投稿介绍自己的案例；
- 实验室：[play.golang.org](https://play.golang.org/)，感觉和tour类似，不过在这里编写的代码可以分享给别人；



# 关键字

- **GOROOT**
GO语言安装路径

- **GOPATH**
代码包所在路径,安装在系统上的GO包,路径为工作区位置,有源文件,相关包对象,执行文件

# GO程序结构
|-- bin  编译后的可执行文件 
|-- pkg  编译后的包文件 (.a)
|-- src  源代码
一般来说,bin和pkg不用创建,go命令自动创建

# 编译运行
两种方式
1、直接执行
```
$go run hello.go  # 实际是编译成A.OUT再执行

## 如果这样运行,目录中必须有main.go,不然会报错
```
2、编译执行
```
$go build hello.go
$./hello
```

# 关于分号
其实，和C一样，Go的正式的语法使用分号来终止语句。和C不同的是，这些分号由词法分析器在扫描源代码过程中使用简单的规则自动插入分号，因此输入源代码多数时候就不需要分号了。


# 包

先来看一个最简单的例子,程序员都知道的hello world
```
package main

import "fmt"

func main() {
   fmt.Println("Hello, World!")
}
```
 package main 就定义了包名。你必须在源文件中非注释的第一行指明这个文件属于哪个包，如：package main。package main表示一个可独立执行的程序，每个 Go 应用程序都包含一个名为 main 的包

- 多个文件组成,编译后与一个文件类似,相互可以直接引用,因此不能有相同的全局函数和变量.不得导入源代码文件中没有用到的package，否则golang编译器会报编译错误

- 每个子目录只能存在一个package,同一个package可以由多个文件组成

- package中每个init()函数都会被调用,如果不同文件,按照文件名字字符串比较"从小到大"的顺序,同一个文件从上到下

- 要生成golang可执行程序，**必须建立一个名为main的package**，并且在该package中**必须包含一个名为main()的函数**。

- **import关键字导入的是package路径，而在源文件中使用package时，才需要package名**。经常可见的import的目录名和源文件中使用的package名一致容易造成import关键字后即是package名的错觉，真正使用时，这两者可以不同

- Go 语言也有 Public 和 Private 的概念，粒度是包。如果类型/接口/方法/函数/字段的首字母大写，则是 Public 的，对其他 package 可见，如果首字母小写，则是 Private 的，对其他 package 不可见

# Modules

[Go Modules](https://github.com/golang/go/wiki/Modules) 是 Go 1.11 版本之后引入的，Go 1.11 之前使用 $GOPATH 机制。Go Modules 可以算作是较为完善的包管理工具

环境变量 GO111MODULE 的值默认为 AUTO，强制使用 Go Modules 进行依赖管理，可以将 GO111MODULE 设置为 ON。  

在一个空文件夹下，初始化一个 Module

```
$ go mod init example
go: creating new go.mod: module example
```

运行 `go run .`，将会自动触发第三方包 `rsc.io/quote`的下载，具体的版本信息也记录在了`go.mod`中：

```
module example

go 1.13

require rsc.io/quote v3.1.0+incompatible
```

# import特殊语法

加载自己写的模块:
```
import "./model"    # 当前文件同一个目录下的model目录
import "url/model"  # 加载GOPATH/src/url/model
```


## 点（.）操作
点（.）操作的含义是：点（.）标识的包导入后，调用该包中函数时可以省略前缀包名。

```
package main

import (
    . "fmt"
    "os"
)

func main() {
    for _, value := range os.Args {
        Println(value)
    }
}
```

## 别名操作
别名操作的含义是：将导入的包命名为另一个容易记忆的别名
```
package main

import (
    f "fmt"
    "os"
)

func main() {
    for _, value := range os.Args {
        f.Println(value)
    }
}
```

## 下划线（_）操作
下划线（_）操作的含义是：导入该包，但不导入整个包，而是执行该包中的init函数，因此无法通过包名来调用包中的其他函数。使用下划线（_）操作往往是为了注册包里的引擎，让外部可以方便地使用。
```
import _ "package1"
import _ "package2"
import _ "package3"
...
```

# 作用域

## 变量作用域



## 函数作用域

函数名称是小写开头的，它的作用域只属于所声明的包内使用，不能被其他包使用;

如果我们把函数名以大写字母开头，该函数的作用域就大了，可以被其他包调用。这也是Go语言中大小写的用处

# 变量

## 变量定义
和C语言是反的
```
var variable_list data_type
```
也可以采用混合型
```
var a,b,c = 3,4,"foo"
```

## **:=**
:= 表示声明变量并赋值
```
d:=100        #系统自动推断类型,不需要var关键字
```

## 字符串中文

```
var str = "hello 你好"  //长度是12个字符
```

**golang中string底层是通过byte数组实现的。中文字符在unicode下占2个字节，在utf-8编码下占3个字节，而golang默认编码正好是utf-8。**

可以将 string 转为 rune 数组,  转换成 `[]rune` 类型后，字符串中的每个字符，无论占多少个字节都用 int32 来表示，因而可以正确处理中文。

```
str2 := "Go语言"
runeArr := []rune(str2)
fmt.Println(reflect.TypeOf(runeArr[2]).Kind()) // int32
fmt.Println(runeArr[0], string(runeArr[0]))    // 71 G
fmt.Println(runeArr[1], string(runeArr[1]))    // 111 o
fmt.Println(runeArr[2], string(runeArr[2]))    // 35821 语
fmt.Println("len(runeArr)：", len(runeArr))    // len(runeArr)： 4
```

## 特殊变量

"\_" 是特殊变量,任何赋值给"_"的值都会被丢弃

## 指针

var var_name *var_type

nil为空指针


# 数组
var variable_name [size] variable_type
```
var balance [10] float32
var balance = [5]float32{1000.0, 2.0, 3.4, 7.0, 50.0}
var balance = []float32{1000.0, 2.0, 3.4, 7.0, 50.0}
```

数组的指针和指针数组是两个概念，数组的指针是`*[5]int`,指针数组是`[5]*int`，注意`*`的位置

# 切片(slice)

slice本身是一个指针, 它只有3个字段的数据结构：一个是指向底层数组的指针，一个是切片的长度，一个是切片的容量

**切片是基于数组来实现的**

```
var identifier []type  //切片不需要说明长度。

// 或者用make来创建
var slice1 []type = make([]type, len)
slice1 := make([]type, len)

slice:=make([]int,5,10)  //这时，我们创建的切片长度是5，容量是10,需要注意的这个容量10其实对应的是切片底层数组的。
```

切片还有nil切片和空切片，它们的长度和容量都是0，但是它们指向底层数组的指针不一样，nil切片意味着指向底层数组的指针为nil，而空切片对应的指针是个地址。

- 注意基于原数组(容量k)或者切片创建一个新的切片[i,j]后，那么新的切片的容量是k-i

- 在创建新切片的时候，最好要让新切片的长度和容量一样，这样我们在追加操作的时候就会生成新的底层数组，和原有数组分离，就不会因为共用底层数组而引起奇怪问题,因为共用数组的时候修改内容，会影响多个切片

- 切片作为函数入参传值的时候, 是把切片本身复制了一份传入, 类似引用传值, 函数内修改值外部会体现出来

- 还可以通过`...`操作符，把一个切片追加到另一个切片里。

  ```
  slice := []int{1, 2, 3, 4, 5}
  newSlice := slice[1:2:3]  //长度为2-1=1，容量为3-1=2的新切片
  
  newSlice=append(newSlice,slice...)
  fmt.Println(newSlice)
  fmt.Println(slice)
  
  [2 1 2 3 4 5]
  [1 2 3 4 5]
  ```

切片使用和python的风格类似

```
a := [5]int{1, 2, 3, 4, 5}
 
b := a[2:4] // a[2] 和 a[3]，但不包括a[4]
fmt.Println(b)
 
b = a[:4] // 从 a[0]到a[4]，但不包括a[4]
fmt.Println(b)
 
b = a[2:] // 从 a[2]到a[4]，且包括a[2]
fmt.Println(b)
```

# Map

- Map是类似引用传递
- 

# 循环与判断

- if语句没有圆括号,但是必须有花括号;
- switch没有break, 匹配到某个 case，执行完该 case 定义的行为后，默认不会继续往下执行。如果需要继续往下执行，需要使用 fallthrough;
- for没有圆括号;(注意go中没有while)
```
//经典的for语句 init; condition; post
for i := 0; i<10; i++{
     fmt.Println(i)
}
 
//精简的for语句 condition
i := 1
for i<10 {
    fmt.Println(i)
    i++
}
```

# 函数
```
func (p myType ) funcName ( [parameter list] ) ( [return_types] ) {
    函数体
    return 语句
}
```
- 函数可以有多个返回值
- (p myType) 表明 函数所属于的类型对象！，**即为特定类型定义方法**，可以省去不写，即为普通的函数

函数还可以输入不定参数,详细用法见例子:
```
func sum(nums ...int) {
    fmt.Print(nums, " ")  //输出如 [1, 2, 3] 之类的数组
    total := 0
    for _, num := range nums { //要的是值而不是下标
        total += num
    }
    fmt.Println(total)
}
func main() {
    sum(1, 2)
    sum(1, 2, 3)
 
    //传数组
    nums := []int{1, 2, 3, 4}
    sum(nums...)
}
```

# 方法
一个方法就是一个包含了接受者的函数，接受者可以是命名类型或者结构体类型的一个值或者是一个指针。所有给定类型的方法属于该类型的方法集。
语法:
```
func (variable_name variable_data_type) function_name() [return_type]{
   /* function body*/
}
```
下面定义一个结构体类型和该类型的一个方法：
```
type User struct {
  Name  string
  Email string
}
func (u User) Notify() error
```
首先我们定义了一个叫做 User 的结构体类型，然后定义了一个该类型的方法叫做 Notify，该方法的接受者是一个 User 类型的值。要调用 Notify 方法我们需要一个 User 类型的值或者指针：
```
// User 类型的值可以调用接受者是值的方法
damon := User{"AriesDevil", "ariesdevil@xxoo.com"}
damon.Notify()

// User 类型的指针同样可以调用接受者是值的方法
alimon := &User{"A-limon", "alimon@ooxx.com"}
alimon.Notify()
```
注意，当接受者不是一个指针时，该方法操作对应接受者的值的副本(意思就是即使你使用了指针调用函数，但是函数的接受者是值类型，所以函数内部操作还是对副本的操作，而不是指针操作),当接受者是指针时，即使用值类型调用那么函数内部也是对指针的操作

# 接口
Go和传统的面向对象的编程语言不太一样,没有类和继承的概念.通过接口来实现面向对象.
语法:
```
type Namer interface {
    Method1(param_list) return_type
    Method2(param_list) return_type
   ...
}
```
> 实现某个接口的类型，除了实现接口的方法外，还可以有自己的方法。
```
package main

import "fmt"

type Shaper interface {
    Area() float64
    //  Perimeter() float64
}

type Rectangle struct {
    length float64
    width  float64
}

// 实现 Shaper 接口中的方法
func (r *Rectangle) Area() float64 {
    return r.length * r.width
}

// Set 是属于 Rectangle 自己的方法
func (r *Rectangle) Set(l float64, w float64) {
    r.length = l
    r.width = w
}

func main() {
    rect := new(Rectangle)
    rect.Set(2, 3)
    areaIntf := Shaper(rect)
    fmt.Printf("The rect has area: %f\n", areaIntf.Area())
}
```
如果去掉 Shaper 中 Perimeter() float64的注释，编译的时候报错误，这是因为 Rectangle 没有实现 Perimeter() 方法。

> - 多个类型可以实现同一个接口。 
- 一个类型可以实现多个接口。

# 内存分配

有new和make

- new:new(T)为一个类型为T的新项目分配了值为零的存储空间并返回其地址，也就是一个类型为*T的值,返回了一个指向新分配的类型为T的零值的指针,内存只是清零但是没有初始化.
- make:仅用于创建切片、map和chan（消息管道），并返回类型T（不是*T）的一个**被初始化**了的（不是零）**实例**。

# 错误处理 – Defer

为了进行错误处理,比如防止资源泄露,go设计了一个defer函数
```
func CopyFile(dstName, srcName string) (written int64, err error) {
    src, err := os.Open(srcName)
    if err != nil {
        return
    }
    defer src.Close()
 
    dst, err := os.Create(dstName)
    if err != nil {
        return
    }
    defer dst.Close()
 
    return io.Copy(dst, src)
}
```
Go的defer语句预设一个函数调用（延期的函数），该调用在函数执行defer返回时立刻运行。该方法显得不同常规，但却是处理上述资源泄露情况很有效，无论函数怎样返回，都必须进行资源释放。

再看一个列子
```
for i := 0; i < 5; i++ {
    defer fmt.Printf("%d ", i)
}
```
被延期的函数以后进先出（LIFO）的顺行执行，因此以上代码在返回时将打印4 3 2 1 0

# 协程goroutine
先来复习下,进程,线程和协程的概念,GoRoutine就是Go的协程

- 进程：分配完整独立的地址空间，拥有自己独立的堆和栈，既不共享堆，亦不共享栈，进程的切换只发生在内核态，由操作系统调度。 
- 线程：和其它本进程的线程共享地址空间，拥有自己独立的栈和共享的堆，共享堆，不共享栈，线程的切换一般也由操作系统调度(标准线程是的)。 
- 协程：和线程类似，共享堆，不共享栈，协程的切换一般由程序员在代码中显式控制。

## goroutine基本概念
GoRoutine主要是使用go关键字来调用函数，你还可以使用匿名函数;
**注意,go routine被调度的的先后顺序是没法保证的**
```
package main
import "fmt"

func f(msg string) {
    fmt.Println(msg)
}

func main(){
    go f("goroutine")

    go func(msg string) {
        fmt.Println(msg)
    }("going")
}
```
下面来看一个常见的错误用法
```
array := []string{"a", "b", "c", "d", "e", "f", "g", "h", "i"}
var i = 0
for index, item := range array {
  go func() {
      fmt.Println("index:", index, "item:", item)
      i++
  }()
}
time.Sleep(time.Second * 1)
fmt.Println("------------------")
//output:
------------------
index: 8 item: i
index: 8 item: i
index: 8 item: i
index: 8 item: i
index: 8 item: i
index: 8 item: i
index: 8 item: i
index: 8 item: i
index: 8 item: i
------------------
```
最初的意图是index与item每次为1,a;2,b;3,c;….这样,结果却不是这样,到底什么原因呢?

这里的go func每个index与item是共享的，并不是局部的，由于for循环的执行是很快的，每次循环启动一个go routine，在for循环结束之后（此时index与item的值分别变成了8与e），但是这个时候第一个启动的goroutine可能还没有开始执行，由于它们是共享变量的，之后所有输出的index与item都是8与e于是出现了上面的效果。

将原来程序做些修改就能满足要求了:
```
for i = 0; i < length; i++ {
  go func(index int) {
  //这里如果打印 array[i]的话 就会index out of range了因为i是全局的(在执行到打印语句的时候i的值已经变成了length+1了)不是新启动的这个goroutine的
  //新启动的goroutine与原来的main routine 是共享占空间的 因此 这个i也是共享的
  fmt.Println("index:", index, "item:", array[index])
  }(i)
```

## goroutine并发
goroutine有个特性，如果一个goroutine没有被阻塞，那么别的goroutine就不会得到执行,这并不是真正的并发，如果你要真正的并发，你需要在你的main函数的第一行加上下面的这段代码：
```
import "runtime"
...
runtime.GOMAXPROCS(4)
```

goroutine并发安全性问题,需要注意:

- 互斥量上锁
```
var mutex = &sync.Mutex{} //可简写成：var mutex sync.Mutex
mutex.Lock()
...
mutex.Unlock()
```
- 原子性操作
```
import "sync/atomic"
......
atomic.AddUint32(&cnt, 1)
......
cntFinal := atomic.LoadUint32(&cnt)//取数据
```

# Channel 信道

## Channel的基本概念

Channal就是用来通信的，像Unix下的管道一样,
它的操作符是箭头" <-" , **箭头的指向就是数据的流向**
```go
ch <- v // 发送值v到Channel ch中
v := <-ch // 从Channel ch中接收数据，并将数据赋值给v
<-ch //从通道里读取值，然后忽略
```

通道分成有缓冲和无缓冲通道, 下面第三个就是有缓冲通道

```go
ch:=make(chan int)
ch:=make(chan int,0)
ch:=make(chan int,2)
```

无缓冲通道通道在接收前没有能力保存任何值，它要求发送goroutine和接收goroutine同时准备好，才可以完成发送和接收操作, 如果没有同时准备好的话，先执行的操作就会阻塞等待，直到另一个相对应的操作准备好为止。这种无缓冲的通道我们也称之为`同步通道`

下面的程序演示了一个goroutine和主程序通信的例程。

```go
package main

import "fmt"

func main() {
    //创建一个string类型的channel
    channel := make(chan string)

    //创建一个goroutine向channel里发一个字符串
    go func() { channel <- "hello" }()

    msg := <- channel
    fmt.Println(msg)
}
```

chan为先入先出的队列,有三种类型,双向,只读,只写,分别为"chan","chan<-","<-chan"

```
chan T          // 可以接收和发送类型为 T 的数据  
chan<- float64  // 只可以用来发送 float64 类型的数据  
<-chan int      // 只可以用来接收 int 类型的数据  
```

注意`<-`操作符的为止，在后面是只能发送，对应发送操作；在前面是只能接收，对应接收操作

初始化时候,可以指定容量`make(chanint,100)`;容量(capacity)代表Channel容纳的最多的元素的数量

## Channel的阻塞
channel默认上是阻塞的，也就是说，如果Channel满了，就阻塞写，如果Channel空了，就阻塞读。于是，我们就可以使用这种特性来同步我们的发送和接收端。
```
package main

import "fmt"
import "time"

func main() {

    channel := make(chan string) //注意: buffer为1

    go func() {
        channel <- "hello"
        fmt.Println("write \"hello\" done!")

        channel <- "World" //Reader在Sleep，这里在阻塞
        fmt.Println("write \"World\" done!")

        fmt.Println("Write go sleep...")
        time.Sleep(3*time.Second)
        channel <- "channel"
        fmt.Println("write \"channel\" done!")
    }()

    time.Sleep(2*time.Second)
    fmt.Println("Reader Wake up...")

    msg := <-channel
    fmt.Println("Reader: ", msg)

    msg = <-channel
    fmt.Println("Reader: ", msg)

    msg = <-channel //Writer在Sleep，这里在阻塞
    fmt.Println("Reader: ", msg)
}
```
结果为
```
Reader Wake up...
Reader:  hello
write "hello" done!
write "World" done!
Write go sleep...
Reader:  World
write "channel" done!
Reader:  channel
```

## 多个Channel的select
```
package main
import "time"
import "fmt"

func main() {
    //创建两个channel - c1 c2
    c1 := make(chan string)
    c2 := make(chan string)

    //创建两个goruntine来分别向这两个channel发送数据
    go func() {
        time.Sleep(time.Second * 1)
        c1 <- "Hello"
    }()
    go func() {
        time.Sleep(time.Second * 1)
        c2 <- "World"
    }()

    //使用select来侦听两个channel
    for i := 0; i < 2; i++ {
        select {
        case msg1 := <-c1:
            fmt.Println("received", msg1)
        case msg2 := <-c2:
            fmt.Println("received", msg2)
        }
    }
}
```

# 定时器
Go语言中可以使用time.NewTimer或time.NewTicker来设置一个定时器，这个定时器会绑定在你的当前channel中，通过channel的阻塞通知机器来通知你的程序。
```
package main

import "time"
import "fmt"

func main() {
    timer := time.NewTimer(2*time.Second)

    <- timer.C
    fmt.Println("timer expired!")
}
```


## 关闭channel
使用close命令
```
close(channel)
```

# 系统调用
Go语言主要是通过两个包完成的。一个是os包，一个是syscall包。
这两个包里提供都是Unix-Like的系统调用，

- syscall里提供了什么Chroot/Chmod/Chmod/Chdir…，Getenv/Getgid/Getpid/Getgroups/Getpid/Getppid…，还有很多如Inotify/Ptrace/Epoll/Socket/…的系统调用。
- os包里提供的东西不多，主要是一个跨平台的调用。它有三个子包，Exec（运行别的命令）, Signal（捕捉信号）和User（通过uid查name之类的）

如执行命令行
```
package main
import "os/exec"
import "fmt"
func main() {
    cmd := exec.Command("ping", "127.0.0.1")
    out, err := cmd.Output()
    if err!=nil {
        println("Command Error!", err.Error())
        return
    }
    fmt.Println(string(out))
}
```

# 锁

## 互斥锁

开箱即用型,使用sync.Mutex

```
var mutex sync.Mutex
mutex.Lock()

defer mutex.Unlock()  #defer语句来保证该互斥锁的及时解锁
```

`defer`语句可以用来保证在该函数被执行结束之前互斥锁mutex一定会被解锁。这省去了我们在所有return语句之前以及异常发生之时重复的附加解锁操作的工作。

## 读写锁

读写锁即是针对于读写操作的互斥锁。它与普通的互斥锁最大的不同就是，它可以分别针对读操作和写操作进行锁定和解锁操作。

读写锁遵循的访问控制规则与互斥锁有所不同。在读写锁管辖的范围内，它允许任意个读操作的同时进行。但是，在同一时刻，它只允许有一个写操作在进行。并且，在某一个写操作被进行的过程中，读操作的进行也是不被允许的。

读写锁由结构体类型sync.RWMutex代表。与互斥锁类似，sync.RWMutex类型的零值就已经是立即可用的读写锁了

```
写操作锁定
func (*RWMutex) Lock
func (*RWMutex) Unlock

# 读操作锁定
func (*RWMutex) RLock
func (*RWMutex) RUnlock
```

# Range关键字

range是用来遍历的，使用非常方便

**注意！注意！注意！range每次取出来的，是数组元素的一个拷贝，所以如果用来赋值，不会改变原数组**

下面来看一组程序

```
type Foo struct {
  bar string
}
func main() {
  list := []Foo{
    {"A"},
    {"B"},
    {"C"},
  }
  list2 := make([]*Foo, len(list))
  for i, value := range list {
    list2[i] = &value
  }
  fmt.Println(list[0], list[1], list[2])
  fmt.Println(list2[0], list2[1], list2[2])
}
```

我们干了什么事情呢

>    1、定义了一个叫做Foo的结构，里面有一个叫bar的field。随后，我们创建了一个基于Foo结构体的slice，名字叫list
     2、我们还创建了一个基于Foo结构体指针类型的slice，叫做list2
     3、在一个for循环中，我们试图遍历list中的每一个元素，获取其指针地址，并赋值到list2中index与之对应的位置。
     4、最后，分别输出list与list2中的每个元素

期望结果为:

```
{A} {B} {C}
&{A} &{B} &{C}
```

实际结果为:

```
{A} {B} {C}
&{C} &{C} &{C}
```

在Go的`for…range`循环中，Go始终使用值拷贝的方式代替被遍历的元素本身，简单来说，就是`for…range`中那个value，是一个值拷贝，而不是元素本身。这样一来，当我们期望用&获取元素的指针地址时，实际上只是取到了value这个临时变量的指针地址，而非list中真正被遍历到的某个元素的指针地址。而在整个`for…range`循环中，value这个临时变量会被重复使用，所以，在上面的例子中，list2被填充了三个相同的指针地址，并且这三个地址都指向value，而在最后一次循环中，value被赋与了`{c}`的指针地址。因此，list2输出的时候显示出了三个`&{c}` 。

同样,下面的写法,得到的结果也是三个`&{c}`

```
var value Foo
for i := 0; i < len(list); i++ {
 value = list[i]
 list2[i] = &value
}
```

那么，怎样才是正确的写法呢？我们应该用index来访问`for…range`中真实的元素，并获取其指针地址：

```
for i, _ := range list {
 list2[i] = &list[i]
}
```

同理,下面的这个通过值来修改的不会成功

```
 for _, e := range array {
 e.field = "foo"
 }
```

必须通过index来修改

```
for i, _ := range array {
 array[i].field = "foo"
}
```

# context

控制并发有两种经典的方式，一种是WaitGroup，另外一种就是Context

通过context，我们可以方便地对同一个请求所产生地goroutine进行约束管理，可以设定超时、deadline，甚至是取消这个请求相关的所有goroutine。

形象地说，假如一个请求过来，需要A去做事情，而A让B去做一些事情，B让C去做一些事情，A、B、C是三个有关联的goroutine，那么问题来了：假如在A、B、C还在处理事情的时候请求被取消了，那么该如何优雅地同时关闭goroutine A、B、C呢？这个时候就轮到context包上场了。

- context包里的方法是线程安全的，可以被多个线程使用
- 当Context被canceled或是timeout, Done返回一个被closed 的channel
- 在Done的channel被closed后, Err代表被关闭的原因
- 如果存在, Deadline 返回Context将要关闭的时间
- 如果存在，Value 返回与 key 相关了的值，不存在返回 nil

对应下面的包核心

```
// context 包的核心
type Context interface {               
    Done() <-chan struct{}      
    Err() error 
    Deadline() (deadline time.Time, ok bool)
    Value(key interface{}) interface{}
}
```

我们不需要手动实现这个接口，context 包已经给我们提供了两个，一个是 Background()，一个是 TODO()，这两个函数都会返回一个 Context 的实例。只是返回的这两个实例都是空 Context

## 主要方法

```
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
func WithValue(parent Context, key interface{}, val interface{}) Context
```

## 使用方法

一般使用是这样使用，创建context然后调用接口

```
ctx,cancel := context.WithCancel(context.Background())
stack := &Stack{}
pkt,err := stack.Read(ctx)
```

那么，它本身就可以支持取消和超时，也就是用户如果需要取消，比如发送了SIGINT信号，程序需要退出，可以在收到信号后调用cancel：

```
sc := make(chan os.Signal, 0)
signal.Notify(sc, syscall.SIGINT, syscall.SIGTERM)
go func() {
    for range sc {
        cancel()
    }
}()
```

如果需要超时，这个API也不用改，只需要调用前设置超时时间：

```
ctx,cancel := context.WithTimeout(context.Background(), 3*time.Second)
defer cancel()
pkt,err := stack.Read(ctx)
```

优势是不影响Read()即用户API的本身的逻辑，在Read中只需要关注context是否Done：

```
func (v *Stack) Read(ctx context.Context) (pkt Packet, err error) {
    select {
    case <- ctx.Done():
        return nil,ctx.Err()
    }
    return
}
```

## 使用规范

- 1、不要将 Contexts 放入结构体，相反context应该作为第一个参数传入，命名为ctx。
```
func DoSomething(ctx context.Context, arg Arg) error {
         ... use ctx ...
}
```
- 2、即使函数允许也不要传递一个nil的Context。如果不确定使用哪种Conetex，传递context.TODO 
- 3、使用context的Value相关方法只应该用于在程序和接口中传递的和请求相关的数据，不要用它来传递一些可选的参数。 
- 4、相同的Context可以在不同的goroutines中传递，Contexts是线程安全的。



# 目录结构

在开发中，一个良好的目录结构是很重要的，好的目录结构不仅能使项目结构清晰，也能让后加入者快速了解项目，便于上手。

```golang
├── conf                         # 配置文件统一存放目录
│   ├── config.yaml              # 配置文件
├── config                       # 专门用来处理配置和配置文件的Go package
│   └── config.go                 
├── db.sql                       # 在部署新环境时，可以登录MySQL客户端，执行source db.sql创建数据库和表
├── handler                      # 类似MVC架构中的C，用来读取输入，并将处理流程转发给实际的处理函数，最后返回结果
│   ├── handler.go
│   ├── sd                       # 健康检查handler
│   │   └── check.go 
│   └── user                     # 核心：用户业务逻辑handler
│       ├── create.go            # 新增用户
│       └── user.go              # 存放用户handler公用的函数、结构体等
├── main.go                      # Go程序唯一入口
├── Makefile                     # Makefile文件，一般大型软件系统都是采用make来作为编译工具
├── model                        # 数据库相关的操作统一放在这里，包括数据库初始化和对表的增删改查
│   ├── init.go                  # 初始化和连接数据库
│   ├── model.go                 # 存放一些公用的go struct
│   └── user.go                  # 用户相关的数据库CURD操作
├── pkg                          # 引用的包
│   ├── constvar                 # 常量统一存放位置
│   │   └── constvar.go
│   ├── errno                    # 错误码存放位置
│   │   ├── code.go
│   │   └── errno.go
├── README.md                    # API目录README
├── router                       # 路由相关处理
│   ├── middleware               # API服务器用的是Gin Web框架，Gin中间件存放位置
│   │   ├── header.go
│   │   ├── logging.go
│   │   └── requestid.go
│   └── router.go                # 路由
├── service                      # 实际业务处理函数存放位置
│   └── service.go
├── util                         # 工具类函数存放目录
│   ├── util.go 
│   └── util_test.go
└── vendor                         # vendor目录用来管理依赖包
    ├── github.com
    ├── golang.org
    ├── gopkg.in
    └── vendor.json
```



# 附录:

Go知识图谱:  https://www.processon.com/view/link/5a9ba4c8e4b0a9d22eb3bdf0#map

Go知乎知识汇总: https://www.zhihu.com/question/30461290

7天用Go从零实现系列: https://github.com/geektutu/7days-golang

更优雅的组织Go程序: [https://medium.com/@draveness/go-%E8%AF%AD%E8%A8%80%E6%98%AF%E4%B8%80%E9%97%A8%E7%AE%80%E5%8D%95-%E6%98%93%E5%AD%A6%E7%9A%84%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80-%E5%AF%B9%E4%BA%8E%E6%9C%89%E7%BC%96%E7%A8%8B%E8%83%8C%E6%99%AF%E7%9A%84%E5%B7%A5%E7%A8%8B%E5%B8%88%E6%9D%A5%E8%AF%B4-%E5%AD%A6%E4%B9%A0-go-3b12c5adc70b](https://medium.com/@draveness/go-语言是一门简单-易学的编程语言-对于有编程背景的工程师来说-学习-go-3b12c5adc70b)