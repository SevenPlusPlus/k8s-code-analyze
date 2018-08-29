## Apiserver端List-Watch请求处理实例
前面Storage实现解析章节分析了对ApiServer对资源对象的存储以及变更event的watch分发等的过程。Apiserver接收到一个来自于其它组件（如kubelet）对一个Pod资源的WATCHLIST请求，其对应的handler函数在哪？ Apiserver的处理逻辑又是怎样的呢？

回到ApiServer的Restful Api注册过程的registerResourceHandlers方法实现
* vendor/k8s.io/apiserver/pkg/endpoints/installer.go:

```
func (a *APIInstaller) registerResourceHandlers(path string, storage rest.Storage, ws *restful.WebService) (*metav1.APIResource, error) {
   ...
   
   switch action.Verb {
       case "WATCHLIST": // Watch all resources of a kind.
			handler := metrics.InstrumentRouteFunc(action.Verb, resource, subresource, requestScope, restfulListResource(lister, watcher, reqScope, true, a.minRequestTimeout))
			route := ws.GET(action.Path).To(handler).
				Doc(doc).
				Param(ws.QueryParameter("pretty", "If 'true', then the output is pretty printed.")).
				Operation("watch"+namespaced+kind+strings.Title(subresource)+"List"+operationSuffix).
				Produces(allMediaTypes...).
				Returns(http.StatusOK, "OK", versionedWatchEvent).
				Writes(versionedWatchEvent)
   }
}
```
请求处理的handler为restfulListResource

```
func restfulListResource(r rest.Lister, rw rest.Watcher, scope handlers.RequestScope, forceWatch bool, minRequestTimeout time.Duration) restful.RouteFunction {
	return func(req *restful.Request, res *restful.Response) {
		handlers.ListResource(r, rw, scope, forceWatch, minRequestTimeout)(res.ResponseWriter, req.Request)
	}
}
```
继续分析ListResource：
* vendor/k8s.io/apiserver/pkg/endpoints/handlers/get.go

ListResource核心步骤如下：
* 调用watcher, err := rw.Watch(ctx, &opts) ，生成一个Watcher接口对象。关于watcher，每种resource都不一样，是根据registerResourceHandlers时传入的rest.Storage不同而不同。
* 创建好Watcher接口对象以后，函数会调用serveWatch(watcher, scope, req, res, timeout)处理传过来的event

```
func ListResource(r rest.Lister, rw rest.Watcher, scope RequestScope, forceWatch bool, minRequestTimeout time.Duration) http.HandlerFunc {
	return func(w http.ResponseWriter, req *http.Request) {
		
		namespace, err := scope.Namer.Namespace(req)
		ctx := req.Context()
		ctx = request.WithNamespace(ctx, namespace)

		opts := metainternalversion.ListOptions{}
               /*此处forceWatch传入的为true*/
		if opts.Watch || forceWatch {

			watcher, err := rw.Watch(ctx, &opts)
			
			requestInfo, _ := request.RequestInfoFrom(ctx)
			metrics.RecordLongRunning(req, requestInfo, func() {
				serveWatch(watcher, scope, req, w, timeout)
			})
			return
		}

		// Log only long List requests (ignore Watch).
		...
	}
}
```
结合调用上下文分析不难得知：rw rest.Watcher其实就是一个Storage即registerResourceHandlers时传入的rest.Storage。
就Pod这种resource来说，其对应的podStorage来源于pkg/registry/core/rest/storage_core.go中func (c LegacyRESTStorageProvider) NewLegacyRESTStorage(restOptionsGetter generic.RESTOptionsGetter) (LegacyRESTStorage, genericapiserver.APIGroupInfo, error) 中

```
restStorageMap := map[string]rest.Storage{
		"pods":             podStorage.Pod,
		"pods/attach":      podStorage.Attach,
		"pods/status":      podStorage.Status,
		"pods/log":         podStorage.Log,
```
因此rw=podStorage.Pod，其定义位于: pkg/registry/core/pod/storage/storage.go
而podStorage.Pod类型为*REST， 而REST继承了*genericregistry.Store类型。
rw.Watch(ctx, &opts)本质上调用了*genericregistry.Store的Watch方法

#### type Store struct
接着上面，继续查看type Store struct，其定义了各种Resource的公共Restful接口实现。其定义在vendor/k8s.io/apiserver/pkg/registry/generic/registry/store.go:

```
// Watch makes a matcher for the given label and field, and calls
// WatchPredicate. If possible, you should customize PredicateFunc to produce
// a matcher that matches by key. SelectionPredicate does this for you
// automatically.
/*
 Watch 根据指定的label and field进行匹配，调用WatchPredicate函数。
	   如果可能，应该自定义PredicateFunc。
	   SelectionPredicate 会完成该功能。
*/
func (e *Store) Watch(ctx context.Context, options *metainternalversion.ListOptions) (watch.Interface, error) {
	label := labels.Everything()
	if options != nil && options.LabelSelector != nil {
		label = options.LabelSelector
	}
	field := fields.Everything()
	if options != nil && options.FieldSelector != nil {
		field = options.FieldSelector
	}
	predicate := e.PredicateFunc(label, field)

	resourceVersion := ""
	if options != nil {
		resourceVersion = options.ResourceVersion
		predicate.IncludeUninitialized = options.IncludeUninitialized
	}
	return e.WatchPredicate(ctx, predicate, resourceVersion)
}

// WatchPredicate starts a watch for the items that matches.
func (e *Store) WatchPredicate(ctx context.Context, p storage.SelectionPredicate, resourceVersion string) (watch.Interface, error) {
	if name, ok := p.MatchesSingle(); ok {
		if key, err := e.KeyFunc(ctx, name); err == nil {
     /*
     /*
				调用e.Storage.Watch(ctx, key, resourceVersion, p)

				e=podStorage.pod, /pkg/registry/core/pod/etcd/etcd.go
				那么e.Storage就是podStorage.pod的Storage，即store.Storage
					==>Storage:     storageInterface,
				所以e.Storage.Watch函数定义在/pkg/storage/cacher.go
				==>func (c *Cacher) Watch(ctx context.Context, key string, resourceVersion string, pred SelectionPredicate)

				开启了cache的时候，e.Storage is: *storage.Cacher
			*/

     */
			w, err := e.Storage.Watch(ctx, key, resourceVersion, p)
			if err != nil {
				return nil, err
			}
			if e.Decorator != nil {
				return newDecoratedWatcher(w, e.Decorator), nil
			}
			return w, nil
		}
		// if we cannot extract a key based on the current context, the
		// optimization is skipped
	}

	w, err := e.Storage.WatchList(ctx, e.KeyRootFunc(ctx), resourceVersion, p)
	if err != nil {
		return nil, err
	}
	if e.Decorator != nil {
		return newDecoratedWatcher(w, e.Decorator), nil
	}
	return w, nil
}
```







