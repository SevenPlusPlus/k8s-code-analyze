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
根据前面章节《ApiServer Storage的使用中》对registry.Store的分析，不难得出此处e.Storage.Watch其实是调用StorageWithCacher方法构建的Cacher对象的Watch方法

w, err := e.Storage.Watch(ctx, key, resourceVersion, p)最终调用的是定义在vendor/k8s.io/apiserver/pkg/storage/cacher/cacher.go的func (c *Cacher) Watch。 分析其流程如下：
* 传入了一个filterWithAttrsFunction，apiserver的watch是带过滤功能的，就是由这个filter实现的。
* 调用newCacheWatcher生成一个watcher，
* 将这个watcher插入到cacher.watchers中去， 也就是说WatchCache中存储着各个订阅者。

```
// Watch implements storage.Interface.
func (c *Cacher) Watch(ctx context.Context, key string, resourceVersion string, pred storage.SelectionPredicate) (watch.Interface, error) {
	watchRV, err := c.versioner.ParseResourceVersion(resourceVersion)
	...
	watcher := newCacheWatcher(watchRV, chanSize, initEvents, filterWithAttrsFunction(key, pred), forget, c.versioner)

	c.watchers.addWatcher(watcher, c.watcherIdx, triggerValue, triggerSupported)
	c.watcherIdx++
	return watcher, nil
}

```
* newCacheWatcher

```
func newCacheWatcher(resourceVersion uint64, chanSize int, initEvents []*watchCacheEvent, filter filterWithAttrsFunc, forget func(bool), versioner storage.Versioner) *cacheWatcher {
       //生成一个新的CacheWatcher
	watcher := &cacheWatcher{
		input:     make(chan *watchCacheEvent, chanSize),
		result:    make(chan watch.Event, chanSize),
		done:      make(chan struct{}),
		filter:    filter,
		stopped:   false,
		forget:    forget,
		versioner: versioner,
	}
      //每一个Watcher都会有一个协程处理对其内部input channel的消费
	go watcher.process(initEvents, resourceVersion)
	return watcher
}
```
每一个cacheWatcher都会有一个协程来消费其内部的 channel input。 而input channel的生产者则是前一节《ApiServer Storage实现解析》中已经介绍过的Cacher的dispatchEvents方法分发的。

```
func (c *cacheWatcher) process(initEvents []*watchCacheEvent, resourceVersion uint64) {
	defer utilruntime.HandleCrash()

	for _, event := range initEvents {
		c.sendWatchCacheEvent(event)
	}

	defer close(c.result)
	defer c.Stop()
	for {
		event, ok := <-c.input
		if !ok {
			return
		}
		// only send events newer than resourceVersion
		if event.ResourceVersion > resourceVersion {
			c.sendWatchCacheEvent(event)
		}
	}
}

// NOTE: sendWatchCacheEvent is assumed to not modify <event> !!!
func (c *cacheWatcher) sendWatchCacheEvent(event *watchCacheEvent) {
     //sendWatchCacheEvent会调用c.filter函数对watch的结果进行过滤

	curObjPasses := event.Type != watch.Deleted && c.filter(event.Key, event.ObjLabels, event.ObjFields, event.ObjUninitialized)
	oldObjPasses := false
	if event.PrevObject != nil {
		oldObjPasses = c.filter(event.Key, event.PrevObjLabels, event.PrevObjFields, event.PrevObjUninitialized)
	}
	if !curObjPasses && !oldObjPasses {
		// Watcher is not interested in that object.
		return
	}

     //然后将过滤后的结果包装成watchEvent
	var watchEvent watch.Event
	switch {
	case curObjPasses && !oldObjPasses:
		watchEvent = watch.Event{Type: watch.Added, Object: event.Object.DeepCopyObject()}
	case curObjPasses && oldObjPasses:
		watchEvent = watch.Event{Type: watch.Modified, Object: event.Object.DeepCopyObject()}
	case !curObjPasses && oldObjPasses:
		// return a delete event with the previous object content, but with the event's resource version
		oldObj := event.PrevObject.DeepCopyObject()
		if err := c.versioner.UpdateObject(oldObj, event.ResourceVersion); err != nil {
			utilruntime.HandleError(fmt.Errorf("failure to version api object (%d) %#v: %v", event.ResourceVersion, oldObj, err))
		}
		watchEvent = watch.Event{Type: watch.Deleted, Object: oldObj}
	}

	
	select {
	case <-c.done:
		return
	default:
	}

    //然后将过滤后的结果包装成watchEvent，发送到c.result这个channel
	select {
	case c.result <- watchEvent:
	case <-c.done:
	}
}
```

最后再来看看c.result的消费者 
回顾前面ListResource的处理流程中第二步，创建好Watcher接口对象以后，函数会调用serveWatch(watcher, scope, req, res, timeout)处理传过来的event。
* vendor/k8s.io/apiserver/pkg/endpoints/handlers/watch.go:
分析其流程如下:
* 设置返回数据包的编码器
* 构建WatchServer
* ServeHTTP开始服务客户端的event watch请求

```
// serveWatch handles serving requests to the server
func serveWatch(watcher watch.Interface, scope RequestScope, req *http.Request, w http.ResponseWriter, timeout time.Duration) {
	// negotiate for the stream serializer
	serializer, err := negotiation.NegotiateOutputStreamSerializer(req, scope.Serializer)
	
	framer := serializer.StreamSerializer.Framer
	streamSerializer := serializer.StreamSerializer.Serializer
	embedded := serializer.Serializer
	
	encoder := scope.Serializer.EncoderForVersion(streamSerializer, scope.Kind.GroupVersion())

	useTextFraming := serializer.EncodesAsText

	// find the embedded serializer matching the media type
	embeddedEncoder := scope.Serializer.EncoderForVersion(embedded, scope.Kind.GroupVersion())

	// TODO: next step, get back mediaTypeOptions from negotiate and return the exact value here
	mediaType := serializer.MediaType
	if mediaType != runtime.ContentTypeJSON {
		mediaType += ";stream=watch"
	}

	ctx := req.Context()
	requestInfo, ok := request.RequestInfoFrom(ctx)

	server := &WatchServer{
		Watching: watcher,
		Scope:    scope,

		UseTextFraming:  useTextFraming,
		MediaType:       mediaType,
		Framer:          framer,
		Encoder:         encoder,
		EmbeddedEncoder: embeddedEncoder,
		Fixup: func(obj runtime.Object) {
			if err := setSelfLink(obj, requestInfo, scope.Namer); err != nil {
				utilruntime.HandleError(fmt.Errorf("failed to set link for object %v: %v", reflect.TypeOf(obj), err))
			}
		},

		TimeoutFactory: &realTimeoutFactory{timeout},
	}

	server.ServeHTTP(w, req)
}
```

WatchServer定义如下：

```
// WatchServer serves a watch.Interface over a websocket or vanilla HTTP.
type WatchServer struct {
	Watching watch.Interface
	Scope    RequestScope

	// true if websocket messages should use text framing (as opposed to binary framing)
	UseTextFraming bool
	// the media type this watch is being served with
	MediaType string
	// used to frame the watch stream
	Framer runtime.Framer
	// used to encode the watch stream event itself
	Encoder runtime.Encoder
	// used to encode the nested object in the watch stream
	EmbeddedEncoder runtime.Encoder
	Fixup           func(runtime.Object)

	TimeoutFactory TimeoutFactory
}
```

ServeHTTP处理客户端watch请求

```
// ServeHTTP serves a series of encoded events via HTTP with Transfer-Encoding: chunked
// or over a websocket connection.
func (s *WatchServer) ServeHTTP(w http.ResponseWriter, req *http.Request) {

	if wsstream.IsWebSocketRequest(req) {
		w.Header().Set("Content-Type", s.MediaType)
		websocket.Handler(s.HandleWS).ServeHTTP(w, req)
		return
	}

	cn, ok := w.(http.CloseNotifier)
	
	flusher, ok := w.(http.Flusher)

	framer := s.Framer.NewFrameWriter(w)
	
	e := streaming.NewEncoder(framer, s.Encoder)

	// begin the stream
	w.Header().Set("Content-Type", s.MediaType)
	w.Header().Set("Transfer-Encoding", "chunked")
	w.WriteHeader(http.StatusOK)
	flusher.Flush()

	var unknown runtime.Unknown
	internalEvent := &metav1.InternalEvent{}
	buf := &bytes.Buffer{}
	ch := s.Watching.ResultChan()
	for {
		select {
		case <-cn.CloseNotify():
			return
		case <-timeoutCh:
			return
		case event, ok := <-ch:
			if !ok {
				// End of results.
				return
			}

			obj := event.Object
			s.Fixup(obj)
			if err := s.EmbeddedEncoder.Encode(obj, buf); err != nil {
				return
			}

			// ContentType is not required here because we are defaulting to the serializer
			// type
			unknown.Raw = buf.Bytes()
			event.Object = &unknown

			// create the external type directly and encode it.  Clients will only recognize the serialization we provide.
			// The internal event is being reused, not reallocated so its just a few extra assignments to do it this way
			// and we get the benefit of using conversion functions which already have to stay in sync
			outEvent := &metav1.WatchEvent{}
			*internalEvent = metav1.InternalEvent(event)
			err := metav1.Convert_v1_InternalEvent_To_v1_WatchEvent(internalEvent, outEvent, nil)
			if err != nil {
				return
			}
			if err := e.Encode(outEvent); err != nil {
				// client disconnect.
				return
			}
			if len(ch) == 0 {
				flusher.Flush()
			}

			buf.Reset()
		}
	}
}
```

至此就可以说kube-apiserver watch的结果已经发送给订阅方。 订阅方是指kube-controller-manager、proxy、scheduler、kubelet这些组件，向kube-apiserver订阅etcd的信息。










