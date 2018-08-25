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
	}


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





