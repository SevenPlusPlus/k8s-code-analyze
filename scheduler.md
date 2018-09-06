## scheduler启动流程

### 调度策略注册

* pkg/scheduler/algorithmprovider/defaults/defaults.go:

```
func init() {
    // Register functions that extract metadata used by predicates and priorities computations.
    factory.RegisterPredicateMetadataProducerFactory(
        func(args factory.PluginFactoryArgs) algorithm.PredicateMetadataProducer {
            return predicates.NewPredicateMetadataFactory(args.PodLister)
        })
    factory.RegisterPriorityMetadataProducerFactory(
        func(args factory.PluginFactoryArgs) algorithm.PriorityMetadataProducer {
            return priorities.NewPriorityMetadataFactory(args.ServiceLister, args.ControllerLister, args.ReplicaSetLister, args.StatefulSetLister)
        })

    registerAlgorithmProvider(defaultPredicates(), defaultPriorities())

    factory.RegisterFitPredicate("PodFitsPorts", predicates.PodFitsHostPorts)
    ...
}

//注册默认的Predicates，如NoVolumeZoneConflictPred、MaxEBSVolumeCountPred。。。
func defaultPredicates() sets.String {
    return sets.NewString(
        // Fit is determined by volume zone requirements.
        factory.RegisterFitPredicateFactory(
            predicates.NoVolumeZoneConflictPred,
            func(args factory.PluginFactoryArgs) algorithm.FitPredicate {
                return predicates.NewVolumeZonePredicate(args.PVInfo, args.PVCInfo, args.StorageClassInfo)
            },
        ),
        // Fit is determined by whether or not there would be too many AWS EBS volumes attached to the node
        factory.RegisterFitPredicateFactory(
            predicates.MaxEBSVolumeCountPred,
            func(args factory.PluginFactoryArgs) algorithm.FitPredicate {
                return predicates.NewMaxPDVolumeCountPredicate(predicates.EBSVolumeFilterType, args.PVInfo, args.PVCInfo)
            },
        ),
    ...
}

//注册默认的Priorities,如SelectorSpreadPriority、InterPodAffinityPriority等
func defaultPriorities() sets.String {
    return sets.NewString(
        // spreads pods by minimizing the number of pods (belonging to the same service or replication controller) on the same node.
        factory.RegisterPriorityConfigFactory(
            "SelectorSpreadPriority",
            factory.PriorityConfigFactory{
                MapReduceFunction: func(args factory.PluginFactoryArgs) (algorithm.PriorityMapFunction, algorithm.PriorityReduceFunction) {
                    return priorities.NewSelectorSpreadPriority(args.ServiceLister, args.ControllerLister, args.ReplicaSetLister, args.StatefulSetLister)
                },
                Weight: 1,
            },
        ),
        // pods should be placed in the same topological domain (e.g. same node, same rack, same zone, same power domain, etc.)
        // as some other pods, or, conversely, should not be placed in the same topological domain as some other pods.
        factory.RegisterPriorityConfigFactory(
            "InterPodAffinityPriority",
            factory.PriorityConfigFactory{
                Function: func(args factory.PluginFactoryArgs) algorithm.PriorityFunction {
                    return priorities.NewInterPodAffinityPriority(args.NodeInfo, args.NodeLister, args.PodLister, args.HardPodAffinitySymmetricWeight)
                },
                Weight: 1,
            },
        ),
   ...
}
```

### 服务启动流程

* cmd/kube-scheduler/scheduler.go:

  ```
  func main() {
   command := app.NewSchedulerCommand()
   command.Execute()
  }
  ```

  新建SchedulerCommand并run

* cmd/kube-scheduler/app/server.go:

启动SchedulerCommand流程如下：

1. 生成scheduler配置
2. 创建调度器scheduler
3. 启动健康检查Healthz和监控Metrics HTTP服务
4. 开启对各种调度所需资源的ListerAndWatcher，开始watch ApiServer，及时获取最新资源并调度前等待所有Informer同步完成
5. 高可用模式下完成选主后，主节点上启动调度器

```
func Run(c schedulerserverconfig.CompletedConfig, stopCh <-chan struct{}) error {
    // Apply algorithms based on feature gates.
    algorithmprovider.ApplyFeatureGates()

    // Build a scheduler config from the provided algorithm source.
    schedulerConfig, err := NewSchedulerConfig(c)
    // Create the scheduler.
    sched := scheduler.NewFromConfig(schedulerConfig)

    // Start up the healthz server.
    if c.InsecureServing != nil {
        separateMetrics := c.InsecureMetricsServing != nil
        handler := buildHandlerChain(newHealthzHandler(&c.ComponentConfig, separateMetrics), nil, nil)
        c.InsecureServing.Serve(handler, 0, stopCh)
    }
    if c.InsecureMetricsServing != nil {
        handler := buildHandlerChain(newMetricsHandler(&c.ComponentConfig), nil, nil)
        c.InsecureMetricsServing.Serve(handler, 0, stopCh)
    }
    if c.SecureServing != nil {
        handler := buildHandlerChain(newHealthzHandler(&c.ComponentConfig, false), c.Authentication.Authenticator, c.Authorization.Authorizer)
        c.SecureServing.Serve(handler, 0, stopCh)
    }

      // Start all informers.
    go c.PodInformer.Informer().Run(stopCh)
    c.InformerFactory.Start(stopCh)
    // Wait for all caches to sync before scheduling.
    c.InformerFactory.WaitForCacheSync(stopCh)
    controller.WaitForCacheSync("scheduler", stopCh, c.PodInformer.Informer().HasSynced)
    // 启动调度器的函数句柄，Prepare a reusable run function.
    run := func(ctx context.Context) {
        sched.Run()
    <-ctx.Done()
    }
// If leader election is enabled, run via LeaderElector until done and exit.
    if c.LeaderElection != nil {
        c.LeaderElection.Callbacks = leaderelection.LeaderCallbacks{
            OnStartedLeading: run,
            OnStoppedLeading: func() {
                utilruntime.HandleError(fmt.Errorf("lost master"))
            },
        }
        leaderElector, err := leaderelection.NewLeaderElector(*c.LeaderElection)
        if err != nil {
            return fmt.Errorf("couldn't create leader elector: %v", err)
        }

        leaderElector.Run(ctx)

        return fmt.Errorf("lost lease")
    }

    // Leader election is disabled, so run inline until done.
    run(ctx)
}
```

#### 生成调度器配置

生成调度器配置首先新建一个configFactory，其内部包含了很多调度器所需的资源类型的ListerAndWatcher，然后有两种模式创建调度器配置，一种是根据用户设置的调度配置（Predicats + Priorities组合），另外一种是采用系统默认提供的调度算法配置。

```
// NewSchedulerConfig creates the scheduler configuration. This is exposed for use by tests.
func NewSchedulerConfig(s schedulerserverconfig.CompletedConfig) (*scheduler.Config, error) {
    var storageClassInformer storageinformers.StorageClassInformer
    if utilfeature.DefaultFeatureGate.Enabled(features.VolumeScheduling) {
        storageClassInformer = s.InformerFactory.Storage().V1().StorageClasses()
    }

    // Set up the configurator which can create schedulers from configs.
    configurator := factory.NewConfigFactory(
        s.ComponentConfig.SchedulerName,
        s.Client,
        s.InformerFactory.Core().V1().Nodes(),
        s.PodInformer,
        s.InformerFactory.Core().V1().PersistentVolumes(),
        s.InformerFactory.Core().V1().PersistentVolumeClaims(),
        s.InformerFactory.Core().V1().ReplicationControllers(),
        s.InformerFactory.Extensions().V1beta1().ReplicaSets(),
        s.InformerFactory.Apps().V1beta1().StatefulSets(),
        s.InformerFactory.Core().V1().Services(),
        s.InformerFactory.Policy().V1beta1().PodDisruptionBudgets(),
        storageClassInformer,
        s.ComponentConfig.HardPodAffinitySymmetricWeight,
        utilfeature.DefaultFeatureGate.Enabled(features.EnableEquivalenceClassCache),
        s.ComponentConfig.DisablePreemption,
    )

    source := s.ComponentConfig.AlgorithmSource
    var config *scheduler.Config
    switch {
    case source.Provider != nil:
        // Create the config from a named algorithm provider.
        sc, err := configurator.CreateFromProvider(*source.Provider)
        if err != nil {
            return nil, fmt.Errorf("couldn't create scheduler using provider %q: %v", *source.Provider, err)
        }
        config = sc
    case source.Policy != nil:
        // Create the config from a user specified policy source.
        ...
    }
    ...
}
```

接着看下configFactory的具体描述

* pkg/scheduler/factory/factory.go:

```
// configFactory is the default implementation of the scheduler.Configurator interface.
type configFactory struct {
    client clientset.Interface
    // queue for pods that need scheduling
    podQueue core.SchedulingQueue
    // a means to list all known scheduled pods.
    scheduledPodLister corelisters.PodLister
    // a means to list all known scheduled pods and pods assumed to have been scheduled.
    podLister algorithm.PodLister
    // a means to list all nodes
    nodeLister corelisters.NodeLister
    // a means to list all PersistentVolumes
    pVLister corelisters.PersistentVolumeLister
    // a means to list all PersistentVolumeClaims
    pVCLister corelisters.PersistentVolumeClaimLister
    ...

    schedulerCache schedulercache.Cache

    // SchedulerName of a scheduler is used to select which pods will be
    // processed by this scheduler, based on pods's "spec.schedulerName".
    schedulerName string

    ...
}
```

configFactory构建过程如下，可以看到构建结构的同时，对各资源Informer添加了变更事件处理Callback

```
// NewConfigFactory initializes the default implementation of a Configurator To encourage eventual privatization of the struct type, we only
// return the interface.
func NewConfigFactory(
	schedulerName string,
	client clientset.Interface,
	nodeInformer coreinformers.NodeInformer,
	podInformer coreinformers.PodInformer,
	pvInformer coreinformers.PersistentVolumeInformer,
	pvcInformer coreinformers.PersistentVolumeClaimInformer,
	replicationControllerInformer coreinformers.ReplicationControllerInformer,
	replicaSetInformer extensionsinformers.ReplicaSetInformer,
	statefulSetInformer appsinformers.StatefulSetInformer,
	serviceInformer coreinformers.ServiceInformer,
	pdbInformer policyinformers.PodDisruptionBudgetInformer,
	storageClassInformer storageinformers.StorageClassInformer,
	hardPodAffinitySymmetricWeight int32,
	enableEquivalenceClassCache bool,
	disablePreemption bool,
) scheduler.Configurator {
	stopEverything := make(chan struct{})
	schedulerCache := schedulercache.New(30*time.Second, stopEverything)

	// storageClassInformer is only enabled through VolumeScheduling feature gate
	var storageClassLister storagelisters.StorageClassLister
	if storageClassInformer != nil {
		storageClassLister = storageClassInformer.Lister()
	}

	c := &configFactory{
		client:                         client,
		podLister:                      schedulerCache,
		podQueue:                       core.NewSchedulingQueue(),
		pVLister:                       pvInformer.Lister(),
		pVCLister:                      pvcInformer.Lister(),
		serviceLister:                  serviceInformer.Lister(),
		controllerLister:               replicationControllerInformer.Lister(),
		replicaSetLister:               replicaSetInformer.Lister(),
		statefulSetLister:              statefulSetInformer.Lister(),
		pdbLister:                      pdbInformer.Lister(),
		storageClassLister:             storageClassLister,
		schedulerCache:                 schedulerCache,
		StopEverything:                 stopEverything,
		schedulerName:                  schedulerName,
		hardPodAffinitySymmetricWeight: hardPodAffinitySymmetricWeight,
		enableEquivalenceClassCache:    enableEquivalenceClassCache,
		disablePreemption:              disablePreemption,
	}

	c.scheduledPodsHasSynced = podInformer.Informer().HasSynced
	// scheduled pod cache
	podInformer.Informer().AddEventHandler(
		cache.FilteringResourceEventHandler{
			FilterFunc: func(obj interface{}) bool {
				switch t := obj.(type) {
				case *v1.Pod:
					return assignedNonTerminatedPod(t)
				case cache.DeletedFinalStateUnknown:
					if pod, ok := t.Obj.(*v1.Pod); ok {
						return assignedNonTerminatedPod(pod)
					}
					runtime.HandleError(fmt.Errorf("unable to convert object %T to *v1.Pod in %T", obj, c))
					return false
				default:
					runtime.HandleError(fmt.Errorf("unable to handle object in %T: %T", c, obj))
					return false
				}
			},
			Handler: cache.ResourceEventHandlerFuncs{
				AddFunc:    c.addPodToCache,
				UpdateFunc: c.updatePodInCache,
				DeleteFunc: c.deletePodFromCache,
			},
		},
	)
	// unscheduled pod queue
	podInformer.Informer().AddEventHandler(
		cache.FilteringResourceEventHandler{
			FilterFunc: func(obj interface{}) bool {
				switch t := obj.(type) {
				case *v1.Pod:
					return unassignedNonTerminatedPod(t) && responsibleForPod(t, schedulerName)
				case cache.DeletedFinalStateUnknown:
					if pod, ok := t.Obj.(*v1.Pod); ok {
						return unassignedNonTerminatedPod(pod) && responsibleForPod(pod, schedulerName)
					}
					runtime.HandleError(fmt.Errorf("unable to convert object %T to *v1.Pod in %T", obj, c))
					return false
				default:
					runtime.HandleError(fmt.Errorf("unable to handle object in %T: %T", c, obj))
					return false
				}
			},
			Handler: cache.ResourceEventHandlerFuncs{
				AddFunc:    c.addPodToSchedulingQueue,
				UpdateFunc: c.updatePodInSchedulingQueue,
				DeleteFunc: c.deletePodFromSchedulingQueue,
			},
		},
	)
	// ScheduledPodLister is something we provide to plug-in functions that
	// they may need to call.
	c.scheduledPodLister = assignedPodLister{podInformer.Lister()}

	nodeInformer.Informer().AddEventHandler(
		cache.ResourceEventHandlerFuncs{
			AddFunc:    c.addNodeToCache,
			UpdateFunc: c.updateNodeInCache,
			DeleteFunc: c.deleteNodeFromCache,
		},
	)
	c.nodeLister = nodeInformer.Lister()

	pdbInformer.Informer().AddEventHandler(
		cache.ResourceEventHandlerFuncs{
			AddFunc:    c.addPDBToCache,
			UpdateFunc: c.updatePDBInCache,
			DeleteFunc: c.deletePDBFromCache,
		},
	)
	c.pdbLister = pdbInformer.Lister()

	// On add and delete of PVs, it will affect equivalence cache items
	// related to persistent volume
	pvInformer.Informer().AddEventHandler(
		cache.ResourceEventHandlerFuncs{
			// MaxPDVolumeCountPredicate: since it relies on the counts of PV.
			AddFunc:    c.onPvAdd,
			UpdateFunc: c.onPvUpdate,
			DeleteFunc: c.onPvDelete,
		},
	)
	c.pVLister = pvInformer.Lister()

	// This is for MaxPDVolumeCountPredicate: add/delete PVC will affect counts of PV when it is bound.
	pvcInformer.Informer().AddEventHandler(
		cache.ResourceEventHandlerFuncs{
			AddFunc:    c.onPvcAdd,
			UpdateFunc: c.onPvcUpdate,
			DeleteFunc: c.onPvcDelete,
		},
	)
	c.pVCLister = pvcInformer.Lister()

	// This is for ServiceAffinity: affected by the selector of the service is updated.
	// Also, if new service is added, equivalence cache will also become invalid since
	// existing pods may be "captured" by this service and change this predicate result.
	serviceInformer.Informer().AddEventHandler(
		cache.ResourceEventHandlerFuncs{
			AddFunc:    c.onServiceAdd,
			UpdateFunc: c.onServiceUpdate,
			DeleteFunc: c.onServiceDelete,
		},
	)
	c.serviceLister = serviceInformer.Lister()

	// Existing equivalence cache should not be affected by add/delete RC/Deployment etc,
	// it only make sense when pod is scheduled or deleted

	if utilfeature.DefaultFeatureGate.Enabled(features.VolumeScheduling) {
		// Setup volume binder
		c.volumeBinder = volumebinder.NewVolumeBinder(client, pvcInformer, pvInformer, storageClassInformer)

		storageClassInformer.Informer().AddEventHandler(
			cache.ResourceEventHandlerFuncs{
				AddFunc:    c.onStorageClassAdd,
				DeleteFunc: c.onStorageClassDelete,
			},
		)
	}

	// Setup cache comparer
	comparer := &cacheComparer{
		podLister:  podInformer.Lister(),
		nodeLister: nodeInformer.Lister(),
		pdbLister:  pdbInformer.Lister(),
		cache:      c.schedulerCache,
		podQueue:   c.podQueue,
	}

	ch := make(chan os.Signal, 1)
	signal.Notify(ch, compareSignal)

	go func() {
		for {
			select {
			case <-c.StopEverything:
				return
			case <-ch:
				comparer.Compare()
			}
		}
	}()

	return c
}
```



