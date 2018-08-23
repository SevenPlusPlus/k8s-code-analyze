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
