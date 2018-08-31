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
// SharedInformerFactory provides shared informers for resources in all known
// API group versions.
type SharedInformerFactory interface {
	internalinterfaces.SharedInformerFactory
	ForResource(resource schema.GroupVersionResource) (GenericInformer, error)
	WaitForCacheSync(stopCh <-chan struct{}) map[reflect.Type]bool

	Admissionregistration() admissionregistration.Interface
	Apps() apps.Interface
	Autoscaling() autoscaling.Interface
	Batch() batch.Interface
	Certificates() certificates.Interface
	Coordination() coordination.Interface
	Core() core.Interface
	...
}
```
可以看到SharedInformerFactory包含了所有group资源informers访问接口，以Core group为例继续深入了解core.Interface
* vendor/k8s.io/client-go/informers/core/interface.go:

```
// Interface provides access to each of this group's versions.
type Interface interface {
	// V1 provides access to shared informers for resources in V1.
	V1() v1.Interface
}
```
可以看到包含了core所有version资源的informers访问接口，继续深入了解v1.Interface
* vendor/k8s.io/client-go/informers/core/v1/interface.go:

```
// Interface provides access to all the informers in this group version.
type Interface interface {
	// Endpoints returns a EndpointsInformer.
	Endpoints() EndpointsInformer
	// Events returns a EventInformer.
	Events() EventInformer
	// LimitRanges returns a LimitRangeInformer.
	LimitRanges() LimitRangeInformer
	// Namespaces returns a NamespaceInformer.
	Namespaces() NamespaceInformer
	// Nodes returns a NodeInformer.
	Nodes() NodeInformer
	// PersistentVolumes returns a PersistentVolumeInformer.
	PersistentVolumes() PersistentVolumeInformer
	// PersistentVolumeClaims returns a PersistentVolumeClaimInformer.
	PersistentVolumeClaims() PersistentVolumeClaimInformer
	// Pods returns a PodInformer.
	Pods() PodInformer
	...
}
```
可以看到包含了core/v1所有资源的Informer访问接口，如NodeInformer、PodInformer等

### type sharedInformerFactory struct
type sharedInformerFactory struct是type SharedInformerFactory interface的实现，具体定义如下：
* vendor/k8s.io/client-go/informers/factory.go:

```
type sharedInformerFactory struct {
	client           kubernetes.Interface
	namespace        string
	tweakListOptions internalinterfaces.TweakListOptionsFunc
	lock             sync.Mutex
	defaultResync    time.Duration
	customResync     map[reflect.Type]time.Duration

	informers map[reflect.Type]cache.SharedIndexInformer
	// startedInformers is used for tracking which informers have been started.
	// This allows Start() to be called multiple times safely.
	startedInformers map[reflect.Type]bool
}
```
下面来看看type sharedInformerFactory struct 提供的功能函数 

* 新建一个sharedInformerFactory方法

```
func NewSharedInformerFactoryWithOptions(client kubernetes.Interface, defaultResync time.Duration, options ...SharedInformerOption) SharedInformerFactory {
	factory := &sharedInformerFactory{
		client:           client,
		namespace:        v1.NamespaceAll,
		defaultResync:    defaultResync,
		informers:        make(map[reflect.Type]cache.SharedIndexInformer),
		startedInformers: make(map[reflect.Type]bool),
		customResync:     make(map[reflect.Type]time.Duration),
	}

	// Apply all options
	for _, opt := range options {
		factory = opt(factory)
	}

	return factory

```
* 初始化启动所有资源的informers

Start函数会把所有注册过的informers都分别启动一个groutine， run起来。

```
// Start initializes all requested informers.
func (f *sharedInformerFactory) Start(stopCh <-chan struct{}) {
	f.lock.Lock()
	defer f.lock.Unlock()

	for informerType, informer := range f.informers {
		if !f.startedInformers[informerType] {
			go informer.Run(stopCh)
			f.startedInformers[informerType] = true
		}
	}
}
```
关于go informer.Run(stopCh)， 是启动一个的informer，会在后面进行讲解。 

* 通过sharedInformerFactory取得具体resource的Informer

#### 回顾创建EndpointController的实现,这里的ctx.InformerFactory即为sharedInformerFactory对象

```
endpointcontroller.NewEndpointController( ctx.InformerFactory.Core().V1().Pods(), 
ctx.InformerFactory.Core().V1().Services(), 
ctx.InformerFactory.Core().V1().Endpoints(), 
ctx.ClientBuilder.ClientOrDie("endpoint-controller")
```
##### 我们跟踪下podInformer创建过程如下

```
func (f *sharedInformerFactory) Core() core.Interface {
	return core.New(f, f.namespace, f.tweakListOptions)
}
```

* 创建core group， vendor/k8s.io/client-go/informers/core/interface.go

```
type group struct {
	factory          internalinterfaces.SharedInformerFactory
	namespace        string
	tweakListOptions internalinterfaces.TweakListOptionsFunc
}
// New returns a new Interface.
func New(f internalinterfaces.SharedInformerFactory, namespace string, tweakListOptions internalinterfaces.TweakListOptionsFunc) Interface {
	return &group{factory: f, namespace: namespace, tweakListOptions: tweakListOptions}
}
```

* 创建 core group v1, vendor/k8s.io/client-go/informers/core/v1/interface.go

```
type version struct {
	factory          internalinterfaces.SharedInformerFactory
	namespace        string
	tweakListOptions internalinterfaces.TweakListOptionsFunc
}
// V1 returns a new v1.Interface.
func (g *group) V1() v1.Interface {
	return v1.New(g.factory, g.namespace, g.tweakListOptions)
}

// New returns a new Interface.
func New(f internalinterfaces.SharedInformerFactory, namespace string, tweakListOptions internalinterfaces.TweakListOptionsFunc) Interface {
	return &version{factory: f, namespace: namespace, tweakListOptions: tweakListOptions}
}

```
* 构建PodInformer vendor/k8s.io/client-go/informers/core/v1/pod.go

```
type podInformer struct {
	factory          internalinterfaces.SharedInformerFactory
	tweakListOptions internalinterfaces.TweakListOptionsFunc
	namespace        string
}

// Pods returns a PodInformer.
func (v *version) Pods() PodInformer {
	return &podInformer{factory: v.factory, namespace: v.namespace, tweakListOptions: v.tweakListOptions}

}
```

#### 继续回顾NewEndpointController方法中获取pod类型的SharedIndexInformer

```
podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		UpdateFunc: e.updatePod,
		DeleteFunc: e.deletePod,
	})
```

可以看到调用了podInformer的Informer方法返回pod类型的SharedIndexInformer

```
func (f *podInformer) Informer() cache.SharedIndexInformer {
 return f.factory.InformerFor(&corev1.Pod{}, f.defaultInformer)
}
```
其内部调用了sharedInformerFactory的InformerFor方法，注册并返回了pod类型共享的SharedIndexInformer

```
// InternalInformerFor returns the SharedIndexInformer for obj using an internal client.
func (f *sharedInformerFactory) InformerFor(obj runtime.Object, newFunc internalinterfaces.NewInformerFunc) cache.SharedIndexInformer {
	informerType := reflect.TypeOf(obj)
	//检查该资源类型的Informer是否存在，已存在则直接返回
	informer, exists := f.informers[informerType]
	if exists {
		return informer
	}

	resyncPeriod, exists := f.customResync[informerType]
	if !exists {
		resyncPeriod = f.defaultResync
	}
	//如果不存在则调用传入的Informer构造方法构造新的Informer
	informer = newFunc(f.client, resyncPeriod)
	f.informers[informerType] = informer

	return informer
}
```
这里看到sharedInformerFactory获取某种资源的Informer时先检查是否已存在，如果存在则直接返回，不存在则根据外部传入的构建方法进行构建，并注册以便后面共享利用。
最后看下pod类型的cache.SharedIndexInformer构造过程如下：

```
func (f *podInformer) defaultInformer(client kubernetes.Interface, resyncPeriod time.Duration) cache.SharedIndexInformer {
	return NewFilteredPodInformer(client, f.namespace, resyncPeriod, cache.Indexers{cache.NamespaceIndex: cache.MetaNamespaceIndexFunc}, f.tweakListOptions)
}

// NewFilteredPodInformer constructs a new informer for Pod type.
// Always prefer using an informer factory to get a shared informer instead of getting an independent
// one. This reduces memory footprint and number of connections to the server.
func NewFilteredPodInformer(client kubernetes.Interface, namespace string, resyncPeriod time.Duration, indexers cache.Indexers, tweakListOptions internalinterfaces.TweakListOptionsFunc) cache.SharedIndexInformer {
	return cache.NewSharedIndexInformer(
		&cache.ListWatch{
			ListFunc: func(options metav1.ListOptions) (runtime.Object, error) {
				if tweakListOptions != nil {
					tweakListOptions(&options)
				}
				return client.CoreV1().Pods(namespace).List(options)
			},
			WatchFunc: func(options metav1.ListOptions) (watch.Interface, error) {
				if tweakListOptions != nil {
					tweakListOptions(&options)
				}
				return client.CoreV1().Pods(namespace).Watch(options)
			},
		},
		&corev1.Pod{},
		resyncPeriod,
		indexers,
	)
}
```

