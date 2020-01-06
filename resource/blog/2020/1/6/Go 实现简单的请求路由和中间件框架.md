
go 中区分函数和方法，方法依附于对象，需要先创建对象，才能调用对象的方法；而函数是包级的，只要是公开的，那么通过包就可以访问。go 中定义新的类型有两种方式，类型别名和结构体:
```go
// 类型别名
type Integer int
type Integer1 = int

// 结构体
type User struct {
    Name string
    Age  int
}
```
此外，类型别名不仅可以用在现有类型上，也可以用在方法上:
```go
type Middleware func(handler http.Handler) http.Handler
```

#### 请求路由
在 Web 框架中，router 是必备的组件，go 的 http 标准库为我们提供了 `DefaultServeMux` 来处理简单的路由，因此用 go 起步写一个简单的 web 服务是很容易的一件事情:
```go
package main
import (
	"log"
	"net/http"
)

func init() {
	http.HandleFunc("/", func(writer http.ResponseWriter, request *http.Request) {
		// handle request
	})
}

func main() {
	log.Println("Listening on port 8080")
	if err := http.ListenAndServe(":8080", nil); err != nil {
		log.Fatal(err)
	}
}
```
如果希望使用我们自己实现的路由组件来分发请求，只需将默认的`DefaultServeMux`替换成我们自己的，http 包中定义了 `Handler` 接口来统一路由处理的入口:
```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```
自定义`Handler`可以在`ListenAndServer`方法中传入。下面的示例中 app.Router 对象所属的结构体实现了`ServeHTTP(ResponseWriter, *Request)`方法:
```go
func main() {
	log.Println("Listening on port 8080")
	if err := http.ListenAndServe(":8080", app.Router); err != nil {
		log.Fatal(err)
	}
}
```
来看下 app.Router 的实现:
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200106112526658.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FpbWVpbWVpVFM=,size_16,color_FFFFFF,t_70)
这里只是简单的用 map 来保存 URL 和 Handler 的对应关系，使用正则来进行 url 和 route 的匹配检测。在 ListenAndServer 方法中 app.Router 已经替代了默认的`DefaultServeMux`，这意味着所有的请求都会首先进入 Router 的 ServeHTTP 方法中，在该方法中再根据 URL 路径找到最终的匹配的 Handler，并把请求分发给它。

URL 和其匹配 Handler 的注册在 Router 的 Path 方法中完成。

#### 中间件
中间件主要用来分离业务代码和非业务代码，典型的需求是日志记录，请求耗时，对请求和响应进行统一处理(如压缩)等。中间件核心功能的实现在于其能在请求被最终 Handler 处理之前，以及请求被 Handler 处理之后收到通知，在一个请求生命周期的起点和终点，这两个端点上处理非业务相关的需求。

在之前请求路由的简单实现中可以看到在请求发送到最终 Handler 之前，首先到达的是 app.Router 的 ServeHTTP 方法，在该方法中会找到最终的 Handler，并调用其 ServeHTTP 方法，那么中间件需要做的就是如下的事情:
```go

func (r *Router) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	handled := false
	for route, handler := range r.mux {
		// ...
		if matched {
			handled = true
			// 中间件逻辑: 请求被最终 Handler 处理前
			handler.ServeHTTP(w, req)
			// 中间件逻辑: 请求被最终 Handler 处理后
			break
		}
	}

	if !handled {
		log.Println("ERROR: no handler find: ", req.URL.Path)
	}
}
```
上面的写法很类似 AOP 的写法，然而在 go 中，得益于其语法特性，可以有更优雅的写法:
```go
func Log(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		log.Println("mw in log start")
		next.ServeHTTP(w, r)
		log.Println("mw in log end")
	})
}
```
上面的 Log 函数即为一个中间件的具体逻辑，在 Log 方法中，入参 next 可先暂时认为是最终的 Handler，而返回值同样也是 Handler 对象，Log 方法所做的事情就是: 对 next 进行包装，加入自己的逻辑，实现中间件的功能，之后再调用正真 Handler 的 ServeHTTP 方法。

http.HandlerFunc 类型定义如下
```go
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```
可见类型 HandlerFunc 是函数 `func(ResponseWriter, *Request)` 的别名，而且实现了 ` ServeHTTP(w ResponseWriter, r *Request)`方法。也就是说 `http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {})`把 `func(w http.ResponseWriter, r *http.Request) {}`包装为一个 Handler 对象，中间件的逻辑在 HandlerFunc 中实现。

那么对于所有需要用到的中间件我们都可以用类似的方法套在最终 Handler 外面。在 Router 的 Path 方法中最终 Handler 会被实例化，我们的包装过程就在这里进行。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200106135125483.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FpbWVpbWVpVFM=,size_16,color_FFFFFF,t_70)
中间件一般不会只有一个，因此应该设计为链式调用的方式。Router 的 Use 方法用于添加新的中间件到路由组件中，而 Path 方法会把中间件倒序套在最终 Handler 外面。

使用时只需按如下的方式调用 Use 方法。
```go
func init() {
	Router.Use(middleware.Log)
	Router.Use(middleware.Cost)
}

var (
	Router = duan.NewRouter()
)
```

完整代码可以在[--这里--](https://github.com/DuanJiaNing/GoStuff/tree/master/web-middleware)找到。