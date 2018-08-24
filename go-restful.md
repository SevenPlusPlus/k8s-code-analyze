## go-restful 框架解析和应用
### go-restful是一个基于go里面net/http构建的一个rest风格的包
### 框架整体结构
![go-restful架构](/assets/go-restful.png)
### 关键组件概念解析
#### Filter
Filter主要作用就是在请求的处理之前或者之后来进行一些额外的操作, 比如记录日志、错误处理等等, go-restful里面针对Container、webService、Route都可以加入filter对象 ， 为了串联起这些filter, go-restful里面使用了ChanFilter来保存当前路由的所有关联filter。
#### Container
Container这个概念比较迷惑人, 通常写web的时候，我们最少要进行两个操作， 写一个业务处理逻辑的handler， 然后定义一个路由吧这个路由绑定到我们的web框架上, 但很多rest框架，都需要很多自定义的逻辑处理, 这时候大家通畅会做一个抽象的实现, 比如router -> dispatch -> handler, 在路由和实际的处理方法之间加入一个dispatch的阶段, 用于自己逻辑的处理和对应请求的转发, go-restful里面的Container主要是实现了一个dispatch方法用来实现上面的chanFilter和router查找功能 。
#### WebService
在rest里面通常需要定义各种各样的资源, 比如用户、商品等, 不同的资源通常都会有一个endpoint来标识这一类比如User、Product等等, WebService其实就可以理解位一个资源的集合, 比如用户服务, 我们可以吧UserResource这类资源定义成一个web service, 所有用户的服务都在这一个wbe serice里面， 同时大家的rootpath也都是易用的/users。
#### Route
Route就比较简单了, 一个Route里面会保存它的路径、请求方法、处理函数等基本问题。
### 处理流程
![](/assets/go-restful处理流程.png)
#### 启动并监控指定端口的 http 服务
```
func ListenAndServe(addr string, handler Handler) error {
 server := &Server{Addr: addr, Handler: handler} 
 return server.ListenAndServe() 
}
```
能看出函数的入口是：Handler 接口 
```
type Handler interface {
 ServeHTTP(ResponseWriter, *Request)
 }
```
httpServer 包含 container . 
```
http.ListenAndServe(":9990", apiServer.Container)
```
一个 Container 包含多个 WebService 
```
type Container struct {
 webServicesLock sync.RWMutex
 webServices []*WebService
 ServeMux *http.ServeMux
 isRegisteredOnRoot bool
 containerFilters []FilterFunction
 doNotRecover bool // default is true 
 recoverHandleFunc RecoverHandleFunction 
 serviceErrorHandleFunc ServiceErrorHandleFunction
 router RouteSelector // default is a CurlyRouter (RouterJSR311 is a slower alternative)
 contentEncodingEnabled bool // default is false
}
```
container 实现的了Handler 接口 
```
func (c *Container) ServeHTTP(httpwriter http.ResponseWriter, httpRequest *http.Request) {
 c.ServeMux.ServeHTTP(httpwriter, httpRequest)
}
```
一个 webservice 包含多个Route 
```
type WebService struct {
 rootPath string
 pathExpr *pathExpression // cached compilation of rootPath as RegExp
 routes []Route
 produces []string 
 consumes []string
 pathParameters []*Parameter
 filters []FilterFunction
 documentation string
 apiVersion string
 typeNameHandleFunc TypeNameHandleFunction
 dynamicRoutes bool // protects 'routes' if dynamic routes are enabled 
 routesLock sync.RWMutex 
}
```
一个 Route 包含HTTP 协议协议相关的HTTP Request 、HTTP Reponse 、方法等处理 
```
type Route struct {

 Method string

 Produces []string

 Consumes []string

 Path string // webservice root path + described path

 Function RouteFunction

 Filters []FilterFunction

 If []RouteSelectionConditionFunction

 // cached values for dispatching

 relativePath string

 pathParts []string

 pathExpr *pathExpression // cached compilation of relativePath as RegExp

 // documentation

 Doc string

 Notes string

 Operation string

 ParameterDocs []*Parameter

 ResponseErrors map[int]ResponseError

 ReadSample, WriteSample interface{} // structs that model an example request or response payload

 // Extra information used to store custom information about the route.

 Metadata map[string]interface{}

 // marks a route as deprecated

 Deprecated bool

}
```






