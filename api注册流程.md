## API注册流程
先来了解下API注册流程概要
![API注册流程概要](/assets/apiserver-register-01.jpg)
开始API注册
* k8s.io/kubernetes/pkg/master/master.go:
```
func (c completedConfig) New(delegationTarget genericapiserver.DelegationTarget) (*Master, error) { 
  ...
  m.InstallLegacyAPI(&c, c.GenericConfig.RESTOptionsGetter, legacyRESTStorageProvider)
  ...
  m.InstallAPIs(c.ExtraConfig.APIResourceConfigSource, c.GenericConfig.RESTOptionsGetter, restStorageProviders...)
  ...
```
### Install /api
* k8s.io/kubernetes/pkg/master/master.go:
```
func (m *Master) InstallLegacyAPI(c *completedConfig, restOptionsGetter generic.RESTOptionsGetter, legacyRESTStorageProvider corerest.LegacyRESTStorageProvider) {
 legacyRESTStorage, apiGroupInfo, err := 		  legacyRESTStorageProvider.NewLegacyRESTStorage(restOptionsGetter)

m.GenericAPIServer.InstallLegacyAPIGroup(genericapiserver.DefaultLegacyAPIPrefix, &apiGroupInfo)
...
}