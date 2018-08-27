## API注册流程
先来了解下API注册流程概要
![API注册流程概要](/assets/apiserver-register-01.jpg)
开始API注册
* /pkg/master/master.go:
```
func (c completedConfig) New(delegationTarget genericapiserver.DelegationTarget) (*Master, error) { 
  ...
  m.InstallLegacyAPI(&c, c.GenericConfig.RESTOptionsGetter, legacyRESTStorageProvider)
  ...
  m.InstallAPIs(c.ExtraConfig.APIResourceConfigSource, c.GenericConfig.RESTOptionsGetter, restStorageProviders...)
  ...
```
### Install /api
* /pkg/master/master.go:

```
func (m *Master) InstallLegacyAPI(c *completedConfig, restOptionsGetter generic.RESTOptionsGetter, legacyRESTStorageProvider corerest.LegacyRESTStorageProvider) {

 legacyRESTStorage, apiGroupInfo, err := 		
       legacyRESTStorageProvider.NewLegacyRESTStorage(restOptionsGetter)

  m.GenericAPIServer.InstallLegacyAPIGroup(
        genericapiserver.DefaultLegacyAPIPrefix, &apiGroupInfo)
...

}
```
* /vendor/k8s.io/apiserver/pkg/server/genericapiserver.go:

```
func (s *GenericAPIServer) InstallLegacyAPIGroup(apiPrefix string, apiGroupInfo *APIGroupInfo) error {
  if err := s.installAPIResources(apiPrefix, apiGroupInfo); err != nil {
		return err
	}
  ...
```
### Install /apis
* /pkg/master/master.go:

```
func (m *Master) InstallAPIs(apiResourceConfigSource serverstorage.APIResourceConfigSource, restOptionsGetter generic.RESTOptionsGetter, restStorageProviders ...RESTStorageProvider) {
 apiGroupsInfo := []genericapiserver.APIGroupInfo{}

 for i := range apiGroupsInfo {
		if err := m.GenericAPIServer.InstallAPIGroup(&apiGroupsInfo[i]); err != nil {
			glog.Fatalf("Error in registering group versions: %v", err)
		}
	}
```
* /vendor/k8s.io/apiserver/pkg/server/genericapiserver.go:

```
func (s *GenericAPIServer) InstallAPIGroup(apiGroupInfo *APIGroupInfo) error {

  if err := s.installAPIResources(APIGroupPrefix, apiGroupInfo); err != nil { 
     return err
  }
  ...
}

```
### all to installAPIResources
* /vendor/k8s.io/apiserver/pkg/server/genericapiserver.go:

```
func (s *GenericAPIServer) installAPIResources(apiPrefix string, apiGroupInfo *APIGroupInfo) error {
  for _, groupVersion := range apiGroupInfo.PrioritizedVersions {
    apiGroupVersion.InstallREST(s.Handler.GoRestfulContainer)
    ...
  }
}

```
* /vendor/k8s.io/apiserver/pkg/endpoints/groupversion.go:

```
func (g *APIGroupVersion) InstallREST(container *restful.Container) error {

   prefix := path.Join(g.Root, g.GroupVersion.Group, g.GroupVersion.Version)
   installer := &APIInstaller{     group: g,     prefix: prefix,   }
   apiResources, ws, registrationErrors := installer.Install()   
   container.Add(ws)
   ...
}
```
* /vendor/k8s.io/apiserver/pkg/endpoints/installer.go:

```
// Install handlers for API resources.
func (a *APIInstaller) Install() ([]metav1.APIResource, *restful.WebService, []error) {
 ...
 ws := a.newWebService()
 for _, path := range paths {
    apiResource, err := a.registerResourceHandlers(path, a.group.Storage[path], ws)
 }
  if apiResource != nil {
    apiResources = append(apiResources, *apiResource)
  } 
}
 return apiResources, ws, errors
}
```
这里可以总结一下主要过程：

1. 新建WebService
2. 将API对应的route新建初始化后加入WebService
3. 将WebService加入Container

### WebService新增router
* /vendor/k8s.io/apiserver/pkg/endpoints/installer.go:

```
func (a *APIInstaller) registerResourceHandlers(path string, storage rest.Storage, ws *restful.WebService) (*metav1.APIResource, error) {
  ...
  creater, isCreater := storage.(rest.Creater)
  actions := []action{}
  ...
  actions = appendIf(actions, action{"POST", resourcePath, resourceParams, namer, false}, isCreater)
  ...
  // Create Routes for the actions.
  for _, action := range actions {
    switch action.Verb {
    case "POST": // Create a resource.
       var handler restful.RouteFunction
	  if isNamedCreater {
		handler = restfulCreateNamedResource(namedCreater, reqScope, admit)
	  } else {
		handler = restfulCreateResource(creater, reqScope, admit)
	  }
       route := ws.POST(action.Path).To(handler).
		Doc(doc).
		Param(ws.QueryParameter("pretty", "If 'true', then the output is pretty printed.")).
		Operation("create"+namespaced+kind+strings.Title(subresource)+operationSuffix).				Produces(append(storageMeta.ProducesMIMETypes(action.Verb), mediaTypes...)...).
		Returns(http.StatusOK, "OK", producedObject).
		// TODO: in some cases, the API may return a v1.Status instead of the versioned object
		// but currently go-restful can't handle multiple different objects being returned.
		Returns(http.StatusCreated, "Created", producedObject).
		Returns(http.StatusAccepted, "Accepted", producedObject).
		Reads(defaultVersionedObject).
		Writes(producedObject)

	    addParams(route, action.Params)
	    routes = append(routes, route)
     ...
   }
   for _, route := range routes {
      ws.Route(route)
   }
```
#### Create Handler
* /vendor/k8s.io/apiserver/pkg/endpoints/installer.go:

```
func restfulCreateNamedResource(r rest.NamedCreater, scope handlers.RequestScope, admit admission.Interface) restful.RouteFunction {
	return func(req *restful.Request, res *restful.Response) {
		handlers.CreateNamedResource(r, scope, admit)(res.ResponseWriter, req.Request)
	}
}

```
* /vendor/k8s.io/apiserver/pkg/endpoints/handlers/create.go:

```
// CreateNamedResource returns a function that will handle a resource creation with name.
func CreateNamedResource(r rest.NamedCreater, scope RequestScope, admission admission.Interface) http.HandlerFunc {
	return createHandler(r, scope, admission, true)
}

func createHandler(r rest.NamedCreater, scope RequestScope, admit admission.Interface, includeName bool) http.HandlerFunc { return func(w http.ResponseWriter, req *http.Request) {
  ...
  original := r.New()
  ...
  transformResponseObject(ctx, scope, req, w, code, result)
}
```
* /vendor/k8s.io/apiserver/pkg/endpoints/handlers/response.go:

```
// transformResponseObject takes an object loaded from storage and performs any necessary transformations.
// Will write the complete response object.
func transformResponseObject(ctx context.Context, scope RequestScope, req *http.Request, w http.ResponseWriter, statusCode int, result runtime.Object) {
  ...
  responsewriters.WriteObject(statusCode, scope.Kind.GroupVersion(), scope.Serializer, result, w, req)
}
```
### 回到最初的问题
最终, handler调用的是rest.Creater.New()
而creater声明的位于/vendor/k8s.io/apiserver/pkg/endpoints/installer.go中
```
 creater, isCreater := storage.(rest.Creater)
```
这里, 想要知道handler最终调用的是哪里定义的方法, 我们需要分析storage的来源
#### 第一步: 链路分析
![creater链路分析](/assets/apiserver-register-04.jpg)
基于图不难分析得出以下结论：
```
// apiGroupInfo.VersionedResourcesStorageMap
storage = apiGroupInfo.VersionedResourcesStorageMap[groupVersion.Version][path]
creater, isCreater := storage.(rest.Creater)
```
接着寻找apiGroupInfo初始化的位置
### 第二步: apiGroupInfo 初始化
* pkg/master/master.go
```
func (m *Master) InstallLegacyAPI(c *Config, restOptionsGetter generic.RESTOptionsGetter, legacyRESTStorageProvider corerest.LegacyRESTStorageProvider) {
    legacyRESTStorage, apiGroupInfo, err := legacyRESTStorageProvider.NewLegacyRESTStorage(restOptionsGetter) // => next
    m.GenericAPIServer.InstallLegacyAPIGroup(genericapiserver.DefaultLegacyAPIPrefix, &apiGroupInfo)
}
```
* pkg/registry/core/rest/storage_core.go
```
func (c LegacyRESTStorageProvider) NewLegacyRESTStorage(restOptionsGetter generic.RESTOptionsGetter) (LegacyRESTStorage, genericapiserver.APIGroupInfo, error) {
  // 初始化: VersionedResourcesStorageMap
    apiGroupInfo := genericapiserver.APIGroupInfo{
        GroupMeta:                    *api.Registry.GroupOrDie(api.GroupName),
        VersionedResourcesStorageMap: map[string]map[string]rest.Storage{},
        Scheme:                      api.Scheme,
        ParameterCodec:              api.ParameterCodec,
        NegotiatedSerializer:        api.Codecs,
        SubresourceGroupVersionKind: map[string]schema.GroupVersionKind{},
    }
    // ......

    // 初始化了一个restStorage的map，然后赋值给APIGroupInfo.VersionedResourcesStorageMap["v1"]
    restStorageMap := map[string]rest.Storage{
        "pods":             podStorage.Pod,
        "pods/attach":      podStorage.Attach,
        "pods/status":      podStorage.Status,
        "services":        serviceRest.Service,
        "nodes":        nodeStorage.Node,
        .....
    }

    apiGroupInfo.VersionedResourcesStorageMap["v1"] = restStorageMap

    return restStorage, apiGroupInfo, nil
}
```
即
```
apiGroupInfo.VersionedResourcesStorageMap["v1"] = map[string]rest.Storage{
        "pods":             podStorage.Pod,
        "pods/attach":      podStorage.Attach,
        "pods/status":      podStorage.Status,
        "services":        serviceRest.Service,
        "nodes":        nodeStorage.Node,
        .....
    }
```
此时, 根据上一节分析得出的结论：
```
// apiGroupInfo.VersionedResourcesStorageMap
storage = apiGroupInfo.VersionedResourcesStorageMap[groupVersion.Version][path]
creater, isCreater := storage.(rest.Creater)
```
我们可以得到
```
storage = apiGroupInfo.VersionedResourcesStorageMap["v1"]["pods"]
// equals
storage = podStorage.Pod
creater, isCreater := (podStorage.Pod).(rest.Creater)
```
然后, 我们再看下podStorage.Pod的实现
### 第三步: podStorage.Pod

* pkg/registry/core/pod/storage/storage.go
```
type PodStorage struct {
  Pod *REST ...
 } 
// REST implements a RESTStorage for pods
 type REST struct {
 *genericregistry.Store // => NOTE
 proxyTransport http.RoundTripper
 }
```
即, PodStorage.Pod 类型是 REST, 而REST.genericregistry.Store, 其定义文件中存在

* vendor/k8s.io/apiserver/pkg/registry/generic/registry/store.go
```
// New implements RESTStorage.New. func (e *Store) New() runtime.Object { 
return e.NewFunc() } func (e *Store) Create(ctx genericapirequest.Context, obj runtime.Object) (runtime.Object, error) { }
```
即,
```
storage = apiGroupInfo.VersionedResourcesStorageMap["v1"]["pods"] // equals storage = podStorage.Pod creater, isCreater := (podStorage.Pod).(rest.Creater) // equals creater, isCreater := (REST).(rest.Creater) creater, isCreater := (*genericregistry.Store).(rest.Creater)
```

### 第四步: creater.New()
![creater.New分析](/assets/apiserver-register-05.jpg)

* vendor/k8s.io/apiserver/pkg/registry/generic/registry/store.go
```
// New implements RESTStorage.New. func (e *Store) New() runtime.Object { return e.NewFunc() }
```
* pkg/registry/core/pod/storage/storage.go
```
func NewStorage(optsGetter generic.RESTOptionsGetter, k client.ConnectionInfoGetter, proxyTransport http.RoundTripper, podDisruptionBudgetClient policyclient.PodDisruptionBudgetsGetter) PodStorage {
 store := &genericregistry.Store{
    NewFunc: func() runtime.Object {
         return &api.Pod{}
    },
     ....
    }
 }
 // pkg/api/types.go
 type Pod struct {
 metav1.TypeMeta
 // +optional metav1.ObjectMeta
 // Spec defines the behavior of a pod.
 // +optional
 Spec PodSpec
 // Status represents the current information about a pod. This data may not be up
 // to date.
 // +optional
 Status PodStatus
 }
```