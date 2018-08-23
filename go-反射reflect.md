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
reflect.TypeOf()是获取pair中的type，reflect.ValueOf()获取pair中的value，示例如下： 
```
package main

import (

 "fmt"

 "reflect"

)

func main() {

 var num float64 = 1.2345

 fmt.Println("type: ", reflect.TypeOf(num))

 fmt.Println("value: ", reflect.ValueOf(num))

}

运行结果:

type: float64

value: 1.2345
```
**说明**

1. reflect.TypeOf： 直接给到了我们想要的type类型，如float64、int、各种pointer、struct 等等真实的类型
2. reflect.ValueOf：直接给到了我们想要的具体的值，如1.2345这个具体数值，或者类似&{1 "Allen.Wu" 25} 这样的结构体struct的值
3. 也就是说明反射可以将“接口类型变量”转换为“反射类型对象”，反射类型指的是reflect.Type和reflect.Value这两种

####从relfect.Value中获取接口interface的信息
当执行reflect.ValueOf(interface)之后，就得到了一个类型为”relfect.Value”变量，可以通过它本身的Interface()方法获得接口变量的真实内容，然后可以通过类型判断进行转换，转换为原有真实类型。不过，我们可能是已知原有类型，也有可能是未知原有类型，因此，下面分两种情况进行说明。

#####已知原有类型【进行“强制转换”】

已知类型后转换为其对应的类型的做法如下，直接通过Interface方法然后强制转换，如下： 
```
realValue := value.Interface().(已知的类型)
```
示例如下:
```
package main

import (

 "fmt"

 "reflect"

)

func main() {

 var num float64 = 1.2345

 pointer := reflect.ValueOf(&num)

 value := reflect.ValueOf(num)

 // 可以理解为“强制转换”，但是需要注意的时候，转换的时候，如果转换的类型不完全符合，则直接panic

 // Golang 对类型要求非常严格，类型一定要完全符合

 // 如下两个，一个是*float64，一个是float64，如果弄混，则会panic

 convertPointer := pointer.Interface().(*float64)

 convertValue := value.Interface().(float64)

 fmt.Println(convertPointer)

 fmt.Println(convertValue)

}

运行结果：

0xc42000e238

1.2345
```
**说明**

* 转换的时候，如果转换的类型不完全符合，则直接panic，类型要求非常严格！
* 转换的时候，要区分是指针还是值
* 也就是说反射可以将“反射类型对象”再重新转换为“接口类型变量”

#####未知原有类型【遍历探测其Filed】

很多情况下，我们可能并不知道其具体类型，那么这个时候，该如何做呢？需要我们进行遍历探测其Filed来得知，示例如下:
```
package main

import (

 "fmt"

 "reflect"

)

type User struct {

 Id int

 Name string

 Age int

}

func (u User) ReflectCallFunc() {

 fmt.Println("Allen.Wu ReflectCallFunc")

}

func main() {

 user := User{1, "Allen.Wu", 25}

 DoFiledAndMethod(user)

}

// 通过接口来获取任意参数，然后一一揭晓

func DoFiledAndMethod(input interface{}) {

 getType := reflect.TypeOf(input)

 fmt.Println("get Type is :", getType.Name())

 getValue := reflect.ValueOf(input)

 fmt.Println("get all Fields is:", getValue)

 // 获取方法字段

 // 1. 先获取interface的reflect.Type，然后通过NumField进行遍历

 // 2. 再通过reflect.Type的Field获取其Field

 // 3. 最后通过Field的Interface()得到对应的value

 for i := 0; i < getType.NumField(); i++ {

 field := getType.Field(i)

 value := getValue.Field(i).Interface()

 fmt.Printf("%s: %v = %v\n", field.Name, field.Type, value)

 }

 // 获取方法

 // 1. 先获取interface的reflect.Type，然后通过.NumMethod进行遍历

 for i := 0; i < getType.NumMethod(); i++ {

 m := getType.Method(i)

 fmt.Printf("%s: %v\n", m.Name, m.Type)

 }

}

运行结果：

get Type is : User

get all Fields is: {1 Allen.Wu 25}

Id: int = 1

Name: string = Allen.Wu

Age: int = 25

ReflectCallFunc: func(main.User)
```





