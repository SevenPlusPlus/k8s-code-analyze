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

```
// Run runs the KubeControllerManagerOptions.  This should never exit.
func Run(c *config.CompletedConfig, stopCh <-chan struct{}) error {
	
	//启动ControllerManager HTTP服务
	var unsecuredMux *mux.PathRecorderMux
	if c.SecureServing != nil {
		unsecuredMux = genericcontrollermanager.NewBaseHandler(&c.ComponentConfig.Debugging)
		handler := genericcontrollermanager.BuildHandlerChain(unsecuredMux, &c.Authorization, &c.Authentication)
		if err := c.SecureServing.Serve(handler, 0, stopCh); err != nil {
			return err
		}
	}
	if c.InsecureServing != nil {
		unsecuredMux = genericcontrollermanager.NewBaseHandler(&c.ComponentConfig.Debugging)
		handler := genericcontrollermanager.BuildHandlerChain(unsecuredMux, &c.Authorization, &c.Authentication)
		if err := c.InsecureServing.Serve(handler, 0, stopCh); err != nil {
			return err
		}
	}
        
       //ControllerManager真正启动方法, 下面详细解析
	run := func(ctx context.Context) {
		...
	}

        //未启用选主时，则直接启动controllermanager，执行run方法
	if !c.ComponentConfig.GenericComponent.LeaderElection.LeaderElect {
		run(context.TODO())
		panic("unreachable")
	}

	//启动leader选举，leader启动controllermanager，执行run方法
	leaderelection.RunOrDie(context.TODO(), leaderelection.LeaderElectionConfig{
		Lock:          rl,
		LeaseDuration: c.ComponentConfig.GenericComponent.LeaderElection.LeaseDuration.Duration,
		RenewDeadline: c.ComponentConfig.GenericComponent.LeaderElection.RenewDeadline.Duration,
		RetryPeriod:   c.ComponentConfig.GenericComponent.LeaderElection.RetryPeriod.Duration,
		Callbacks: leaderelection.LeaderCallbacks{
			OnStartedLeading: run,
			OnStoppedLeading: func() {
				glog.Fatalf("leaderelection lost")
			},
		},
	})
	panic("unreachable")
}
```
接下来深入分析下ControllerManager启动过程

```
run := func(ctx context.Context) {
	//构建ControllerClientBuilder
		var clientBuilder controller.ControllerClientBuilder
		
	//创建Controller上下文对象
		controllerContext, err := CreateControllerContext(c, rootClientBuilder, clientBuilder, ctx.Done())
	//设置saTokenController初始化方法
		saTokenControllerInitFunc := serviceAccountTokenControllerStarter{rootClientBuilder: rootClientBuilder}.startServiceAccountTokenController
	//启动内置所有Controller, 每个controller承担不同的工作，譬如EndpointController负责维护Service的endpoint
		if err := StartControllers(controllerContext, saTokenControllerInitFunc, NewControllerInitializers(controllerContext.LoopMode), unsecuredMux); err != nil {
			glog.Fatalf("error starting controllers: %v", err)
		}
	//启动初始化所有请求的informer对象
	//Start initializes all requested informers.
		controllerContext.InformerFactory.Start(controllerContext.Stop)

		close(controllerContext.InformersStarted)
		select {}
	}
```
启动每个具体的Controller的时候都会传入ControllerContext，其具体定义如下：

```
type ControllerContext struct {
	// ClientBuilder will provide a client for this controller to use
	//ClientBuilder用于Controller创建client
	ClientBuilder controller.ControllerClientBuilder

	// InformerFactory gives access to informers for the controller.
	//InformerFactory 用于访问各API group versions的资源的informers
	InformerFactory informers.SharedInformerFactory

	// ComponentConfig provides access to init options for a given controller
       //用于访问各Controller的配置选项
	ComponentConfig componentconfig.KubeControllerManagerConfiguration

	// DeferredDiscoveryRESTMapper is a RESTMapper that will defer
	// initialization of the RESTMapper until the first mapping is
	// requested.
	RESTMapper *restmapper.DeferredDiscoveryRESTMapper

	// AvailableResources is a map listing currently available resources
	AvailableResources map[schema.GroupVersionResource]bool

	// Cloud is the cloud provider interface for the controllers to use.
	// It must be initialized and ready to use.
	Cloud cloudprovider.Interface

	// Control for which control loops to be run
	// IncludeCloudLoops is for a kube-controller-manager running all loops
	// ExternalLoops is for a kube-controller-manager running with a cloud-controller-manager
	LoopMode ControllerLoopMode

	// Stop is the stop channel
	Stop <-chan struct{}

	// InformersStarted is closed after all of the controllers have been initialized and are running.  After this point it is safe,
	// for an individual controller to start the shared informers. Before it is closed, they should not.
	InformersStarted chan struct{}

	// ResyncPeriod generates a duration each time it is invoked; this is so that
	// multiple controllers don't get into lock-step and all hammer the apiserver
	// with list requests simultaneously.
	ResyncPeriod func() time.Duration
}
```
##### 创建ControllerContext

```
// CreateControllerContext creates a context struct containing references to resources needed by the controllers such as the cloud provider and clientBuilder. 
func CreateControllerContext(s *config.CompletedConfig, rootClientBuilder, clientBuilder controller.ControllerClientBuilder, stop <-chan struct{}) (ControllerContext, error) {
	//创建sharedInformerFactory实例，各Controllers通过sharedInformers实现对资源的watch-list
	versionedClient := rootClientBuilder.ClientOrDie("shared-informers")
	sharedInformers := informers.NewSharedInformerFactory(versionedClient, ResyncPeriod(s)())

	
	// Use a discovery client capable of being refreshed.
	discoveryClient := rootClientBuilder.ClientOrDie("controller-discovery")
	cachedClient := cacheddiscovery.NewMemCacheClient(discoveryClient.Discovery())
	restMapper := restmapper.NewDeferredDiscoveryRESTMapper(cachedClient)
	go wait.Until(func() {
		restMapper.Reset()
	}, 30*time.Second, stop)

	//获取当前所有groups和versions支持的resouces
	availableResources, err := GetAvailableResources(rootClientBuilder)
	if err != nil {
		return ControllerContext{}, err
	}

	cloud, loopMode, err := createCloudProvider(s.ComponentConfig.CloudProvider.Name, s.ComponentConfig.ExternalCloudVolumePlugin,
		s.ComponentConfig.CloudProvider.CloudConfigFile, s.ComponentConfig.KubeCloudShared.AllowUntaggedCloud, sharedInformers)
	if err != nil {
		return ControllerContext{}, err
	}

	ctx := ControllerContext{
		ClientBuilder:      clientBuilder,
		InformerFactory:    sharedInformers,
		ComponentConfig:    s.ComponentConfig,
		RESTMapper:         restMapper,
		AvailableResources: availableResources,
		Cloud:              cloud,
		LoopMode:           loopMode,
		Stop:               stop,
		InformersStarted:   make(chan struct{}),
		ResyncPeriod:       ResyncPeriod(s),
	}
	return ctx, nil
}
```
### 启动Controllers的实现
* cmd/kube-controller-manager/app/controllermanager.go:

```
func StartControllers(ctx ControllerContext, startSATokenController InitFunc, controllers map[string]InitFunc, unsecuredMux *mux.PathRecorderMux) error {
	// Always start the SA token controller first using a full-power client, since it needs to mint tokens for the rest
	// If this fails, just return here and fail since other controllers won't be able to get credentials.
	//先启动SATokenController
	if _, _, err := startc(ctx); err != nil {
		return err
	}

	// Initialize the cloud provider with a reference to the clientBuilder only after token controller
	// has started in case the cloud provider uses the client builder.
	if ctx.Cloud != nil {
		ctx.Cloud.Initialize(ctx.ClientBuilder)
	}

	//Loop初始化启动各个Controller
	for controllerName, initFn := range controllers {
		if !ctx.IsControllerEnabled(controllerName) {
			glog.Warningf("%q is disabled", controllerName)
			continue
		}

		glog.V(1).Infof("Starting %q", controllerName)
		debugHandler, started, err := initFn(ctx)
		if err != nil {
			glog.Errorf("Error starting %q", controllerName)
			return err
		}
		if !started {
			glog.Warningf("Skipping %q", controllerName)
			continue
		}
		if debugHandler != nil && unsecuredMux != nil {
			basePath := "/debug/controllers/" + controllerName
			unsecuredMux.UnlistedHandle(basePath, http.StripPrefix(basePath, debugHandler))
			unsecuredMux.UnlistedHandlePrefix(basePath+"/", http.StripPrefix(basePath, debugHandler))
		}
		glog.Infof("Started %q", controllerName)
	}

	return nil
}
```
以endpoint Controller为例分析具体Controller初始化启动过程，前面NewControllerInitializers中不难找到endpoint Controller的初始化方法

```
controllers["endpoint"] = startEndpointController
```
#### 启动EndpointController
* cmd/kube-controller-manager/app/core.go:

```
func startEndpointController(ctx ControllerContext) (http.Handler, bool, error) {
	go endpointcontroller.NewEndpointController(
		ctx.InformerFactory.Core().V1().Pods(),
		ctx.InformerFactory.Core().V1().Services(),
		ctx.InformerFactory.Core().V1().Endpoints(),
		ctx.ClientBuilder.ClientOrDie("endpoint-controller"),
	).Run(int(ctx.ComponentConfig.EndPointController.ConcurrentEndpointSyncs), ctx.Stop)
	return nil, true, nil
}
```
新建EndpointController
* pkg/controller/endpoint/endpoints_controller.go:

EndpointController定义如下：

```
// EndpointController manages selector-based service endpoints.
type EndpointController struct {
	client clientset.Interface

	// serviceLister is able to list/get services and is populated by the shared informer passed to
	// NewEndpointController.
	//方便get/list services列表的帮助类接口
	serviceLister corelisters.ServiceLister
	// servicesSynced returns true if the service shared informer has been synced at least once.
	// Added as a member to the struct to allow injection for testing.
	servicesSynced cache.InformerSynced

	// podLister is able to list/get pods and is populated by the shared informer passed to
	// NewEndpointController.
	//方便get/list pods列表的帮助类接口
	podLister corelisters.PodLister
	// podsSynced returns true if the pod shared informer has been synced at least once.
	// Added as a member to the struct to allow injection for testing.
	podsSynced cache.InformerSynced

	// endpointsLister is able to list/get endpoints and is populated by the shared informer passed to
	// NewEndpointController.
	//方便get/list endpoints列表的帮助类接口
	endpointsLister corelisters.EndpointsLister
	// endpointsSynced returns true if the endpoints shared informer has been synced at least once.
	// Added as a member to the struct to allow injection for testing.
	endpointsSynced cache.InformerSynced

	// Services that need to be updated. A channel is inappropriate here,
	// because it allows services with lots of pods to be serviced much
	// more often than services with few pods; it also would cause a
	// service that's inserted multiple times to be processed more than
	// necessary.
	queue workqueue.RateLimitingInterface

	// workerLoopPeriod is the time between worker runs. The workers process the queue of service and pod changes.
	workerLoopPeriod time.Duration
}

// NewEndpointController returns a new *EndpointController.
func NewEndpointController(podInformer coreinformers.PodInformer, serviceInformer coreinformers.ServiceInformer,
	endpointsInformer coreinformers.EndpointsInformer, client clientset.Interface) *EndpointController {
	
	e := &EndpointController{
		client:           client,
		queue:            workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), "endpoint"),
		workerLoopPeriod: time.Second,
	}

	serviceInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: e.enqueueService,
		UpdateFunc: func(old, cur interface{}) {
			e.enqueueService(cur)
		},
		DeleteFunc: e.enqueueService,
	})
	e.serviceLister = serviceInformer.Lister()
	e.servicesSynced = serviceInformer.Informer().HasSynced

	podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    e.addPod,
		UpdateFunc: e.updatePod,
		DeleteFunc: e.deletePod,
	})
	e.podLister = podInformer.Lister()
	e.podsSynced = podInformer.Informer().HasSynced

	e.endpointsLister = endpointsInformer.Lister()
	e.endpointsSynced = endpointsInformer.Informer().HasSynced

	return e
}
```



