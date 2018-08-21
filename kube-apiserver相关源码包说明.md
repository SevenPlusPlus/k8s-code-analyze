## cmd\/kube-apiserver: kube-apiserver启动命令相关

### -apiserver.go:启动命令入口

### -app:启动命令及其执行参数选项

## k8s.io: k8s API服务的注册相关

### -api: 各API Group的资源类型定义及注册

-admission:  admission.k8s.io\/v1beta1

-admissionregistration: admissionregistration.k8s.io\/v1alpha1, admissionregistration.k8s.io\/v1beta1

-apps: apps\/v1, apps\/v1beta1, apps\/v1beta2

-authentication: authentication.k8s.io\/v1, authentication.k8s.io\/v1beta1

-authorization: authorization.k8s.io\/v1, authorization.k8s.io\/v1beta1

-autoscaling: autoscaling\/v1, autoscaling\/v2beta1

-batch: batch\/v1, batch\/v1beta1, batch\/v2alpha1

-certificates: certificates.k8s.io\/v1beta1

-coordination: coordination.k8s.io\/v1beta1

-core: ""\/v1

eg:

```go
scheme.AddKnownTypes(SchemeGroupVersion,   &Pod{},   &PodList{},   &PodStatusResult{}, 
  &PodTemplate{},   &PodTemplateList{},   &ReplicationController{},   &ReplicationControllerList{},
   &Service{},   &ServiceProxyOptions{},   &ServiceList{},   &Endpoints{},   &EndpointsList{},  
 &Node{},   &NodeList{},   &NodeProxyOptions{},   &Binding{},   &Event{},   &EventList{},   &List{},  
 &LimitRange{},   &LimitRangeList{},   &ResourceQuota{},   &ResourceQuotaList{},   &Namespace{},   &NamespaceList{},
   &Secret{},   &SecretList{},   &ServiceAccount{},   &ServiceAccountList{},   &PersistentVolume{}, 
  &PersistentVolumeList{},   &PersistentVolumeClaim{},   &PersistentVolumeClaimList{},   &PodAttachOptions{}, 
  &PodLogOptions{},   &PodExecOptions{},   &PodPortForwardOptions{},   &PodProxyOptions{},  
 &ComponentStatus{},   &ComponentStatusList{},   &SerializedReference{},   &RangeAllocation{},   
&ConfigMap{},   &ConfigMapList{},)
// Add common types
scheme.AddKnownTypes(SchemeGroupVersion, &metav1.Status{})
```

-events: events.k8s.io\/v1beta1

-extensions: extensions\/v1beta1

-imagepolicy: imagepolicy.k8s.io\/v1alpha1

-networking: networking.k8s.io\/v1

-policy: policy\/v1beta1

-rbac: rbac.authorization.k8s.io\/v1, rbac.authorization.k8s.io\/v1alpha1, rbac.authorization.k8s.io\/v1beta1

-scheduling: scheduling.k8s.io\/v1alpha1, scheduling.k8s.io\/v1beta1

```
scheme.AddKnownTypes(SchemeGroupVersion,   &PriorityClass{},   &PriorityClassList{},)
```

-settings: settings.k8s.io\/v1alpha1

-storage: storage.k8s.io\/v1, storage.k8s.io\/v1alpha1, storage.k8s.io\/v1beta1

### -apimachinery: k8s API对象的typing, encoding, decoding, and conversion相关

官方文档中关于该包的作用的描述：This library is a shared dependency for servers and clients to work with Kubernetes API infrastructure without direct type dependencies.  Its first consumers are \`k8s.io\/kubernetes\`, \`k8s.io\/client-go\`, and \`k8s.io\/apiserver\`.

-runtime：

* schema: 

### -apiextensions-apiserver:

-apiserver:

-client-go:

-kube-aggregator:

-kube-openapi:

-metrics:

