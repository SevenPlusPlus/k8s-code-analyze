##Golang的反射reflect深入理解和示例
---
###interface 和 反射
在讲反射之前，先来看看Golang关于类型设计的一些原则 
* 变量包括（type, value）两部分  
理解这一点就知道为什么nil != nil了
* type 包括 static type和concrete type. 简单来说 static type是你在编码是看见的类型(如int、string)，concrete type是runtime系统看见的类型

* 类型断言能否成功，取决于变量的concrete type，而不是static type. 因此，一个 reader变量如果它的concrete type也实现了write方法的话，它也可以被类型断言为writer.

作者：吴德宝AllenWu链接：https://juejin.im/post/5a75a4fb5188257a82110544来源：掘金著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
