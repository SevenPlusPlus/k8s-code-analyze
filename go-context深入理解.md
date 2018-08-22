## Golang Context深入理解
### Context应用场景
>golang 的 Context包，是专门用来简化对于处理单个请求的多个goroutine之间与请求域的数据、取消信号、截止时间等相关操作，这些操作可能涉及多个 API 调用。   

比如有一个网络请求Request，每个Request都需要开启一个goroutine做一些事情，这些goroutine又可能会开启其他的goroutine。这样的话， 我们就可以通过Context，来跟踪这些goroutine，并且通过Context来控制他们的目的，这就是Go语言为我们提供的Context，中文可以称之为“上下文”。

另外一个实际例子是，在Go服务器程序中，每个请求都会有一个goroutine去处理。然而，处理程序往往还需要创建额外的goroutine去访问后端资源，比如数据库、RPC服务等。由于这些goroutine都是在处理同一个请求，所以它们往往需要访问一些共享的资源，比如用户身份信息、认证token、请求截止时间等。而且如果请求超时或者被取消后，所有的goroutine都应该马上退出并且释放相关的资源。这种情况也需要用Context来为我们取消掉所有goroutine。

### Context一个直观的应用实例
代码示例：
```
package mainimport ( "context" "log" "net/http" _ "net/http/pprof" "time")func main() { go http.ListenAndServe(":8989", nil) ctx, cancel := context.WithCancel(context.Background()) go func() { time.Sleep(3 * time.Second) log.Println("After 3 seconds, ready to cancel") cancel() }() log.Println(A(ctx)) select {}}func C(ctx context.Context) string { select { case <-ctx.Done(): return "C Done" } return ""}func B(ctx context.Context) string { ctx, _ = context.WithCancel(ctx) go log.Println(C(ctx)) select { case <-ctx.Done(): return "B Done" } return ""}func A(ctx context.Context) string { go log.Println(B(ctx)) select { case <-ctx.Done(): return "A Done" } return ""}
```

### Context定义
