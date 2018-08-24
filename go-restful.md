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
 


