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

}


