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
###存储后端创建工厂方法实现(storagebackend/factory/)
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

