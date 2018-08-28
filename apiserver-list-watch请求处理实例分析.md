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

