## ControllerManager的List-Watch机制之sharedInformerFactory
分析ControllerManager对资源的watch-list的时候，需要注意的一点是： 一个资源是分为共享型和独占型的，两种类型的资源watch机制是不一样的。

比如说，一类是replication controller，另一类是pods。 这两类资源刚好属于两个不同的范畴，pods是许多Controller共享的，像endpoint controller也需要对pods进行watch，而replication controller是独享的。因此对他们的watch机制也不一样。

所以informer也分为两类，共享和非共享。这两类informer本质上都是对Reflector的封装。

本文首先以对pod资源的List-Watch的主线，对SharedInformer进行解析。
接上文ControllerManager启动过程，我们创建了一个SharedInformerFactory。
### type SharedInformerFactory interface
SharedInformerFactory为所有已知的API group versions中的资源提供了shared informers,其具体定义为：
* vendor/k8s.io/client-go/informers/factory.go:

```

```