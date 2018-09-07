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
    // 待调度pod队列，queue for pods that need scheduling
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

configFactory构建过程如下，可以看到构建结构的同时，对各资源Informer添加了特定变更事件处理Callback，通过对特定资源变更事件的处理来触发调度过程或者更新内部调度数据辅助调度算法的决策。

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
    //已调度pod队列缓存的处理 scheduled pod cache
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
    //待调度pod队列缓存处理 unscheduled pod queue
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

我们以podInformer对unscheduled pod queue的操作维护过程为例来解析，其首先通过FilterFunc来过滤pod.Spec.NodeName为空的pod对象，然后根据变更事件调用相应Callback对待调度pod队列进行增删改。

```
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
```

尚未调度并未结束pod过滤方法实现为

```
// unassignedNonTerminatedPod selects pods that are unassigned and non-terminal.
func unassignedNonTerminatedPod(pod *v1.Pod) bool {
    if len(pod.Spec.NodeName) != 0 {
        return false
    }
    if pod.Status.Phase == v1.PodSucceeded || pod.Status.Phase == v1.PodFailed {
        return false
    }
    return true
}
```

添加未调度pod到待调度podQueue中

```
func (c *configFactory) addPodToSchedulingQueue(obj interface{}) {
    if err := c.podQueue.Add(obj.(*v1.Pod)); err != nil {
        runtime.HandleError(fmt.Errorf("unable to queue %T: %v", obj, err))
    }
}
```

跟踪分析各个Informer的回调处理句柄不难发现他们主要操作维护了configFactory的两个重要的数据字段，待调度pod队列c.podQueue和调度器信息缓存c.schedulerCache, 而其类型分别为core.SchedulingQueue和schedulercache.Cache，下面对这两个字段类型重点分析下。

##### core.SchedulingQueue

* pkg/scheduler/core/scheduling\_queue.go:

```
// SchedulingQueue is an interface for a queue to store pods waiting to be scheduled.
// The interface follows a pattern similar to cache.FIFO and cache.Heap and
// makes it easy to use those data structures as a SchedulingQueue.
type SchedulingQueue interface {
    Add(pod *v1.Pod) error
    AddIfNotPresent(pod *v1.Pod) error
    AddUnschedulableIfNotPresent(pod *v1.Pod) error
    Pop() (*v1.Pod, error)
    Update(oldPod, newPod *v1.Pod) error
    Delete(pod *v1.Pod) error
    MoveAllToActiveQueue()
    AssignedPodAdded(pod *v1.Pod)
    AssignedPodUpdated(pod *v1.Pod)
    WaitingPodsForNode(nodeName string) []*v1.Pod
    WaitingPods() []*v1.Pod
}
```

SchedulingQueue有两种实现，若开启pod优先级调度则采用PriorityQueue实现否则采用简单的FIFO实现，比较分析两种实现可以看到接口中一些操作只对优先级队列有效。我们看下优先级队列的实现：

```
// PriorityQueue implements a scheduling queue. It is an alternative to FIFO.
// The head of PriorityQueue is the highest priority pending pod. This structure
// has two sub queues. One sub-queue holds pods that are being considered for
// scheduling. This is called activeQ and is a Heap. Another queue holds
// pods that are already tried and are determined to be unschedulable. The latter
// is called unschedulableQ.
// 其内部维护了两个队列，一个为等待调度的队列activeQ，另一个为尝试过确定为不可调度的队列unschedulableQ
type PriorityQueue struct {
    lock sync.RWMutex
    cond sync.Cond

    // activeQ is heap structure that scheduler actively looks at to find pods to
    // schedule. Head of heap is the highest priority pod.
    activeQ *Heap
    // unschedulableQ holds pods that have been tried and determined unschedulable.
    unschedulableQ *UnschedulablePodsMap
    // nominatedPods is a map keyed by a node name and the value is a list of
    // pods which are nominated to run on the node. These are pods which can be in
    // the activeQ or unschedulableQ.
    nominatedPods map[string][]*v1.Pod
    // receivedMoveRequest is set to true whenever we receive a request to move a
    // pod from the unschedulableQ to the activeQ, and is set to false, when we pop
    // a pod from the activeQ. It indicates if we received a move request when a
    // pod was in flight (we were trying to schedule it). In such a case, we put
    // the pod back into the activeQ if it is determined unschedulable.
    receivedMoveRequest bool
}
```

##### schedulercache.Cache

* pkg/scheduler/cache/interface.go:

Cache收集pod的信息并且提供节点级的聚合信息，主要用于调度器高效的查询

```
// Cache collects pods' information and provides node-level aggregated information.
// It's intended for generic scheduler to do efficient lookup.
// Cache's operations are pod centric. It does incremental updates based on pod events.
// Pod events are sent via network. We don't have guaranteed delivery of all events:
// We use Reflector to list and watch from remote.
// Reflector might be slow and do a relist, which would lead to missing events.
//
// State Machine of a pod's events in scheduler's cache:
//
//
//   +-------------------------------------------+  +----+
//   |                            Add            |  |    |
//   |                                           |  |    | Update
//   +      Assume                Add            v  v    |
//Initial +--------> Assumed +------------+---> Added <--+
//   ^                +   +               |       +
//   |                |   |               |       |
//   |                |   |           Add |       | Remove
//   |                |   |               |       |
//   |                |   |               +       |
//   +----------------+   +-----------> Expired   +----> Deleted
//         Forget             Expire
//
//
// Note that an assumed pod can expire, because if we haven't received Add event notifying us
// for a while, there might be some problems and we shouldn't keep the pod in cache anymore.
//
// Note that "Initial", "Expired", and "Deleted" pods do not actually exist in cache.
// Based on existing use cases, we are making the following assumptions:
// - No pod would be assumed twice
// - A pod could be added without going through scheduler. In this case, we will see Add but not Assume event.
// - If a pod wasn't added, it wouldn't be removed or updated.
// - Both "Expired" and "Deleted" are valid end states. In case of some problems, e.g. network issue,
//   a pod might have changed its state (e.g. added and deleted) without delivering notification to the cache.
type Cache interface {
    // AssumePod assumes a pod scheduled and aggregates the pod's information into its node.
    // The implementation also decides the policy to expire pod before being confirmed (receiving Add event).
    // After expiration, its information would be subtracted.
    AssumePod(pod *v1.Pod) error

    // FinishBinding signals that cache for assumed pod can be expired
    FinishBinding(pod *v1.Pod) error

    // ForgetPod removes an assumed pod from cache.
    ForgetPod(pod *v1.Pod) error

    // AddPod either confirms a pod if it's assumed, or adds it back if it's expired.
    // If added back, the pod's information would be added again.
    AddPod(pod *v1.Pod) error

    // UpdatePod removes oldPod's information and adds newPod's information.
    UpdatePod(oldPod, newPod *v1.Pod) error

    // RemovePod removes a pod. The pod's information would be subtracted from assigned node.
    RemovePod(pod *v1.Pod) error

    // GetPod returns the pod from the cache with the same namespace and the
    // same name of the specified pod.
    GetPod(pod *v1.Pod) (*v1.Pod, error)

    // IsAssumedPod returns true if the pod is assumed and not expired.
    IsAssumedPod(pod *v1.Pod) (bool, error)

    // AddNode adds overall information about node.
    AddNode(node *v1.Node) error

    // UpdateNode updates overall information about node.
    UpdateNode(oldNode, newNode *v1.Node) error

    // RemoveNode removes overall information about node.
    RemoveNode(node *v1.Node) error

    // AddPDB adds a PodDisruptionBudget object to the cache.
    AddPDB(pdb *policy.PodDisruptionBudget) error

    // UpdatePDB updates a PodDisruptionBudget object in the cache.
    UpdatePDB(oldPDB, newPDB *policy.PodDisruptionBudget) error

    // RemovePDB removes a PodDisruptionBudget object from the cache.
    RemovePDB(pdb *policy.PodDisruptionBudget) error

    // List lists all cached PDBs matching the selector.
    ListPDBs(selector labels.Selector) ([]*policy.PodDisruptionBudget, error)

    // UpdateNodeNameToInfoMap updates the passed infoMap to the current contents of Cache.
    // The node info contains aggregated information of pods scheduled (including assumed to be)
    // on this node.
    UpdateNodeNameToInfoMap(infoMap map[string]*NodeInfo) error

    // List lists all cached pods (including assumed ones).
    List(labels.Selector) ([]*v1.Pod, error)

    // FilteredList returns all cached pods that pass the filter.
    FilteredList(filter PodFilter, selector labels.Selector) ([]*v1.Pod, error)

    // Snapshot takes a snapshot on current cache
    Snapshot() *Snapshot

    // IsUpToDate returns true if the given NodeInfo matches the current data in the cache.
    IsUpToDate(n *NodeInfo) bool
}

// Snapshot is a snapshot of cache state
type Snapshot struct {
    AssumedPods map[string]bool
    Nodes       map[string]*NodeInfo
    Pdbs        map[string]*policy.PodDisruptionBudget
}
```

##### schedulerCache为Cache接口的默认实现

* pkg/scheduler/cache/cache.go:

从schedulerCache的描述中就可以看到其内部含pod状态信息、节点级的调度信息，这些信息会在不同的调度算法中查询使用。

```
type schedulerCache struct {
    stop   <-chan struct{}
    ttl    time.Duration
    period time.Duration

    // This mutex guards all fields within this cache struct.
    mu sync.RWMutex
    // a set of assumed pod keys.
    // The key could further be used to get an entry in podStates.
    assumedPods map[string]bool
    // a map from pod key to podState.
    podStates map[string]*podState
    nodes     map[string]*NodeInfo
    pdbs      map[string]*policy.PodDisruptionBudget
    // A map from image name to its imageState.
    imageStates map[string]*imageState
}

type podState struct {
    pod *v1.Pod
    // Used by assumedPod to determinate expiration.
    deadline *time.Time
    // Used to block cache from expiring assumedPod if binding still runs
    bindingFinished bool
}

type imageState struct {
    // Size of the image
    size int64
    // A set of node names for nodes having this image present
    nodes sets.String
}

// NodeInfo is node level aggregated information.
type NodeInfo struct {
    // Overall node information.
    node *v1.Node

    pods             []*v1.Pod
    podsWithAffinity []*v1.Pod
    usedPorts        util.HostPortInfo

    // Total requested resource of all pods on this node.
    // It includes assumed pods which scheduler sends binding to apiserver but
    // didn't get it as scheduled yet.
    requestedResource *Resource
    nonzeroRequest    *Resource
    // We store allocatedResources (which is Node.Status.Allocatable.*) explicitly
    // as int64, to avoid conversions and accessing map.
    allocatableResource *Resource

    // Cached taints of the node for faster lookup.
    taints    []v1.Taint
    taintsErr error

    // imageStates holds the entry of an image if and only if this image is on the node. The entry can be used for
    // checking an image's existence and advanced usage (e.g., image locality scheduling policy) based on the image
    // state information.
    imageStates map[string]*ImageStateSummary

    // TransientInfo holds the information pertaining to a scheduling cycle. This will be destructed at the end of
    // scheduling cycle.
    // TODO: @ravig. Remove this once we have a clear approach for message passing across predicates and priorities.
    TransientInfo *transientSchedulerInfo

    // Cached conditions of node for faster lookup.
    memoryPressureCondition v1.ConditionStatus
    diskPressureCondition   v1.ConditionStatus
    pidPressureCondition    v1.ConditionStatus

    // Whenever NodeInfo changes, generation is bumped.
    // This is used to avoid cloning it if the object didn't change.
    generation int64
}
```

回过头继续分析生成调度器配置的第二步应用调度器指定的调度算法配置创建调度配置

##### CreateFromProvider

```
//根据调度名获取调度算法配置
// Creates a scheduler from the name of a registered algorithm provider.
func (c *configFactory) CreateFromProvider(providerName string) (*scheduler.Config, error) {
    glog.V(2).Infof("Creating scheduler from algorithm provider '%v'", providerName)
    /*
    // AlgorithmProviderConfig is used to store the configuration of algorithm providers.
    type AlgorithmProviderConfig struct {
        FitPredicateKeys     sets.String
        PriorityFunctionKeys sets.String
    }
    */
    provider, err := GetAlgorithmProvider(providerName)

    return c.CreateFromKeys(provider.FitPredicateKeys, provider.PriorityFunctionKeys, []algorithm.SchedulerExtender{})
}

// Creates a scheduler from a set of registered fit predicate keys and priority keys.
func (c *configFactory) CreateFromKeys(predicateKeys, priorityKeys sets.String, extenders []algorithm.SchedulerExtender) (*scheduler.Config, error) {
    glog.V(2).Infof("Creating scheduler with fit predicates '%v' and priority functions '%v'", predicateKeys, priorityKeys)

    predicateFuncs, err := c.GetPredicates(predicateKeys)
    priorityConfigs, err := c.GetPriorityFunctionConfigs(priorityKeys)

    priorityMetaProducer, err := c.GetPriorityMetadataProducer()
    predicateMetaProducer, err := c.GetPredicateMetadataProducer()
    //创建真正的调度器GenericScheduler
    algo := core.NewGenericScheduler(
        c.schedulerCache,
        c.equivalencePodCache,
        c.podQueue,
        predicateFuncs,
        predicateMetaProducer,
        priorityConfigs,
        priorityMetaProducer,
        extenders,
        c.volumeBinder,
        c.pVCLister,
        c.alwaysCheckAllPredicates,
        c.disablePreemption,
    )

    podBackoff := util.CreateDefaultPodBackoff()
    return &scheduler.Config{
        SchedulerCache: c.schedulerCache,
        Ecache:         c.equivalencePodCache,
        // The scheduler only needs to consider schedulable nodes.
        NodeLister:          &nodeLister{c.nodeLister},
        Algorithm:           algo,
        GetBinder:           c.getBinderFunc(extenders),
        PodConditionUpdater: &podConditionUpdater{c.client},
        PodPreemptor:        &podPreemptor{c.client},
        WaitForCacheSync: func() bool {
            return cache.WaitForCacheSync(c.StopEverything, c.scheduledPodsHasSynced)
        },
        NextPod: func() *v1.Pod {
            return c.getNextPod()
        },
        Error:          c.MakeDefaultErrorFunc(podBackoff, c.podQueue),
        StopEverything: c.StopEverything,
        VolumeBinder:   c.volumeBinder,
    }, nil
}
```
回顾前面SchedulerCommand执行过程

