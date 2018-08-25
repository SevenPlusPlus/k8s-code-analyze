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

* vendor/k8s.io/apiserver/pkg/server/config.go
```
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


