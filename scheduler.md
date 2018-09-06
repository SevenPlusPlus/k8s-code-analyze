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



