## ApiServer服务启动
### 服务启动流程
![ApiServer启动流程](/assets/apiserver-start-01.jpg)
* cmd/kube-apiserver/apiserver.go
```
func main() {
command := app.NewAPIServerCommand(server.SetupSignalHandler())
err := command.Execute()
```
* cmd/kube-apiserver/app/server.go

```
func Run(completeOptions completedServerRunOptions, stopCh <-chan struct{}) error {
  server, err := CreateServerChain(completeOptions, stopCh)
  return server.PrepareRun().Run(stopCh)
}

func CreateServerChain(completedOptions completedServerRunOptions, stopCh <-chan struct{}) (*genericapiserver.GenericAPIServer, error) {
  kubeAPIServerConfig, sharedInformers, versionedInformers, insecureServingOptions, serviceResolver, pluginInitializer, admissionPostStartHook, err := CreateKubeAPIServerConfig(completedOptions, nodeTunneler, proxyTransport)

  kubeAPIServer, err := CreateKubeAPIServer(kubeAPIServerConfig, apiExtensionsServer.GenericAPIServer, sharedInformers, versionedInformers, admissionPostStartHook)

  return kubeAPIServer, nil
}
```
* vendor/k8s.io/apiserver/pkg/server/genericapiserver.go

```
// PrepareRun does post API installation setup steps.func (s *GenericAPIServer) PrepareRun() preparedGenericAPIServer {
  routes.Swagger{Config: s.swaggerConfig}.Install(s.Handler.GoRestfulContainer)
  s.installHealthz()
  return preparedGenericAPIServer{s}
}
func (s preparedGenericAPIServer) Run(stopCh <-chan struct{}) error {
	err := s.NonBlockingRun(stopCh)
}

// NonBlockingRun spawns the secure http server. An error is
// returned if the secure port cannot be listened on.
func (s preparedGenericAPIServer) NonBlockingRun(stopCh <-chan struct{}) error {
  if s.SecureServingInfo != nil && s.Handler != nil {
		if err := s.SecureServingInfo.Serve(s.Handler, s.ShutdownTimeout, internalStopCh); err != nil {
			close(internalStopCh)
			return err
		}
	}
}
```
* vendor/k8s.io/apiserver/pkg/server/serve.go

```
// serveSecurely runs the secure http server. It fails only if certificates cannot
// be loaded or the initial listen call fails. The actual server loop (stoppable by closing
// stopCh) runs in a go routine, i.e. serveSecurely does not block.

func (s *SecureServingInfo) Serve(handler http.Handler, shutdownTimeout time.Duration, stopCh <-chan struct{}) error {

 secureServer := &http.Server{
   Addr: s.Listener.Addr().String(),
   Handler: handler,
 }
 return RunServer(secureServer, s.Listener, shutdownTimeout, stopCh)
}

// RunServer listens on the given port if listener is not given,
// then spawns a go-routine continuously serving
// until the stopCh is closed. This function does not block.
// TODO: make private when insecure serving is gone from the kube-apiserver
func RunServer(
 server *http.Server,
 ln net.Listener,
 shutDownTimeout time.Duration,
 stopCh <-chan struct{},
) error {
  go func() {
   var listener net.Listener
   listener = tcpKeepAliveListener{ln.(*net.TCPListener)}
   if server.TLSConfig != nil {
      listener = tls.NewListener(listener, server.TLSConfig)
   }
   err := server.Serve(listener)
  }()

 return nil
}

```

### Container初始化
* cmd/kube-apiserver/app/server.go
```
func CreateKubeAPIServerConfig(...) {
  genericConfig, ... = buildGenericConfig(s.ServerRunOptions, proxyTransport)
}

func buildGenericConfig(s *options.ServerRunOptions, proxyTransport *http.Transport,)(...) {
  genericConfig = genericapiserver.NewConfig(legacyscheme.Codecs)
}


* vendor/k8s.io/apiserver/pkg/server/config.go

```
/ NewConfig returns a Config struct with the default values
func NewConfig(codecs serializer.CodecFactory) *Config {
 return &Config{
 Serializer: codecs,
 BuildHandlerChainFunc: DefaultBuildHandlerChain,
 HandlerChainWaitGroup: new(utilwaitgroup.SafeWaitGroup),
 LegacyAPIGroupPrefixes: sets.NewString(DefaultLegacyAPIPrefix),
 ... 
 }
}

func DefaultBuildHandlerChain(apiHandler http.Handler, c *Config) http.Handler {
 handler := genericapifilters.WithAuthorization(apiHandler, c.Authorization.Authorizer, c.Serializer)
 handler = genericfilters.WithMaxInFlightLimit(handler, c.MaxRequestsInFlight, c.MaxMutatingRequestsInFlight, c.LongRunningFunc)
 handler = genericapifilters.WithImpersonation(handler, c.Authorization.Authorizer, c.Serializer)
  handler = genericapifilters.WithAuthentication(handler, c.Authentication.Authenticator, failedHandler)
 ... 
 return handler
}



// New creates a new server which logically combines the handling chain with the passed server.
func (c completedConfig) New(name string, delegationTarget DelegationTarget) (*GenericAPIServer, error) {
  handlerChainBuilder := func(handler http.Handler) http.Handler {
		return c.BuildHandlerChainFunc(handler, c.Config)
  }
  apiServerHandler := NewAPIServerHandler(name, c.Serializer, handlerChainBuilder, delegationTarget.UnprotectedHandler())
  s := &GenericAPIServer{
		legacyAPIGroupPrefixes: c.LegacyAPIGroupPrefixes,
		SecureServingInfo: c.SecureServing,

		Handler: apiServerHandler,
   }
```
* vendor/k8s.io/apiserver/pkg/server/handler.go
ApiServerHandler为最终APIServer中http.Server的handler。

```
// APIServerHandlers holds the different http.Handlers used by the API server.
// This includes the full handler chain, the director (which chooses between gorestful and nonGoRestful,
// the gorestful handler (used for the API) which falls through to the nonGoRestful handler on unregistered paths,
// and the nonGoRestful handler (which can contain a fallthrough of its own)
// FullHandlerChain -> Director -> {GoRestfulContainer,NonGoRestfulMux} based on inspection of registered web services
type APIServerHandler struct {
 // FullHandlerChain is the one that is eventually served with. It should include the full filter chain and then call the Director.
 FullHandlerChain http.Handler

 // The registered APIs. InstallAPIs uses this. Other servers probably shouldn't access this directly.
 GoRestfulContainer *restful.Container

 // NonGoRestfulMux is the final HTTP handler in the chain.
 // It comes after all filters and the API handling
 // This is where other servers can attach handler to various parts of the chain.
 NonGoRestfulMux *mux.PathRecorderMux

 // Director is here so that we can properly handle fall through and proxy cases.
 // This looks a bit bonkers, but here's what's happening. We need to have /apis handling registered in gorestful in order to have
 // swagger generated for compatibility. Doing that with `/apis` as a webservice, means that it forcibly 404s (no defaulting allowed)
 // all requests which are not /apis or /apis/. We need those calls to fall through behind goresful for proper delegation. Trying to
 // register for a pattern which includes everything behind it doesn't work because gorestful negotiates for verbs and content encoding
 // and all those things go crazy when gorestful really just needs to pass through. In addition, openapi enforces unique verb constraints
 // which we don't fit into and it still muddies up swagger. Trying to switch the webservices into a route doesn't work because the
 // containing webservice faces all the same problems listed above. 
// This leads to the crazy thing done here. Our mux does what we need, so we'll place it in front of gorestful. It will introspect to
 // decide if the route is likely to be handled by goresful and route there if needed. Otherwise, it goes to PostGoRestful mux in
 // order to handle "normal" paths and delegation. Hopefully no API consumers will ever have to deal with this level of detail. I think
 // we should consider completely removing gorestful.
 // Other servers should only use this opaquely to delegate to an API server.
 Director http.Handler
}

```
APIServerHandler和Director都实现了http.Handler接口ServeHTTP方法：
```
// ServeHTTP makes it an http.Handler
func (a *APIServerHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
 a.FullHandlerChain.ServeHTTP(w, r)
}

func (d director) ServeHTTP(w http.ResponseWriter, req *http.Request) {
 path := req.URL.Path
 // check to see if our webservices want to claim this path
 for _, ws := range d.goRestfulContainer.RegisteredWebServices() {
 switch {
   case ws.RootPath() == "/apis":
        if path == "/apis" || path == "/apis/" {  
             d.goRestfulContainer.Dispatch(w, req) 
 	     return
 	} 
   case strings.HasPrefix(path, ws.RootPath()):
 // ensure an exact match or a path boundary match
      if len(path) == len(ws.RootPath()) || path[len(ws.RootPath())] == '/' {
             d.goRestfulContainer.Dispatch(w, req)
             return
      }
  }
 } // if we didn't find a match, then we just skip gorestful altogether
  glog.V(5).Infof("%v: %v %q satisfied by nonGoRestful", d.name, req.Method, path)
  d.nonGoRestfulMux.ServeHTTP(w, req)
}

```

```
func NewAPIServerHandler(name string, s runtime.NegotiatedSerializer, handlerChainBuilder HandlerChainBuilderFn, notFoundHandler http.Handler) *APIServerHandler {
	nonGoRestfulMux := mux.NewPathRecorderMux(name)
	if notFoundHandler != nil {
		nonGoRestfulMux.NotFoundHandler(notFoundHandler)
	}
	gorestfulContainer := restful.NewContainer()
	gorestfulContainer.ServeMux = http.NewServeMux()
	gorestfulContainer.Router(restful.CurlyRouter{}) // e.g. for proxy/{kind}/{name}/{*}
	
	director := director{
		name:               name,
		goRestfulContainer: gorestfulContainer,
		nonGoRestfulMux:    nonGoRestfulMux,
	}


	return &APIServerHandler{
		FullHandlerChain:   handlerChainBuilder(director),
		GoRestfulContainer: gorestfulContainer,
		NonGoRestfulMux:    nonGoRestfulMux,
		Director:           director,
	}
}

```

### 整体思维脑图
![apiserver脑图](/assets/apiserver-start-02.jpg)



