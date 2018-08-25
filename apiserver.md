## ApiServer中Storage的前世今生
从ApiServer的注册流程可以看到，ApiGroupInfo中装载了各类的Storage，然后GenericApiServer根据传入的ApiGroupInfo中的Storage，来自动生成REST Handler。

ApiServer在存取资源时，最终是通过各个storage完成，这里探究一下storage怎样创建的，又是怎样与etcd关联上的。 
###代码结构
registry目录下收录的就是kubernetes的每类资源的REST实现代码。 
```
k8s.io/kubernetes/pkg/registry/
 ▾ apps/
 	▸ petset/ 
 	▸ rest/ storage_apps.go
 ▸ authentication/
 ▸ authorization/
 ▸ autoscaling/
 ▸ batch/
 ▸ cachesize/
 ▸ certificates/
 ▸ core/ #core/rest中实现了NewLegacyRESTStorage()
 ▸ extensions/
 ▸ policy/
 ▸ rbac/
 ▸ registrytest/
 ▸ settings/
 ▸ storage/
```
每类资源目录下都有一个rest目录，其中实现了各自的storage。例如apps/rest中的代码定义了可以提供给GenericAPIServer的storage。 

### Storage的注册装载过程
cmd/kube-apiserver/app/server.go: 
```
// Run runs the specified APIServer. This should never exit.func Run(completeOptions completedServerRunOptions, stopCh <-chan struct{}) error {
 server, err := CreateServerChain(completeOptions, stopCh) return server.PrepareRun().Run(stopCh)}
```


