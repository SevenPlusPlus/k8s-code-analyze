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
// Run runs the specified APIServer. This should never exit.func 
Run(completeOptions completedServerRunOptions, stopCh <-chan struct{}) error {
 server, err := CreateServerChain(completeOptions, stopCh) 
 return server.PrepareRun().Run(stopCh)
}
```
创建GenericApiServer
cmd/kube-apiserver/app/server.go:
```
// CreateServerChain creates the apiservers connected via delegation.func 
CreateServerChain(completedOptions completedServerRunOptions, stopCh <-chan struct{}) (*genericapiserver.GenericAPIServer, error) { 
	kubeAPIServerConfig, sharedInformers, versionedInformers, insecureServingOptions, serviceResolver, pluginInitializer, admissionPostStartHook, err := CreateKubeAPIServerConfig(completedOptions, nodeTunneler, proxyTransport)
	kubeAPIServer, err := CreateKubeAPIServer(kubeAPIServerConfig, apiExtensionsServer.GenericAPIServer, sharedInformers, versionedInformers, admissionPostStartHook)
	kubeAPIServer.GenericAPIServer.PrepareRun()
	return aggregatorServer.GenericAPIServer, nil
}
```
```
func CreateKubeAPIServer(kubeAPIServerConfig *master.Config, delegateAPIServer genericapiserver.DelegationTarget, sharedInformers informers.SharedInformerFactory, versionedInformers clientgoinformers.SharedInformerFactory, admissionPostStartHook genericapiserver.PostStartHookFunc) (*master.Master, error) {
 kubeAPIServer, err := kubeAPIServerConfig.Complete(versionedInformers).New(delegateAPIServer)  return kubeAPIServer, nil
}
```
设置Storage：
k8s.io/kubernetes/pkg/master/master.go: 
```
func (c completedConfig) New(delegationTarget genericapiserver.DelegationTarget) (*Master, error) {
	s, err := c.GenericConfig.New("kube-apiserver", delegationTarget)
	m := &Master{
		GenericAPIServer: s,
	}}
	// install legacy rest storage
	legacyRESTStorageProvider := corerest.LegacyRESTStorageProvider{
			StorageFactory:              c.ExtraConfig.StorageFactory,
			ProxyTransport:              c.ExtraConfig.ProxyTransport,
			KubeletClientConfig:         c.ExtraConfig.KubeletClientConfig,
			EventTTL:                    c.ExtraConfig.EventTTL,
			ServiceIPRange:              c.ExtraConfig.ServiceIPRange,
			ServiceNodePortRange:        c.ExtraConfig.ServiceNodePortRange,
			LoopbackClientConfig:        c.GenericConfig.LoopbackClientConfig,
			ServiceAccountIssuer:        c.ExtraConfig.ServiceAccountIssuer,
			ServiceAccountAPIAudiences:  c.ExtraConfig.ServiceAccountAPIAudiences,
			ServiceAccountMaxExpiration: c.ExtraConfig.ServiceAccountMaxExpiration,
		}
	m.InstallLegacyAPI(&c, c.GenericConfig.RESTOptionsGetter, legacyRESTStorageProvider)
	


	restStorageProviders := []RESTStorageProvider{

		authenticationrest.RESTStorageProvider{Authenticator: c.GenericConfig.Authentication.Authenticator},
		authorizationrest.RESTStorageProvider{Authorizer: c.GenericConfig.Authorization.Authorizer, RuleResolver: c.GenericConfig.RuleResolver},
		autoscalingrest.RESTStorageProvider{},
		batchrest.RESTStorageProvider{},
		certificatesrest.RESTStorageProvider{},
		coordinationrest.RESTStorageProvider{},
		extensionsrest.RESTStorageProvider{},
		networkingrest.RESTStorageProvider{},
		policyrest.RESTStorageProvider{},
		rbacrest.RESTStorageProvider{Authorizer: c.GenericConfig.Authorization.Authorizer},
		schedulingrest.RESTStorageProvider{},
		settingsrest.RESTStorageProvider{},
		storagerest.RESTStorageProvider{},
		// keep apps after extensions so legacy clients resolve the extensions versions of shared resource names.
		// See https://github.com/kubernetes/kubernetes/issues/42392
		appsrest.RESTStorageProvider{},
		admissionregistrationrest.RESTStorageProvider{},
		eventsrest.RESTStorageProvider{TTL: c.ExtraConfig.EventTTL},
	}

	m.InstallAPIs(c.ExtraConfig.APIResourceConfigSource, c.GenericConfig.RESTOptionsGetter, restStorageProviders...)


	return m, nil
```
留意这里创建的legacyRESTStorageProvider和restStorageProviders，通过接下来的过程可以看到storage最终是由它们创建的。 

### m.InstallLegacyAPI()
k8s.io/kubernetes/pkg/master/master.go: 
```
func (m *Master) InstallLegacyAPI(c *completedConfig, restOptionsGetter generic.RESTOptionsGetter, legacyRESTStorageProvider corerest.LegacyRESTStorageProvider) {

 legacyRESTStorage, apiGroupInfo, err := legacyRESTStorageProvider.NewLegacyRESTStorage(restOptionsGetter) 
 m.GenericAPIServer.InstallLegacyAPIGroup(genericapiserver.DefaultLegacyAPIPrefix, &apiGroupInfo); 
}
```
可以看到，这里生成了一个apiGroupInfo，然后将其装载到了m.GenericAPIServer中，与GenericAPIServer的工作方式衔接上了。 

### legacyRESTStorageProvider
apiGroupInfo是通过调用legacyRESTStorageProvider.NewLegacyRESTStorage()创建的.
k8s.io/kubernetes/pkg/registry/core/rest/storage_core.go:
```
func (c LegacyRESTStorageProvider) NewLegacyRESTStorage(restOptionsGetter generic.RESTOptionsGetter) (LegacyRESTStorage, genericapiserver.APIGroupInfo, error) {
     apiGroupInfo := genericapiserver.APIGroupInfo{
 	PrioritizedVersions: legacyscheme.Scheme.PrioritizedVersionsForGroup(""), 
	VersionedResourcesStorageMap: map[string]map[string]rest.Storage{},
 	Scheme: legacyscheme.Scheme,
 	ParameterCodec: legacyscheme.ParameterCodec,
 	NegotiatedSerializer: legacyscheme.Codecs,
 	}
       restStorage := LegacyRESTStorage{}
	...
	nodeStorage, err := nodestore.NewStorage(restOptionsGetter, c.KubeletClientConfig, c.ProxyTransport)
	...
	//按资源创建了Storage
	podTemplateStorage := podtemplatestore.NewREST(restOptionsGetter)
	eventStorage := eventstore.NewREST(restOptionsGetter, uint64(c.EventTTL.Seconds()))
	limitRangeStorage := limitrangestore.NewREST(restOptionsGetter)
	...
	podStorage := podstore.NewStorage(
			restOptionsGetter,
			nodeStorage.KubeletConnectionInfo,
			c.ProxyTransport,
			podDisruptionClient,
			)
	...
	
	//将新建的storage保存到
	restStorageMap := map[string]rest.Storage{
		"pods":             podStorage.Pod,
		"pods/attach":      podStorage.Attach,
		"pods/status":      podStorage.Status,
		"pods/log":         podStorage.Log,
		...
		"nodes":        nodeStorage.Node,
	...
	//restStorageMap被装载到了apiGroupInfo
	apiGroupInfo.VersionedResourcesStorageMap["v1"] = restStorageMap
	...
```
重点分析几个具体的storage，了解它们的实现。 
#### nodeStorage
在上面的代码中可以看到nodeStorage是通过调用nodestore.NewStorage创建的。
k8s.io/kubernetes/pkg/registry/core/node/storage/storage.go: 
```
func NewStorage(optsGetter generic.RESTOptionsGetter, kubeletClientConfig client.KubeletClientConfig, proxyTransport http.RoundTripper) (*NodeStorage, error) {
 store := &genericregistry.Store{ 
 NewFunc:                  func() runtime.Object { return &api.Node{} },
		
 NewListFunc:              func() runtime.Object { return &api.NodeList{} },
... 
 }

 // Set up REST handlers
	nodeREST := &REST{Store: store, proxyTransport: proxyTransport}
	statusREST := &StatusREST{store: &statusStore}
	proxyREST := &noderest.ProxyREST{Store: store, ProxyTransport: proxyTransport}

 return &NodeStorage{
 Node: nodeREST,
 Status: statusREST,
 Proxy: proxyREST,
 KubeletConnectionInfo: connectionInfoGetter,
 }, nil 
...

```
NodeStorage的成员变量Status实现了Get()、New()、Update(), Status类型是｀StatusREST`。 
```
// StatusREST implements the REST endpoint for changing the status of a node.
type StatusREST struct {
 store *genericregistry.Store
}
func (r *StatusREST) New() runtime.Object {
 return &api.Node{}
}
// Get retrieves the object from the storage. It is required to support Patch.
func (r *StatusREST) Get(ctx context.Context, name string, options *metav1.GetOptions) (runtime.Object, error) {
 return r.store.Get(ctx, name, options)
}
// Update alters the status subset of an object.
func (r *StatusREST) Update(ctx context.Context, name string, objInfo rest.UpdatedObjectInfo, createValidation rest.ValidateObjectFunc, updateValidation rest.ValidateObjectUpdateFunc, forceAllowCreate bool, options *metav1.UpdateOptions) (runtime.Object, bool, error) {
  ...}

```
回到创建NodeStorage的函数中，找到变量StatusREST.store的创建。 可以看到，StatusREST.store是genericregistry.Store。genericregistry.Store中包含了NewFunc、NewListFunc等函数变量，需要看一下genericregistry.Store的Get等方法是怎样实现的。
### generic.registry.Store
k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/registry/generic/registry/store.go: 
```
// Store implements pkg/api/rest.StandardStorage. It's intended to be
 // embeddable and allows the consumer to implement any non-generic functions    // that are required. This object is intended to be copyable so that it can be // used in different ways but share the same underlying behavior.
 //
 // All fields are required unless specified. 
//
 // The intended use of this type is embedding within a Kind specific
 // RESTStorage implementation. This type provides CRUD semantics on a Kubelike 
// resource, handling details like conflict detection with ResourceVersion and
 // semantics. The RESTCreateStrategy, RESTUpdateStrategy, and
 // RESTDeleteStrategy are generic across all backends, and encapsulate logic // specific to the API.

```
registry.Store的成员: 
```
-+Store : struct
 [fields]
 +AfterCreate : ObjectFunc
 +AfterDelete : ObjectFunc
 +AfterUpdate : ObjectFunc
 +Copier : runtime.ObjectCopier
 +CreateStrategy : rest.RESTCreateStrategy
 +Decorator : ObjectFunc
 +DeleteCollectionWorkers : int
 +DeleteStrategy : rest.RESTDeleteStrategy
 +DestroyFunc : func()
 +EnableGarbageCollection : bool
 +ExportStrategy : rest.RESTExportStrategy
 +KeyFunc : func(ctx genericapirequest.Context, name string) string, error +KeyRootFunc : func(ctx genericapirequest.Context) string
 +NewFunc : func() runtime.Object
 +NewListFunc : func() runtime.Object
 +ObjectNameFunc : func(obj runtime.Object) string, error
 +PredicateFunc : func(label labels.Selector, field fields.Selector) storage.SelectionPredicate
 +QualifiedResource : schema.GroupResource
 +ReturnDeletedObject : bool
 +Storage : storage.Interface
 +TTLFunc : func(obj runtime.Object, existing uint64, update bool) uint64, error
 +UpdateStrategy : rest.RESTUpdateStrategy
 +WatchCacheSize : int
 [methods]
 +CompleteWithOptions(options *generic.StoreOptions) : error
 +Create(ctx genericapirequest.Context, obj runtime.Object) : runtime.Object, error
 +Delete(ctx genericapirequest.Context, name string, options *metav1.DeleteOptions) : runtime.Object, bool, error
 +DeleteCollection(ctx genericapirequest.Context, options *metav1.DeleteOptions, listOptions *metainternalversion.ListOptions) : runtime.Object, error
 +Export(ctx genericapirequest.Context, name string, opts metav1.ExportOptions) : runtime.Object, error
 +Get(ctx genericapirequest.Context, name string, options *metav1.GetOptions) : runtime.Object, error
 +List(ctx genericapirequest.Context, options *metainternalversion.ListOptions) : runtime.Object, error
 +ListPredicate(ctx genericapirequest.Context, p storage.SelectionPredicate, options *metainternalversion.ListOptions) : runtime.Object, error
 +New() : runtime.Object
 +NewList() : runtime.Object
 +Update(ctx genericapirequest.Context, name string, objInfo rest.UpdatedObjectInfo) : runtime.Object, bool, error
 +Watch(ctx genericapirequest.Context, options *metainternalversion.ListOptions) : watch.Interface, error
 +WatchPredicate(ctx genericapirequest.Context, p storage.SelectionPredicate, resourceVersion string) : watch.Interface, error

```
看一下Create()方法的实现: 

```
// Create inserts a new item according to the unique key from the object.
func (e *Store) Create(ctx context.Context, obj runtime.Object, createValidation rest.ValidateObjectFunc, options *metav1.CreateOptions) (runtime.Object, error) { name, err := e.ObjectNameFunc(obj)
 key, err := e.KeyFunc(ctx, name)
 qualifiedResource := e.qualifiedResourceFromContext(ctx)
 ttl, err := e.calculateTTL(obj, 0, false)
 out := e.NewFunc()
 e.Storage.Create(ctx, key, obj, out, ttl, dryrun.IsDryRun(options.DryRun))

 if e.AfterCreate != nil {... } }
 if e.Decorator != nil { ... } }
 return out, nil
}
```










