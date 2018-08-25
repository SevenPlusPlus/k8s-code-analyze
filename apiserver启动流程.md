## ApiServer服务启动
启动流程
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
// serveSecurely runs the secure http server. It fails only if certificates cannot// be loaded or the initial listen call fails. The actual server loop (stoppable by closing// stopCh) runs in a go routine, i.e. serveSecurely does not block.func (s *SecureServingInfo) Serve(handler http.Handler, shutdownTimeout time.Duration, stopCh <-chan struct{}) error { if s.Listener == nil { return fmt.Errorf("listener must not be nil") } secureServer := &http.Server{   Addr: s.Listener.Addr().String(), Handler: handler, MaxHeaderBytes: 1 << 20, TLSConfig: &tls.Config{ NameToCertificate: s.SNICerts, // Can't use SSLv3 because of POODLE and BEAST // Can't use TLSv1.0 because of POODLE and BEAST using CBC cipher // Can't use TLSv1.1 because of RC4 cipher usage MinVersion: tls.VersionTLS12, // enable HTTP2 for go's 1.7 HTTP Server NextProtos: []string{"h2", "http/1.1"}, }, }

```


