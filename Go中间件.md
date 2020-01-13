# go框架

[toc]

## base

使用中间件技术，将业务代码和非业务代码进行解耦。

先来看一段代码：

```go
package main

import "net/http"

func hello(wr http.ResponseWriter, r *http.Request){
	wr.Write([]byte("hello"))
}

func main() {
	http.HandleFunc("/",hello)
	err := http.ListenAndServe(":7777", nil)
}
```

这是一个典型的web服务，挂载了一个简单的路由。我们在线上的服务一般也是从这样简单的服务开始逐渐拓展开去的。

现在突然来了一个新的需求，我们想统计之前写的hello服务的处理耗时，需求很简单，我们对上面的程序进行少量的修改。

```go
var logger = log.New(os.Stdout, "", 0)

func hello(wr http.ResponseWriter, r *http.Request){
    timeStart := time.Now()
    wr.Write([]byte("hello"))
    timeElapsed := time.Since(timeStart)
    logger.Println(timeElapsed)
}
```

这样，在每次接到 http 请求时，打印出当前请求所消耗的时间。

完成了这个需求之后，我们继续进行业务开发，提供的 api 逐渐增加，现在我们的路由看起来是这个样子：

```go
// middleware/hello_with_more_routes.go
// 省略了一些相同的代码
package main

func helloHandler(wr http.ResponseWriter, r *http.Request) {
    ...
}

func showInfoHandler(wr http.ResponseWriter, r *http.Request) {
    ...
}

func showEmailHandler(wr http.ResponseWriter, r *http.Request) {
    ...
}

func showFriendsHandler(wr http.ResponseWriter, r *http.Request) {
    timeStart := time.Now()
    wr.Write([]byte("your friends is tom and alex"))
    timeElapsed := time.Since(timeStart)
    logger.Println(timeElapsed)
}

func main() {
    http.HandleFunc("/", helloHandler)
    http.HandleFunc("/info/show", showInfoHandler)
    http.HandleFunc("/email/show", showEmailHandler)
    http.HandleFunc("/friends/show", showFriendsHandler)
    ...
}
```

**每一个handler里都有之前提到的记录运行时间的代码，代码重复太多**。

渐渐的，我们的系统增加了30个路由和handler函数，每次增加新的handler，我们做的第一件事，就是把之前写的和业务逻辑无关的周边代码先copy过来。

然后系统平稳的运行了一段时间......突然有一天，你的boss来找你，说我们最近开发了新的监控系统，为了系统运行更加可控，要把每个接口运行的耗时数据上报到监控系统(metrics)里。现在我们需要修改代码并把耗时通过http post的方式发送给metrics。来修改一下：

```go
func helloHandler(wr http.ResponseWriter, r *http.Request) {
    timeStart := time.Now()
    wr.Write([]byte("hello"))
    timeElapsed := time.Since(timeStart)
    logger.Println(timeElapsed)
    // 新增耗时上报
    metrics.Upload("timeHandler", timeElapsed)
}
```

修改到这里，就会发现一个困境。无论未来我们对这个web系统有任何其他的非功能或统计需求，我们的修改必定会牵一发而动全身。只要增加一个非常简单的非业务统计，我们就需要去几十个 handler 里增加这些业务无关的代码。

**要有一点改动，所有的handler都需要改动**

### 使用middleware剥离非业务逻辑

我们来分析一下，一开始哪里做错了吗？我们只是一步一步的满足需求，把我们需要的逻辑按照流程写下去啊！

实际是，我们最大的错误，就是**把业务和非业务代码糅合在了一起**。对大多数场景来讲，非业务的需求都是在http请求处理前做一些事情，或者在响应完成后做一些事情。我们有没有办法使用一些重构思路，把这些公共的非业务代码剥离出去呢？回到刚开头的例子，我们需要给我们的helloHandler增加超时时间统计，我们用一种叫`function adapter`的方法来对`helloHandler`进行包装：

```go
func hello(wr http.ResponseWriter, r *http.Request) {
    wr.Write([]byte("hello"))
}

func timeMiddleware(next http.Handler) http.Handler{
	return func(wr http.ResponseWriter, r*http.Request) {
		timeStart := time.Now()

		// next handler
		next.ServeHTTP(wr, r)

		timeElapsed := time.Since(timeStart)
		logger.Println(timeElapsed)
	}
}

func main() {
    http.HandleFunc("/", timeMiddleware(hello))
    err := http.ListenAndServe(":8080", nil)
    ...
}
```

这样，就很轻松的实现了业务与非业务之间的剥离。魔法就在于这个timeMiddleware。可以从代码中看到，我们的timeMiddleware也是一个函数，其参数为http.Handler，http.Handler 的定义在 net/http 包中：

```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

任何方法实现了 `ServeHTTP()`，即是一个合法的` http.Handler`，读到这里你可能会有一些混乱，我们先来梳理一下 http 库的 `Handler`，`HandlerFunc` 和 `ServeHTTP` 的关系：

```go
type Handler interface {
        ServeHTTP(ResponseWriter, *Request)
}

type HandlerFunc func(ResponseWriter, *Request)

func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request)
     f(w, r)
}
```

实际上只要你的 handler 函数签名是：

```go
func (ResponseWriter, *Request)
```

那么这个 handler 就是一个 HandlerFunc 类型了，也就相当于实现了 http.Handler 这个接口（duck typing)。

在http库需要调用你的handler函数来处理http请求时，会调用HandlerFunc的ServerHTTP函数，可见一个请求的基本调用连是酱紫的：

```
h = getHandler() => h.ServeHttp(w, r) => h(w, r)
```

知道 handler 是怎么一回事，我们的中间件通过包装 handler，再返回一个新的 handler 就好理解了。

总结一下，我们的中间件要做的事情，就是通过一个or多个函数对handler进行包装，返回一个包含了各个中间件逻辑的函数链。我们把上面的包装做的再复杂一些。

```go
customizedHandler = logger(timeout(ratelimit(helloHandler)))
```

再直白一些，这个流程在进行请求处理的时候实际上就是不断地进行函数压栈再出栈，有一些类似于递归的执行流：

......

功能实现了，但在上面的使用过程中我们也看到了，这种函数套函数的用法不是很美观，同时也不具备什么可读性。

### 更优雅的middleware写法

上一节中解决了业务功能代码和非业务功能代码的解耦，但也提到了，看起来并不美观。如果需要修改这些函数的顺序，或者增删middleware还是有点费劲。本节我们来进行一些“写法”上的优化。

看一个例子：

```go
r = NewRouter()
r.Use(logger)
r.Use(timeout)
r.Use(ratelimit)
r.Add("/", helloHandler)
```

通过多步设置，我们拥有了和上一节差不多的执行函数链。胜在直观易懂，如果我们要增加或者删除 middleware，只要简单地增加删除对应的 Use 调用就可以了。非常方便。



## new

### 以类的形式

先看一个最简单的handler实现

```go
package main

import "net/http"

func myHandler(w http.ResponseWriter, r *http.Request){
	w.Write([]byte("Hello World"))
}

func main() {
	http.ListenAndServe(":7777", http.HandlerFunc(myHandler))
}
```

这部分代码，我们定义一个myHandler，它接受`http.ResponseWriter`和 `*http.Request`两个参数，然后向`ResponseWriter`中写入`Hello World`。在`main`函数中，我们直接使用了`ListenAndServe`方法监听本地的7777端口。

注意，**第二个参数必须为Handler类型**。而要是Handler类型，必须实现`ServeHTTP`方法。但是Go的源码中，其实已经为`HandlerFunc`实现了这个方法，所以我们只要实现了`HandlerFunc`即可——也就是**函数是`func(ResponseWriter, *Request)`类型就是Handler类型**。

```go
func ListenAndServe(addr string, handler Handler) error {}

type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}

// 定义handlerfunc类型
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```

Go虽然实现了ServeHTTP方法，但是它本身没有任何逻辑，需要我们自己来实现——也就是自己定义一个HandlerFunc函数。

可以看到，我们通过curl请求本地的8000端口，返回我们一个HelloWorld。这便是一个最简单的Handler实现了。

但是我们的目标是中间件，通过上面的方法，我们可以大致明白，`myHandler`函数应该作为最后的调用，在它之前才是中间件该作用的地方。

**我们可以实现一个逻辑，包含了这个myHandler，但是这个函数本身也必须是Handler。**

先大概阐述一下这个中间件的作用，它会拦截非预期host的请求。

```go
package main

import "net/http"

type SingleHost struct {
	handler http.Handler	// myHandler
	allowedHost string		// 允许的host
}

func(this *SingleHost)ServeHTTP(w http.ResponseWriter, r *http.Request){
	if (r.Host == this.allowedHost){
		this.handler.ServeHTTP(w, r)
	}else{
		w.WriteHeader(403)
	}
}

func myHandler(w http.ResponseWriter, r *http.Request){
	w.Write([]byte("Hello World"))
}

func main() {
	single := &SingleHost{
		handler:http.HandlerFunc(myHandler),
		allowedHost:"localhost:7777",
	}
	http.ListenAndServe(":7777", single)
}
```

### 以函数形式实现

在上面，我们实现了以类型为基础的中间件。下面是用函数为基础来实现。首先，因为我们是以函数来实现中间件，因此这个**函数返回的必须是Handler**，它会接受两个参数，一个myHandler，一个allowedHost。

```go
package main

import "net/http"

func SingleHost(handler http.Handler, allowedHost string) http.Handler {
	fn := func(w http.ResponseWriter, r *http.Request) {
		if r.Host == allowedHost {
			handler.ServeHTTP(w, r)
		} else {
			w.WriteHeader(403)
		}
	}
	return http.HandlerFunc(fn)
}

func myHandler(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("Hello World"))
}

func main() {
  single := SingleHost(http.HandlerFunc(myHandler), "localhsot:7777")
	http.ListenAndServe(":7777", single)
}
```

## Reference

- [go实现中间件](https://juejin.im/post/5a644654f265da3e393a844f)
- [middleware](https://juejin.im/post/5a644654f265da3e393a844f)

