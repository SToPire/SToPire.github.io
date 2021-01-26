---
title: Golang net/http流程
date: 2021-01-26 19:32:42
categories: Tips
tags: [Golang]
---

参考资料：https://juejin.cn/post/6844903998869209095
<!-- more -->

### 简单的http  server

```go
func myHandlerFunc(w http.ResponseWriter, r *http.Request){
	fmt.Println("myHandlerFunc")
	w.Write([]byte("From myHandlerFunc"))
}

type myHandler struct {

}

func (handler *myHandler) ServeHTTP(w http.ResponseWriter, r *http.Request){
	fmt.Println("myHandler")
	w.Write([]byte("From myHandler"))
}

func main(){
	http.HandleFunc("/one",myHandlerFunc)
	http.Handle("/two",&myHandler{})
	http.ListenAndServe(":9090",nil)
}
```

注意到注册路由的两种方式，即`http.HandleFunc`和`http.Handle`

非成员方法会默认以`DefaultServeMux`为receiver，可见这两种方式最终都会调用`DefaultServeMux.Handle`.

```go
// server.go
// HandleFunc registers the handler function for the given pattern.
func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	if handler == nil {
		panic("http: nil handler")
	}
	mux.Handle(pattern, HandlerFunc(handler))
}

// Handle registers the handler for the given pattern
// in the DefaultServeMux.
// The documentation for ServeMux explains how patterns are matched.
func Handle(pattern string, handler Handler) { DefaultServeMux.Handle(pattern, handler) }

// HandleFunc registers the handler function for the given pattern
// in the DefaultServeMux.
// The documentation for ServeMux explains how patterns are matched.
func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	DefaultServeMux.HandleFunc(pattern, handler)
}
```

### ServeMux

`ServeMux`是HTTP request的multiplexer.所谓multiplexer，就是Server会处理对于多个不同url的请求，应该根据最长url匹配的原则把请求转发给指定的handler.`DefaultServeMux`是一个`ServerMux`的实例，在上例中调用全局的`Handle` `HandleFunc`等函数时，实际是在配置`DefaultServeMux`。监听时`ListenAndServe`的第二个参数留空，同样也是默认使用`DefaultServeMux`.

```GO
type ServeMux struct {
	mu    sync.RWMutex // 互斥锁
	m     map[string]muxEntry
	es    []muxEntry // slice of entries sorted from longest to shortest.
	hosts bool       // whether any patterns contain hostnames
}

type muxEntry struct {
	h       Handler
	pattern string
}
```

`ServerMux`中的`mu`字段维护了一个从pattern(路由表达式)到`muxEntry`的映射。`muxEntry`简单的把路由表达式和handler组合在一起。

### Handler

Handler是一个接口，只有一个方法`ServeHTTP()`.一个`Handler`(以及它的唯一方法)的职责是处理Request并构建Response.这也是golang的http服务暴露给开发者的入口，许多web框架自定义实现了`Handler`.

值得注意的是，`ServeMux`也实现了`Handler`接口，它将请求转发给最匹配的url所对应的handler。也就是说，我们可以使用`ServeMux`对象或者干脆使用`DefaultServeMux`来作为`Handler`.

```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

#### 小Tip

回到一开始`ServeMux`的`HandleFunc`成员函数：

```go
func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	if handler == nil {
		panic("http: nil handler")
	}
	mux.Handle(pattern, HandlerFunc(handler))
}
```

注意第5行的`HandlerFunc(handler)`，它的作用是把一个具有`func(Response Writer, *Request)`签名的函数转变为实现了`Handler`接口的对象。

```go
// The HandlerFunc type is an adapter to allow the use of
// ordinary functions as HTTP handlers. If f is a function
// with the appropriate signature, HandlerFunc(f) is a
// Handler that calls f.
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```

通过一层wrapper优雅地实现了这一点，上述的`HandlerFunc(handler)`不是函数调用，而是一个类型转换。

### 回到`DefaultServeMux.Handle`

```go
func (mux *ServeMux) Handle(pattern string, handler Handler) {
	mux.mu.Lock()
	defer mux.mu.Unlock()

	if pattern == "" {
		panic("http: invalid pattern")
	}
	if handler == nil {
		panic("http: nil handler")
	}
	if _, exist := mux.m[pattern]; exist {
		panic("http: multiple registrations for " + pattern)
	}

	if mux.m == nil {
		mux.m = make(map[string]muxEntry)
	}
	e := muxEntry{h: handler, pattern: pattern}
	mux.m[pattern] = e
	if pattern[len(pattern)-1] == '/' {
		mux.es = appendSorted(mux.es, e)
	}

	if pattern[0] != '/' {
		mux.hosts = true
	}
}
```

这个函数主要做了以下工作：

1. 建立从url模式到对应handler的映射。
2. 如果模式以'/'结尾，对应一个子树中所有的url，把它加入es列表中，按照最长有限匹配的原则从长到短排序。
3. 不以'/'开头的模式表示只匹配某个主机名下的URL.

### `ListenAndServe`

`http`的`ListenAndSere`方法创建了一个`Server`对象并调用了其同名方法。`Server`的`ListenAndServe`会初始化监听地址，调用`Listen`方法设置监听，把`Listen`返回的监听器对象传入`Serve`方法。`Serve`方法对每一个连接创建`conn`即连接对象，每个连接对象调用自己的`serve`方法，该方法中实际处理请求的语句是`serverHandler{c.server}.ServeHTTP(w, w.req)`.

```go
// serverHandler delegates to either the server's Handler or
// DefaultServeMux and also handles "OPTIONS *" requests.
type serverHandler struct {
	srv *Server
}

func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
	handler := sh.srv.Handler
	if handler == nil {
		handler = DefaultServeMux
	}
	if req.RequestURI == "*" && req.Method == "OPTIONS" {
		handler = globalOptionsHandler{}
	}
	handler.ServeHTTP(rw, req)
}
```

`serverHandler`开启了一系列调用流程，并最终调用了用户在`ListenAndServe`中指定的`handler`，如果为`nil`则会使用`DefaultServeMux`.

### Best Practice

我们可以看出，为了把url模式绑定到指定的handler，我们有多种方式：

1. 直接使用`http.HandleFunc`绑定，`ListenAndServe`时使用`nil`作为`handler`.这是使用了默认的`DefaultServeMux`.

2. 使用`HandlerFunc`（注意之前提及的，这是一个强制类型转换的adaptor）把一个函数转化为`Handler`对象并用于`ListenAndServe`.

   ```go
   func foo(w http.ResponseWriter, r *http.Request){
   	//balabala
   }
   
   handler := http.HandlerFunc(foo)
   http.ListenAndServe(":9090",handler)
   ```

3. 创建一个自定义的`ServeMux`，在里面注册handler.

   ```go
   mux := http.NewServeMux()
   mux.HandleFunc("/one", HandlerOne)
   mux.HandleFunc("/two", HandlerTwo)
   http.ListenAndServe(":8001", mux)
   ```

**建议使用第三种方式。**
