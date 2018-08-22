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

runtime包主要包含了一些处理API对象辅助方法，主要有以下几点约定：

1. API对象都有一个公共元数据结构成员，TypeMeta
2. 你的API代码中引入一组内部的API对象
3. 在不同的包中，会有一组外部的API对象
4. 外部集合是版本化的，不能做出破坏性的更改（版本之间需要向下兼容）
5. 随着你的API迭代，随着主要变更的产生，你会有一个另外版本的API实现包
6. 新版本化的包中需要包含对内部版本对象的转换方法实现
7. 你可以持续的支持老版本的API，也因为第六条很容易更新老版本到新版本
8. 所有序列化和反序列化在统一的地方处理

内部实现主要有：

* schema: gv, gvk, gvr, gr, gk \(group, version, kind, resource\) 类型定义, ObjectKind接口定义（这个接口用于序列化时设置Schema的gvk信息到反序列化的API版本对象中）
* serializer: api版本化对象的序列化编解码实现（json\/yaml\/protobuf）
* types.go: \(TypeMeta\/Unknown\/VersionedObjects\) the types provided in this file are not versioned and are intended to be safe to use from within all versions of every API object
* scheme.go: 对象类型注册中心，gvkToType\(gvk 到对象go类型的映射\)， typeToGVK\(go对象类型到gvk列表的映射\)，unversionedTypes， unverionedKinds\(公共对象类型到go对象类型映射\)，versionPriority\(apigroup的版本优先级映射\)，fieldLabelConversionFuncs\(gvk到资源字段convert到内部版本是的转化方法映射\)，defaulterFuncs, converter\(存储所有注册的convertion方法\)
* interfaces.go: GroupVersioner, Encoder, Decoder, Serializer（序列化编解码）, Codec\( Serializer \), ParameterCodec\(序列化编解码url.Values和API对象\)，NegotiatedSerializer（NegotiatedSerializer is an interface used for obtaining encoders, decoders, and serializers for multiple supported media types），Object（返回一个no-op ObjectKindAccessor）, Unstructured\(Unstructured objects store values as map\[string\]interface{}, with only values that can be serialized to JSON allowed\)，ObjectCreater（实例化一个api对象by gvk的接口），ResourceVersioner（设置或返回资源版本信息的接口），SelfLinker（设置或返回API对象SelfLink字段的接口），ObjectTyper（包含提取对象中gvk信息方法的接口），ObjectDefaulter（为对象设置默认值的接口），ObjectConvertor（转换一个对象到不同版本的接口），GroupVersioner（GroupVersioner refines a set of possible conversion targets into a single option， codec\/internalGroupVersioner则是通过一组可能的目标gvk，返回API内部版本_APIVersionInternal的GVK_）
* Codec.go: 提供了统一包装的Encode\/Decode\/DecodeInto方法实现

-types：k8s通用类型实现（NamespacedName、UID、NodeName...）
-labels: 实现了简单的label系统，解析和匹配label选择器

-fields: implements a simple field system, parsing and matching selectors with sets of fields.

-conversion: provides go object versioning.

-api:

----meta：提供从对象获取API元数据的方法， RESTMapper用于映射resources to kind

----erro：api字段验证的错误类型定义

-apis: unstructured\/unstructured\_list, 元数据获取相关接口   Object\/ObjectMetaAccessor\/ListMetaAccessor\/Common\/ListInterface\/Type,元数据相关类型TypeMeta\/ListMeta\/ObjectMeta\/OwnerReference\/ListOptions\/ExportOptions\/GetOptions\/DeleteOptions\/CreateOptions\/Status

-watch: contains a generic watchable interface

### -apiserver:

-plugin\/pkg\/audit: implementations for pkg\/audit\/AuditBackend interface, eg:log\/fake\/buffered\/truncate\/webhook

-plugin\/pkg\/authenticator: implementations for pkg\/auth\/authenticator interfaces, eg:password\/request\/token

-plugin\/pkg\/authorizer: Package webhook implements the authorizer.Authorizer interface using HTTP webhooks.

-pkg:

----admission:

----apis:

----audit:  

----authentication:

----authorization:

----endpoints:

----features:

----registry:

----server:

----storage: 存储操作相关接口

----util:

### -apiextensions-apiserver:

-client-go:

-kube-aggregator:

-kube-openapi:

-metrics:

