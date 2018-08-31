## ControllerManager启动流程
### 构建ControllerManagerCommand并启动
* cmd/kube-controller-manager/controller-manager.go:

```
func main() {
	command := app.NewControllerManagerCommand()

	if err := command.Execute(); err != nil {
		os.Exit(1)
	}
}
```

ControllerManagerCommand::Run
* cmd/kube-controller-manager/app/controllermanager.go:

```
Run: func(cmd *cobra.Command, args []string) {
 //构建一个默认的KubeControllerManagerOptions，其内部包含了各种Controller的默认配置选项
 s, err := options.NewKubeControllerManagerOptions()
 c, err := s.Config(KnownControllers(), ControllersDisabledByDefault.List())
 Run(c.Complete(), wait.NeverStop)
}
```
#### 获取默认启用的的Controller列表
KnownControllers->NewControllerInitializers

```
func KnownControllers() []string {
	ret := sets.StringKeySet(NewControllerInitializers(IncludeCloudLoops))

	// add "special" controllers that aren't initialized normally.  These controllers cannot be initialized
	// using a normal function.  The only known special case is the SA token controller which *must* be started
	// first to ensure that the SA tokens for future controllers will exist.  Think very carefully before adding
	// to this list.
	ret.Insert(
		saTokenControllerName,
	)
	return ret.List()
}

//返回默认Controller名集群初始化方法的map
func NewControllerInitializers(loopMode ControllerLoopMode) map[string]InitFunc {
	controllers := map[string]InitFunc{}
	controllers["endpoint"] = startEndpointController
	controllers["replicationcontroller"] = startReplicationController
	controllers["podgc"] = startPodGCController
	controllers["resourcequota"] = startResourceQuotaController
	controllers["namespace"] = startNamespaceController
	controllers["serviceaccount"] = startServiceAccountController
	controllers["deployment"] = startDeploymentController
	controllers["replicaset"] = startReplicaSetController
	...

	return controllers
}
```
#### 配置返回ControllerManager的配置对象

```
func (s KubeControllerManagerOptions) Config(allControllers []string, disabledByDefaultControllers []string) (*kubecontrollerconfig.Config, error) {
	//通过master url和kubeconfig文件构造restclient.Config
	kubeconfig, err := clientcmd.BuildConfigFromFlags(s.Master, s.Kubeconfig)
      //根据kubeconfig创建新的ApiServer各group的rest client集合Clientset
	client, err := clientset.NewForConfig(restclient.AddUserAgent(kubeconfig, KubeControllerManagerUserAgent))
	
	// shallow copy, do not modify the kubeconfig.Timeout.
	config := *kubeconfig
	config.Timeout = s.GenericComponent.LeaderElection.RenewDeadline.Duration
      //构建仅用于选主的Clientset
	leaderElectionClient := clientset.NewForConfigOrDie(restclient.AddUserAgent(&config, "leader-election"))
      //构建controllermanager的config对象
	c := &kubecontrollerconfig.Config{
		Client:               client,
		Kubeconfig:           kubeconfig,
		EventRecorder:        eventRecorder,
		LeaderElectionClient: leaderElectionClient,
	}
      //将配置的options应用到controllermanager的配置对象config中
	if err := s.ApplyTo(c); err != nil {
		return nil, err
	}

	return c, nil
}
```
其中Clientset定义如下：
* vendor/k8s.io/client-go/kubernetes/clientset.go

```
// Clientset contains the clients for groups. Each group has exactly one
// version included in a Clientset.
type Clientset struct {
	*discovery.DiscoveryClient
	admissionregistrationV1alpha1 *admissionregistrationv1alpha1.AdmissionregistrationV1alpha1Client
	admissionregistrationV1beta1  *admissionregistrationv1beta1.AdmissionregistrationV1beta1Client
	appsV1beta1                   *appsv1beta1.AppsV1beta1Client
	appsV1beta2                   *appsv1beta2.AppsV1beta2Client
	appsV1                        *appsv1.AppsV1Client
	authenticationV1              *authenticationv1.AuthenticationV1Client
	authenticationV1beta1         *authenticationv1beta1.AuthenticationV1beta1Client
       ...
}
```
#### 启动ControllerManager
