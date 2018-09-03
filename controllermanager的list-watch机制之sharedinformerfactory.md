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
type PodInformer interface {
 Informer() cache.SharedIndexInformer
 Lister() v1.PodLister
}
//实现了PodInformer接口
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

#### 继续回顾NewEndpointController方法中通过PodInformer获取pod类型的SharedIndexInformer的调用

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
				return client.CoreV1().Pods(namespace).List(options)
			},
			WatchFunc: func(options metav1.ListOptions) (watch.Interface, error) {
				return client.CoreV1().Pods(namespace).Watch(options)
			},
		},
		&corev1.Pod{},
		resyncPeriod,
		indexers,
	)
}
```
可以看到这里面定义了cache.ListWatch类型的ListFunc和WatchFunc方法类型字段，声明了Reflector机制的List-Watch的实现。ListFunc和WatchFunc的实现即通过Clientset获取相应版本资源的restClient向ApiServer发起请求。

接着我们继续NewSharedIndexInformer创建SharedIndexInformer接口对象的过程,

```
// NewSharedIndexInformer creates a new instance for the listwatcher.
func NewSharedIndexInformer(lw ListerWatcher, objType runtime.Object, defaultEventHandlerResyncPeriod time.Duration, indexers Indexers) SharedIndexInformer {
	sharedIndexInformer := &sharedIndexInformer{
		processor:                       &sharedProcessor{clock: realClock},
		indexer:                         NewIndexer(DeletionHandlingMetaNamespaceKeyFunc, indexers),
		listerWatcher:                   lw,
		objectType:                      objType,
		resyncCheckPeriod:               defaultEventHandlerResyncPeriod,
		defaultEventHandlerResyncPeriod: defaultEventHandlerResyncPeriod,
		cacheMutationDetector:           NewCacheMutationDetector(fmt.Sprintf("%T", objType)),
		clock: realClock,
	}
	return sharedIndexInformer
}
```

### type sharedIndexInformer struct
* vendor/k8s.io/client-go/tools/cache/shared_informer.go:

sharedIndexInformer结构实现了SharedIndexInformer接口，SharedIndexInformer接口继承了SharedInformer,如下代码所示：

```
// SharedInformer has a shared data cache and is capable of distributing notifications for changes
// to the cache to multiple listeners who registered via AddEventHandler.
//SharedInformer有一个共享的数据缓存，并且可以分发缓存数据变化的通知给所有注册到其上的监听者
type SharedInformer interface {
	// AddEventHandler adds an event handler to the shared informer using the shared informer's resync
	// period.  Events to a single handler are delivered sequentially, but there is no coordination
	// between different handlers.
	AddEventHandler(handler ResourceEventHandler)
	// AddEventHandlerWithResyncPeriod adds an event handler to the shared informer using the
	// specified resync period.  Events to a single handler are delivered sequentially, but there is
	// no coordination between different handlers.
	AddEventHandlerWithResyncPeriod(handler ResourceEventHandler, resyncPeriod time.Duration)
	// GetStore returns the Store.
	GetStore() Store
	// GetController gives back a synthetic interface that "votes" to start the informer
	GetController() Controller
	// Run starts the shared informer, which will be stopped when stopCh is closed.
	Run(stopCh <-chan struct{})
	// HasSynced returns true if the shared informer's store has synced.
	HasSynced() bool
	// LastSyncResourceVersion is the resource version observed when last synced with the underlying
	// store. The value returned is not synchronized with access to the underlying store and is not
	// thread-safe.
	LastSyncResourceVersion() string
}

type SharedIndexInformer interface {
	SharedInformer
	// AddIndexers add indexers to the informer before it starts.
	AddIndexers(indexers Indexers) error
	GetIndexer() Indexer
}

type sharedIndexInformer struct {
	indexer    Indexer
	controller Controller

	processor             *sharedProcessor //记录了注册的所有Controller
	cacheMutationDetector CacheMutationDetector

	// This block is tracked to handle late initialization of the controller
	listerWatcher ListerWatcher
	objectType    runtime.Object
	...
}

type sharedProcessor struct {
	listenersStarted bool
	listenersLock    sync.RWMutex
	listeners        []*processorListener
	syncingListeners []*processorListener
	clock            clock.Clock
	wg               wait.Group
}
```
再次回顾NewEndpointController方法，可以看到EndpointController通过AddEventHandler向pod类型资源的sharedIndexInformer注册了监听事件处理Handler。最后前面sharedInformerFactory的start方法提到的，sharedInformerFactory负责启动了其内部的所有sharedinformers，像下面的代码这样：
```
go informer.Run(stopCh)
```
接着来看看type sharedIndexInformer struct的AddEventHandler函数处理过程
* podInformer.AddEventHandler

```
func (s *sharedIndexInformer) AddEventHandler(handler ResourceEventHandler) {
	s.AddEventHandlerWithResyncPeriod(handler, s.defaultEventHandlerResyncPeriod)
}

func (s *sharedIndexInformer) AddEventHandlerWithResyncPeriod(handler ResourceEventHandler, resyncPeriod time.Duration) {
	...

	listener := newProcessListener(handler, resyncPeriod, determineResyncPeriod(resyncPeriod, s.resyncCheckPeriod), s.clock.Now(), initialBufferSize)

	if !s.started {
		s.processor.addListener(listener)
		return
	}

	// in order to safely join, we have to
	// 1. stop sending add/update/delete notifications
	// 2. do a list against the store
	// 3. send synthetic "Add" events to the new handler
	// 4. unblock
	s.blockDeltas.Lock()
	defer s.blockDeltas.Unlock()

	s.processor.addListener(listener)
	for _, item := range s.indexer.List() {
		listener.add(addNotification{newObj: item})
	}
}
```
可以看到其主要实现流程为：

1. 将handler(ResourceEventHandler)包装成listerner(processorListener)
2. 将listener添加到sharedIndexInformer的processor成员的listeners列表中

processorListener结构定义为：

```
type processorListener struct {
	nextCh chan interface{}
	addCh  chan interface{}

	handler ResourceEventHandler

	// pendingNotifications is an unbounded ring buffer that holds all notifications not yet distributed.
	// There is one per listener, but a failing/stalled listener will have infinite pendingNotifications
	// added until we OOM.
	// TODO: This is no worse than before, since reflectors were backed by unbounded DeltaFIFOs, but
	// we should try to do something better.
	pendingNotifications buffer.RingGrowing

	// requestedResyncPeriod is how frequently the listener wants a full resync from the shared informer
	requestedResyncPeriod time.Duration
	// resyncPeriod is how frequently the listener wants a full resync from the shared informer. This
	// value may differ from requestedResyncPeriod if the shared informer adjusts it to align with the
	// informer's overall resync check period.
	resyncPeriod time.Duration
	// nextResync is the earliest time the listener should get a full resync
	nextResync time.Time
	// resyncLock guards access to resyncPeriod and nextResync
	resyncLock sync.Mutex
}

```
### 一个shared informer run起来之后是如何运行的
在前面提到过sharedInformerFactory的start方法， 最后会调用定义在shared_informer.go的func (s *sharedIndexInformer) Run(stopCh <-chan struct{}) 来启动一个informer。
其流程如下：
* NewDeltaFIFO创建了一个type DeltaFIFO struct对象
* 构建一个controller，controller的作用就是构建一个reflector，然后将watch到的资源放入fifo这个cache里面。
* 放入之后Process: s.HandleDeltas会对资源进行处理。
* 在启动controller之前，先启动了s.processor.run(stopCh)，启动在前面已经向sharedIndexInformer注册了的各个listener。

```
func (s *sharedIndexInformer) Run(stopCh <-chan struct{}) {
	fifo := NewDeltaFIFO(MetaNamespaceKeyFunc, s.indexer)

	cfg := &Config{
		Queue:            fifo,
		ListerWatcher:    s.listerWatcher,
		ObjectType:       s.objectType,
		FullResyncPeriod: s.resyncCheckPeriod,
		RetryOnError:     false,
		ShouldResync:     s.processor.shouldResync,
		//这里定义了event的分发函数
		Process: s.HandleDeltas,
	}

	func() {
		s.startedLock.Lock()
		defer s.startedLock.Unlock()
		/*构建一个controller,其作用就是构建一个reflector，然后将watch到的资源放入fifo这个cache里面。放入之后Process: s.HandleDeltas会对资源进行处理*/
		s.controller = New(cfg)
		s.controller.(*controller).clock = s.clock
		s.started = true
	}()

	// Separate stop channel because Processor should be stopped strictly after controller
	processorStopCh := make(chan struct{})
	var wg wait.Group
	defer wg.Wait()              // Wait for Processor to stop
	defer close(processorStopCh) // Tell Processor to stop

	wg.StartWithChannel(processorStopCh, s.cacheMutationDetector.Run)
	wg.StartWithChannel(processorStopCh, s.processor.run)

	defer func() {
		s.startedLock.Lock()
		defer s.startedLock.Unlock()
		s.stopped = true // Don't want any new listeners
	}()
	s.controller.Run(stopCh)
}
```
s.processor.run(stopCh) 中包含了一个生产消费者模型。 这种模式也kubernetes中非常常见的。 通过两个groutine来构造一个生产消费者模型。

s.controller.Run(stopCh) 会完成消息的分发，把watch到的信息分发到各个listener中。
#### type Controller struct
controller的作用就是构建一个reflector，然后将watch到的资源放入fifo这个cache里面。 放入之后Process成员: s.HandleDeltas会对资源进行处理，完成消息的分发。

首先来看看Process成员: s.HandleDeltas的定义,其会在controller.run启动的processLoop()方法中对fifo队列中的资源Pop时调用。

```
func (s *sharedIndexInformer) HandleDeltas(obj interface{}) error {
	...
	// from oldest to newest
	for _, d := range obj.(Deltas) {
		switch d.Type {
		case Sync, Added, Updated:
			isSync := d.Type == Sync
			s.cacheMutationDetector.AddObject(d.Object)
			if old, exists, err := s.indexer.Get(d.Object); err == nil && exists {
				if err := s.indexer.Update(d.Object); err != nil {
					return err
				}
				s.processor.distribute(updateNotification{oldObj: old, newObj: d.Object}, isSync)
			} else {
				if err := s.indexer.Add(d.Object); err != nil {
					return err
				}
				s.processor.distribute(addNotification{newObj: d.Object}, isSync)
			}
		case Deleted:
			if err := s.indexer.Delete(d.Object); err != nil {
				return err
			}
			s.processor.distribute(deleteNotification{oldObj: d.Object}, false)
		}
	}
	return nil
}
```
主要是调用distribute函数来完成信息的分发，把消息发送给所有的listener。

```
func (p *sharedProcessor) distribute(obj interface{}, sync bool) {
	...	
	if sync {
		for _, listener := range p.syncingListeners {
			listener.add(obj)
		}
	} else {
		for _, listener := range p.listeners {
			listener.add(obj)
		}
	}
}

func (p *processorListener) add(notification interface{}) {
	p.addCh <- notification
}
```
* vendor/k8s.io/client-go/tools/cache/controller.go: type controller struct

type controller struct 是一个通用的controller框架, 体现了Reflector机制。

```
// Controller is a generic controller framework.
type controller struct {
	config         Config
	reflector      *Reflector
	reflectorMutex sync.RWMutex
	clock          clock.Clock
}

// Config contains all the settings for a Controller.
type Config struct {
	// The queue for your objects; either a FIFO or
	// a DeltaFIFO. Your Process() function should accept
	// the output of this Queue's Pop() method.
	Queue

	// Something that can list and watch your objects.
	ListerWatcher

	// Something that can process your objects.
	Process ProcessFunc

	...
}
// New makes a new Controller from the given Config.
func New(c *Config) Controller {
	ctlr := &controller{
		config: *c,
		clock:  &clock.RealClock{},
	}
	return ctlr
}

// Run begins processing items, and will continue until a value is sent down stopCh.
// Run blocks; call via go.
func (c *controller) Run(stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()
	go func() {
		<-stopCh
		c.config.Queue.Close()
	}()
	/*
		首先构建了一个reflector,这里体现了List-Watch
		从这里看出informer只是包装了reflector
	*/
	r := NewReflector(
		c.config.ListerWatcher,
		c.config.ObjectType,
		c.config.Queue, // Reflector机制的store，即前面的NewDeltaFIFO构建的type DeltaFIFO struct对象
		c.config.FullResyncPeriod,
	)
	r.ShouldResync = c.config.ShouldResync
	r.clock = c.clock

	c.reflectorMutex.Lock()
	c.reflector = r
	c.reflectorMutex.Unlock()

	var wg wait.Group
	defer wg.Wait()
	//启动reflector
	wg.StartWithChannel(stopCh, r.Run)

	/*
	启动processLoop消费event,其对应的生产者是上面的Reflector机制,
	reflector往fifo里面添加数据，而processLoop就不停地去消费这里这些数据。
	*/
	wait.Until(c.processLoop, time.Second, stopCh)
}
func (c *controller) processLoop() {
	for {
		//这里的PopProcessFunc(c.config.Process)即为前面定义的s.HandleDeltas
		obj, err := c.config.Queue.Pop(PopProcessFunc(c.config.Process))
		if err != nil {
			...
		}
	}
}
```
目前只需要知道PopProcessFunc(c.config.Process)就是上面的func (s *sharedIndexInformer) HandleDeltas(obj interface{})， 也就是说Controller完成了event的分发。

#### 启动sharedProcessor，分发event到注册的Listeners

在上面的informer启动方法中启动了s.processor.run(stopCh)，启动在前面已经向sharedIndexInformer注册了的各个listener。这里的p.listeners正是前面的func (s *sharedIndexInformer) AddEventHandler(handler ResourceEventHandler)中一个controller向shareInformer注册时添加的一个个listener。

```
func (p *sharedProcessor) run(stopCh <-chan struct{}) {
	func() {
		p.listenersLock.RLock()
		defer p.listenersLock.RUnlock()
		for _, listener := range p.listeners {
			p.wg.Start(listener.run)
			p.wg.Start(listener.pop)
		}
		p.listenersStarted = true
	}()
	<-stopCh
	p.listenersLock.RLock()
	defer p.listenersLock.RUnlock()
	for _, listener := range p.listeners {
		close(listener.addCh) // Tell .pop() to stop. .pop() will tell .run() to stop
	}
	p.wg.Wait() // Wait for all .pop() and .run() to stop
}
```

