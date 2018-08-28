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

```
func ListResource(r rest.Lister, rw rest.Watcher, scope RequestScope, forceWatch bool, minRequestTimeout time.Duration) http.HandlerFunc {
	return func(w http.ResponseWriter, req *http.Request) {
		
		namespace, err := scope.Namer.Namespace(req)
		if err != nil {
			scope.err(err, w, req)
			return
		}

		// Watches for single objects are routed to this function.
		// Treat a name parameter the same as a field selector entry.
		hasName := true
		_, name, err := scope.Namer.Name(req)
		if err != nil {
			hasName = false
		}

		ctx := req.Context()
		ctx = request.WithNamespace(ctx, namespace)

		opts := metainternalversion.ListOptions{}
		if err := metainternalversion.ParameterCodec.DecodeParameters(req.URL.Query(), scope.MetaGroupVersion, &opts); err != nil {
			err = errors.NewBadRequest(err.Error())
			scope.err(err, w, req)
			return
		}

		// transform fields
		// TODO: DecodeParametersInto should do this.
		if opts.FieldSelector != nil {
			fn := func(label, value string) (newLabel, newValue string, err error) {
				return scope.Convertor.ConvertFieldLabel(scope.Kind, label, value)
			}
			if opts.FieldSelector, err = opts.FieldSelector.Transform(fn); err != nil {
				// TODO: allow bad request to set field causes based on query parameters
				err = errors.NewBadRequest(err.Error())
				scope.err(err, w, req)
				return
			}
		}

		if hasName {
			// metadata.name is the canonical internal name.
			// SelectionPredicate will notice that this is a request for
			// a single object and optimize the storage query accordingly.
			nameSelector := fields.OneTermEqualSelector("metadata.name", name)

			// Note that fieldSelector setting explicitly the "metadata.name"
			// will result in reaching this branch (as the value of that field
			// is propagated to requestInfo as the name parameter.
			// That said, the allowed field selectors in this branch are:
			// nil, fields.Everything and field selector matching metadata.name
			// for our name.
			if opts.FieldSelector != nil && !opts.FieldSelector.Empty() {
				selectedName, ok := opts.FieldSelector.RequiresExactMatch("metadata.name")
				if !ok || name != selectedName {
					scope.err(errors.NewBadRequest("fieldSelector metadata.name doesn't match requested name"), w, req)
					return
				}
			} else {
				opts.FieldSelector = nameSelector
			}
		}

		if opts.Watch || forceWatch {
			if rw == nil {
				scope.err(errors.NewMethodNotSupported(scope.Resource.GroupResource(), "watch"), w, req)
				return
			}
			// TODO: Currently we explicitly ignore ?timeout= and use only ?timeoutSeconds=.
			timeout := time.Duration(0)
			if opts.TimeoutSeconds != nil {
				timeout = time.Duration(*opts.TimeoutSeconds) * time.Second
			}
			if timeout == 0 && minRequestTimeout > 0 {
				timeout = time.Duration(float64(minRequestTimeout) * (rand.Float64() + 1.0))
			}
			glog.V(3).Infof("Starting watch for %s, rv=%s labels=%s fields=%s timeout=%s", req.URL.Path, opts.ResourceVersion, opts.LabelSelector, opts.FieldSelector, timeout)

			watcher, err := rw.Watch(ctx, &opts)
			if err != nil {
				scope.err(err, w, req)
				return
			}
			requestInfo, _ := request.RequestInfoFrom(ctx)
			metrics.RecordLongRunning(req, requestInfo, func() {
				serveWatch(watcher, scope, req, w, timeout)
			})
			return
		}

		// Log only long List requests (ignore Watch).
		defer trace.LogIfLong(500 * time.Millisecond)
		trace.Step("About to List from storage")
		result, err := r.List(ctx, &opts)
		if err != nil {
			scope.err(err, w, req)
			return
		}
		trace.Step("Listing from storage done")
		numberOfItems, err := setListSelfLink(result, ctx, req, scope.Namer)
		if err != nil {
			scope.err(err, w, req)
			return
		}
		trace.Step("Self-linking done")
		// Ensure empty lists return a non-nil items slice
		if numberOfItems == 0 && meta.IsListType(result) {
			if err := meta.SetList(result, []runtime.Object{}); err != nil {
				scope.err(err, w, req)
				return
			}
		}

		transformResponseObject(ctx, scope, req, w, http.StatusOK, result)
		trace.Step(fmt.Sprintf("Writing http response done (%d items)", numberOfItems))
	}
}
```

