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
 +EnableGarbageCollection : bool
 +ExportStrategy : rest.RESTExportStrategy
 +KeyFunc : func(ctx genericapirequest.Context, name string) string, error +KeyRootFunc : func(ctx genericapirequest.Context) string
 +NewFunc : func() runtime.Object
 +NewListFunc : func() runtime.Object
 +ObjectNameFunc : func(obj runtime.Object) string, error
 +PredicateFunc : func(label labels.Selector, field fields.Selector) storage.SelectionPredicate
 +QualifiedResource : schema.GroupResource
 +ReturnDeletedObject : bool
 +Storage : DryRunnableStorage 
 +DestroyFunc : func()
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
其中的Storage成员类型为DryRunnableStorage：
```
type DryRunnableStorage struct {
 Storage storage.Interface
 Codec runtime.Codec
}
```
看一下Create()方法的实现: 

```
// Create inserts a new item according to the unique key from the object.
func (e *Store) Create(ctx context.Context, obj runtime.Object, createValidation rest.ValidateObjectFunc, options *metav1.CreateOptions) (runtime.Object, error) {
 name, err := e.ObjectNameFunc(obj)
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
e.NewFunc()返回的对象是调用创建方法时传入的变量，对于NodeStorage而言是: 
```
 store := &genericregistry.Store{
 	Copier: api.Scheme,
 	NewFunc: func() runtime.Object { return &api.Node{} },
 	NewListFunc: func() runtime.Object { return &api.NodeList{} }, 
	ObjectNameFunc: func(obj runtime.Object) (string, error) { return obj.(*api.Node).Name, nil },
 	PredicateFunc: node.MatchNode,
 	QualifiedResource: api.Resource("nodes"),
 	WatchCacheSize: cachesize.GetWatchCacheSizeByResource("nodes"), 
	CreateStrategy: node.Strategy,
 	UpdateStrategy: node.Strategy,
 	DeleteStrategy: node.Strategy,
 	ExportStrategy: node.Strategy,
 }
```
创建时在调用了e.NewFunc()之后，又调用了e.Storage.Create(ctx, key, obj, out, ttl)，需要继续回溯找到创建e.Storage的地方。 NodeStorage创建store时没有为store的Storage字段赋值，所以应该是后续进行的赋值操作。回溯NodeStorage的创建代码，发现了store.CompleteWithOptions()，kubernetes的代码中经常会用这种方式来补全一个结构体的成员变量。
回到k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/registry/generic/registry/store.go: 
```
func (e *Store) CompleteWithOptions(options *generic.StoreOptions) error {

 ...
 opts, err := options.RESTOptions.GetRESTOptions(e.QualifiedResource)
 ...
 if e.Storage == nil {
 	e.Storage.Codec = opts.StorageConfig.Codec

	e.Storage.Storage, e.DestroyFunc = opts.Decorator(
			opts.StorageConfig,
			e.NewFunc(),
			prefix,
			keyFunc,
			e.NewListFunc,
			attrFunc,
			triggerFunc,
		)
 }
 ...
```
不出所料，e.Storage就是在这里创建的，还需要继续回溯，找到opts.Decorator()的实现。 
#### RESTOptions opts.Decorator
要找到opts.Decorator()的实现，看它是如何创建了e.Storage。 在上面的代码中可以看到opts是通过options.RESTOptions.GetRESTOptions()创建的。而options则是在NodeStorage的NewStorage中调用store.CompletWithOptions之前创建的：
k8s.io/kubernetes/pkg/registry/core/node/storage/storage.go:
```
func NewStorage(optsGetter generic.RESTOptionsGetter, kubeletClientConfig client.KubeletClientConfig, proxyTransport http.RoundTripper) (*NodeStorage, error) {
...
options := &generic.StoreOptions{RESTOptions: optsGetter, AttrFunc: node.GetAttrs, TriggerFunc: node.NodeNameTriggerFunc}

...

```
options.RESTOptions就是变量optsGetter，继续回溯，找到optsGetter的实现： 
k8s.io/kubernetes/pkg/master/master.go: 
```
func (c completedConfig) New() (*Master, error) {
 ...
 m := &Master{ GenericAPIServer: s, }
 if c.APIResourceConfigSource.AnyResourcesForVersionEnabled(apiv1.SchemeGroupVersion) {
 ...
 //装载pod、service的资源操作的REST api
 //参数2就是optsGetter
 m.InstallLegacyAPI(c.Config, 
 c.Config.GenericConfig.RESTOptionsGetter, legacyRESTStorageProvider)
 }
...
```
c.Config.GenericConfig.RESTOptionsGetter就是optsGetter，而c.Config就是一开始就提醒要记住的kubeAPIServerConfig: 
k8s.io/kubernetes/cmd/kube-apiserver/app/server.go: 
```
func Run(runOptions *options.ServerRunOptions, stopCh <-chan struct{}) error {
 kubeAPIServerConfig, sharedInformers, insecureServingOptions, err := CreateKubeAPIServerConfig(runOptions)
 ...

```
要找到kubeAPIServerConfig.GenericConfig.RESTOptionsGetter。 
此处的kubeAPIServerConfig.GenericConfig即master.config.GenericConfig
其在CreateKubeAPIServerConfig方法中调用buildGenericConfig得到的。
```
func buildGenericConfig(s *options.ServerRunOptions) (*genericapiserver.Config, informers.SharedInformerFactory, *kubeserver.InsecureServingInfo, error) {
 genericConfig := genericapiserver.NewConfig(api.Codecs)
 ...
if lastErr = s.Etcd.ApplyWithStorageFactoryTo(storageFactory, genericConfig); lastErr != nil {
 return
}

```
进入到s.Etcd.ApplyWithStorageFactoryTo()中，才猛然发现: 
k8s.io/kubernetes/vendor/src/k8s.io/apiserver/pkg/server/options/etcd.go:
```
func (s *EtcdOptions) ApplyWithStorageFactoryTo(factory serverstorage.StorageFactory, c *server.Config) error {
 c.RESTOptionsGetter = &storageFactoryRestOptionsFactory{Options: *s, StorageFactory: factory}
 return nil
}
```
而且注意s的类型是EtcdOptions! 
我们最终要找的是e.Storage -> opts.Decorator -> opts -> options.RESTOptions.GetRESTOptions()即
k8s.io/kubernetes/vendor/src/k8s.io/apiserver/pkg/server/options/etcd.go:
```
func (f *SimpleRestOptionsFactory) GetRESTOptions(resource schema.GroupResource) (generic.RESTOptions, error) {

	ret := generic.RESTOptions{
		StorageConfig:           &f.Options.StorageConfig,
		Decorator:               generic.UndecoratedStorage,
		EnableGarbageCollection: f.Options.EnableGarbageCollection,
		DeleteCollectionWorkers: f.Options.DeleteCollectionWorkers,
		ResourcePrefix:          resource.Group + "/" + resource.Resource,
		CountMetricPollPeriod:   f.Options.StorageConfig.CountMetricPollPeriod,
	}

	if f.Options.EnableWatchCache {
		sizes, err := ParseWatchCacheSizes(f.Options.WatchCacheSizes)
		if err != nil {
			return generic.RESTOptions{}, err
		}
		cacheSize, ok := sizes[resource]
		if !ok {
			cacheSize = f.Options.DefaultWatchCacheSize
		}
		ret.Decorator = genericregistry.StorageWithCacher(cacheSize)
	}
	return ret, nil
}
```
找到Decorator方法实现了，e.Storage.Storage就是通过调用这个函数得到的，也就是genericregistry.StorageWithCacher方法，不开启watchcache时Decorator的实现为generic.UndecoratedStorage方法: 
#### genericregistry.StorageWithCacher方法
别忘了NodeStorage的Create方法最终操作落到e.Storage上，而e.Storage.Storage是通过opts.Decorator()创建的。
而opt.Decorator真正实现就是genericregistry.StorageWithCacher()
k8s.io/kubernetes/vendor/src/k8s.io/apiserver/pkg/registry/generic/registry/storage_factory.go:

```
// Creates a cacher based given storageConfig.
func StorageWithCacher(capacity int) generic.StorageDecorator {
 return func( storageConfig *storagebackend.Config, objectType runtime.Object, resourcePrefix string, keyFunc func(obj runtime.Object) (string, error), newListFunc func() runtime.Object, getAttrsFunc storage.AttrFunc, triggerFunc storage.TriggerPublisherFunc) (storage.Interface, factory.DestroyFunc) {
    s, d := generic.NewRawStorage(storageConfig)
 // TODO: we would change this later to make storage always have cacher and hide low level KV layer inside.
 // Currently it has two layers of same storage interface -- cacher and low level kv.
   cacherConfig := cacherstorage.Config{
 	CacheCapacity: capacity,
	Storage: s,
 	Versioner: etcdstorage.APIObjectVersioner{},
 	Type: objectType,
 	ResourcePrefix: resourcePrefix,
 	KeyFunc: keyFunc,
 	NewListFunc: newListFunc,
 	GetAttrsFunc: getAttrsFunc,
 	TriggerPublisherFunc: triggerFunc,
 	Codec: storageConfig.Codec,
   }
   cacher := cacherstorage.NewCacherFromConfig(cacherConfig)

   destroyFunc := func() {
 	cacher.Stop()
 	d()
   }
   RegisterStorageCleanup(destroyFunc)

   return cacher, destroyFunc
 }
}
```
StorageWithCache又是一个很复杂的过程，它将会真正与etcd通信。 
将会在下一节**ApiServer Storage实现解析**中重点解析。








