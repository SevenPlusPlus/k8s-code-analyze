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

```
func Run(c schedulerserverconfig.CompletedConfig, stopCh <-chan struct{}) error {
    // Apply algorithms based on feature gates.
    algorithmprovider.ApplyFeatureGates()

    // Build a scheduler config from the provided algorithm source.
    schedulerConfig, err := NewSchedulerConfig(c)
    // Create the scheduler.
    sched := scheduler.NewFromConfig(schedulerConfig)
}

```
