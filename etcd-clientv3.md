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
SA


