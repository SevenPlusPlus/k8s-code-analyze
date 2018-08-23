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
 _, _, err := codec.Decode(value, nil, objPtr)
 // being unable to set the version does not prevent the object from being extracted
 versioner.UpdateObject(objPtr, uint64(rev))
 return nil
}

```


