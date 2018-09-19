## kubelet启动流程
---

### kubelet cmd启动
* cmd/kubelet/kubelet.go:
```
func main() {
	command := app.NewKubeletCommand(server.SetupSignalHandler())
	if err := command.Execute(); err != nil {
		os.Exit(1)
	}
}
```
#### 新建Kubelet command
新建Kubelet command主要工作：

* 创建kubelet默认启动参数，kubeletFlags := options.NewKubeletFlags()
* 新建kubelet服务配置，kubeletConfig, err := options.NewKubeletConfiguration()
其中KubeletConfiguration定义如下(pkg/kubelet/apis/kubeletconfig/types.go)：
```
// KubeletConfiguration contains the configuration for the Kubelet
type KubeletConfiguration struct {
    ...
	// staticPodPath is the path to the directory containing local (static) pods to run, or the path to a single static pod file.
	StaticPodPath string
	// staticPodURL is the URL for accessing static pods to run
	StaticPodURL string
	// address is the IP address for the Kubelet to serve on (set to 0.0.0.0
	// for all interfaces)
	Address string
	// port is the port for the Kubelet to serve on.
	Port int32
	// readOnlyPort is the read-only port for the Kubelet to serve on with
	// no authentication/authorization (set to 0 to disable)
	ReadOnlyPort int32
	// tlsCertFile is the file containing x509 Certificate for HTTPS.  
	TLSCertFile string
    ...
	// oomScoreAdj is The oom-score-adj value for kubelet process. Values
	// must be within the range [-1000, 1000].
	OOMScoreAdj int32
	// clusterDomain is the DNS domain for this cluster. If set, kubelet will
	// configure all containers to search this domain in addition to the
	// host's search domains.
	ClusterDomain string
	// clusterDNS is a list of IP addresses for a cluster DNS server. If set,
	// kubelet will configure all containers to use this for DNS resolution
	// instead of the host's DNS servers.
	ClusterDNS []string
	// nodeStatusUpdateFrequency is the frequency that kubelet posts node
	// status to master. Note: be cautious when changing the constant, it
	// must work with nodeMonitorGracePeriod in nodecontroller.
	NodeStatusUpdateFrequency metav1.Duration
	// imageMinimumGCAge is the minimum age for an unused image before it is
	// garbage collected.
	ImageMinimumGCAge metav1.Duration
	// How frequently to calculate and cache volume disk usage for all pods
	VolumeStatsAggPeriod metav1.Duration
	// driver that the kubelet uses to manipulate cgroups on the host (cgroupfs or systemd)
	CgroupDriver string
	// maxPods is the number of pods that can run on this Kubelet.
	MaxPods int32
	// The CIDR to use for pod IP addresses, only used in standalone mode.
	// In cluster mode, this is obtained from the master.
	PodCIDR string
	// ResolverConfig is the resolver configuration file used as the basis
	// for the container DNS resolution configuration.
	ResolverConfig string
	// serializeImagePulls when enabled, tells the Kubelet to pull images one at a time.
	SerializeImagePulls bool
	// Map of signal names to quantities that defines hard eviction thresholds. For example: {"memory.available": "300Mi"}.
	EvictionHard map[string]string
	// Map of signal names to quantities that defines soft eviction thresholds.  For example: {"memory.available": "300Mi"}.
	EvictionSoft map[string]string
	// Map of signal names to quantities that defines grace periods for each soft eviction signal. For example: {"memory.available": "30s"}.
	EvictionSoftGracePeriod map[string]string
	// Duration for which the kubelet has to wait before transitioning out of an eviction pressure condition.
	EvictionPressureTransitionPeriod metav1.Duration
	// Maximum allowed grace period (in seconds) to use when terminating pods in response to a soft eviction threshold being met.
	EvictionMaxPodGracePeriod int32
	// Map of signal names to quantities that defines minimum reclaims, which describe the minimum
	// amount of a given resource the kubelet will reclaim when performing a pod eviction while
	// that resource is under pressure. For example: {"imagefs.available": "2Gi"}
	EvictionMinimumReclaim map[string]string
	// featureGates is a map of feature names to bools that enable or disable alpha/experimental
	// features. This field modifies piecemeal the built-in default values from
	// "k8s.io/kubernetes/pkg/features/kube_features.go".
	FeatureGates map[string]bool
	// Tells the Kubelet to fail to start if swap is enabled on the node.
	FailSwapOn bool
	// A quantity defines the maximum size of the container log file before it is rotated. For example: "5Mi" or "256Ki".
	ContainerLogMaxSize string
	// Maximum number of container log files that can be present for a container.
	ContainerLogMaxFiles int32
	// ConfigMapAndSecretChangeDetectionStrategy is a mode in which config map and secret managers are running.
	ConfigMapAndSecretChangeDetectionStrategy ResourceChangeDetectionStrategy

	/* the following fields are meant for Node Allocatable */

	// A set of ResourceName=ResourceQuantity (e.g. cpu=200m,memory=150G) pairs
	// that describe resources reserved for non-kubernetes components.
	// Currently only cpu and memory are supported.
	// See http://kubernetes.io/docs/user-guide/compute-resources for more detail.
	SystemReserved map[string]string
	// A set of ResourceName=ResourceQuantity (e.g. cpu=200m,memory=150G) pairs
	// that describe resources reserved for kubernetes system components.
	// Currently cpu, memory and local ephemeral storage for root file system are supported.
	// See http://kubernetes.io/docs/user-guide/compute-resources for more detail.
	KubeReserved map[string]string
	// This flag helps kubelet identify absolute name of top level cgroup used to enforce `SystemReserved` compute resource reservation for OS system daemons.
	// Refer to [Node Allocatable](https://git.k8s.io/community/contributors/design-proposals/node/node-allocatable.md) doc for more information.
	SystemReservedCgroup string
	// This flag helps kubelet identify absolute name of top level cgroup used to enforce `KubeReserved` compute resource reservation for Kubernetes node system daemons.
	// Refer to [Node Allocatable](https://git.k8s.io/community/contributors/design-proposals/node/node-allocatable.md) doc for more information.
	KubeReservedCgroup string
	// This flag specifies the various Node Allocatable enforcements that Kubelet needs to perform.
	// This flag accepts a list of options. Acceptable options are `pods`, `system-reserved` & `kube-reserved`.
	EnforceNodeAllocatable []string
}
```

#### kubelet命令启动Run执行流程如下，cmd/kubelet/app/server.go:

* validateConfig 主要对kubelet的KubeletConfiguration配置信息进行参数校验。
* 构建KubeletServer
* 通过KubeletServer构建kubelet运行时的依赖kubelet.Dependencies
* run启动kubelet

```
Run: func(cmd *cobra.Command, args []string) {
			// We always validate the local configuration (command line + config file).
			if err := kubeletconfigvalidation.ValidateKubeletConfiguration(kubeletConfig); err != nil             {
			}

			// use dynamic kubelet config, if enabled
		    ...
			// construct a KubeletServer from kubeletFlags and kubeletConfig
			kubeletServer := &options.KubeletServer{
				KubeletFlags:         *kubeletFlags,
				KubeletConfiguration: *kubeletConfig,
			}

			// use kubeletServer to construct the default KubeletDeps
			kubeletDeps, err := UnsecuredDependencies(kubeletServer)
		
			// add the kubelet config controller to kubeletDeps
			kubeletDeps.KubeletConfigController = kubeletConfigController

			// run the kubelet
			if err := Run(kubeletServer, kubeletDeps, stopCh); err != nil {
			}
		}
```
其中Dependencies包含了kubelet运行时需要的各种依赖组件，如ContainerManager管理节点上容器、VolumePlugins管理各种存储卷插件等，其定义如下：
```
// Dependencies is a bin for things we might consider "injected dependencies" -- objects constructed
// at runtime that are necessary for running the Kubelet. This is a temporary solution for grouping
// these objects while we figure out a more comprehensive dependency injection story for the Kubelet.
type Dependencies struct {
	Options []Option

	// Injected Dependencies
	Auth                    server.AuthInterface
	CAdvisorInterface       cadvisor.Interface
	Cloud                   cloudprovider.Interface
	ContainerManager        cm.ContainerManager
	DockerClientConfig      *dockershim.ClientConfig
	EventClient             v1core.EventsGetter
	HeartbeatClient         v1core.CoreV1Interface
	OnHeartbeatFailure      func()
	KubeClient              clientset.Interface
	ExternalKubeClient      clientset.Interface
	Mounter                 mount.Interface
	OOMAdjuster             *oom.OOMAdjuster
	OSInterface             kubecontainer.OSInterface
	PodConfig               *config.PodConfig
	Recorder                record.EventRecorder
	VolumePlugins           []volume.VolumePlugin
	DynamicPluginProber     volume.DynamicPluginProber
	TLSOptions              *server.TLSOptions
	KubeletConfigController *kubeletconfig.Controller
}
```
### 启动kubelet
主要启动流程如下：

* 校验配置
* 向ApiServer请求该节点认证证书，然后创建并设置kubelet运行时依赖的各种连接ApiServer的客户端
* 构建Kubelet所需的认证、授权接口对象
* 构建cAdvisor并且暴露其API接口
* 构建ContainerManager
* 最后RunKubelet真正构建并运行kubelet

```
func run(s *options.KubeletServer, kubeDeps *kubelet.Dependencies, stopCh <-chan struct{}) (err error) {
	// validate the initial KubeletServer (we set feature gates first, because this validation depends on feature gates)
	if err := options.ValidateKubeletServer(s); err != nil {
		return err
	}
	done := make(chan struct{})

	// About to get clients and such, detect standaloneMode
	//此处为非standalone模式
	standaloneMode := true
	if len(s.KubeConfig) > 0 {
		standaloneMode = false
	}

	hostName, err := nodeutil.GetHostname(s.HostnameOverride)
	nodeName, err := getNodeName(kubeDeps.Cloud, hostName)

    //向ApiServer请求该节点客户端认证证书并存储在指定证书目录下
	if s.BootstrapKubeconfig != "" {
		if err := bootstrap.LoadClientCert(s.KubeConfig, s.BootstrapKubeconfig, s.CertDirectory, nodeName); err != nil {
			return err
		}
	}

	// if in standalone mode, indicate as much by setting all clients to nil
	if standaloneMode {
		...
	} else if kubeDeps.KubeClient == nil || kubeDeps.ExternalKubeClient == nil || kubeDeps.EventClient == nil || kubeDeps.HeartbeatClient == nil {
	//创建并设置kubelet运行时依赖的各种连接ApiServer的客户端
		// initialize clients if not standalone mode and any of the clients are not provided
		var kubeClient clientset.Interface
		var eventClient v1core.EventsGetter
		var heartbeatClient v1core.CoreV1Interface
		var externalKubeClient clientset.Interface

		clientConfig, err := createAPIServerClientConfig(s)

		kubeClient, err = clientset.NewForConfig(clientConfig)
		externalKubeClient, err = clientset.NewForConfig(clientConfig)

		// make a separate client for events
		eventClientConfig := *clientConfig
		eventClientConfig.QPS = float32(s.EventRecordQPS)
		eventClientConfig.Burst = int(s.EventBurst)
		eventClient, err = v1core.NewForConfig(&eventClientConfig)

		// make a separate client for heartbeat with throttling disabled and a timeout attached
		heartbeatClientConfig := *clientConfig
		heartbeatClientConfig.Timeout = s.KubeletConfiguration.NodeStatusUpdateFrequency.Duration
		heartbeatClientConfig.QPS = float32(-1)
		heartbeatClient, err = v1core.NewForConfig(&heartbeatClientConfig)

		kubeDeps.KubeClient = kubeClient
		kubeDeps.ExternalKubeClient = externalKubeClient
		if heartbeatClient != nil {
			kubeDeps.HeartbeatClient = heartbeatClient
			kubeDeps.OnHeartbeatFailure = closeAllConns
		}
		if eventClient != nil {
			kubeDeps.EventClient = eventClient
		}
	}
    //构建Kubelet所需的认证、授权接口对象
	if kubeDeps.Auth == nil {
		auth, err := BuildAuth(nodeName, kubeDeps.ExternalKubeClient, s.KubeletConfiguration)
		kubeDeps.Auth = auth
	}
    //构建cAdvisor并且暴露其API接口
	if kubeDeps.CAdvisorInterface == nil {
		imageFsInfoProvider := cadvisor.NewImageFsInfoProvider(s.ContainerRuntime, s.RemoteRuntimeEndpoint)
		kubeDeps.CAdvisorInterface, err = cadvisor.New(imageFsInfoProvider, s.RootDirectory, cadvisor.UsingLegacyCadvisorStats(s.ContainerRuntime, s.RemoteRuntimeEndpoint))
	}

	// Setup event recorder if required.
	makeEventRecorder(kubeDeps, nodeName)
    //构建ContainerManager
	if kubeDeps.ContainerManager == nil {
		if s.CgroupsPerQOS && s.CgroupRoot == "" {
			glog.Infof("--cgroups-per-qos enabled, but --cgroup-root was not specified.  defaulting to /")
			s.CgroupRoot = "/"
		}
		//k8s系统组件预留资源设置（CPU、mem、磁盘存储空间）
		kubeReserved, err := parseResourceList(s.KubeReserved)
		//为非k8s组件预留的系统资源（CPU、mem、磁盘存储空间）
		systemReserved, err := parseResourceList(s.SystemReserved)
	
		var hardEvictionThresholds []evictionapi.Threshold
		// If the user requested to ignore eviction thresholds, then do not set valid values for hardEvictionThresholds here.
		if !s.ExperimentalNodeAllocatableIgnoreEvictionThreshold {
			hardEvictionThresholds, err = eviction.ParseThresholdConfig([]string{}, s.EvictionHard, nil, nil, nil)
		}
		experimentalQOSReserved, err := cm.ParseQOSReserved(s.QOSReserved)

		devicePluginEnabled := utilfeature.DefaultFeatureGate.Enabled(features.DevicePlugins)

		kubeDeps.ContainerManager, err = cm.NewContainerManager(
			kubeDeps.Mounter,
			kubeDeps.CAdvisorInterface,
			cm.NodeConfig{
				RuntimeCgroupsName:    s.RuntimeCgroups,
				SystemCgroupsName:     s.SystemCgroups,
				KubeletCgroupsName:    s.KubeletCgroups,
				ContainerRuntime:      s.ContainerRuntime,
				CgroupsPerQOS:         s.CgroupsPerQOS,
				CgroupRoot:            s.CgroupRoot,
				CgroupDriver:          s.CgroupDriver,
				KubeletRootDir:        s.RootDirectory,
				ProtectKernelDefaults: s.ProtectKernelDefaults,
				NodeAllocatableConfig: cm.NodeAllocatableConfig{
					KubeReservedCgroupName:   s.KubeReservedCgroup,
					SystemReservedCgroupName: s.SystemReservedCgroup,
					EnforceNodeAllocatable:   sets.NewString(s.EnforceNodeAllocatable...),
					KubeReserved:             kubeReserved,
					SystemReserved:           systemReserved,
					HardEvictionThresholds:   hardEvictionThresholds,
				},
				QOSReserved:                           *experimentalQOSReserved,
				ExperimentalCPUManagerPolicy:          s.CPUManagerPolicy,
				ExperimentalCPUManagerReconcilePeriod: s.CPUManagerReconcilePeriod.Duration,
				ExperimentalPodPidsLimit:              s.PodPidsLimit,
				EnforceCPULimits:                      s.CPUCFSQuota,
			},
			s.FailSwapOn,
			devicePluginEnabled,
			kubeDeps.Recorder)
	}
    //检查kubelet执行权限
	if err := checkPermissions(); err != nil {
		glog.Error(err)
	}

	// TODO(vmarmol): Do this through container config.
	oomAdjuster := kubeDeps.OOMAdjuster
	if err := oomAdjuster.ApplyOOMScoreAdj(0, int(s.OOMScoreAdj)); err != nil {
		glog.Warning(err)
	}
    //真正构建并执行kubelet
	if err := RunKubelet(s, kubeDeps, s.RunOnce); err != nil {
		return err
	}
    //暴露kubelet服务健康检查API服务
	if s.HealthzPort > 0 {
		healthz.DefaultHealthz()
		go wait.Until(func() {
			err := http.ListenAndServe(net.JoinHostPort(s.HealthzBindAddress, strconv.Itoa(int(s.HealthzPort))), nil)
		}, 5*time.Second, wait.NeverStop)
	}

	if s.RunOnce {
		return nil
	}

	// If systemd is used, notify it that we have started
	go daemon.SdNotify(false, "READY=1")

	select {
	case <-done:
		break
	case <-stopCh:
		break
	}

	return nil
}
```
### RunKubelet实现
RunKubelet实现主要包括两个部分，1）构建初始化对象kubelet.Bootstrap；2）启动kubelet，开始处理各pod来源的pod更新
```
// RunKubelet is responsible for setting up and running a kubelet.  
func RunKubelet(kubeServer *options.KubeletServer, kubeDeps *kubelet.Dependencies, runOnce bool) error {
	hostname, err := nodeutil.GetHostname(kubeServer.HostnameOverride)
	nodeName, err := getNodeName(kubeDeps.Cloud, hostname)
	// Setup event recorder if required.
	makeEventRecorder(kubeDeps, nodeName)

    //设置系统操作接口实现，如Stat\Symlink\Chmod etc.
	if kubeDeps.OSInterface == nil {
		kubeDeps.OSInterface = kubecontainer.RealOS{}
	}
    //创建初始化kubelet对象
	k, err := CreateAndInitKubelet(&kubeServer.KubeletConfiguration,
		kubeDeps,
		&kubeServer.ContainerRuntimeOptions,
		kubeServer.ContainerRuntime,
		kubeServer.RuntimeCgroups,
		...
		kubeServer.NodeStatusMaxImages)

	// NewMainKubelet should have set up a pod source config if one didn't exist when the builder was run. This is just a precaution.
	podCfg := kubeDeps.PodConfig

	rlimit.RlimitNumFiles(uint64(kubeServer.MaxOpenFiles))

	// process pods and exit.
	if runOnce {
		if _, err := k.RunOnce(podCfg.Updates()); err != nil {
			return fmt.Errorf("runonce failed: %v", err)
		}
		glog.Infof("Started kubelet as runonce")
	} else {
	    //启动kubelet处理各pod来源的pod更新
		startKubelet(k, podCfg, &kubeServer.KubeletConfiguration, kubeDeps, kubeServer.EnableServer)
		glog.Infof("Started kubelet")
	}
	return nil
}
```
#### 构建初始化kubelet.Bootstrap接口对象

```
func CreateAndInitKubelet(kubeCfg *kubeletconfiginternal.KubeletConfiguration,
	kubeDeps *kubelet.Dependencies,
	crOptions *config.ContainerRuntimeOptions,
	containerRuntime string,
	...
	nodeStatusMaxImages int32) (k kubelet.Bootstrap, err error) {
	// TODO: block until all sources have delivered at least one update to the channel, or break the sync loop
	// up into "per source" synchronizations
    //构建Kubelet对象其实现了Bootstrap接口
	k, err = kubelet.NewMainKubelet(kubeCfg,
		kubeDeps,
		crOptions,
		...
		nodeStatusMaxImages)

    //发送kubelet启动成功的event
	k.BirthCry()
    //启动垃圾回收线程
	k.StartGarbageCollection()

	return k, nil
}
```
继续分析Kubelet构建构成NewMainKubelet之前，我们先看下Kubelet结构及其接口Bootstrap定义(pkg/kubelet/kubelet.go)：
```
// Bootstrap is a bootstrapping interface for kubelet, targets the initialization protocol
type Bootstrap interface {
	GetConfiguration() kubeletconfiginternal.KubeletConfiguration
	BirthCry()
	StartGarbageCollection()
	ListenAndServe(address net.IP, port uint, tlsOptions *server.TLSOptions, auth server.AuthInterface, enableDebuggingHandlers, enableContentionProfiling bool)
	ListenAndServeReadOnly(address net.IP, port uint)
	Run(<-chan kubetypes.PodUpdate)
	RunOnce(<-chan kubetypes.PodUpdate) ([]RunPodResult, error)
}

// Kubelet is the main kubelet implementation.
type Kubelet struct {
	kubeletConfiguration kubeletconfiginternal.KubeletConfiguration
	hostname        string
	nodeName        types.NodeName
	runtimeCache    kubecontainer.RuntimeCache
	kubeClient      clientset.Interface
	heartbeatClient v1core.CoreV1Interface
	iptClient       utilipt.Interface //iptables 命令操作接口客户端
	rootDirectory   string

	// podWorkers handle syncing Pods in response to events.
	//podWorkers处理Pods的同步
	podWorkers PodWorkers

	// resyncInterval is the interval between periodic full reconciliations of
	// pods on this node.
	resyncInterval time.Duration
    //记录当前kubelet可用的pod sources
	sourcesReady config.SourcesReady
    //Pod管理接口
	// podManager is a facade that abstracts away the various sources of pods
	// this Kubelet services.
	podManager kubepod.Manager
    //节点驱逐管理
	// Needed to observe and respond to situations that could impact node stability
	evictionManager eviction.Manager
	// Optional, defaults to /logs/ from /var/log
	logServer http.Handler
	// Optional, defaults to simple Docker implementation，执行容器内命令实现接口
	runner kubecontainer.ContainerCommandRunner
	// cAdvisor used for container information.
	cadvisor cadvisor.Interface
    //标记该节点是否已注册到ApiServer
	// Set to true to have the node register itself with the apiserver.
	registerNode bool
	//该节点上的taints列表
	// List of taints to add to a node object when the kubelet registers itself.
	registerWithTaints []api.Taint
	//标记该节点是否可作为调度节点（调度器可以调度pod到该节点上）
	// Set to true to have the node register itself as schedulable.
	registerSchedulable bool
	// for internal book keeping; access only from within registerWithApiserver
	registrationCompleted bool
    //该节点上启动Pod时，容器DNS配置设置
	// dnsConfigurer is used for setting up DNS resolver configuration when launching pods.
	dnsConfigurer *dns.Configurer
	// masterServiceNamespace is the namespace that the master service is exposed in.
	masterServiceNamespace string
	//List Services接口实现
	// serviceLister knows how to list services
	serviceLister serviceLister
	//获取节点信息的接口实现
	// nodeInfo knows how to get information about the node for this kubelet.
	nodeInfo predicates.NodeInfo
    //该节点上的标签信息
	// a list of node labels to register
	nodeLabels map[string]string
	// Last timestamp when runtime responded on ping.
	// Mutex is used to protect this value.
	runtimeState *runtimeState
	// Volume plugins.已注册的存储卷插件管理
	volumePluginMgr *volume.VolumePluginMgr
	// Handles container probing. 容器监控检查管理器接口实现
	probeManager prober.Manager
	// Manages container health check results.容器监控检查结果管理接口实现
	livenessManager proberesults.Manager
	// How long to keep idle streaming command execution/port forwarding
	// connections open before terminating them
	streamingConnectionIdleTimeout time.Duration
	// The EventRecorder to use
	recorder record.EventRecorder
    //死亡容器的gc策略实现
	// Policy for handling garbage collection of dead containers.
	containerGC kubecontainer.ContainerGC
    //容器镜像回收策略实现
	// Manager for image garbage collection.
	imageManager images.ImageGCManager
    //容器日志生命周期管理接口实现
	// Manager for container logs.
	containerLogManager logs.ContainerLogManager
	// Secret manager. Secret管理接口实现
	secretManager secret.Manager
	// ConfigMap manager. ConfigMap管理接口实现
	configMapManager configmap.Manager
	// Cached MachineInfo returned by cadvisor.
	machineInfo *cadvisorapi.MachineInfo
	//Cached RootFsInfo returned by cadvisor
	rootfsInfo *cadvisorapiv2.FsInfo
	// Handles certificate rotations.
	serverCertificateManager certificate.Manager
	// Syncs pods statuses with apiserver; also used as a cache of statuses.
	statusManager status.Manager
    //存储卷管理接口实现
	// VolumeManager runs a set of asynchronous loops that figure out which
	// volumes need to be attached/mounted/unmounted/detached based on the pods
	// scheduled on this node and makes it so.
	volumeManager volumemanager.VolumeManager
	// Reference to this node.
	nodeRef *v1.ObjectReference
	// The name of the container runtime
	containerRuntimeName string
	// redirectContainerStreaming enables container streaming redirect.
	redirectContainerStreaming bool
	// Container runtime.
	containerRuntime kubecontainer.Runtime
	// Streaming runtime handles container streaming.
	streamingRuntime kubecontainer.StreamingRuntime
	// Container runtime service (needed by container runtime Start()).
	runtimeService internalapi.RuntimeService
	// reasonCache caches the failure reason of the last creation of all containers, which is used for generating ContainerStatus.
	reasonCache *ReasonCache

	// nodeStatusUpdateFrequency specifies how often kubelet posts node status to master.
	nodeStatusUpdateFrequency time.Duration

    //缓存节点内所有Pods的PodStatus
	podCache kubecontainer.Cache

	// os is a facade for various syscalls that need to be mocked during testing.
	os kubecontainer.OSInterface

	// Watcher of out of memory events.
	oomWatcher OOMWatcher
    //统计节点上资源的消耗情况
	resourceAnalyzer serverstats.ResourceAnalyzer

	// Whether or not we should have the QOS cgroup hierarchy for resource management
	cgroupsPerQOS bool

	// If non-empty, pass this to the container runtime as the root cgroup.
	cgroupRoot string
	// Mounter to use for volumes. 存储卷mount操作接口实现
	mounter mount.Interface
	// Manager of non-Runtime containers.
	containerManager cm.ContainerManager
	// Maximum Number of Pods which can be run by this Kubelet
	maxPods int
	// Monitor Kubelet's sync loop
	syncLoopMonitor atomic.Value
	// Container restart Backoff； 容器重启时Backoff策略
	backOff *flowcontrol.Backoff
	// Channel for sending pods to kill.
	podKillingCh chan *kubecontainer.PodPair
	// Information about the ports which are opened by daemons on Node running this Kubelet server.
	daemonEndpoints *v1.NodeDaemonEndpoints
	// A queue used to trigger pod workers. pod workers工作队列
	workQueue queue.WorkQueue
	// oneTimeInitializer is used to initialize modules that are dependent on the runtime to be up.
	oneTimeInitializer sync.Once
	// handlers called during the tryUpdateNodeStatus cycle
	setNodeStatusFuncs []func(*v1.Node) error
	lastNodeUnschedulableLock sync.Mutex
	// maintains Node.Spec.Unschedulable value from previous run of tryUpdateNodeStatus()
	lastNodeUnschedulable bool
	// TODO: think about moving this to be centralized in PodWorkers in follow-on.
	// the list of handlers to call during pod admission.
	admitHandlers lifecycle.PodAdmitHandlers
	// softAdmithandlers are applied to the pod after it is admitted by the Kubelet, but before it is
	// run. A pod rejected by a softAdmitHandler will be left in a Pending state indefinitely. If a
	// rejected pod should not be recreated, or the scheduler is not aware of the rejection rule, the
	// admission rule should be applied by a softAdmitHandler.
	softAdmitHandlers lifecycle.PodAdmitHandlers
	// the list of handlers to call during pod sync loop.
	lifecycle.PodSyncLoopHandlers
	// the list of handlers to call during pod sync.
	lifecycle.PodSyncHandlers
	// the number of allowed pods per core
	podsPerCore int
	// enableControllerAttachDetach indicates the Attach/Detach controller
	// should manage attachment/detachment of volumes scheduled to this node,
	// and disable kubelet from executing any attach/detach operations
	enableControllerAttachDetach bool
	// trigger deleting containers in a pod
	containerDeletor *podContainerDeletor
	// The AppArmor validator for checking whether AppArmor is supported.
	appArmorValidator apparmor.Validator
	// The handler serving CRI streaming calls (exec/attach/port-forward).
	criHandler http.Handler
    ...
}
```
##### NewMainKubelet实例化kubelet对象并其他内部各模块进行初始化
分析前面Kubelet对象结构，不难看出其内部包含了很多组件来协助kubelet的工作，因此NewMainKubelet需要一一对其进行初始化，主要工作如下：

* 创建pod元数据来源（File、URL、apiserver）
* 构建kubelet对象
* 初始化configMap和secret管理接口对象
* pod健康检查结果管理接口对象
* 实例化并初始化pod管理接口对象
* 创建并启动CRI shim提供容器运行时grpc接口服务
* 构建kubelet容器运行时和镜像服务管理接口实现
* 构建一个kubeGenericRuntimeManager，封装了kubelet全部运行时管理接口方法实现
* 构建 containerGC
* 构建镜像GC管理器
* 构建kubelet pod 状态管理器保持pod真实状态与apiserver同步
* 构建pod健康检查probing管理器
* 构建service account tokens管理器
* 构建卷插件管理器
* 构建存储卷管理器
* 构建pod驱逐管理器
* 增加各种pod准入检查处理句柄

```
// NewMainKubelet instantiates a new Kubelet object along with all the required internal modules.
// No initialization of Kubelet and its modules should happen here.
func NewMainKubelet(kubeCfg *kubeletconfiginternal.KubeletConfiguration,
	kubeDeps *Dependencies,
	crOptions *config.ContainerRuntimeOptions,
	containerRuntime string,
	runtimeCgroups string,
	hostnameOverride string,
	nodeIP string,
	providerID string,
	cloudProvider string,
	certDirectory string,
	rootDirectory string,
	registerNode bool,
	registerWithTaints []api.Taint,
	allowedUnsafeSysctls []string,
	remoteRuntimeEndpoint string,
	remoteImageEndpoint string,
	experimentalMounterPath string,
	experimentalKernelMemcgNotification bool,
	experimentalCheckNodeCapabilitiesBeforeMount bool,
	experimentalNodeAllocatableIgnoreEvictionThreshold bool,
	minimumGCAge metav1.Duration,
	maxPerPodContainerCount int32,
	maxContainerCount int32,
	masterServiceNamespace string,
	registerSchedulable bool,
	nonMasqueradeCIDR string,
	keepTerminatedPodVolumes bool,
	nodeLabels map[string]string,
	seccompProfileRoot string,
	bootstrapCheckpointPath string,
	nodeStatusMaxImages int32) (*Kubelet, error) {

	hostname, err := nodeutil.GetHostname(hostnameOverride)
	// Query the cloud provider for our node name, default to hostname
	nodeName := types.NodeName(hostname)
    //创建pod元数据来源（File、URL、apiserver）
	if kubeDeps.PodConfig == nil {
		kubeDeps.PodConfig, err = makePodSourceConfig(kubeCfg, kubeDeps, nodeName, bootstrapCheckpointPath)
	}
    //初始化容器回收策略
	containerGCPolicy := kubecontainer.ContainerGCPolicy{
		MinAge:             minimumGCAge.Duration,
		MaxPerPodContainer: int(maxPerPodContainerCount),
		MaxContainers:      int(maxContainerCount),
	}

	daemonEndpoints := &v1.NodeDaemonEndpoints{
		KubeletEndpoint: v1.DaemonEndpoint{Port: kubeCfg.Port},
	}
    //初始化镜像回收策略
	imageGCPolicy := images.ImageGCPolicy{
		MinAge:               kubeCfg.ImageMinimumGCAge.Duration,
		HighThresholdPercent: int(kubeCfg.ImageGCHighThresholdPercent),
		LowThresholdPercent:  int(kubeCfg.ImageGCLowThresholdPercent),
	}

	enforceNodeAllocatable := kubeCfg.EnforceNodeAllocatable
	if experimentalNodeAllocatableIgnoreEvictionThreshold {
		// Do not provide kubeCfg.EnforceNodeAllocatable to eviction threshold parsing if we are not enforcing Evictions
		enforceNodeAllocatable = []string{}
	}
	thresholds, err := eviction.ParseThresholdConfig(enforceNodeAllocatable, kubeCfg.EvictionHard, kubeCfg.EvictionSoft, kubeCfg.EvictionSoftGracePeriod, kubeCfg.EvictionMinimumReclaim)
    //初始化节点驱逐配置
	evictionConfig := eviction.Config{
		PressureTransitionPeriod: kubeCfg.EvictionPressureTransitionPeriod.Duration,
		MaxPodGracePeriodSeconds: int64(kubeCfg.EvictionMaxPodGracePeriod),
		Thresholds:               thresholds,
		KernelMemcgNotification:  experimentalKernelMemcgNotification,
		PodCgroupRoot:            kubeDeps.ContainerManager.GetPodCgroupRoot(),
	}
    //初始化Service本地缓存索引
	serviceIndexer := cache.NewIndexer(cache.MetaNamespaceKeyFunc, cache.Indexers{cache.NamespaceIndex: cache.MetaNamespaceIndexFunc})
	if kubeDeps.KubeClient != nil {
		serviceLW := cache.NewListWatchFromClient(kubeDeps.KubeClient.CoreV1().RESTClient(), "services", metav1.NamespaceAll, fields.Everything())
		r := cache.NewReflector(serviceLW, &v1.Service{}, serviceIndexer, 0)
		go r.Run(wait.NeverStop)
	}
	serviceLister := corelisters.NewServiceLister(serviceIndexer)
    //初始化节点本地缓存索引
	nodeIndexer := cache.NewIndexer(cache.MetaNamespaceKeyFunc, cache.Indexers{})
	if kubeDeps.KubeClient != nil {
		fieldSelector := fields.Set{api.ObjectNameField: string(nodeName)}.AsSelector()
		nodeLW := cache.NewListWatchFromClient(kubeDeps.KubeClient.CoreV1().RESTClient(), "nodes", metav1.NamespaceAll, fieldSelector)
		r := cache.NewReflector(nodeLW, &v1.Node{}, nodeIndexer, 0)
		go r.Run(wait.NeverStop)
	}
	nodeInfo := &predicates.CachedNodeInfo{NodeLister: corelisters.NewNodeLister(nodeIndexer)}

	// TODO: get the real node object of ourself,
	// and use the real node name and UID.
	nodeRef := &v1.ObjectReference{
		Kind:      "Node",
		Name:      string(nodeName),
		UID:       types.UID(nodeName),
		Namespace: "",
	}

	containerRefManager := kubecontainer.NewRefManager()
    //创建并初始化OOMWatcher
	oomWatcher := NewOOMWatcher(kubeDeps.CAdvisorInterface, kubeDeps.Recorder)

	clusterDNS := make([]net.IP, 0, len(kubeCfg.ClusterDNS))
	for _, ipEntry := range kubeCfg.ClusterDNS {
		ip := net.ParseIP(ipEntry)
		if ip == nil {
			glog.Warningf("Invalid clusterDNS ip '%q'", ipEntry)
		} else {
			clusterDNS = append(clusterDNS, ip)
		}
	}
	httpClient := &http.Client{}
	parsedNodeIP := net.ParseIP(nodeIP)
    //构建kubelet对象
	klet := &Kubelet{
		hostname:                       hostname,
		nodeName:                       nodeName,
		kubeClient:                     kubeDeps.KubeClient,
		heartbeatClient:                kubeDeps.HeartbeatClient,
		onRepeatedHeartbeatFailure:     kubeDeps.OnHeartbeatFailure,
		rootDirectory:                  rootDirectory,
		resyncInterval:                 kubeCfg.SyncFrequency.Duration,
		sourcesReady:                   config.NewSourcesReady(kubeDeps.PodConfig.SeenAllSources),
		registerNode:                   registerNode,
		registerWithTaints:             registerWithTaints,
		registerSchedulable:            registerSchedulable,
		dnsConfigurer:                  dns.NewConfigurer(kubeDeps.Recorder, nodeRef, parsedNodeIP, clusterDNS, kubeCfg.ClusterDomain, kubeCfg.ResolverConfig),
		serviceLister:                  serviceLister,
		nodeInfo:                       nodeInfo,
		masterServiceNamespace:         masterServiceNamespace,
		streamingConnectionIdleTimeout: kubeCfg.StreamingConnectionIdleTimeout.Duration,
		recorder:                       kubeDeps.Recorder,
		cadvisor:                       kubeDeps.CAdvisorInterface,
		cloud:                          kubeDeps.Cloud,
		externalCloudProvider:     cloudprovider.IsExternal(cloudProvider),
		providerID:                providerID,
		nodeRef:                   nodeRef,
		nodeLabels:                nodeLabels,
		nodeStatusUpdateFrequency: kubeCfg.NodeStatusUpdateFrequency.Duration,
		os:                         kubeDeps.OSInterface,
		oomWatcher:                 oomWatcher,
		cgroupsPerQOS:              kubeCfg.CgroupsPerQOS,
		cgroupRoot:                 kubeCfg.CgroupRoot,
		mounter:                    kubeDeps.Mounter,
		maxPods:                    int(kubeCfg.MaxPods),
		podsPerCore:                int(kubeCfg.PodsPerCore),
		syncLoopMonitor:            atomic.Value{},
		daemonEndpoints:            daemonEndpoints,
		containerManager:           kubeDeps.ContainerManager,
		containerRuntimeName:       containerRuntime,
		redirectContainerStreaming: crOptions.RedirectContainerStreaming,
		nodeIP:          parsedNodeIP,
		nodeIPValidator: validateNodeIP,
		clock:           clock.RealClock{},
		enableControllerAttachDetach:            kubeCfg.EnableControllerAttachDetach,
		iptClient:                               utilipt.New(utilexec.New(), utildbus.New(), utilipt.ProtocolIpv4),
		makeIPTablesUtilChains:                  kubeCfg.MakeIPTablesUtilChains,
		iptablesMasqueradeBit:                   int(kubeCfg.IPTablesMasqueradeBit),
		iptablesDropBit:                         int(kubeCfg.IPTablesDropBit),
		experimentalHostUserNamespaceDefaulting: utilfeature.DefaultFeatureGate.Enabled(features.ExperimentalHostUserNamespaceDefaultingGate),
		keepTerminatedPodVolumes:                keepTerminatedPodVolumes,
		nodeStatusMaxImages:                     nodeStatusMaxImages,
		enablePluginsWatcher:                    utilfeature.DefaultFeatureGate.Enabled(features.KubeletPluginsWatcher),
	}

    //初始化configMap和secret管理接口对象
	var secretManager secret.Manager
	var configMapManager configmap.Manager
	switch kubeCfg.ConfigMapAndSecretChangeDetectionStrategy {
	case kubeletconfiginternal.WatchChangeDetectionStrategy:
		secretManager = secret.NewWatchingSecretManager(kubeDeps.KubeClient)
		configMapManager = configmap.NewWatchingConfigMapManager(kubeDeps.KubeClient)
	case kubeletconfiginternal.TTLCacheChangeDetectionStrategy:
		secretManager = secret.NewCachingSecretManager(
			kubeDeps.KubeClient, manager.GetObjectTTLFromNodeFunc(klet.GetNode))
		configMapManager = configmap.NewCachingConfigMapManager(
			kubeDeps.KubeClient, manager.GetObjectTTLFromNodeFunc(klet.GetNode))
	case kubeletconfiginternal.GetChangeDetectionStrategy:
		secretManager = secret.NewSimpleSecretManager(kubeDeps.KubeClient)
		configMapManager = configmap.NewSimpleConfigMapManager(kubeDeps.KubeClient)
	default:
		return nil, fmt.Errorf("unknown configmap and secret manager mode: %v", kubeCfg.ConfigMapAndSecretChangeDetectionStrategy)
	}

	klet.secretManager = secretManager
	klet.configMapManager = configMapManager
	
	machineInfo, err := klet.cadvisor.MachineInfo()
	klet.machineInfo = machineInfo

	imageBackOff := flowcontrol.NewBackOff(backOffPeriod, MaxContainerBackOff)
    //pod健康检查结果管理接口对象
	klet.livenessManager = proberesults.NewManager()
    //节点上pod缓存
	klet.podCache = kubecontainer.NewCache()
	//检查点管理接口对象
	var checkpointManager checkpointmanager.CheckpointManager
	if bootstrapCheckpointPath != "" {
		checkpointManager, err = checkpointmanager.NewCheckpointManager(bootstrapCheckpointPath)
		}
	}
	//实例化并初始化pod管理接口对象
	// podManager is also responsible for keeping secretManager and configMapManager contents up-to-date.
	klet.podManager = kubepod.NewBasicPodManager(kubepod.NewBasicMirrorClient(klet.kubeClient), secretManager, configMapManager, checkpointManager)

    //设置The unix socket for kubelet <-> dockershim communication.
	if remoteRuntimeEndpoint != "" {
		// remoteImageEndpoint is same as remoteRuntimeEndpoint if not explicitly specified
		if remoteImageEndpoint == "" {
			remoteImageEndpoint = remoteRuntimeEndpoint
		}
	}

	//为docker shim初始化网络插件配置
	pluginSettings := dockershim.NetworkPluginSettings{
		HairpinMode:        kubeletconfiginternal.HairpinMode(kubeCfg.HairpinMode),
		NonMasqueradeCIDR:  nonMasqueradeCIDR,
		PluginName:         crOptions.NetworkPluginName,
		PluginConfDir:      crOptions.CNIConfDir,
		PluginBinDirString: crOptions.CNIBinDir,
		MTU:                int(crOptions.NetworkPluginMTU),
	}
    //初始化节点资源消耗统计分析器
	klet.resourceAnalyzer = serverstats.NewResourceAnalyzer(klet, kubeCfg.VolumeStatsAggPeriod.Duration)

	switch containerRuntime {
	case kubetypes.DockerContainerRuntime:
	    //创建并启动CRI shim提供容器运行时grpc接口服务
		// Create and start the CRI shim running as a grpc server.
		streamingConfig := getStreamingConfig(kubeCfg, kubeDeps, crOptions)
		ds, err := dockershim.NewDockerService(kubeDeps.DockerClientConfig, crOptions.PodSandboxImage, streamingConfig,
			&pluginSettings, runtimeCgroups, kubeCfg.CgroupDriver, crOptions.DockershimRootDirectory, !crOptions.RedirectContainerStreaming)
	
		if crOptions.RedirectContainerStreaming {
			klet.criHandler = ds
		}

		glog.V(2).Infof("Starting the GRPC server for the docker CRI shim.")
		//启动docker CRI shim接口grpc服务
		server := dockerremote.NewDockerServer(remoteRuntimeEndpoint, ds)
		if err := server.Start(); err != nil {
			return nil, err
		}
	case kubetypes.RemoteContainerRuntime:
		// No-op.
		break
	default:
		return nil, fmt.Errorf("unsupported CRI runtime: %q", containerRuntime)
	}
	//构建kubelet容器运行时和镜像服务管理接口实现
	runtimeService, imageService, err := getRuntimeAndImageServices(remoteRuntimeEndpoint, remoteImageEndpoint, kubeCfg.RuntimeRequestTimeout)

	klet.runtimeService = runtimeService
	//构建一个kubeGenericRuntimeManager，封装了kubelet全部运行时管理接口方法实现
	runtime, err := kuberuntime.NewKubeGenericRuntimeManager(
		kubecontainer.FilterEventRecorder(kubeDeps.Recorder),
		klet.livenessManager,
		seccompProfileRoot,
		containerRefManager,
		machineInfo,
		klet,
		kubeDeps.OSInterface,
		klet,
		httpClient,
		imageBackOff,
		kubeCfg.SerializeImagePulls,
		float32(kubeCfg.RegistryPullQPS),
		int(kubeCfg.RegistryBurst),
		kubeCfg.CPUCFSQuota,
		runtimeService,
		imageService,
		kubeDeps.ContainerManager.InternalContainerLifecycle(),
		legacyLogProvider,
	)

	klet.containerRuntime = runtime
	klet.streamingRuntime = runtime
	klet.runner = runtime

    //构造节点状态统计信息Provider
	if cadvisor.UsingLegacyCadvisorStats(containerRuntime, remoteRuntimeEndpoint) {
		klet.StatsProvider = stats.NewCadvisorStatsProvider(
			klet.cadvisor,
			klet.resourceAnalyzer,
			klet.podManager,
			klet.runtimeCache,
			klet.containerRuntime)
	} else {
		klet.StatsProvider = stats.NewCRIStatsProvider(
			klet.cadvisor,
			klet.resourceAnalyzer,
			klet.podManager,
			klet.runtimeCache,
			runtimeService,
			imageService,
			stats.NewLogMetricsService())
	}

	klet.pleg = pleg.NewGenericPLEG(klet.containerRuntime, plegChannelCapacity, plegRelistPeriod, klet.podCache, clock.RealClock{})
	klet.runtimeState = newRuntimeState(maxWaitForContainerRuntime)
	klet.runtimeState.addHealthCheck("PLEG", klet.pleg.Healthy)
	klet.updatePodCIDR(kubeCfg.PodCIDR)

	// 构建 containerGC
	containerGC, err := kubecontainer.NewContainerGC(klet.containerRuntime, containerGCPolicy, klet.sourcesReady)
	klet.containerGC = containerGC
	klet.containerDeletor = newPodContainerDeletor(klet.containerRuntime, integer.IntMax(containerGCPolicy.MaxPerPodContainer, minDeadContainerInPod))

	// 构建镜像GC管理器
	imageManager, err := images.NewImageGCManager(klet.containerRuntime, klet.StatsProvider, kubeDeps.Recorder, nodeRef, imageGCPolicy, crOptions.PodSandboxImage)
	klet.imageManager = imageManager

	if containerRuntime == kubetypes.RemoteContainerRuntime && utilfeature.DefaultFeatureGate.Enabled(features.CRIContainerLogRotation) {
	...
	} else {
		klet.containerLogManager = logs.NewStubContainerLogManager()
	}
    //构建kubelet pod 状态管理器保持pod真实状态与apiserver同步
	klet.statusManager = status.NewManager(klet.kubeClient, klet.podManager, klet)

	if kubeCfg.ServerTLSBootstrap && kubeDeps.TLSOptions != nil && utilfeature.DefaultFeatureGate.Enabled(features.RotateKubeletServerCertificate) {
		klet.serverCertificateManager, err = kubeletcertificate.NewKubeletServerCertificateManager(klet.kubeClient, kubeCfg, klet.nodeName, klet.getLastObservedNodeAddresses, certDirectory)
		}
		kubeDeps.TLSOptions.Config.GetCertificate = func(*tls.ClientHelloInfo) (*tls.Certificate, error) {
			cert := klet.serverCertificateManager.Current()
			}
			return cert, nil
		}
	}
    //构建pod健康检查probing管理器
	klet.probeManager = prober.NewManager(
		klet.statusManager,
		klet.livenessManager,
		klet.runner,
		containerRefManager,
		kubeDeps.Recorder)
    //构建service account tokens管理器
	tokenManager := token.NewManager(kubeDeps.KubeClient)

    //构建卷插件管理器
	klet.volumePluginMgr, err =
		NewInitializedVolumePluginMgr(klet, secretManager, configMapManager, tokenManager, kubeDeps.VolumePlugins, kubeDeps.DynamicPluginProber)

	if klet.enablePluginsWatcher {
		klet.pluginWatcher = pluginwatcher.NewWatcher(klet.getPluginsDir())
	}

	// If the experimentalMounterPathFlag is set, we do not want to
	// check node capabilities since the mount path is not the default
	if len(experimentalMounterPath) != 0 {
		experimentalCheckNodeCapabilitiesBeforeMount = false
		// Replace the nameserver in containerized-mounter's rootfs/etc/resolve.conf with kubelet.ClusterDNS
		// so that service name could be resolved
		klet.dnsConfigurer.SetupDNSinContainerizedMounter(experimentalMounterPath)
	}

	//构建存储卷管理器
	klet.volumeManager = volumemanager.NewVolumeManager(
		kubeCfg.EnableControllerAttachDetach,
		nodeName,
		klet.podManager,
		klet.statusManager,
		klet.kubeClient,
		klet.volumePluginMgr,
		klet.containerRuntime,
		kubeDeps.Mounter,
		klet.getPodsDir(),
		kubeDeps.Recorder,
		experimentalCheckNodeCapabilitiesBeforeMount,
		keepTerminatedPodVolumes)

	runtimeCache, err := kubecontainer.NewRuntimeCache(klet.containerRuntime)

	klet.runtimeCache = runtimeCache
	klet.reasonCache = NewReasonCache() //出错原因缓存
	klet.workQueue = queue.NewBasicWorkQueue(klet.clock) //pod worker工作队列
	//kubelet pod工作线程，保证节点上的pod都能出于期望的状态
	klet.podWorkers = newPodWorkers(klet.syncPod, kubeDeps.Recorder, klet.workQueue, klet.resyncInterval, backOffPeriod, klet.podCache)

	klet.backOff = flowcontrol.NewBackOff(backOffPeriod, MaxContainerBackOff)
	klet.podKillingCh = make(chan *kubecontainer.PodPair, podKillingChannelCapacity)

	//构建pod驱逐管理器
	evictionManager, evictionAdmitHandler := eviction.NewManager(klet.resourceAnalyzer, evictionConfig, killPodNow(klet.podWorkers, kubeDeps.Recorder), klet.imageManager, klet.containerGC, kubeDeps.Recorder, nodeRef, klet.clock)

	klet.evictionManager = evictionManager
	klet.admitHandlers.AddPodAdmitHandler(evictionAdmitHandler)

	if utilfeature.DefaultFeatureGate.Enabled(features.Sysctls) {
		//增加sysctl admission检查是否容器运行时支持sysctls
		runtimeSupport, err := sysctl.NewRuntimeAdmitHandler(klet.containerRuntime)
		// Safe, whitelisted sysctls can always be used as unsafe sysctls in the spec.
		// Hence, we concatenate those two lists.
		safeAndUnsafeSysctls := append(sysctlwhitelist.SafeSysctlWhitelist(), allowedUnsafeSysctls...)
		sysctlsWhitelist, err := sysctl.NewWhitelist(safeAndUnsafeSysctls)
		
		klet.admitHandlers.AddPodAdmitHandler(runtimeSupport)
		klet.admitHandlers.AddPodAdmitHandler(sysctlsWhitelist)
	}

	// enable active deadline handler
	activeDeadlineHandler, err := newActiveDeadlineHandler(klet.statusManager, kubeDeps.Recorder, klet.clock)
	
	klet.AddPodSyncLoopHandler(activeDeadlineHandler)
	klet.AddPodSyncHandler(activeDeadlineHandler)
    
    //新增CriticalPodAdmissionFailureHandler，处理关键pod准入检查失败的情况，如果其准入检查失败原因仅为资源不足，那么就会发起pod驱逐，最终使该关键pod可以通过准入检查
	criticalPodAdmissionHandler := preemption.NewCriticalPodAdmissionHandler(klet.GetActivePods, killPodNow(klet.podWorkers, kubeDeps.Recorder), kubeDeps.Recorder)
	klet.admitHandlers.AddPodAdmitHandler(lifecycle.NewPredicateAdmitHandler(klet.getNodeAnyWay, criticalPodAdmissionHandler, klet.containerManager.UpdatePluginResources))
	// apply functional Option's
	for _, opt := range kubeDeps.Options {
		opt(klet)
	}

	klet.appArmorValidator = apparmor.NewValidator(containerRuntime)
	klet.softAdmitHandlers.AddPodAdmitHandler(lifecycle.NewAppArmorAdmitHandler(klet.appArmorValidator))
	klet.softAdmitHandlers.AddPodAdmitHandler(lifecycle.NewNoNewPrivsAdmitHandler(klet.containerRuntime))
	// Finally, put the most recent version of the config on the Kubelet, so
	// people can see how it was configured.
	klet.kubeletConfiguration = *kubeCfg

	// Generating the status funcs should be the last thing we do,
	// since this relies on the rest of the Kubelet having been constructed.
	klet.setNodeStatusFuncs = klet.defaultNodeStatusFuncs()

	return klet, nil
}
```
#### 启动kubelet
回溯前面RunKubelet的实现，创建并初始化Kubelet后，接下来就启动kubelet。其主要流程：首先通过goroutine调用kubelet.Run方法启动kubelet，然后创建kubelet http server对外暴露接口服务。
```
func startKubelet(k kubelet.Bootstrap, podCfg *config.PodConfig, kubeCfg *kubeletconfiginternal.KubeletConfiguration, kubeDeps *kubelet.Dependencies, enableServer bool) {
	// start the kubelet
	go wait.Until(func() {
		k.Run(podCfg.Updates())
	}, 0, wait.NeverStop)

	// start the kubelet server
	if enableServer {
		go k.ListenAndServe(net.ParseIP(kubeCfg.Address), uint(kubeCfg.Port), kubeDeps.TLSOptions, kubeDeps.Auth, kubeCfg.EnableDebuggingHandlers, kubeCfg.EnableContentionProfiling)

	}
	if kubeCfg.ReadOnlyPort > 0 {
		go k.ListenAndServeReadOnly(net.ParseIP(kubeCfg.Address), uint(kubeCfg.ReadOnlyPort))
	}
}
```
##### 启动kubelet开始处理各pod来源的pod变更请求
启动kubelet工作依赖的各组件，每个组件都是以goroutine来运行的，最终pod变更处理的入口在syncLoop这里开始。
```
// Run starts the kubelet reacting to config updates
func (kl *Kubelet) Run(updates <-chan kubetypes.PodUpdate) {
	if kl.logServer == nil {
		kl.logServer = http.StripPrefix("/logs/", http.FileServer(http.Dir("/var/log/")))
	}

	// Start the cloud provider sync manager
	if kl.cloudResourceSyncManager != nil {
		go kl.cloudResourceSyncManager.Run(wait.NeverStop)
	}

	if err := kl.initializeModules(); err != nil {
		kl.recorder.Eventf(kl.nodeRef, v1.EventTypeWarning, events.KubeletSetupFailed, err.Error())
		glog.Fatal(err)
	}

	// Start volume manager
	go kl.volumeManager.Run(kl.sourcesReady, wait.NeverStop)

	if kl.kubeClient != nil {
		// Start syncing node status immediately, this may set up things the runtime needs to run.
		go wait.Until(kl.syncNodeStatus, kl.nodeStatusUpdateFrequency, wait.NeverStop)
	}
	go wait.Until(kl.updateRuntimeUp, 5*time.Second, wait.NeverStop)

	// Start loop to sync iptables util rules
	if kl.makeIPTablesUtilChains {
		go wait.Until(kl.syncNetworkUtil, 1*time.Minute, wait.NeverStop)
	}

	// Start a goroutine responsible for killing pods (that are not properly
	// handled by pod workers).
	go wait.Until(kl.podKiller, 1*time.Second, wait.NeverStop)

	// Start component sync loops.
	kl.statusManager.Start()
	kl.probeManager.Start()

	// Start the pod lifecycle event generator.
	kl.pleg.Start()
	kl.syncLoop(updates, kl)
}
```
##### syncLoop kubelet主要工作循环体
syncLoop 是kubelet的主循环方法，它从不同的管道(FILE,URL, API-SERVER)监听到pod的变化，并把它们汇聚起来。当有新的变化发生时，它会调用对应的函数，保证Pod处于期望的状态。
```
// syncLoop is the main loop for processing changes. It watches for changes from
// three channels (file, apiserver, and http) and creates a union of them. For
// any new change seen, will run a sync against desired state and running state. If
// no changes are seen to the configuration, will synchronize the last known desired
// state every sync-frequency seconds. Never returns.
func (kl *Kubelet) syncLoop(updates <-chan kubetypes.PodUpdate, handler SyncHandler) {
	glog.Info("Starting kubelet main sync loop.")
	// The resyncTicker wakes up kubelet to checks if there are any pod workers
	// that need to be sync'd. A one-second period is sufficient because the
	// sync interval is defaulted to 10s.
	syncTicker := time.NewTicker(time.Second)
	defer syncTicker.Stop()
	housekeepingTicker := time.NewTicker(housekeepingPeriod)
	defer housekeepingTicker.Stop()
	plegCh := kl.pleg.Watch()
	const (
		base   = 100 * time.Millisecond
		max    = 5 * time.Second
		factor = 2
	)
	duration := base
	for {
		if rs := kl.runtimeState.runtimeErrors(); len(rs) != 0 {
			glog.Infof("skipping pod synchronization - %v", rs)
			// exponential backoff
			time.Sleep(duration)
			duration = time.Duration(math.Min(float64(max), factor*float64(duration)))
			continue
		}
		// reset backoff if we have a success
		duration = base

		if !kl.syncLoopIteration(updates, handler, syncTicker.C, housekeepingTicker.C, plegCh) {
			break
		}
	}
}
```
##### syncLoopIteration
syncLoopIteration这个方法会对多个管道进行遍历，如果有pod动作，则会调用相应的Handler。Handler的接口定义如下：
```
type SyncHandler interface {
	HandlePodAdditions(pods []*v1.Pod)
	HandlePodUpdates(pods []*v1.Pod)
	HandlePodRemoves(pods []*v1.Pod)
	HandlePodReconcile(pods []*v1.Pod)
	HandlePodSyncs(pods []*v1.Pod)
	HandlePodCleanups() error
}
```
而这里Handler的接口实现对象为Kubelet。
```
// syncLoopIteration reads from various channels and dispatches pods to the
// given handler.
//
// Arguments:
// 1.  configCh:       a channel to read config events from
// 2.  handler:        the SyncHandler to dispatch pods to
// 3.  syncCh:         a channel to read periodic sync events from
// 4.  houseKeepingCh: a channel to read housekeeping events from
// 5.  plegCh:         a channel to read PLEG updates from
//
// Events are also read from the kubelet liveness manager's update channel.
//
// The workflow is to read from one of the channels, handle that event, and
// update the timestamp in the sync loop monitor.

func (kl *Kubelet) syncLoopIteration(configCh <-chan kubetypes.PodUpdate, handler SyncHandler,
	syncCh <-chan time.Time, housekeepingCh <-chan time.Time, plegCh <-chan *pleg.PodLifecycleEvent) bool {
	select {
	case u, open := <-configCh:
		// Update from a config source; dispatch it to the right handler
		// callback.
		if !open {
			glog.Errorf("Update channel is closed. Exiting the sync loop.")
			return false
		}

		switch u.Op {
		case kubetypes.ADD:
			glog.V(2).Infof("SyncLoop (ADD, %q): %q", u.Source, format.Pods(u.Pods))
			handler.HandlePodAdditions(u.Pods)
		case kubetypes.UPDATE:
			glog.V(2).Infof("SyncLoop (UPDATE, %q): %q", u.Source, format.PodsWithDeletiontimestamps(u.Pods))
			handler.HandlePodUpdates(u.Pods)
		case kubetypes.REMOVE:
			glog.V(2).Infof("SyncLoop (REMOVE, %q): %q", u.Source, format.Pods(u.Pods))
			handler.HandlePodRemoves(u.Pods)
		case kubetypes.RECONCILE:
			glog.V(4).Infof("SyncLoop (RECONCILE, %q): %q", u.Source, format.Pods(u.Pods))
			handler.HandlePodReconcile(u.Pods)
		case kubetypes.DELETE:
			glog.V(2).Infof("SyncLoop (DELETE, %q): %q", u.Source, format.Pods(u.Pods))
			// DELETE is treated as a UPDATE because of graceful deletion.
			handler.HandlePodUpdates(u.Pods)
		case kubetypes.RESTORE:
			glog.V(2).Infof("SyncLoop (RESTORE, %q): %q", u.Source, format.Pods(u.Pods))
			// These are pods restored from the checkpoint. Treat them as new
			// pods.
			handler.HandlePodAdditions(u.Pods)
		case kubetypes.SET:
			// TODO: Do we want to support this?
			glog.Errorf("Kubelet does not support snapshot update")
		}

		if u.Op != kubetypes.RESTORE {
			kl.sourcesReady.AddSource(u.Source)
		}
	case e := <-plegCh:
		if isSyncPodWorthy(e) {
			// PLEG event for a pod; sync it.
			if pod, ok := kl.podManager.GetPodByUID(e.ID); ok {
				glog.V(2).Infof("SyncLoop (PLEG): %q, event: %#v", format.Pod(pod), e)
				handler.HandlePodSyncs([]*v1.Pod{pod})
			} else {
				glog.V(4).Infof("SyncLoop (PLEG): ignore irrelevant event: %#v", e)
			}
		}

		if e.Type == pleg.ContainerDied {
			if containerID, ok := e.Data.(string); ok {
				kl.cleanUpContainersInPod(e.ID, containerID)
			}
		}
	case <-syncCh:
		// Sync pods waiting for sync
		podsToSync := kl.getPodsToSync()
		if len(podsToSync) == 0 {
			break
		}
		glog.V(4).Infof("SyncLoop (SYNC): %d pods; %s", len(podsToSync), format.Pods(podsToSync))
		handler.HandlePodSyncs(podsToSync)
	case update := <-kl.livenessManager.Updates():
		if update.Result == proberesults.Failure {
			pod, ok := kl.podManager.GetPodByUID(update.PodUID)
			if !ok {
				// If the pod no longer exists, ignore the update.
				glog.V(4).Infof("SyncLoop (container unhealthy): ignore irrelevant update: %#v", update)
				break
			}
			glog.V(1).Infof("SyncLoop (container unhealthy): %q", format.Pod(pod))
			handler.HandlePodSyncs([]*v1.Pod{pod})
		}
	case <-housekeepingCh:
		if !kl.sourcesReady.AllReady() {
			glog.V(4).Infof("SyncLoop (housekeeping, skipped): sources aren't ready yet.")
		} else {
			glog.V(4).Infof("SyncLoop (housekeeping)")
			if err := handler.HandlePodCleanups(); err != nil {
				glog.Errorf("Failed cleaning pods: %v", err)
			}
		}
	}
	return true
}
```
我们以创建Pod为例,它会调用对应的HandlePodAdditions Handler进行处理。HandlePodAdditions做的任务就是通过canAdmitPod方法校验Pod能否在该计算节点创建(如:磁盘空间)。之后把创建Pod的事交给dispatchWork。
```
// HandlePodAdditions is the callback in SyncHandler for pods being added from
// a config source.
func (kl *Kubelet) HandlePodAdditions(pods []*v1.Pod) {
	sort.Sort(sliceutils.PodsByCreationTime(pods))
	for _, pod := range pods {
		// Responsible for checking limits in resolv.conf
		if kl.dnsConfigurer != nil && kl.dnsConfigurer.ResolverConfig != "" {
			kl.dnsConfigurer.CheckLimitsForResolvConf()
		}
		existingPods := kl.podManager.GetPods()
		// Always add the pod to the pod manager. Kubelet relies on the pod
		// manager as the source of truth for the desired state. If a pod does
		// not exist in the pod manager, it means that it has been deleted in
		// the apiserver and no action (other than cleanup) is required.
		kl.podManager.AddPod(pod)

		if kubepod.IsMirrorPod(pod) {
			kl.handleMirrorPod(pod, start)
			continue
		}

		if !kl.podIsTerminated(pod) {
			// Only go through the admission process if the pod is not terminated. We failed pods that we rejected, so activePods include all admitted pods that are alive.
			activePods := kl.filterOutTerminatedPods(existingPods)

			// Check if we can admit the pod; if not, reject it.
			if ok, reason, message := kl.canAdmitPod(activePods, pod); !ok {
				kl.rejectPod(pod, reason, message)
				continue
			}
		}
		mirrorPod, _ := kl.podManager.GetMirrorPodByPod(pod)
		kl.dispatchWork(pod, kubetypes.SyncPodCreate, mirrorPod, start)
		kl.probeManager.AddPod(pod)
	}
}
```
接下来dispatchWork主要工作就是把接收到的参数封装成UpdatePodOptions，调用 UpdatePod 方法.
```
// dispatchWork starts the asynchronous sync of the pod in a pod worker.
// If the pod is terminated, dispatchWork
func (kl *Kubelet) dispatchWork(pod *v1.Pod, syncType kubetypes.SyncPodType, mirrorPod *v1.Pod, start time.Time) {
	if kl.podIsTerminated(pod) {
		if pod.DeletionTimestamp != nil {
			// If the pod is in a terminated state, there is no pod worker to
			// handle the work item. Check if the DeletionTimestamp has been
			// set, and force a status update to trigger a pod deletion request
			// to the apiserver.
			kl.statusManager.TerminatePod(pod)
		}
		return
	}
	// Run the sync in an async worker.
	kl.podWorkers.UpdatePod(&UpdatePodOptions{
		Pod:        pod,
		MirrorPod:  mirrorPod,
		UpdateType: syncType,
		OnCompleteFunc: func(err error) {
			if err != nil {
				metrics.PodWorkerLatency.WithLabelValues(syncType.String()).Observe(metrics.SinceInMicroseconds(start))
			}
		},
	})
	// Note the number of containers for new pods.
	if syncType == kubetypes.SyncPodCreate {
	 metrics.ContainersPerPodCount.Observe(float64(len(pod.Spec.Containers)))
	}
}
```
podWorkers调用UpdatePod方法为新pod创建一个新的pod worker，处理该pod的podUpdates channel中所有同步更新请求。(pkg/kubelet/pod_workers.go)

```
// Apply the new setting to the specified pod.
func (p *podWorkers) UpdatePod(options *UpdatePodOptions) {
	pod := options.Pod
	uid := pod.UID
	var podUpdates chan UpdatePodOptions
	var exists bool

	if podUpdates, exists = p.podUpdates[uid]; !exists {
		// We need to have a buffer here, because checkForUpdates() method that
		// puts an update into channel is called from the same goroutine where
		// the channel is consumed. However, it is guaranteed that in such case
		// the channel is empty, so buffer of size 1 is enough.
		podUpdates = make(chan UpdatePodOptions, 1)
		p.podUpdates[uid] = podUpdates

		// Creating a new pod worker either means this is a new pod, or that the kubelet just restarted. In either case the kubelet is willing to believe
 the status of the pod for the first pod worker sync.
		go func() {
			defer runtime.HandleCrash()
			p.managePodLoop(podUpdates)
		}()
	}
	if !p.isWorking[pod.UID] {
		p.isWorking[pod.UID] = true
		podUpdates <- *options
	} else {
		// if a request to kill a pod is pending, we do not let anything overwrite that request.
		update, found := p.lastUndeliveredWorkUpdate[pod.UID]
		if !found || update.UpdateType != kubetypes.SyncPodKill {
			p.lastUndeliveredWorkUpdate[pod.UID] = *options
		}
	}
}
```
pod worker循环处理podUpdates channel中的更新请求，调用syncPodFn来同步pod，这里的syncPodFn其实为Kubelet.syncPod方法。
```
func (p *podWorkers) managePodLoop(podUpdates <-chan UpdatePodOptions) {
	var lastSyncTime time.Time
	for update := range podUpdates {
		err := func() error {
			podUID := update.Pod.UID

			err = p.syncPodFn(syncPodOptions{
				mirrorPod:      update.MirrorPod,
				pod:            update.Pod,
				podStatus:      status,
				killPodOptions: update.KillPodOptions,
				updateType:     update.UpdateType,
			})
			lastSyncTime = time.Now()
			return err
		}()
		// notify the call-back function if the operation succeeded or not
		if update.OnCompleteFunc != nil {
			update.OnCompleteFunc(err)
		}
		p.wrapUp(update.Pod.UID, err)
	}
}
```
syncPod是处理单个pod同步操作句柄，其最终回调kl.containerRuntime.SyncPod方法来使节点上运行的pod处于期望的状态。
```
// syncPod is the transaction script for the sync of a single pod.
// The workflow is:
// * If the pod is being created, record pod worker start latency
// * Call generateAPIPodStatus to prepare an v1.PodStatus for the pod
// * If the pod is being seen as running for the first time, record pod
//   start latency
// * Update the status of the pod in the status manager
// * Kill the pod if it should not be running
// * Create a mirror pod if the pod is a static pod, and does not
//   already have a mirror pod
// * Create the data directories for the pod if they do not exist
// * Wait for volumes to attach/mount
// * Fetch the pull secrets for the pod
// * Call the container runtime's SyncPod callback
// * Update the traffic shaping for the pod's ingress and egress limits
func (kl *Kubelet) syncPod(o syncPodOptions) error {
	// pull out the required options
	pod := o.pod
	mirrorPod := o.mirrorPod
	podStatus := o.podStatus
	updateType := o.updateType

	// kill a pod
	if updateType == kubetypes.SyncPodKill {
		killPodOptions := o.killPodOptions
		apiPodStatus := killPodOptions.PodStatusFunc(pod, podStatus)
		kl.statusManager.SetPodStatus(pod, apiPodStatus)
		// we kill the pod with the specified grace period since this is a termination
		if err := kl.killPod(pod, nil, podStatus, killPodOptions.PodTerminationGracePeriodSecondsOverride); err != nil {
			// there was an error killing the pod, so we return that error directly
			utilruntime.HandleError(err)
			return err
		}
		return nil
	}

	// Record pod worker start latency if being created
	if updateType == kubetypes.SyncPodCreate {
	    ...
	}

	// Generate final API pod status with pod and status manager status
	apiPodStatus := kl.generateAPIPodStatus(pod, podStatus)
	// The pod IP may be changed in generateAPIPodStatus if the pod is using host network. 
	podStatus.IP = apiPodStatus.PodIP

	// Record the time it takes for the pod to become running.
    ...

	runnable := kl.canRunPod(pod)
	if !runnable.Admit {
		// Pod is not runnable; update the Pod and Container statuses to why.
	    ...
	}

	// Update status in the status manager
	kl.statusManager.SetPodStatus(pod, apiPodStatus)

	// Kill pod if it should not be running
	if !runnable.Admit || pod.DeletionTimestamp != nil || apiPodStatus.Phase == v1.PodFailed {
	    。。。
		return syncErr
	}

	// If the network plugin is not ready, only start the pod if it uses the host network
	if rs := kl.runtimeState.networkErrors(); len(rs) != 0 && !kubecontainer.IsHostNetworkPod(pod) {
		return fmt.Errorf("network is not ready: %v", rs)
	}

	// Create Cgroups for the pod and apply resource parameters
	// to them if cgroups-per-qos flag is enabled.
	pcm := kl.containerManager.NewPodContainerManager()
	//为pod创建或更新cgroup设置
	if !kl.podIsTerminated(pod) {
	    ...
	}

	// Create Mirror Pod for Static Pod if it doesn't already exist
	//处理static pod的情况
	if kubepod.IsStaticPod(pod) {
		...
	}

	// Make data directories for the pod 
	//为POD新建数据目录
	if err := kl.makePodDataDirs(pod); err != nil {
		glog.Errorf("Unable to make pod data directories for pod %q: %v", format.Pod(pod), err)
		return err
	}

	// Volume manager will not mount volumes for terminated pods
	if !kl.podIsTerminated(pod) {
		// 等待pod卷挂载完成
		if err := kl.volumeManager.WaitForAttachAndMount(pod); err != nil {
			glog.Errorf("Unable to mount volumes for pod %q: %v; skipping pod", format.Pod(pod), err)
			return err
		}
	}

	// Fetch the pull secrets for the pod
	//拉取pod所需的secrets
	pullSecrets := kl.getPullSecretsForPod(pod)

	// Call the container runtime's SyncPod callback
	//调用真正的容器运行时SyncPod回调方法使Pod处于期望的状态
	result := kl.containerRuntime.SyncPod(pod, apiPodStatus, podStatus, pullSecrets, kl.backOff)
	kl.reasonCache.Update(pod.UID, result)

	return nil
}
```
回溯前面创建初始化kubelet的过程不难发现kubelet.containerRuntime其实现为NewKubeGenericRuntimeManager创建的kubeGenericRuntimeManager对象，那么kubelet.containerRuntime.SyncPod的实现即为kubeGenericRuntimeManager的SyncPod方法实现，而此处SyncPod是创建Pod的核心逻辑。

```
// SyncPod syncs the running pod into the desired pod by executing following steps:
//
//  1. Compute sandbox and container changes.
//  2. Kill pod sandbox if necessary.
//  3. Kill any containers that should not be running.
//  4. Create sandbox if necessary.
//  5. Create init containers.
//  6. Create normal containers.
func (m *kubeGenericRuntimeManager) SyncPod(pod *v1.Pod, _ v1.PodStatus, podStatus *kubecontainer.PodStatus, pullSecrets []v1.Secret, backOff *flowcontrol.Backoff) (result kubecontainer.PodSyncResult) {
	// Step 1: Compute sandbox and container changes.
	podContainerChanges := m.computePodActions(pod, podStatus)
    //是否需要创建沙箱
	if podContainerChanges.CreateSandbox {
	    。。。
	}

	// Step 2: Kill the pod if the sandbox has changed.
	//是否停止该pod的所有容器及沙箱
	if podContainerChanges.KillPod {
		killResult := m.killPodWithSyncResult(pod, kubecontainer.ConvertPodStatusToRunningPod(m.runtimeName, podStatus), nil)
		result.AddPodSyncResult(killResult)
        //如果需要创建沙箱，那么需要删除pod所有的init容器
		if podContainerChanges.CreateSandbox {
			m.purgeInitContainers(pod, podStatus)
		}
	} else {
		// Step 3: kill any running containers in this pod which are not to keep.
		for containerID, containerInfo := range podContainerChanges.ContainersToKill {
			killContainerResult := kubecontainer.NewSyncResult(kubecontainer.KillContainer, containerInfo.name)
			result.AddSyncResult(killContainerResult)
			if err := m.killContainer(pod, containerID, containerInfo.name, containerInfo.message, nil); err != nil {
				killContainerResult.Fail(kubecontainer.ErrKillContainer, err.Error())
				glog.Errorf("killContainer %q(id=%q) for pod %q failed: %v", containerInfo.name, containerID, format.Pod(pod), err)
				return
			}
		}
	}

	// Keep terminated init containers fairly aggressively controlled
	// This is an optimization because container removals are typically handled
	// by container garbage collector.
	m.pruneInitContainersBeforeStart(pod, podStatus)

	// We default to the IP in the passed-in pod status, and overwrite it if the sandbox needs to be (re)started.
	podIP := ""
	if podStatus != nil {
		podIP = podStatus.IP
	}

	// Step 4: Create a sandbox for the pod if necessary.
	podSandboxID := podContainerChanges.SandboxID
	if podContainerChanges.CreateSandbox {
		var msg string
		var err error

		glog.V(4).Infof("Creating sandbox for pod %q", format.Pod(pod))
		createSandboxResult := kubecontainer.NewSyncResult(kubecontainer.CreatePodSandbox, format.Pod(pod))
		result.AddSyncResult(createSandboxResult)
		//为pod创建沙箱
		podSandboxID, msg, err = m.createPodSandbox(pod, podContainerChanges.Attempt)
		glog.V(4).Infof("Created PodSandbox %q for pod %q", podSandboxID, format.Pod(pod))
        //查询获取当前pod沙箱状态
		podSandboxStatus, err := m.runtimeService.PodSandboxStatus(podSandboxID)

		// If we ever allow updating a pod from non-host-network to
		// host-network, we may use a stale IP.
		if !kubecontainer.IsHostNetworkPod(pod) {
			// Overwrite the podIP passed in the pod status, since we just started the pod sandbox.
			podIP = m.determinePodSandboxIP(pod.Namespace, pod.Name, podSandboxStatus)
		}
	}

	// Get podSandboxConfig for containers to start.
	configPodSandboxResult := kubecontainer.NewSyncResult(kubecontainer.ConfigPodSandbox, podSandboxID)
	result.AddSyncResult(configPodSandboxResult)
	podSandboxConfig, err := m.generatePodSandboxConfig(pod, podContainerChanges.Attempt)

	// Step 5: start the init container. 启动init容器
	if container := podContainerChanges.NextInitContainerToStart; container != nil {
		// Start the next init container.
		startContainerResult := kubecontainer.NewSyncResult(kubecontainer.StartContainer, container.Name)
		result.AddSyncResult(startContainerResult)
	
		if msg, err := m.startContainer(podSandboxID, podSandboxConfig, container, pod, podStatus, pullSecrets, podIP, kubecontainer.ContainerTypeInit); err != nil {
			startContainerResult.Fail(err, msg)
			utilruntime.HandleError(fmt.Errorf("init container start failed: %v: %s", err, msg))
			return
		}
	}

	// Step 6: start containers in podContainerChanges.ContainersToStart.
	//启动pod内容器
	for _, idx := range podContainerChanges.ContainersToStart {
		container := &pod.Spec.Containers[idx]
		startContainerResult := kubecontainer.NewSyncResult(kubecontainer.StartContainer, container.Name)
		result.AddSyncResult(startContainerResult)

		if msg, err := m.startContainer(podSandboxID, podSandboxConfig, container, pod, podStatus, pullSecrets, podIP, kubecontainer.ContainerTypeRegular); err != nil {
			startContainerResult.Fail(err, msg)
			// known errors that are logged in other places are logged at higher levels here to avoid
			// repetitive log spam
			switch {
			case err == images.ErrImagePullBackOff:
				glog.V(3).Infof("container start failed: %v: %s", err, msg)
			default:
				utilruntime.HandleError(fmt.Errorf("container start failed: %v: %s", err, msg))
			}
			continue
		}
	}

	return
}
```
可以看到其主要流程为：


* computePodContainerChanges 根据最新拿到的pod配置与当前运行的容器配置进行对比，计算其中的变化(一个具体的hash值)，得到需要重启的容器的信息。
* createPodSandBox 创建一个PodSandBox。
```
// createPodSandbox creates a pod sandbox and returns (podSandBoxID, message, error).
func (m *kubeGenericRuntimeManager) createPodSandbox(pod *v1.Pod, attempt uint32) (string, string, error) {
    podSandboxConfig, err := m.generatePodSandboxConfig(pod, attempt)
    if err != nil {
         //......
    }

    // Create pod logs directory
    err = m.osInterface.MkdirAll(podSandboxConfig.LogDirectory, 0755)
    if err != nil {
        //......
    }

    podSandBoxID, err := m.runtimeService.RunPodSandbox(podSandboxConfig)
    if err != nil {
        //......
    }

    return podSandBoxID, "", nil
}
```
* generatePodSandboxConfig 获取PodSandbox的配置(如:metadata,clusterDNS,容器的端口映射等)

* RunPodSandbox 创建并开启一个Pod级别的sandbox。
```
func (ds *dockerService) RunPodSandbox(config *runtimeapi.PodSandboxConfig) (id string, err error) {
    // Step 1: Pull the image for the sandbox.
    image := defaultSandboxImage
    podSandboxImage := ds.podSandboxImage
    if len(podSandboxImage) != 0 {
        image = podSandboxImage
    }
    if err := ensureSandboxImageExists(ds.client, image); err != nil {
        return "", err
    }

    // Step 2: Create the sandbox container.
    createConfig, err := ds.makeSandboxDockerConfig(config, image)
    createResp, err := ds.client.CreateContainer(*createConfig)

    // Step 3: Create Sandbox Checkpoint.
    if err = ds.checkpointHandler.CreateCheckpoint(createResp.ID, constructPodSandboxCheckpoint(config)); err != nil {
        return createResp.ID, err
    }

    // Step 4: Start the sandbox container.
    err = ds.client.StartContainer(createResp.ID)

    // Step 5: Setup networking for the sandbox.
    cID := kubecontainer.BuildContainerID(runtimeName, createResp.ID)
    err = ds.network.SetUpPod(config.GetMetadata().Namespace, config.GetMetadata().Name, cID, config.Annotations)

    return createResp.ID
}
```
* RunPodSandbox主要流程为：
1. ensureSandboxImageExists 检测用户是否设置了自己的pause镜像，如果没有设置则使用默认的gcr.io/google_containers/pause-amd64:3.0镜像。
2. makeSandboxDockerConfig 生成创建pause容器的配置信息。
3. CreateContainer 创建容器
4. StartContainer 启动容器
5. network.SetUpPod 设置容器的网络(kubelet加载cni插件对容器的网络进行设置等)

上面的这些操作就把我们Pod中的第一个pause容器创建并启动了。之后要做的就是把该Pod中的其它业务容器逐一的启动。但是在启动真正的业务容器之前,首先会检查用户是否设置了init_container。如果设置了，则会按init_container设置的顺序依次的执行init_container(注意:当其中的init_container执行失败了，则Pod会异常，并且业务容器不会被创建)。当init_container执行完成之后，我们真正的业务容器才会被逐一的启动。

* 业务容器启动的逻辑和Pod的初始化pause容器的启动的流程基本一致。下面的代码是循环的启动业务容器。
```
for idx := range podContainerChanges.ContainersToStart {
        container := &pod.Spec.Containers[idx]

        //.....

        glog.V(4).Infof("Creating container %+v in pod %v", container, format.Pod(pod))
        if msg, err := m.startContainer(podSandboxID, podSandboxConfig, container, pod, podStatus, pullSecrets, podIP); err != nil {
            //.....
            continue
        }
}
```
启动业务容器实现：
```
func (m *kubeGenericRuntimeManager) startContainer(podSandboxID string, podSandboxConfig *runtimeapi.PodSandboxConfig, container *v1.Container, pod *v1.Pod, podStatus *kubecontainer.PodStatus, pullSecrets []v1.Secret, podIP string) (string, error) {
    // Step 1: pull the image.
    imageRef, msg, err := m.imagePuller.EnsureImageExists(pod, container, pullSecrets)

    // Step 2: create the container.    
    containerConfig, err := m.generateContainerConfig(container, pod, restartCount, podIP, imageRef)
    containerID, err := m.runtimeService.CreateContainer(podSandboxID, containerConfig, podSandboxConfig)

    // Step 3: start the container.
    err = m.runtimeService.StartContainer(containerID)

    // Step 4: execute the post start hook.
    if container.Lifecycle != nil && container.Lifecycle.PostStart != nil {
        kubeContainerID := kubecontainer.ContainerID{
            Type: m.runtimeName,
            ID:   containerID,
        }
        msg, handlerErr := m.runner.Run(kubeContainerID, pod, container, container.Lifecycle.PostStart)
        if handlerErr != nil {
            m.recordContainerEvent(pod, container, kubeContainerID.ID, v1.EventTypeWarning, events.FailedPostStartHook, msg)
            m.killContainer(pod, kubeContainerID, container.Name, "FailedPostStartHook", nil)
            return "PostStart Hook Failed", err
        }
    }

    return "", nil
}
```
1. EnsureImageExists 检查业务镜像是否存在，不存在则到Docker Registry或是Private Registry拉取镜像。
2. generateContainerConfig 生成业务容器的配置信息
3. CreateContainer 通过client.CreateContainer调用docker engine-api创建业务容器。
4. StartContainer 启动业务容器
5. runner.Run 这个方法的主要作用就是在业务容器起来的时候，首先会执行一个container hook(PostStart和PreStop),做一些预处理工作。只有container hook执行成功才会运行具体的业务服务，否则容器异常。

### Pod创建流程总结
![POD创建流程图](/assets/pod_create_routine.png)

这样Pod大体的启动流程就描述完了，但是对于kubelet中其它的中间服务,如: volumeManager,diskSpaceManager,secretManager,configMapManager等等就需要更深层的了解了。

