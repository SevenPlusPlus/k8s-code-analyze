##ApiServer后端存储机制接口实现
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
* vendor/k8s.io/apiserver/pkg/storage/cacher/cacher.go:

```
// NewCacherFromConfig creates a new Cacher responsible for servicing WATCH and LIST requests from
// its internal cache and updating its cache in the background based on the
// given configuration.
func NewCacherFromConfig(config Config) *Cacher {
  watchCache := newWatchCache(config.CacheCapacity, config.KeyFunc, config.GetAttrsFunc, config.Versioner)
  listerWatcher := newCacherListerWatcher(config.Storage, config.ResourcePrefix, config.NewListFunc)

  cacher := &Cacher{
		storage:     config.Storage,
		objectType:  reflect.TypeOf(config.Type),
		watchCache:  watchCache,
		reflector:   cache.NewNamedReflector(reflectorName, listerWatcher, config.Type, watchCache, 0),
		watchers: indexedWatchers{
			allWatchers:   make(map[int]*cacheWatcher),
			valueWatchers: make(map[string]watchersMap),
		},
		...
	}
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
}
```


