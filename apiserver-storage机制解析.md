// cacherListerWatcher opaques storage.Interface to expose cache.ListerWatcher.
type cacherListerWatcher struct {
	storage        storage.Interface
	resourcePrefix string
	newListFunc    func() runtime.Object
}##ApiServer后端存储机制接口实现
---
主要代码目录为: k8s.io/apiserver/pkg/storage
###存储后端(storagebackend/)
后端存储配置主要支持etcd2/etcd3, 后端存储配置定义为：
```
// Config is configuration for creating a storage backend.
type Config struct {
 //后端存储类型 e.g. "etcd2", etcd3". Default ("") is "etcd3".
 Type string
 // Prefix is the prefix to all keys passed to storage.Interface methods.
 Prefix string
 // 存储后端服务地址列表
 ServerList []string
 // 后端TLS认证相关配置
 KeyFile string
 CertFile string
 CAFile string
 Quorum bool
 Paging bool
 Codec runtime.Codec // Transformer 用于键值持久化前的转换
 Transformer value.Transformer
}
```
####存储后端创建工厂方法实现(storagebackend/factory/)
根据后端存储类型创建对应的后端存储健康检查方法实现, 传入后端存储配置Config，返回后端存储健康检查方法实现func()error, 关键实现代码为：

```

func CreateHealthCheck(c storagebackend.Config) (func() error, error) {
 switch c.Type {
 case storagebackend.StorageTypeETCD2:
   return newETCD2HealthCheck(c)
 case storagebackend.StorageTypeUnset, storagebackend.StorageTypeETCD3:
   return newETCD3HealthCheck(c)
 default:
   return nil, fmt.Errorf("unknown storage type: %s", c.Type)
 }

}

```

根据后端存储配置类型调用对应的存储接口实现构建方法，传入参数为后端存储配置Config，返回存储接口对象storage.Interface, 实现代码如下:
```
func Create(c storagebackend.Config) (storage.Interface, DestroyFunc, error) {
 switch c.Type {
 case storagebackend.StorageTypeETCD2:
 return newETCD2Storage(c)
 case storagebackend.StorageTypeUnset, storagebackend.StorageTypeETCD3:
 return newETCD3Storage(c)
 default:
 return nil, nil, fmt.Errorf("unknown storage type: %s", c.Type)
 }
}
```
我们接着以ETCD3后端存储创建实现为例分析newETCD3Storage方法，传入后端存储配置信息，返回创建的后端存储接口对象，以及存储对象销毁方法，关键实现代码如下：
```
func newETCD3Storage(c storagebackend.Config) (storage.Interface, DestroyFunc, error) {
 //构建Etcd3客户端访问对象
 client, err := newETCD3Client(c)
 //创建ETCd3客户端的Context对象
 ctx, cancel := context.WithCancel(context.Background())
 //构造Etcd3客户端的销毁方法
 destroyFunc := func() {
 cancel()
 client.Close()
 }
 transformer := c.Transformer
//返回Etcd3存储接口实现
 return etcd3.NewWithNoQuorumRead(client, c.Codec, c.Prefix, transformer, c.Paging), destroyFunc, nil
}
```
通过etcd/clientv3创建ETCD3访问客户端对象，实现代码如下：
```
func newETCD3Client(c storagebackend.Config) (*clientv3.Client, error) {
 tlsInfo := transport.TLSInfo{
 CertFile: c.CertFile,
 KeyFile: c.KeyFile,
 CAFile: c.CAFile,
 }
 tlsConfig, err := tlsInfo.ClientConfig()
//构建clientv3配置对象
 cfg := clientv3.Config{
 DialTimeout: dialTimeout,
 DialKeepAliveTime: keepaliveTime,
 DialKeepAliveTimeout: keepaliveTimeout,
 DialOptions: []grpc.DialOption{
 grpc.WithUnaryInterceptor(grpcprom.UnaryClientInterceptor),
 grpc.WithStreamInterceptor(grpcprom.StreamClientInterceptor),
 },
 Endpoints: c.ServerList,
 TLS: tlsConfig,
 }
 //创建clientv3对象
 client, err := clientv3.New(cfg)
 return client, err
}
```
下一节继续深入分析ETCD3存储接口对象的相关实现
###ETCD3 存储接口对象实现(etcd3/)
接上文etcd3.NewWithNoQuorumRead来创建基于etcd3的后端存储接口实现，本质上为构建了一个etcd3.store对象，而该对象实现了storage.Interface存储接口，实现代码如下：
```
func NewWithNoQuorumRead(c *clientv3.Client, codec runtime.Codec, prefix string, transformer value.Transformer, pagingEnabled bool) storage.Interface {
 return newStore(c, false, pagingEnabled, codec, prefix, transformer)
}
func newStore(c *clientv3.Client, quorumRead, pagingEnabled bool, codec runtime.Codec, prefix string, transformer value.Transformer) *store {
 versioner := etcd.APIObjectVersioner{}
 result := &store{ client: c, codec: codec, versioner: versioner, transformer: transformer, pagingEnabled: pagingEnabled, pathPrefix: path.Join("/", prefix), watcher: newWatcher(c, codec, versioner, transformer), leaseManager: newDefaultLeaseManager(c), 
} 
return result
}
```
我们重点了解下store对象结构及其关键接口方法实现，store结构实现如下:
```
type store struct {
 //etcd3访问客户端
 client *clientv3.Client
 // getOpts contains additional options that should be passed
 // to all Get() calls.
 getOps []clientv3.OpOption
 //数据编解码接口对象
 codec runtime.Codec
 //获取或设置对象元数据字段的接口对象
 versioner storage.Versioner
 transformer value.Transformer
 //前缀路径
 pathPrefix string
 //用于创建watch接口对象(watchChan)
 watcher *watcher
 pagingEnabled bool
 //管理etcd请求的leases
 leaseManager *leaseManager
}
```
store对象实现了storage.Interface接口，首先解析一下存储接口中Get方法的实现过程,根据对象key从ETCD存储中获取并解析得到API对象实例：
```
func (s *store) Get(ctx context.Context, key string, resourceVersion string, out runtime.Object, ignoreNotFound bool) error {
 //得到对象存储path
 key = path.Join(s.pathPrefix, key)
 //调用etcd client获取对象path，获取存储数据返回包
 getResp, err := s.client.KV.Get(ctx, key, s.getOps...)
 //对象键值对不存在情况的处理
 if len(getResp.Kvs) == 0 {
 if ignoreNotFound {
 return runtime.SetZeroValue(out)
 }
 return storage.NewKeyNotFoundError(key, 0)
 }
 //获取对象键值对
 kv := getResp.Kvs[0] 
 //对象值解码前数据转换
 data, _, err := s.transformer.TransformFromStorage(kv.Value, authenticatedDataString(key)) 
 //对象值解码到输出API对象中
 return decode(s.codec, s.versioner, data, out, kv.ModRevision)
}

func decode(codec runtime.Codec, versioner storage.Versioner, value []byte, objPtr runtime.Object, rev int64) error {
 if _, err := conversion.EnforcePtr(objPtr); err != nil { panic("unable to convert output object to pointer") }
 //对象值解码
 _, _, err := codec.Decode(value, nil, objPtr)
 // being unable to set the version does not prevent the object from being extracted
 versioner.UpdateObject(objPtr, uint64(rev))
 return nil
}
```
store对storage.Interface的Create方法实现解析如下：
```
func (s *store) Create(ctx context.Context, key string, obj, out runtime.Object, ttl uint64) error {
 //为API存储对象的SelfLink、ResourceVersion设置空值
 if err := s.versioner.PrepareObjectForStorage(obj); err != nil {
 return fmt.Errorf("PrepareObjectForStorage failed: %v", err)
 }
 //将要存储的API对象编码为字节数组
 data, err := runtime.Encode(s.codec, obj)
 //生成API对象存储路径
 key = path.Join(s.pathPrefix, key) 
 //设置ttl
 opts, err := s.ttlOpts(ctx, int64(ttl))
 //对象值存储前数据加密转换操作
 newData, err := s.transformer.TransformToStorage(data, authenticatedDataString(key)) 
 //API对象键值对kv存储提交
 txnResp, err := s.client.KV.Txn(ctx).If( notFound(key), ).Then( clientv3.OpPut(key, string(newData), opts...), ).Commit()
 if !txnResp.Succeeded {
 return storage.NewKeyExistsError(key, 0)
 } if out != nil {
 //返回存储后的API对象（含存储的ResourceVersion信息）
 putResp := txnResp.Responses[0].GetResponsePut()
 return decode(s.codec, s.versioner, data, out, putResp.Header.Revision)
 }
 return nil
}
```
store对storage.Interface的Watch方法实现解析如下：
```
func (s *store) watch(ctx context.Context, key string, rv string, pred storage.SelectionPredicate, recursive bool) (watch.Interface, error) { rev, err := s.versioner.ParseResourceVersion(rv) if err != nil { return nil, err }
 //生成对象存储path key = path.Join(s.pathPrefix, key) return s.watcher.Watch(ctx, key, int64(rev), recursive, pred)}
```
接下来重点解析存储watch过程的实现，store结构中包含一个watcher对象实现watch方法，watcher对象的Watch方法实现中会创建一个watchChan对象并调用其run方法启动watch goroutine，而该watchChan对象实现了watch.Interface接口作为结果返回。
watcher结构描述如下：
```
type watcher struct {
 //ETCD v3客户端对象
 client *clientv3.Client
 //API对象编解码接口对象
 codec runtime.Codec
 versioner storage.Versioner
 transformer value.Transformer
}
```
watch.Interface接口用于watch和report存储的变更：
```
type Interface interface { 
// Stops watching. Will close the channel returned by ResultChan(). Releases // any resources used by the watch.
 Stop()
 // 返回一个channel用户接收所有变更事件. If an error occurs // or Stop() is called, this channel will be closed, in which case the // watch should be completely cleaned up.
 ResultChan() <-chan Event
}
```
watchChan结构实现了watch Interface接口，其结构描述如下：
```
type watchChan struct {
 //创建自己的外部watcher对象引用
 watcher *watcher
 //watch的API对象的path
 key string
 //watch的初始版本信息
 initialRev int64
 //是否watch其子节点列表
 recursive bool
 internalPred storage.SelectionPredicate
 //通过外部watch调用传入的Context参数WithCancel得到当前watch子routine的Context及其Cancel方法实现
 ctx context.Context
 cancel context.CancelFunc
 //通过clientv3客户端watch到的ETCD节点变更event暂时存于该channel，由precessEvent routine消费处理
 incomingEventChan chan *event
 //用于返回从存储中watch得到的对象变更事件channel
 resultChan chan watch.Event
 //通过clientv3客户端watch到错误事件暂存于该channel
 errChan chan error
}
```
watchChan调用run开始对对象存储节点的watch，其启动步骤如下：
```
func (wc *watchChan) run() {
 //创建channel用于接收底层客户端watch操作结束事件
  watchClosedCh := make(chan struct{})
 //调用底层etcd客户端开始watch节点变更事件
 go wc.startWatching(watchClosedCh)

 var resultChanWG sync.WaitGroup
 resultChanWG.Add(1)
 //处理后端etcd存储收到的变更事件并发送结果到resultChan
 go wc.processEvent(&resultChanWG)
 select {
 case err := <-wc.errChan: //收到后端错误事件
 	if err == context.Canceled {
 	break
 	}
 	errResult := transformErrorToEvent(err)
 	if errResult != nil {
 	// error result is guaranteed to be received by user before closing 	ResultChan.
 		select {
 		case wc.resultChan <- *errResult:
 		case <-wc.ctx.Done(): // user has given up all results
 		}
 	}
 case <-watchClosedCh: //底层客户端watch操作结束
 case <-wc.ctx.Done(): // 用户取消操作
 }
 // We use wc.ctx to reap all goroutines. Under whatever condition, we should stop them all.
 // It's fine to double cancel.
 wc.cancel()
 // we need to wait until resultChan wouldn't be used anymore
 resultChanWG.Wait()
 //watch结束前关闭对外发布变更的resultChan
 close(wc.resultChan)
}
```
接着分析下调用底层etcd客户端真正开启节点watch的过程wc.startWatching：
```
func (wc *watchChan) startWatching(watchClosedCh chan struct{}) {
 //准备clientv3调用watch操作选项options
 opts := []clientv3.OpOption{clientv3.WithRev(wc.initialRev + 1), clientv3.WithPrevKV()}
 if wc.recursive {
 opts = append(opts, clientv3.WithPrefix())
 }
 //调用etcd clientv3的watch方法，返回其内部的WatchChan即（<-chan WatchResponse）
 wch := wc.watcher.client.Watch(wc.ctx, wc.key, opts...)
 //循环处理watch到的回包
 for wres := range wch {
        //收到错误回包，则发送错误事件到内部的errChan
 	if wres.Err() != nil {
 		err := wres.Err()
 		// If there is an error on server (e.g. compaction), the channel will return it before closed. 
               wc.sendError(err)
 		return
 	}
	//收到正常的变更事件包，则发送到内部的incomingEventChan以便后续processEvent routine去消费
 	for _, e := range wres.Events {
 		wc.sendEvent(parseEvent(e))
 	}
 }
 // When we come to this point, it's only possible that client side ends the watch.
 // e.g. cancel the context, close the client.
 // If this watch chan is broken and context isn't cancelled, other goroutines will still hang.
 // We should notify the main thread that this goroutine has exited.
 //关闭watchClosedCh，通知外部routine watch操作结束
 close(watchClosedCh)
}
```

前面解析了基于ETCD V3的Storage实现，下面我们接着上一节遗留问题，解析StorageWithCache的实现。
### StorageWithCache
* vendor/k8s.io/apiserver/pkg/registry/generic/registry/storage_factory.go:
```
// Creates a cacher based given storageConfig.
func StorageWithCacher(capacity int) generic.StorageDecorator {
   cacherConfig := cacherstorage.Config{
			CacheCapacity:        capacity,
			Storage:              s,
			Versioner:            etcdstorage.APIObjectVersioner{},
			Type:                 objectType,
			ResourcePrefix:       resourcePrefix,
			KeyFunc:              keyFunc,
			NewListFunc:          newListFunc,
			GetAttrsFunc:         getAttrsFunc,
			TriggerPublisherFunc: triggerFunc,
			Codec:                storageConfig.Codec,
		}
		cacher := cacherstorage.NewCacherFromConfig(cacherConfig)
    ...
```
### Cacher
* vendor/k8s.io/apiserver/pkg/storage/cacher/cacher.go:

Cacher负责从其内部缓存提供给定资源的WATCH和LIST请求，并根据底层存储内容在后台更新其缓存。 Cacher实现storage.Interface（虽然大部分的调用只是委托给底层的存储）。 Cacher的核心是reflector机制。

Cacher接口必然也实现了storage.Interface接口所需要的方法。 因为该Cacher只用于WATCH和LIST的request，所以可以看下cacher提供的API,除了WATCH和LIST相关的之外的接口都是调用了之前创建的storage的API。

Cacher的四个重要的成员：storage、watchCache、reflector、watchers。

* storage，数据源（可以简单理解为etcd、带cache的etcd），一个资源的etcd handler

* watchCache，用来存储apiserver从etcd那里watch到的对象

* reflector，包含两个重要的数据成员listerWatcher和watchCache。reflector的工作主要是将watch到的config.Type类型的对象存放到watcherCache中。

* watchers， 当kubelet、kube-scheduler需要watch某类资源时，他们会向kube-apiserver发起watch请求，kube-apiserver就会生成一个cacheWatcher，他们负责将watch的资源通过http从apiserver传递到kubelet、kube-scheduler这些订阅方。watcher是kube-apiserver watch的发布方和订阅方的枢纽。

```
type Cacher struct {

	// Incoming events that should be dispatched to watchers.
	/** Incoming events 会被分发到watchers **/
	incoming chan watchCacheEvent

	// Underlying storage.Interface.
	storage Interface

	// Expected type of objects in the underlying cache.
	objectType reflect.Type

	// "sliding window" of recent changes of objects and the current state.
	watchCache *watchCache
	reflector  *cache.Reflector
	// watchers is mapping from the value of trigger function that a
	// watcher is interested into the watchers

	watcherIdx int
	watchers   indexedWatchers
        ...
}
```

#### NewCacherFromConfig 
```
// NewCacherFromConfig creates a new Cacher responsible for servicing WATCH and LIST requests from
// its internal cache and updating its cache in the background based on the
// given configuration.
func NewCacherFromConfig(config Config) *Cacher {
  //新建watchCache，用来存储apiserver从etcd那里watch到的对象
  watchCache := newWatchCache(config.CacheCapacity, config.KeyFunc, config.GetAttrsFunc, config.Versioner)
  //新建cacherListerWatcher将内部storage暴露为cache.ListerWacher的接口实现
  listerWatcher := newCacherListerWatcher(config.Storage, config.ResourcePrefix, config.NewListFunc)

  //新建Cacher实现，四个重要成员: storage、watchCache、reflector、watchers
  cacher := &Cacher{
		storage:     config.Storage,
		objectType:  reflect.TypeOf(config.Type),
		watchCache:  watchCache,
      //reflector主要工作是将watch到的config.Type类型的对象存放到watchCache中
		reflector:   cache.NewNamedReflector(reflectorName, listerWatcher, config.Type, watchCache, 0),

//allWatchers、valueWatchers 都是一个map，map的值类型为cacheWatcher，
//当kubelet、kube-scheduler需要watch某类资源时， 他们会向kube-apiserver发起watch请求，
//kube-apiserver就会生成一个cacheWatcher， 他们负责将watch的资源通过http从apiserver传递到kubelet、kube-scheduler
// ==>event分发功能是在下面的 go cacher.dispatchEvents()中完成

		watchers: indexedWatchers{
			allWatchers:   make(map[int]*cacheWatcher),
			valueWatchers: make(map[string]watchersMap),
		},
		...
	}

/* 完成event分发功能，把event分发到对应的watchers中。 是incoming chan watchCacheEvent的消费者 */
   go cacher.dispatchEvents()

   go func() {
		defer cacher.stopWg.Done()
		wait.Until(
			func() {
				if !cacher.isStopped() {
					cacher.startCaching(stopCh)
				}
			}, time.Second, stopCh,
		)
	}()
   return cacher
}
```
#### watchCache
* vendor/k8s.io/apiserver/pkg/storage/cacher/watch_cache.go:
watchCache实现了cache.Store(vendor/k8s.io/client-go/tools/cache/store.go)接口，其内部包含两个重要的成员：cache和store
* cache中存储的是event(watchCacheElement/watchCacheEvent)(add\delete\update)
* store存储的是资源对象

```
type watchCache struct {
	sync.RWMutex
	cond *sync.Cond
	// Maximum size of history window.
	capacity int
	// keyFunc is used to get a key in the underlying storage for a given object.
	keyFunc func(runtime.Object) (string, error)
	// getAttrsFunc is used to get labels and fields of an object.
	getAttrsFunc func(runtime.Object) (labels.Set, fields.Set, bool, error)

	// cache is used a cyclic buffer - its first element (with the smallest
	// resourceVersion) is defined by startIndex, its last element is defined
	// by endIndex (if cache is full it will be startIndex + capacity).
	// Both startIndex and endIndex can be greater than buffer capacity -
	// you should always apply modulo capacity to get an index in cache array.
	cache      []watchCacheElement
	startIndex int
	endIndex   int

	// store will effectively support LIST operation from the "end of cache
	// history" i.e. from the moment just after the newest cached watched event.
	// It is necessary to effectively allow clients to start watching at now.
	// NOTE: We assume that <store> is thread-safe.
	store cache.Store

	// ResourceVersion up to which the watchCache is propagated.
	resourceVersion uint64

	// This handler is run at the end of every successful Replace() method.
	onReplace func()

	// This handler is run at the end of every Add/Update/Delete method
	// and additionally gets the previous value of the object.
	onEvent func(*watchCacheEvent)

	// for testing timeouts.
	clock clock.Clock

	// An underlying storage.Versioner.
	versioner storage.Versioner
}
```
新建一个watchCache 

```
func newWatchCache(capacity int, keyFunc func(runtime.Object) (string, error)) *watchCache {
	wc := &watchCache{
		capacity:   capacity,
		keyFunc:    keyFunc,
		cache:      make([]watchCacheElement, capacity),
		startIndex: 0,
		endIndex:   0,
		/*
			定义在/pkg/client/cache/store.go
				==>func NewStore(keyFunc KeyFunc) Store
		*/
		store:           cache.NewStore(storeElementKey),
		resourceVersion: 0,
		clock:           clock.RealClock{},
	}
	wc.cond = sync.NewCond(wc.RLocker())
	return wc
}
```
新建一个cache.Store
* vendor/k8s.io/client.go/tools/cache/store.go:

```
// NewStore returns a Store implemented simply with a map and a lock.
func NewStore(keyFunc KeyFunc) Store {
	return &cache{
		cacheStorage: NewThreadSafeStore(Indexers{}, Indices{}),
		keyFunc:      keyFunc,
	}
}

```
watchCache实现了Add、Update、processEvent等一系列函数对cache中的event数据流进行处理。 

#### cacherListerWatcher
cacherListerWatcher内部的storage.Interface暴露给cache.ListerWatcher，其实现了List和Watch方法，但其实都是在调用之前通过newETCD3Storage(c)创建的定义在vendor/k8s.io/apiserver/pkg/storage/etcd3/store.go中List和Watch方法。

```
// cacherListerWatcher opaques storage.Interface to expose cache.ListerWatcher.
type cacherListerWatcher struct {
	storage        storage.Interface
	resourcePrefix string
	newListFunc    func() runtime.Object
}
```
#### Reflector
Reflector主要是watch一个指定的resource，会把resource发生的任何变化反映到指定的store中。 
* vendor/k8s.io/client-go/tools/cache/reflector.go:

```
// Reflector watches a specified resource and causes all changes to be reflected in the given store.
type Reflector struct {

	// The type of object we expect to place in the store.
	expectedType reflect.Type
	// The destination to sync up with the watch source
	store Store
	// listerWatcher is used to perform lists and watches.
	listerWatcher ListerWatcher
	// period controls timing between one watch ending and
	// the beginning of the next one.
	period       time.Duration
	resyncPeriod time.Duration
	ShouldResync func() bool
	// clock allows tests to manipulate time
	clock clock.Clock
	// lastSyncResourceVersion is the resource version token last
	// observed when doing a sync with the underlying store
	// it is thread safe, but not synchronized with the underlying store
	lastSyncResourceVersion string
	// lastSyncResourceVersionMutex guards read/write access to lastSyncResourceVersion
	lastSyncResourceVersionMutex sync.RWMutex
}
```
新建一个Reflector

创建一个新的type Reflector struct对象，Reflector会保持‘store中存储的expectedType’和etcd端的内容同步更新。 Reflector保证只会把符合expectedType类型的对象存放到store中，除非expectedType的值为nil。 如果resyncPeriod非0，那么list操作会间隔resyncPeriod执行一次， 所以可以使用reflectors周期性处理所有的数据、后续更新。

```

// NewNamedReflector same as NewReflector, but with a specified name for logging
func NewNamedReflector(name string, lw ListerWatcher, expectedType interface{}, store Store, resyncPeriod time.Duration) *Reflector {
	reflectorSuffix := atomic.AddInt64(&reflectorDisambiguator, 1)
	r := &Reflector{
		name: name,
		// we need this to be unique per process (some names are still the same) but obvious who it belongs to
		metrics:       newReflectorMetrics(makeValidPrometheusMetricLabel(fmt.Sprintf("reflector_"+name+"_%d", reflectorSuffix))),
		listerWatcher: lw,
		store:         store,
		expectedType:  reflect.TypeOf(expectedType),
		period:        time.Second,
		resyncPeriod:  resyncPeriod,
		clock:         &clock.RealClock{},
	}
	return r
}
```

### 启动Cacher（Cacher.startCaching）
分析其流程如下：
1. 首先会通过terminateAllWatchers注销所有的cachewatcher,因为这个时候apiserver还处于初始化阶段，因此不可能接受其他组件的watch，也就不可能有watcher。
2. 调用c.reflector.ListAndWatch函数，完成前面说过的功能：reflector主要将apiserver组件从etcd中watch到的资源存储到watchCache中。

```
func (c *Cacher) startCaching(stopChannel <-chan struct{}) {
  ...
  c.terminateAllWatchers()
  if err := c.reflector.ListAndWatch(stopChannel); err != nil {
		glog.Errorf("unexpected ListAndWatch error: %v", err)
	}
}
``` 
#### 调用Reflector的ListAndWatch
分析其流程如下：

1. 调用listerWatcher的List方法
2. 调用listerWatcher的Watch方法
3. 调用func (r *Reflector) watchHandler

```
ListAndWatch 首先会list所有的items，得到resource version；
然后使用该resource version去watch。
// ListAndWatch first lists all items and get the resource version at the moment of call,
// and then use the resource version to watch.
func (r *Reflector) ListAndWatch(stopCh <-chan struct{}) error {
	
	var resourceVersion string
	// Explicitly set "0" as resource version - it's fine for the List()
	// to be served from cache and potentially be delayed relative to
	// etcd contents. Reflector framework will catch up via Watch() eventually.
	options := metav1.ListOptions{ResourceVersion: "0"}
	list, err := r.listerWatcher.List(options)
	
	listMetaInterface, err := meta.ListAccessor(list)
	resourceVersion = listMetaInterface.GetResourceVersion()
	items, err := meta.ExtractList(list)
	if err := r.syncWith(items, resourceVersion); err != nil {
	}
	r.setLastSyncResourceVersion(resourceVersion)

	for {
		options = metav1.ListOptions{
			ResourceVersion: resourceVersion,
			TimeoutSeconds: &timeoutSeconds,
		}
		w, err := r.listerWatcher.Watch(options)

		r.watchHandler(w, &resourceVersion, resyncerrc, stopCh)
	}
}
```
从Reflector在NewCacherFromConfig方法的创建过程可知，其内部的listerWatch成员即为前面分析过的cacherListerWatcher。其List、Watch实现即为：

```
// Implements cache.ListerWatcher interface.
func (lw *cacherListerWatcher) List(options metav1.ListOptions) (runtime.Object, error) {
	list := lw.newListFunc()
	if err := lw.storage.List(context.TODO(), lw.resourcePrefix, "", storage.Everything, list); err != nil {
		return nil, err
	}
	return list, nil
}

// Implements cache.ListerWatcher interface.
func (lw *cacherListerWatcher) Watch(options metav1.ListOptions) (watch.Interface, error) {
	return lw.storage.WatchList(context.TODO(), lw.resourcePrefix, options.ResourceVersion, storage.Everything)
}
```
可以分析得出其本质上是调用其内部的storage成员来实现的，根据前面对cacherListerWatcher分析可知最终调用到了位于（vendor/k8s.io/apiserver/pkg/storage/etcd3/store.go）的etcd3.store的实现。
##### Reflector的watchHandler实现
将event对象从上一步Watch操作返回的watch.Interface的ResultChan中读取出来，然后根据event.Type去操作r.store，即操作watchCache。 

```
// watchHandler watches w and keeps *resourceVersion up to date.
func (r *Reflector) watchHandler(w watch.Interface, resourceVersion *string, errc chan error, stopCh <-chan struct{}) error {
	
	defer w.Stop()
loop:
	for {
		select {
		case <-stopCh:
			return errorStopRequested
		case err := <-errc:
			return err
		case event, ok := <-w.ResultChan():
			if !ok {
				break loop
			}
			
			meta, err := meta.Accessor(event.Object)
			newResourceVersion := meta.GetResourceVersion()

			switch event.Type {
			case watch.Added:
				err := r.store.Add(event.Object)
			case watch.Modified:
				err := r.store.Update(event.Object)
			case watch.Deleted:
				err := r.store.Delete(event.Object)
			default:
				utilruntime.HandleError(fmt.Errorf("%s: unable to understand watch event %#v", r.name, event))
			}
			*resourceVersion = newResourceVersion
			r.setLastSyncResourceVersion(newResourceVersion)
		}
	}

	return nil
}

```
接着我们以watchCache的Add方法实现为例继续深入分析
* vendor/k8s.io/apiserver/pkg/storage/cacher/watch_cache.go:

```
// Add takes runtime.Object as an argument.
func (w *watchCache) Add(obj interface{}) error {
	object, resourceVersion, err := w.objectToVersionedRuntimeObject(obj)
	event := watch.Event{Type: watch.Added, Object: object}

	f := func(elem *storeElement) error { return w.store.Add(elem) }
	return w.processEvent(event, resourceVersion, f)
}
```
继续调用processEvent方法处理event，其流程如下：

```
func (w *watchCache) processEvent(event watch.Event, resourceVersion uint64, updateFunc func(*storeElement) error) error {
	key, err := w.keyFunc(event.Object)
	if err != nil {
		return fmt.Errorf("couldn't compute key: %v", err)
	}
	elem := &storeElement{Key: key, Object: event.Object}
	elem.Labels, elem.Fields, elem.Uninitialized, err = w.getAttrsFunc(event.Object)
	if err != nil {
		return err
	}

	watchCacheEvent := &watchCacheEvent{
		Type:             event.Type,
		Object:           elem.Object,
		ObjLabels:        elem.Labels,
		ObjFields:        elem.Fields,
		ObjUninitialized: elem.Uninitialized,
		Key:              key,
		ResourceVersion:  resourceVersion,
	}

	// TODO: We should consider moving this lock below after the watchCacheEvent
	// is created. In such situation, the only problematic scenario is Replace(
	// happening after getting object from store and before acquiring a lock.
	// Maybe introduce another lock for this purpose.
	w.Lock()
	defer w.Unlock()
	previous, exists, err := w.store.Get(elem)
	if err != nil {
		return err
	}
	if exists {
		previousElem := previous.(*storeElement)
		watchCacheEvent.PrevObject = previousElem.Object
		watchCacheEvent.PrevObjLabels = previousElem.Labels
		watchCacheEvent.PrevObjFields = previousElem.Fields
		watchCacheEvent.PrevObjUninitialized = previousElem.Uninitialized
	}

	if w.onEvent != nil {
		w.onEvent(watchCacheEvent)
	}
	w.updateCache(resourceVersion, watchCacheEvent)
	w.resourceVersion = resourceVersion
	w.cond.Broadcast()
	return updateFunc(elem)
}
```




