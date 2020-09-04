# 安装

如果是基于`GOPATH`开发的，你需要先使用`go get -u github.com/gin-gonic/gin` 下载gin，然后`import`导入即可。

如果你是用Go Module这种方式，使用`import`直接导入使用，然后你在`go run`运行的时候，会自动的下载`gin`包编译使用。当然你也可以通过`go mod tidy`来下载依赖的模块。

# Hello Gin

入门简单例子

```
package main

import "github.com/gin-gonic/gin"

func main() {
	r := gin.Default()
	r.GET("/", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"Blog":"chenzongshu.github.io",
			"wechat":"BruceVampire",
		})
	})
	r.Run(":8080")
}
```

然后我们运行它，打开浏览器，输入`http://localhost:8080/`就可以看到如下内容：

```
{"Blog":"chenzongshu.github.io","wechat":"BruceVampire"}
```

- `r := gin.Default()`是实例化一个默认的gin示例

- `c.JSON`方法就是返回一个JSON格式的字符串

  > ```go
  > func (c *Context) JSON(code int, obj interface{})
  > ```

- `gin.H`其实是一个`map[string]interface{}`,声明为`H`类型，便于操作



# 路由参数

如果是多个值的URL, 如下

```
/users/123
/users/456
/users/23456
```

不可能一个个注册, 可以使用路由参数

```
func main() {
	r := gin.Default()

	r.GET("/users/:id", func(c *gin.Context) {
		id := c.Param("id")
		c.String(200, "The user id is  %s", id)
	})
	r.Run(":8080")
}
```

我们运行如上代码，打开浏览器，输入`http://localhost:8080/users/123`，就可以看到如下信息：

```
The user id is  123
```

我们可以更换`http://localhost:8080/users/123`中的id `123` 为其他字符串，会发现都可以正常打印，这就是路由匹配、路由正则，或者路由参数。

`Gin`的路由采用的是`httprouter`，所以它的路由参数的定义和`httprouter`也是一样的。

`/users/:id` 就是一种路由匹配模式，也是一个通配符，其中`:id`就是一个路由参数，我们可以通过`c.Param("id")`获取定义的路由参数的值，然后用来做事情，比如打印出来。

- `Gin`路径中的匹配都是字符串，它是不区分数字、字母和汉字的，都匹配
- `Gin`的路由是单一的，不能有重复。比如这里我们注册了`/users/:id`,那么我们就不能再注册`/users/list`这种路由



还有一种 "*"的路由匹配模式不建议使用



# 参数获取

## 单参数

在`Gin`中，为我们提供了简便的方法来获取查询参数的值，我们只需要知道查询参数的key（参数名）就可以了。

```
func main() {
	r := gin.Default()

	r.GET("/", func(c *gin.Context) {
		c.String(200, c.Query("wechat"))
	})
	r.Run(":8080")
}
```

运行这段代码，打开浏览器访问`http://localhost:8080/?wechat=flysnow_org`,就可以看到`flysnow_org`文字。这表示我们通过`c.Query("wechat")`获取到了查询参数`wechat`的值是`flysnow_org`。

`Query`方法为我们提供了获取对应`key`的值的能力，如果该`key`不存在，则返回`""`字符串。如果对于一些数字参数，比如`id`如果返回为空的话，我们进行字符串转数字的时候会报错，这时候，我们就可以通过`DefaultQuery`方法指定一个默认值：

```
c.DefaultQuery("wechat", "flysnow_org")

c.DefaultQuery("id", "0")
```

## 数组参数

如果一个key, 有多个value, 可以作为数组传入, URL可以这样( 媒体途径有2种: 博客和微信)

```
http://localhost:8080/?media=blog&media=wechat
```

获取方式:

```
func main() {
	r := gin.Default()

	r.GET("/", func(c *gin.Context) {
		c.JSON(200, c.QueryArray("media"))
	})
	r.Run(":8080")
}
```

得到结果为:

```
["blog","wechat"]
```

## Map参数

这种参数情况使用较少, 比如假设有a,b,c三个人，他们对应的id是123,456,789, 用Map参数就是

```
http://localhost:8080/map?ids[a]=123&ids[b]=456&ids[c]=789
```

代码使用

```
	r.GET("/map", func(c *gin.Context) {
		c.JSON(200, c.QueryMap("ids"))
	})
```

可以看到结果

```
{"a":"123","b":"456","c":"789"}
```



# 表单获取

对于Form表单，我们不会陌生，比如`input`文本框、密码框等等，可以让我们输入一些数据，然后点击「保存」、「提交」等按钮，把数据提交到服务器的。

对于Form表单来说，有两种提交方式`GET`和`POST`。其中`GET`方式就是我们前两篇文章的URL查询参数的方式，参考即可获得对应的参数键值对，这篇文章主要介绍`POST`的方式的表单，而`Gin`处理的也是这种表单。

```
func main() {
	r := gin.Default()
	r.POST("/", func(c *gin.Context) {
		wechat := c.PostForm("wechat")
		c.String(200, wechat)
	})

	r.Run(":8080")
}
```

运行这段代码，然后打开终端输入`curl -d wechat=flysnow_org http://localhost:8080/` 回车，就会看到打印的如下信息：

```
flysnow_org
```

这里我们通过`curl`这个工具来模拟POST请求，当然你可以可以使用`Postman`比较容易操作的可视化工具。

在这个`Gin`示例中，使用`PostForm`方法来获取相应的键值对，它接收一个`key`，也就是我们`html`中`input`这类表单标签的`name`属性值。

`PostForm`方法和查询参数的`Query`是一样的，如果对应的`key`不存在则返回空字符串。

## PostForm系列方法

和查询参数方法一样，对于表单的参数接收，`Gin`也提供了一系列的方法，他们的用法和查询参数的一样。

| 查询参数      | Form表单         | 说明                                    |
| :------------ | :--------------- | :-------------------------------------- |
| Query         | PostForm         | 获取key对应的值，不存在为空字符串       |
| GetQuery      | GetPostForm      | 多返回一个key是否存在的结果             |
| QueryArray    | PostFormArray    | 获取key对应的数组，不存在返回一个空数组 |
| GetQueryArray | GetPostFormArray | 多返回一个key是否存在的结果             |
| QueryMap      | PostFormMap      | 获取key对应的map，不存在返回空map       |
| GetQueryMap   | GetPostFormMap   | 多返回一个key是否存在的结果             |
| DefaultQuery  | DefaultPostForm  | key不存在的话，可以指定返回的默认值     |



# 分组路由

## 分组路由

在我们开发定义路由的时候，可能会遇到很多部分重复的路由：

```
/admin/users
/admin/manager
/admin/photo
```

以上等等，这些路由最前面的部分`/admin/`是相同的，如果我们一个个写也没问题，但是不免会觉得琐碎、重复，无用劳动，那么有没有一种更好的办法来解决呢？`Gin`为我们提供的解决方案就是分组路由

上面的示例，就是分好组的路由，分组的原因有很多，比如基于模块化，把同样模块的放在一起，比如基于版本，把相同版本的API放一起，便于使用。在有的框架中，分组路由也被称之为命名空间。

```
func main() {
	r := gin.Default()

    //V1版本的API
	v1Group := r.Group("/v1")
	v1Group.GET("/users", func(c *gin.Context) {
		c.String(200, "/v1/users")
	})
	v1Group.GET("/products", func(c *gin.Context) {
		c.String(200, "/v1/products")
	})

    //V2版本的API
	v2Group := r.Group("/v2")
	v2Group.GET("/users", func(c *gin.Context) {
		c.String(200, "/v2/users")
	})
	v2Group.GET("/products", func(c *gin.Context) {
		c.String(200, "/v2/products")
	})

	r.Run(":8080")
}
```

只需要通过`Group`方法就可以生成一个分组，然后用这个分组来注册不同路由，用法和我们直接使用`r`变量一样，非常简单。这里为了便于阅读，一般都是把不同分组的，用`{}`括起来。

```
	v1Group := r.Group("/v1")
	{
		v1Group.GET("/users", func(c *gin.Context) {
			c.String(200, "/v1/users")
		})
		v1Group.GET("/products", func(c *gin.Context) {
			c.String(200, "/v1/products")
		})
	}

	v2Group := r.Group("/v2")
	{
		v2Group.GET("/users", func(c *gin.Context) {
			c.String(200, "/v2/users")
		})
		v2Group.GET("/products", func(c *gin.Context) {
			c.String(200, "/v2/products")
		})
	}
```

## 路由中间件

通过`Group`方法的定义，我们可以看到，它是可以接收两个参数的：

第一个就是我们注册的分组路由（命名空间）；第二个是一个`...HandlerFunc`，可以把它理解为这个分组路由的中间件，所以这个分组路由下的子路由在执行的时候，都会调用它。

这样就给我们带来很多的便利，比如请求的统一处理，比如`/admin`分组路由下的授权校验处理

```
	v1Group := r.Group("/v1", func(c *gin.Context) {
		fmt.Println("/v1中间件")
	})
```

这样不管你是访问`/v1/users`，还是访问`/v1/products`,控制台都会打印出`/v1中间件`。

从`Group`的方法定义，还可以看到，我们可以注册多个`HandlerFunc`，对分组路由进行多次处理。

## 分组路由嵌套

我们不光可以定义一个分组路由，还可以在这个分组路由中再添加一个分组路由，达到分组路由嵌套的目的，这种业务场景也不少，比如在上面增加如下场景：

```
/v1/admin/users
/v1/admin/manager
/v1/admin/photo
```

V1版本下的`admin`模块，我们使用`Gin`可以这么实现。

```
	v1AdminGroup := v1Group.Group("/admin")
	{
		v1AdminGroup.GET("/users", func(c *gin.Context) {
			c.String(200, "/v1/admin/users")
		})
		v1AdminGroup.GET("/manager", func(c *gin.Context) {
			c.String(200, "/v1/admin/manager")
		})
		v1AdminGroup.GET("/photo", func(c *gin.Context) {
			c.String(200, "/v1/admin/photo")
		})
	}
```

# JSON输出

## JSON

- 可以输出json格式
- 可以输出json数组
- 可以把struct转换给json格式

```
func main() {
	r := gin.Default()

	r.GET("/users/123", func(c *gin.Context) {
		c.JSON(200, user{ID: 123, Name: "张三", Age: 20})
	})

	r.Run(":8080")
}

type user struct {
	ID   int
	Name string
	Age  int
}
```

## IndentedJSON 

增加JSON的缩进

## PureJSON

对于JSON字符串中特殊的字符串，比如`<`，Gin默认是转义的，比如变成`\ u003c`，但是有时候我们为了可读性，需要保持原来的字符，不进行转义，这时候我们就可以使用`PureJSON`

## AsciiJSON

把非`Ascii`字符串转为unicode编码



# ProtoBuf

## 序列化

```go
func main() {
	r := gin.Default()
	r.GET("/protobuf", func(c *gin.Context) {

		data := &module.User{
			Name: "张三",
			Age:  20,
		}
		c.ProtoBuf(http.StatusOK, data)
	})
	r.Run(":8080")
}
```

在Gin中，我们直接使用生成的`module.User`即可，把它作为参数传给`c.ProtoBuf`方法，这样Gin就帮我们自动序列化（其实内部实现还是golang protobuf库），然后我们就可以通过`http://localhost:8080/protobuf`获取的这个序列化数据了。这个就是 Protocol Buffer API。

## 客户端反序列化ProtoBuf数据

反序列化也很简单，我们先启动上面的服务端 Protocol Buffer API 服务。

```
func main() {
	resp, err := http.Get("http://localhost:8080/protobuf")
	if err != nil {
		fmt.Println(err)
	} else {
		defer resp.Body.Close()
		body, err := ioutil.ReadAll(resp.Body)
		if err != nil {
			fmt.Println(err)
		} else {
			user := &module.User{}
			proto.UnmarshalMerge(body, user)
			fmt.Println(*user)
		}

	}
}
```

以上就是反序列化，得到`User`对象的例子。我们运行这段代码，可以看到`{张三 20 {} [] 0}`，拿到了我们想要的信息。这里的关键点，就是通过`proto.UnmarshalMerge(body, user)`反序列化。



# 中间件

## Gin默认中间件

在Gin中，我们可以通过Gin提供的默认函数，来构建一个自带默认中间件的`*Engine`。

```
r := gin.Default()
```

`Default`函数会默认绑定两个已经准备好的中间件，它们就是Logger 和 Recovery，帮助我们打印日志输出和`painc`处理。

```
func Default() *Engine {
	debugPrintWARNINGDefault()
	engine := New()
	engine.Use(Logger(), Recovery())
	return engine
}
```

从中我们可以看到，Gin的中间件是通过`Use`方法设置的，它接收一个可变参数，所以我们同时可以设置多个中间件。

```
func (engine *Engine) Use(middleware ...HandlerFunc) IRoutes
```

到了这里其实我们应该更加明白了，一个Gin的中间件，其实就是Gin定义的一个`HandlerFunc`,而它在我们Gin中经常使用，比如：

```
r.GET("/", func(c *gin.Context) {
		fmt.Println("首页")
		c.JSON(200, "")
	})
```

后面的`func(c *gin.Context)`这部分其实就是一个`HandlerFunc`。

## 中间件实现HTTP Basic Authorization

HTTP Basic Authorization 是HTTP常用的认证方案，它通过Authorization 请求消息头含有服务器用于验证用户代理身份的凭证，格式为：

```
Authorization: Basic <credentials>
```

如果认证不成功，服务器返回401 Unauthorized 状态码以及WWW-Authenticate 消息头，让客户端输入用户名和密码进一步认证。

在Gin中，为我们提供了`gin.BasicAuth`帮我们生成基本认证的中间件，方便我们的开发。

```
	r := gin.Default()

	r.Use(gin.BasicAuth(gin.Accounts{
		"admin": "123456",
	}))
	
	r.GET("/", func(c *gin.Context) {
		c.JSON(200, "首页")
	})
	
	r.Run(":8080")
```

我们添加一个用户名为`admin`,密码是`123456`的账户，用于HTTP 基本认证。现在我们运行启动，访问`http://localhost:8080/`，这时候只有我们输入正确的用户名和密码，才能看到`首页`，否则是看不到的，这样我们就达到了授权的目的,就是这么简单。

## 针对特定URL的Basic Authorization

其实在实际的项目开发中，我们基本上不太可能对所有的URL都进行认证的，一般只有一些需要认证访问的数据才需要认证，比如网站的后台，那么这时候我们就可以用分组路由来处理。

```
func main() {
	r := gin.Default()

	r.GET("/", func(c *gin.Context) {
		c.JSON(200, "首页")
	})

	adminGroup := r.Group("/admin")
	adminGroup.Use(gin.BasicAuth(gin.Accounts{
		"admin": "123456",
	}))

	adminGroup.GET("/index", func(c *gin.Context) {
		c.JSON(200, "后台首页")
	})

	r.Run(":8080")
}
```

现在我们运行访问`/`首页是可以正常显示的，但是我们访问`/admin/index`会提示输入密码，其实所有`/admin/*`下的URL都会让输入密码才能访问，这就是我们分组路由的好处，我们通过把中间件加到`/admin`这个分组路由上，就可以达到我们的目的。

通过分组路由的控制，我们可以比较灵活的设置HTTP认证，粒度可以自己随意控制。

## 自定义中间件

我们已经知道，Gin的中间件其实就是一个`HandlerFunc`,那么只要我们自己实现一个`HandlerFunc`，就可以自定义一个自己的中间件。现在我们以统计每次请求的执行时间为例，来演示如何自定义一个中间件。

```
func costTime() gin.HandlerFunc {
	return func(c *gin.Context) {
		//请求前获取当前时间
		nowTime := time.Now()

		//请求处理
		c.Next()

		//处理后获取消耗时间
		costTime := time.Since(nowTime)
		url := c.Request.URL.String()
		fmt.Printf("the request URL %s cost %v\n", url, costTime)
	}
}
```

以上我们就实现了一个Gin中间件，比较简单，而且有注释加以说明，这里要注意的是`c.Next`方法，这个是执行后续中间件请求处理的意思（含没有执行的中间件和我们定义的GET方法处理），这样我们才能获取执行的耗时。也就是在`c.Next`方法前后分别记录时间，就可以得出耗时。

有了自定义的中间件，我们就可以这么使用。

```
func main() {
	r := gin.New()

	r.Use(costTime())

	r.GET("/", func(c *gin.Context) {
		c.JSON(200, "首页")
	})

	r.Run(":8080")
}
```



# 文件托管

## 托管一个静态文件

在项目的开发中，你可能需要这么一个功能：把服务器上的JS文件暴露出来以供访问，比如让网站调用里面的JS函数等。对于这种情形，我们可以使用Gin提供的`StaticFile`方法很方便的完成。

```
func main() {
	router := gin.Default()
	router.StaticFile("/adobegc.log", "/tmp/adobegc.log")
	router.Run(":8080")
}
```

通过`StaticFile`方法，把文件`/tmp/adobegc.log`托管在网络上，并且设置访问路径为`/adobegc.log`,这样我们通过`http://localhost:8080/adobegc.log`就可以访问这个文件，看到它的内容了。

通过这种方式可以托管任何类型的文件，并且我们不用指定`Content-Type`,因为会自动识别。

## 托管一个目录

一般情况下，我们会把我们的静态文件放在一个目录中，比如我们使用Gin做网站开发的时候，可以把CSS、JS和Image这些静态资源文件都放在一个目录中，然后使用`Static`方法把整个目录托管，这样就可以自由访问这个目录中的所有文件了。

```
router.Static("/static", "/tmp")
```

只需要新增这样一行代码就可以实现，非常简单。`Static`方法的第一个参数是设置的相对路径，第二个参数是本机目录的绝对路径。

现在我们就可以通过`http://localhost:8080/static/adobegc.log`访问上一节里演示的那个`adobegc.log`的内容了，和直接访问`http://localhost:8080/adobegc.log`效果是一样的。

## 实现一个FTP服务器

上一节的例子，如果你在浏览器里访问`http://localhost:8080/static/`，你会得到404的错误，这是因为Gin做了安全措施，防止第三方恶意罗列获取你服务器上的所有文件。

但是你的需求正好是要搭建一个类似FTP的服务器，就是想把服务器上的文件件共享给其他人使用，比如下载电影等。这时候你就需要一个可以列出目录的功能了，也就是我们访问`http://localhost:8080/static/`可以看到`/tmp/`目录下的所有文件（包括文件夹），点击文件夹还可以展开看到里面的文件和文件夹，选择合适的文件进行下载，这样就是一个完整的FTP服务器了。

```
router.StaticFS("/static1", gin.Dir("/tmp", true))
```

这里我们用到了`StaticFS`方法，也是一行代码就可以搞定，为了和上个例子区分，我这里采用`/static1`作为相对路径。

这里的关键点在于`gin.Dir`函数的第二个参数，`true`代表可以列目录的意思。

## 自定义托管内容类型

以上的示例都是托管一个静态文件或者目录，我们并没有太多的自定义能力，比如设置内容类型，托管一个文件的部分内容等等。

对于这类需求，Gin为我们提供了`Data`方法来实现,以第一节中的`adobegc.log`为例。

```
router.GET("/adobegc.log", func(c *gin.Context) {
	data, err := ioutil.ReadFile("/tmp/adobegc.log")
	if err != nil {
		c.AbortWithError(500, err)
	} else {
		c.Data(200, "text/plain; charset=utf-8", data)
	}
})
```

这个例子实现的效果和上面的是一样的，不一样的是我们通过`c.Data`这个方法来实现，这个方法有三个参数

```go
func (c *Context) Data(code int, contentType string, data []byte) 
```

这就为为我们自定义提供了便利，比如可以指定`contentType`和内容`data`，这种能力很有用，比如我们可以把我们储存在数据库中的图片二进制数据，作为一张图片显示在网站上。

## 功能更强大的Reader托管。

除了可以从一个字节数组`[]byte`中读取数据显示外，Gin还为我们提供了从一个`io.Reader`中获取数据，并且提供了更强大的自定义能力,它就是`DataFromReader`方法。

```go
func (c *Context) DataFromReader(code int, contentLength int64, contentType string, reader io.Reader, extraHeaders map[string]string)
```