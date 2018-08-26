## API注册流程
先来了解下API注册流程概要
![API注册流程概要](/assets/apiserver-register-01.jpg)
开始API注册
* /pkg/master/master.go:
```
func (c completedConfig) New(delegationTarget genericapiserver.DelegationTarget) (*Master, error) { 
  ...
  m.InstallLegacyAPI(&c, c.GenericConfig.RESTOptionsGetter, legacyRESTStorageProvider)
  ...
  m.InstallAPIs(c.ExtraConfig.APIResourceConfigSource, c.GenericConfig.RESTOptionsGetter, restStorageProviders...)
  ...
```
### Install /api
* /pkg/master/master.go:

```
func (m *Master) InstallLegacyAPI(c *completedConfig, restOptionsGetter generic.RESTOptionsGetter, legacyRESTStorageProvider corerest.LegacyRESTStorageProvider) {

 legacyRESTStorage, apiGroupInfo, err := 		
       legacyRESTStorageProvider.NewLegacyRESTStorage(restOptionsGetter)

  m.GenericAPIServer.InstallLegacyAPIGroup(
        genericapiserver.DefaultLegacyAPIPrefix, &apiGroupInfo)
...

}
```
* /vendor/k8s.io/apiserver/pkg/server/genericapiserver.go:

```
func (s *GenericAPIServer) InstallLegacyAPIGroup(apiPrefix string, apiGroupInfo *APIGroupInfo) error {
  if err := s.installAPIResources(apiPrefix, apiGroupInfo); err != nil {
		return err
	}
  ...
```
### Install /apis
* /pkg/master/master.go:

```
func (m *Master) InstallAPIs(apiResourceConfigSource serverstorage.APIResourceConfigSource, restOptionsGetter generic.RESTOptionsGetter, restStorageProviders ...RESTStorageProvider) {
 apiGroupsInfo := []genericapiserver.APIGroupInfo{}

 for i := range apiGroupsInfo {
		if err := m.GenericAPIServer.InstallAPIGroup(&apiGroupsInfo[i]); err != nil {
			glog.Fatalf("Error in registering group versions: %v", err)
		}
	}
```
* /vendor/k8s.io/apiserver/pkg/server/genericapiserver.go:

```
func (s *GenericAPIServer) InstallAPIGroup(apiGroupInfo *APIGroupInfo) error {

  if err := s.installAPIResources(APIGroupPrefix, apiGroupInfo); err != nil { 
     return err
  }
  ...
}

```
### all to installAPIResources
* /vendor/k8s.io/apiserver/pkg/server/genericapiserver.go:

```
func (s *GenericAPIServer) installAPIResources(apiPrefix string, apiGroupInfo *APIGroupInfo) error {
  for _, groupVersion := range apiGroupInfo.PrioritizedVersions {
    apiGroupVersion.InstallREST(s.Handler.GoRestfulContainer)
    ...
  }
}

```
* /vendor/k8s.io/apiserver/pkg/endpoints/groupversion.go:

```

```

