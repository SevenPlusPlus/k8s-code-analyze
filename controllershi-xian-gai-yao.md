## Controller开发的一般模式

Controller开发的大致结构如下图所示：

![](/assets/general-pattern-of-controller.jpg)

典型的 controller 一般会有 1 个或者多个 informer，来跟踪某一类 resource，跟 APIserver 保持通讯，把最新的状态反映到本地的 cache 中。只要这些资源有变化，informal 会调用 callback。这些 callbacks 只是做一些非常简单的预处理，把不关心的的变化过滤掉，然后把关心的变更的 Object 放到 workqueue 里面。其实真正的 business logic 都是在 worker 里面， 一般 1 个 Controller 会启动很多 goroutines 跑 Workers，处理 workqueue 里的 items。它会计算用户想要达到的状态和当前的状态有多大的区别，然后通过 clients 向 APIserver 发送请求，来驱动这个集群向用户要求的状态演化。图里面蓝色的是 client-go 的原件，红色是自己写 controller 时填的代码。

