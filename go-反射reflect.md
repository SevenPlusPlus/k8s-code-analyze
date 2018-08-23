##Golang的反射reflect深入理解和示例
---
###interface 和 反射
在讲反射之前，先来看看Golang关于类型设计的一些原则 
* 变量包括（type, value）两部分  
 * 理解这一点就知道为什么nil != nil了
* type 包括 static type和concrete type. 简单来说 static type是你在编码是看见的类型(如int、string)，concrete type是runtime系统看见的类型
* 类型断言能否成功，取决于变量的concrete type，而不是static type. 因此，一个 reader变量如果它的concrete type也实现了write方法的话，它也可以被类型断言为writer.

接下来要讲的反射，就是建立在类型之上的，Golang的指定类型的变量的类型是静态的（也就是指定int、string这些的变量，它的type是static type），在创建变量的时候就已经确定，反射主要与Golang的interface类型相关（它的type是concrete type），只有interface类型才有反射一说。

在Golang的实现中，每个interface变量都有一个对应pair，pair中记录了实际变量的值和类型: 
```
(value, type)
```
value是实际变量值，type是实际变量的类型。一个interface{}类型的变量包含了2个指针，一个指针指向值的类型【对应concrete type】，另外一个指针指向实际的值【对应value】。 
例如，创建类型为*os.File的变量，然后将其赋给一个接口变量r： 
```
tty, err := os.OpenFile("/dev/tty", os.O_RDWR, 0)
var r io.Reader
r = tty
```
接口变量r的pair中将记录如下信息：(tty, *os.File)，这个pair在接口变量的连续赋值过程中是不变的，将接口变量r赋给另一个接口变量w: 
```
var w io.Writer

w = r.(io.Writer)
```
接口变量w的pair与r的pair相同，都是:(tty, *os.File)，即使w是空接口类型，pair也是不变的。  

interface及其pair的存在，是Golang中实现反射的前提，理解了pair，就更容易理解反射。反射就是用来检测存储在接口变量内部(值value；类型concrete type) pair对的一种机制。 
###Golang的反射reflect
---
**reflect的基本功能TypeOf和ValueOf**

既然反射就是用来检测存储在接口变量内部(值value；类型concrete type) pair对的一种机制。那么在Golang的reflect反射包中有什么样的方式可以让我们直接获取到变量内部的信息呢？ 它提供了两种类型（或者说两个方法）让我们可以很容易的访问接口变量内容，分别是reflect.ValueOf() 和 reflect.TypeOf()，看看官方的解释

```
// ValueOf returns a new Value initialized to the concrete value

// stored in the interface i. ValueOf(nil) returns the zero

func ValueOf(i interface{}) Value {...}

翻译一下：ValueOf用来获取输入参数接口中的数据的值，如果接口为空则返回0

// TypeOf returns the reflection Type that represents the dynamic type of i.

// If i is a nil interface value, TypeOf returns nil.

func TypeOf(i interface{}) Type {...}

翻译一下：TypeOf用来动态获取输入参数接口中的值的类型，如果接口为空则返回nil
```

