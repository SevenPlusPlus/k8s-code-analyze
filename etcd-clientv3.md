## clientv3是官方提供的etcd v3的go客户端实现

### clientv3用法详解

#### 创建client
要访问etcd第一件事就是创建client，它需要传入一个Config配置，这里传了2个选项：
* Endpoints：etcd的多个节点服务地址，因为我是单点测试，所以只传1个。
* DialTimeout：创建client的首次连接超时，这里传了5秒，如果5秒都没有连接成功就会返回err；值得注意的是，一旦client创建成功，我们就不用再关心后续底层连接的状态了，client内部会重连。

代码示例如下:
```
cli, err := clientv3.New(clientv3.Config{
   Endpoints:   []string{"localhost:2378"},
   DialTimeout: 5 * time.Second,
})
```
创建的client结构如下：
```
type Client struct {
    Cluster
    KV
    Lease
    Watcher
    Auth
    Maintenance

    // Username is a user name for authentication.
    Username string
    // Password is a password for authentication.
    Password string
    // contains filtered or unexported fields
}
```
Cluster：向集群里增加etcd服务端节点之类，属于管理员操作。
KV：我们主要使用的功能，即操作K-V。
Lease：租约相关操作，比如申请一个TTL=10秒的租约。
Watcher：观察订阅，从而监听最新的数据变化。
Auth：管理etcd的用户和权限，属于管理员操作。
Maintenance：维护etcd，比如主动迁移etcd的leader节点，属于管理员操作。
#### 获取KV对象
实际上client.KV是一个interface，提供了关于k-v操作的所有方法：
```
type KV interface {
	// Put puts a key-value pair into etcd.
	// Note that key,value can be plain bytes array and string is
	// an immutable representation of that bytes array.
	// To get a string of bytes, do string([]byte{0x10, 0x20}).
	Put(ctx context.Context, key, val string, opts ...OpOption) (*PutResponse, error)

	// Get retrieves keys.
	// By default, Get will return the value for "key", if any.
	// When passed WithRange(end), Get will return the keys in the range [key, end).
	// When passed WithFromKey(), Get returns keys greater than or equal to key.
	// When passed WithRev(rev) with rev > 0, Get retrieves keys at the given revision;
	// if the required revision is compacted, the request will fail with ErrCompacted .
	// When passed WithLimit(limit), the number of returned keys is bounded by limit.
	// When passed WithSort(), the keys will be sorted.
	Get(ctx context.Context, key string, opts ...OpOption) (*GetResponse, error)

	// Delete deletes a key, or optionally using WithRange(end), [key, end).
	Delete(ctx context.Context, key string, opts ...OpOption) (*DeleteResponse, error)

	// Compact compacts etcd KV history before the given rev.
	Compact(ctx context.Context, rev int64, opts ...CompactOption) (*CompactResponse, error)

	// Do applies a single Op on KV without a transaction.
	// Do is useful when creating arbitrary operations to be issued at a
	// later time; the user can range over the operations, calling Do to
	// execute them. Get/Put/Delete, on the other hand, are best suited
	// for when the operation should be issued at the time of declaration.
	Do(ctx context.Context, op Op) (OpResponse, error)

	// Txn creates a transaction.
	Txn(ctx context.Context) Txn
}
```
但是我们一般并不是直接获取client.KV来使用，而是通过一个方法来获得一个经过装饰的KV实现（内置错误重试机制的高级KV）来操作ETCD中的数据：
```
kv := clientv3.NewKV(cli)
```
##### Put操作
函数原型为：
```
	// Put puts a key-value pair into etcd.
	// Note that key,value can be plain bytes array and string is
	// an immutable representation of that bytes array.
	// To get a string of bytes, do string([]byte{0x10, 0x20}).
	Put(ctx context.Context, key, val string, opts ...OpOption) (*PutResponse, error)
```
除了我们传递的参数，还支持一个可变参数，主要是传递一些控制项来影响Put的行为，例如可以携带一个lease ID来支持key过期，这个后面再说。

上述Put操作返回的是PutResponse，不同的KV操作对应不同的response结构。
```
type PutResponse struct {
	Header *ResponseHeader `protobuf:"bytes,1,opt,name=header" json:"header,omitempty"`
	// if prev_kv is set in the request, the previous key-value pair will be returned.
	PrevKv *mvccpb.KeyValue `protobuf:"bytes,2,opt,name=prev_kv,json=prevKv" json:"prev_kv,omitempty"`
}
```
Header里保存的主要是本次更新的revision信息，而PrevKv可以返回Put覆盖之前的value是什么
```
	// 再写一个孩子
	kv.Put(context.TODO(),"/test/b", "another")

	// 再写一个同前缀的干扰项
	kv.Put(context.TODO(), "/testxxx", "干扰")
```


