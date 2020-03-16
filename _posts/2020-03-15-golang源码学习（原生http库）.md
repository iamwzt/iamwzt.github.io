
## 一个简单的 web 服务 demo
首先是一个基于 go 的标准库 net/http 的 web 服务的 demo：
```go
package main

import "net/http"

func main() {
    // 方法1
    http.Handle("/", MyHandler{})
    // 方法2
    http.HandleFunc("/hello", hello)
    // 方法3
    http.Handle("/bye", http.HandlerFunc(goodbye))
    
    http.ListenAndServe(":8090", nil)
}

func goodbye(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("Goodbye."))
}

func hello(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("Hello World."))
}

type MyHandler struct{}

func (m MyHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("MyHandler ServerHTTP."))
}
```

## 如何注册 handler
上面的 demo 中用了三种看似不同的方式注册了三个 handler，但实际上用的都是 `net/http` 包中的两个函数：
```go
func Handle(pattern string, handler Handler) { 
    DefaultServeMux.Handle(pattern, handler) 
}
func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
    DefaultServeMux.HandleFunc(pattern, handler)
}
```
其中，`pattern` 是请求的路径，也就是 URL，而第二个参数略有不同，从函数的名字可以看出，一个是实现了 `Handler` 接口的类型，而另一个则是一个函数类型的参数，其函数签名是 `func(ResponseWriter, *Request)`。事实上，`Handler` 接口是这样子的：
```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

而第三种中的 `http.HandlerFunc()` 实际上就是一个适配器：
```go
type HandlerFunc func(ResponseWriter, *Request)

func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
    f(w, r)
}
```
所以三种方法本质都是一样的，都会交给 `DefaultServeMux` 来处理，而 `DefaultServeMux` 是个 `ServerMux` 类型，以 key-value 的形式保存了所有的 url-handler 映射
```go
type ServeMux struct {
    mu    sync.RWMutex
    m     map[string]muxEntry
    es    []muxEntry // slice of entries sorted from longest to shortest.
    hosts bool       // whether any patterns contain hostnames
}

type muxEntry struct {
    h       Handler
    pattern string
}
```
来看一下这个映射是怎么保存（注册）的：
```go
func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
    if handler == nil {
        panic("http: nil handler")
    }
    // 适配器那么一包啊
    mux.Handle(pattern, HandlerFunc(handler))
}
// 得，真的全都到这个方法里来了
func (mux *ServeMux) Handle(pattern string, handler Handler) {
    mux.mu.Lock()
    defer mux.mu.Unlock()

    if pattern == "" {
        panic("http: invalid pattern")
    }
    if handler == nil {
        panic("http: nil handler")
    }
    // 若有重复则 panic
    if _, exist := mux.m[pattern]; exist {
        panic("http: multiple registrations for " + pattern)
    }
    // 第一个注册时，将会建个新的 map
    if mux.m == nil {
        mux.m = make(map[string]muxEntry)
    }
    e := muxEntry{h: handler, pattern: pattern}
    mux.m[pattern] = e
    // 如果 URL 以 “/” 结束，将会按长度插入到 mux.es 这个切片中，由短到长的顺序
    if pattern[len(pattern)-1] == '/' {
        mux.es = appendSorted(mux.es, e)
    }
    // 若 URL 不以 “/” 开头
    if pattern[0] != '/' {
        mux.hosts = true
    }
}
```

OK，到现在为止，handler 的注册过程基本捋通了，还算简单。那么问题来了，`http.ListenAndServe(":8090", nil)` 开启 web 服务后，是怎么和上面注册的 handler 关联上的呢？？

## 请求是怎么到 handler 上的
如果我们从 `http.ListenAndServe` 一路点进去的话，就会发现。。。迷失了，这里稍稍过一下：
```go
func ListenAndServe(addr string, handler Handler) error {
    server := &Server{Addr: addr, Handler: handler}
    return server.ListenAndServe()
}
```
```go
func (srv *Server) ListenAndServe() error {
    if srv.shuttingDown() {
        return ErrServerClosed
    }
    addr := srv.Addr
    if addr == "" {
        addr = ":http"
    }
    ln, err := net.Listen("tcp", addr)
    if err != nil {
        return err
    }
    // 这里继续
    return srv.Serve(ln)
}
```
这个方法有点长，先只看最关键的地方：
```go
func (srv *Server) Serve(l net.Listener) error {
    // 省略...
    for {
        rw, e := l.Accept()
        // 继续省略
        c := srv.newConn(rw)
        // 省略 +10086
        go c.serve(connCtx)
    }
}
```
可以看到经典的几个步骤：accept ➡️ 建立新连接 ➡️ 开启一个 gorotine 来处理

这个 `c.server()` 方法也很长，找到这一行：
```go
serverHandler{c.server}.ServeHTTP(w, w.req)
```
```go
func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
    handler := sh.srv.Handler
    if handler == nil {
    // 这里就是上面注册了 handler 的那个 ServeMux 了
        handler = DefaultServeMux
    }
    if req.RequestURI == "*" && req.Method == "OPTIONS" {
        handler = globalOptionsHandler{}
    }
    // 这里干正事了
    handler.ServeHTTP(rw, req)
}
```

好了，到这里，算是关联上了，那么是怎么执行呢？继续往下：
```go
func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
    if r.RequestURI == "*" {
        if r.ProtoAtLeast(1, 1) {
            w.Header().Set("Connection", "close")
        }
        w.WriteHeader(StatusBadRequest)
        return
    }
    // 从map中找到匹配的handler
    h, _ := mux.Handler(r)
    h.ServeHTTP(w, r)
}
```
当然，这个里头还是有一定的道道的，这边就不展开了。
