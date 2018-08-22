## Golang Context深入理解
### Context应用场景
>golang 的 Context包，是专门用来简化对于处理单个请求的多个goroutine之间与请求域的数据、取消信号、截止时间等相关操作，这些操作可能涉及多个 API 调用。   

比如有一个网络请求Request，每个Request都需要开启一个goroutine做一些事情，这些goroutine又可能会开启其他的goroutine。这样的话， 我们就可以通过Context，来跟踪这些goroutine，并且通过Context来控制他们的目的，这就是Go语言为我们提供的Context，中文可以称之为“上下文”。

另外一个实际例子是，在Go服务器程序中，每个请求都会有一个goroutine去处理。然而，处理程序往往还需要创建额外的goroutine去访问后端资源，比如数据库、RPC服务等。由于这些goroutine都是在处理同一个请求，所以它们往往需要访问一些共享的资源，比如用户身份信息、认证token、请求截止时间等。而且如果请求超时或者被取消后，所有的goroutine都应该马上退出并且释放相关的资源。这种情况也需要用Context来为我们取消掉所有goroutine。

### Context一个直观的应用实例
代码示例：
```
package main
import (
 "context"
 "log"
 "net/http"
 _ "net/http/pprof"
 "time"
)

func main() {
 go http.ListenAndServe(":8989", nil)
 ctx, cancel := context.WithCancel(context.Background())
 go func() {
 time.Sleep(3 * time.Second)
 log.Println("After 3 seconds, ready to cancel")
 cancel()
 }()
 log.Println(A(ctx))
 select {}
}

func C(ctx context.Context) string {
 select {
 case <-ctx.Done():
 return "C Done"
 }
 return ""
}

func B(ctx context.Context) string {
 ctx, _ = context.WithCancel(ctx)
 go log.Println(C(ctx))
 select {
 case <-ctx.Done():
 return "B Done"
 }
 return ""
}

func A(ctx context.Context) string {
 go log.Println(B(ctx)) 
 select {
 case <-ctx.Done(): return "A Done"
 } 
return ""
}

```
运行结果为：
![run result](/assets/context-eg.png)

### Context定义
Context的主要数据结构是一种嵌套的结构或者说是单向的继承关系的结构，比如最初的context是一个小盒子，里面装了一些数据，之后从这个context继承下来的children就像在原本的context中又套上了一个盒子，然后里面装着一些自己的数据。或者说context是一种分层的结构，根据使用场景的不同，每一层context都具备有一些不同的特性，这种层级式的组织也使得context易于扩展，职责清晰。
context 包的核心是 struct Context，声明如下： 
```
type Context interface {

Deadline() (deadline time.Time, ok bool)

Done() <-chan struct{}

Err() error

Value(key interface{}) interface{}

}
```
可以看到Context是一个interface，在golang里面，interface是一个使用非常广泛的结构，它可以接纳任何类型。Context定义很简单，一共4个方法，我们需要能够很好的理解这几个方法: 
 
1. Deadline方法是获取设置的截止时间的意思，第一个返回式是截止时间，到了这个时间点，Context会自动发起取消请求；第二个返回值ok==false时表示没有设置截止时间，如果需要取消的话，需要调用取消函数进行取消。
2. Done方法返回一个只读的chan，类型为struct{}，我们在goroutine中，如果该方法返回的chan可以读取，则意味着parent context已经发起了取消请求，我们通过Done方法收到这个信号后，就应该做清理操作，然后退出goroutine，释放资源。之后，Err 方法会返回一个错误，告知为什么 Context 被取消。
3. Err方法返回取消的错误原因，因为什么Context被取消。
4. Value方法获取该Context上绑定的值，是一个键值对，所以要通过一个Key才可以获取对应的值，这个值一般是线程安全的。
### Context的实现方法
Context 虽然是个接口，但是并不需要使用方实现，golang内置的context 包，已经帮我们实现了2个方法，一般在代码中，开始上下文的时候都是以这两个作为最顶层的parent context，然后再衍生出子context。这些 Context 对象形成一棵树：当一个 Context 对象被取消时，继承自它的所有 Context 都会被取消。两个实现如下：  
```
var (

 background = new(emptyCtx)

 todo = new(emptyCtx)

)

func Background() Context {

 return background

}

func TODO() Context {

 return todo

}
```



