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
回顾前面SchedulerCommand执行过程,生成调度器配置后，根据调度器配置创建调度器
```
// NewFromConfig returns a new scheduler using the provided Config.
func NewFromConfig(config *Config) *Scheduler {
	metrics.Register()
	return &Scheduler{
		config: config,
	}
}
```
可以看出此处的调度器其实是对调度器配置的包装，真正的调度器genericScheduler前面已经创建了，接下来就是启动所有Informer，对ApiServer进行ListAndWatch直到同步完成，最后一步就是选主完成后在主节点上执行sched.Run开启Pod的调度过程。
### Scheduler启动调度过程scheduler.Run
```
// Run begins watching and scheduling. It waits for cache to be synced, then starts a goroutine and returns immediately.
func (sched *Scheduler) Run() {
	if !sched.config.WaitForCacheSync() {
		return
	}

	if utilfeature.DefaultFeatureGate.Enabled(features.VolumeScheduling) {
		go sched.config.VolumeBinder.Run(sched.bindVolumesWorker, sched.config.StopEverything)
	}

	go wait.Until(sched.scheduleOne, 0, sched.config.StopEverything)
}
```
可以看到其主要实现就是启动goroutine，循环反复执行Scheduler.scheduleOne方法，直到收到shut down scheduler的信号

Scheduler.scheduleOne开始真正的调度逻辑，每次负责一个Pod的调度，其主要流程如下:
* 取出一个待调度的Pod
* 执行调度算法为Pod选择出一个合适的Node
* sched.assume(assumedPod, suggestedHost),假设该Pod已经被成功调度，更新SchedulerCache中Pod的状态(AssumePod)，同时对本地缓存中node资源占用进行更新，以便下次调度时以此最新缓存作为参考
* sched.bind(assumedPod, &v1.Binding）异步将Pod的Binding信息（Pod和Node的绑定，从而让ApiServer知道Pod已经调度完成）写入ApiServer中，从而一次调度完成。
```
// scheduleOne does the entire scheduling workflow for a single pod.  It is serialized on the scheduling algorithm's host fitting.
func (sched *Scheduler) scheduleOne() {
        //取出一个待调度Pod
	pod := sched.config.NextPod()

	glog.V(3).Infof("Attempting to schedule pod: %v/%v", pod.Namespace, pod.Name)

	// Synchronously attempt to find a fit for the pod.
        //调度算法选出一个合适的Node
	suggestedHost, err := sched.schedule(pod)
	
	// Tell the cache to assume that a pod now is running on a given node, even though it hasn't been bound yet.
	// This allows us to keep scheduling without waiting on binding to occur.
	assumedPod := pod.DeepCopy()

	// Assume volumes first before assuming the pod.
	err = sched.assumeAndBindVolumes(assumedPod, suggestedHost)

	// assume modifies `assumedPod` by setting NodeName=suggestedHost
	err = sched.assume(assumedPod, suggestedHost)
	
 	// bind the pod to its host asynchronously (we can do this b/c of the assumption step above).
	go func() {
		err := sched.bind(assumedPod, &v1.Binding{
			ObjectMeta: metav1.ObjectMeta{Namespace: assumedPod.Namespace, Name: assumedPod.Name, UID: assumedPod.UID},
			Target: v1.ObjectReference{
				Kind: "Node",
				Name: suggestedHost,
			},
		})
	}()
}
```
这里取出待调度Pod的实现即为前面configFactory中的podQueue队列pop而来
```
func (c *configFactory) getNextPod() *v1.Pod {
	pod, err := c.podQueue.Pop()
	if err == nil {
		return pod
	}
	return nil
}
```
接下来应用调度算法为该Pod选出一个合适的Node
```
// schedule implements the scheduling algorithm and returns the suggested host.
func (sched *Scheduler) schedule(pod *v1.Pod) (string, error) {
	host, err := sched.config.Algorithm.Schedule(pod, sched.config.NodeLister)
	return host, err
}
```
回溯前面调度器配置生成过程，不难得知这里的sched.config即为scheduler.Config，而sched.config.Algorithm即为前面创建的genericScheduler，而其参数sched.Config.NodeLister即为configFactory通过nodeInformer得到的NodeLister，那么我们继续深入分析genericScheduler为Pod选合适Node的过程。
```
// Schedule tries to schedule the given pod to one of the nodes in the node list.
// If it succeeds, it will return the name of the node.
// If it fails, it will return a FitError error with reasons.
func (g *genericScheduler) Schedule(pod *v1.Pod, nodeLister algorithm.NodeLister) (string, error) {
	//首先检查Pod所需的PVC是否都已存在
	if err := podPassesBasicChecks(pod, g.pvcLister); 
	//从缓存中获得当前节点列表
	nodes, err := nodeLister.List()	
	if len(nodes) == 0 {
		return "", ErrNoNodesAvailable
	}

	// Used for all fit and priority funcs.

	//根据当前本地Cache中的节点信息更新传入的节点信息，此时节点信息包含所有已调度在该节点上的Pod聚合信息（包括已假设调度在其上的Pod）
	err = g.cache.UpdateNodeNameToInfoMap(g.cachedNodeInfoMap)

        //执行Predicate预选过程
	filteredNodes, failedPredicateMap, err := g.findNodesThatFit(pod, nodes)

	if len(filteredNodes) == 0 {
		return "", &FitError{
			Pod:              pod,
			NumAllNodes:      len(nodes),
			FailedPredicates: failedPredicateMap,
		}
	}
	

	trace.Step("Prioritizing")
	// When only one node after predicate, just use it.
	if len(filteredNodes) == 1 {
		metrics.SchedulingAlgorithmPriorityEvaluationDuration.Observe(metrics.SinceInMicroseconds(startPriorityEvalTime))
		return filteredNodes[0].Name, nil
	}
	//执行Priorities优选过程
	metaPrioritiesInterface := g.priorityMetaProducer(pod, g.cachedNodeInfoMap)
	priorityList, err := PrioritizeNodes(pod, g.cachedNodeInfoMap, metaPrioritiesInterface, g.prioritizers, filteredNodes, g.extenders)

	//从优选结果中选一个得分最高的节点返回，如果有多个得分最高的节点则随机选取一个
	trace.Step("Selecting host")
	return g.selectHost(priorityList)
}
```
##### Predicate预选过程
checkNode会调用podFitsOnNode完成配置的所有Predicates Policies对该Node的检查。
podFitsOnNode循环执行所有配置的Predicates Polic对应的predicateFunc。只有全部策略都通过，该node才符合要求。
```
// Filters the nodes to find the ones that fit based on the given predicate functions
// Each node is passed through the predicate functions to determine if it is a fit
func (g *genericScheduler) findNodesThatFit(pod *v1.Pod, nodes []*v1.Node) ([]*v1.Node, FailedPredicateMap, error) {
	var filtered []*v1.Node
	failedPredicateMap := FailedPredicateMap{}

	if len(g.predicates) == 0 {
		filtered = nodes
	} else {
		// Create filtered list with enough space to avoid growing it
		// and allow assigning.
		filtered = make([]*v1.Node, len(nodes))
		errs := errors.MessageCountMap{}
		var predicateResultLock sync.Mutex
		var filteredLen int32

		// We can use the same metadata producer for all nodes.
		meta := g.predicateMetaProducer(pod, g.cachedNodeInfoMap)

		checkNode := func(i int) {
			var nodeCache *equivalence.NodeCache
			nodeName := nodes[i].Name
			fits, failedPredicates, err := podFitsOnNode(
				pod,
				meta,
				g.cachedNodeInfoMap[nodeName],
				g.predicates,
				g.cache,
				nodeCache,
				g.schedulingQueue,
				g.alwaysCheckAllPredicates,
				equivClass,
			)
			if fits {
				filtered[atomic.AddInt32(&filteredLen, 1)-1] = nodes[i]
			} else {
				predicateResultLock.Lock()
				failedPredicateMap[nodeName] = failedPredicates
				predicateResultLock.Unlock()
			}
		}
		workqueue.Parallelize(16, len(nodes), checkNode)
		filtered = filtered[:filteredLen]
		if len(errs) > 0 {
			return []*v1.Node{}, FailedPredicateMap{}, errors.CreateAggregateFromMessageCountMap(errs)
		}
	}
	
	//如果有扩展实现的调度实现，则继续根据扩展策略的Filter实现对前面的预选结果继续过滤
	if len(filtered) > 0 && len(g.extenders) != 0 {
		for _, extender := range g.extenders {
			if !extender.IsInterested(pod) {
				continue
			}
			filteredList, failedMap, err := extender.Filter(pod, filtered, g.cachedNodeInfoMap)

			for failedNodeName, failedMsg := range failedMap {
				if _, found := failedPredicateMap[failedNodeName]; !found {
					failedPredicateMap[failedNodeName] = []algorithm.PredicateFailureReason{}
				}
				failedPredicateMap[failedNodeName] = append(failedPredicateMap[failedNodeName], predicates.NewFailureReason(failedMsg))
			}
			filtered = filteredList
			if len(filtered) == 0 {
				break
			}
		}
	}
	return filtered, failedPredicateMap, nil
}

//循环执行所有配置的Predicates Polic对应的predicateFunc
// podFitsOnNode checks whether a node given by NodeInfo satisfies the given predicate functions.
// For given pod, podFitsOnNode will check if any equivalent pod exists and try to reuse its cached
// predicate results as possible.
func podFitsOnNode(
	pod *v1.Pod,
	meta algorithm.PredicateMetadata,
	info *schedulercache.NodeInfo,
	predicateFuncs map[string]algorithm.FitPredicate,
	cache schedulercache.Cache,
	nodeCache *equivalence.NodeCache,
	queue SchedulingQueue,
	alwaysCheckAllPredicates bool,
	equivClass *equivalence.Class,
) (bool, []algorithm.PredicateFailureReason, error) {
	var (
		eCacheAvailable  bool
		failedPredicates []algorithm.PredicateFailureReason
	)

	podsAdded := false
	for i := 0; i < 2; i++ {
		metaToUse := meta
		nodeInfoToUse := info
		if i == 0 {
			podsAdded, metaToUse, nodeInfoToUse = addNominatedPods(util.GetPodPriority(pod), meta, info, queue)
		} else if !podsAdded || len(failedPredicates) != 0 {
			break
		}
		// Bypass eCache if node has any nominated pods.
		// TODO(bsalamat): consider using eCache and adding proper eCache invalidations
		// when pods are nominated or their nominations change.
		eCacheAvailable = equivClass != nil && nodeCache != nil && !podsAdded
		for _, predicateKey := range predicates.Ordering() {
			var (
				fit     bool
				reasons []algorithm.PredicateFailureReason
				err     error
			)
			//TODO (yastij) : compute average predicate restrictiveness to export it as Prometheus metric
			if predicate, exist := predicateFuncs[predicateKey]; exist {
				if eCacheAvailable {
					fit, reasons, err = nodeCache.RunPredicate(predicate, predicateKey, pod, metaToUse, nodeInfoToUse, equivClass, cache)
				} else {
					fit, reasons, err = predicate(pod, metaToUse, nodeInfoToUse)
				}
				if err != nil {
					return false, []algorithm.PredicateFailureReason{}, err
				}

				if !fit {
					// eCache is available and valid, and predicates result is unfit, record the fail reasons
					failedPredicates = append(failedPredicates, reasons...)
					// if alwaysCheckAllPredicates is false, short circuit all predicates when one predicate fails.
					if !alwaysCheckAllPredicates {
						glog.V(5).Infoln("since alwaysCheckAllPredicates has not been set, the predicate" +
							"evaluation is short circuited and there are chances" +
							"of other predicates failing as well.")
						break
					}
				}
			}
		}
	}
```
##### Priorities优选过程
先使用PriorityFunction计算节点的分数，在向priorityFunctionMap注册PriorityFunction时，会指定该PriorityFunction对应的weight，然后再累加每个PriorityFunction和weight相乘的积，这就样就得到了这个节点的分数。

```
// PrioritizeNodes prioritizes the nodes by running the individual priority functions in parallel.
// Each priority function is expected to set a score of 0-10
// 0 is the lowest priority score (least preferred node) and 10 is the highest
// Each priority function can also have its own weight
// The node scores returned by the priority function are multiplied by the weights to get weighted scores
// All scores are finally combined (added) to get the total weighted scores of all nodes
func PrioritizeNodes(
	pod *v1.Pod,
	nodeNameToInfo map[string]*schedulercache.NodeInfo,
	meta interface{},
	priorityConfigs []algorithm.PriorityConfig,
	nodes []*v1.Node,
	extenders []algorithm.SchedulerExtender,
) (schedulerapi.HostPriorityList, error) {
	// If no priority configs are provided, then the EqualPriority function is applied
	// This is required to generate the priority list in the required format
	if len(priorityConfigs) == 0 && len(extenders) == 0 {
		result := make(schedulerapi.HostPriorityList, 0, len(nodes))
		for i := range nodes {
			hostPriority, err := EqualPriorityMap(pod, meta, nodeNameToInfo[nodes[i].Name])
			if err != nil {
				return nil, err
			}
			result = append(result, hostPriority)
		}
		return result, nil
	}

	var (
		mu   = sync.Mutex{}
		wg   = sync.WaitGroup{}
		errs []error
	)
	appendError := func(err error) {
		mu.Lock()
		defer mu.Unlock()
		errs = append(errs, err)
	}

	results := make([]schedulerapi.HostPriorityList, len(priorityConfigs), len(priorityConfigs))

	/*
	根据所有配置到Priorities Policies对所有预选后的Nodes进行优选打分
	每个Priorities policy对每个node打分范围为0-10分，分越高表示越合适
	*/
	for i, priorityConfig := range priorityConfigs {
		if priorityConfig.Function != nil {
			// DEPRECATED
			wg.Add(1)
			go func(index int, config algorithm.PriorityConfig) {
				defer wg.Done()
				var err error
				results[index], err = config.Function(pod, nodeNameToInfo, nodes)
				if err != nil {
					appendError(err)
				}
			}(i, priorityConfig)
		} else {
			results[i] = make(schedulerapi.HostPriorityList, len(nodes))
		}
	}
	processNode := func(index int) {
		nodeInfo := nodeNameToInfo[nodes[index].Name]
		var err error
		for i := range priorityConfigs {
			if priorityConfigs[i].Function != nil {
				continue
			}
			results[i][index], err = priorityConfigs[i].Map(pod, meta, nodeInfo)
			if err != nil {
				appendError(err)
				return
			}
		}
	}
	workqueue.Parallelize(16, len(nodes), processNode)
	for i, priorityConfig := range priorityConfigs {
		if priorityConfig.Reduce == nil {
			continue
		}
		wg.Add(1)
		go func(index int, config algorithm.PriorityConfig) {
			defer wg.Done()
			if err := config.Reduce(pod, meta, nodeNameToInfo, results[index]); err != nil {
				appendError(err)
			}
			if glog.V(10) {
				for _, hostPriority := range results[index] {
					glog.Infof("%v -> %v: %v, Score: (%d)", pod.Name, hostPriority.Host, config.Name, hostPriority.Score)
				}
			}
		}(i, priorityConfig)
	}
	// Wait for all computations to be finished.
	wg.Wait()
	if len(errs) != 0 {
		return schedulerapi.HostPriorityList{}, errors.NewAggregate(errs)
	}

	// Summarize all scores.
	result := make(schedulerapi.HostPriorityList, 0, len(nodes))

	for i := range nodes {
		result = append(result, schedulerapi.HostPriority{Host: nodes[i].Name, Score: 0})
		for j := range priorityConfigs {
			result[i].Score += results[j][i].Score * priorityConfigs[j].Weight
		}
	}

	//如果配置了Extender，则再执行Extender的优选打分方法Extender.Prioritize
	if len(extenders) != 0 && nodes != nil {
		combinedScores := make(map[string]int, len(nodeNameToInfo))
		for _, extender := range extenders {
			if !extender.IsInterested(pod) {
				continue
			}
			wg.Add(1)
			go func(ext algorithm.SchedulerExtender) {
				defer wg.Done()
				prioritizedList, weight, err := ext.Prioritize(pod, nodes)
				if err != nil {
					// Prioritization errors from extender can be ignored, let k8s/other extenders determine the priorities
					return
				}
				mu.Lock()
				for i := range *prioritizedList {
					host, score := (*prioritizedList)[i].Host, (*prioritizedList)[i].Score
					combinedScores[host] += score * weight
				}
				mu.Unlock()
			}(extender)
		}
		// wait for all go routines to finish
		wg.Wait()
		for i := range result {
			result[i].Score += combinedScores[result[i].Host]
		}
	}

	if glog.V(10) {
		for i := range result {
			glog.V(10).Infof("Host %s => Score %d", result[i].Host, result[i].Score)
		}
	}
	return result, nil
}
```
##### selectHost选出最终结果节点

```
// selectHost takes a prioritized list of nodes and then picks one
// in a round-robin manner from the nodes that had the highest score.
func (g *genericScheduler) selectHost(priorityList schedulerapi.HostPriorityList) (string, error) {
	if len(priorityList) == 0 {
		return "", fmt.Errorf("empty priorityList")
	}

	maxScores := findMaxScores(priorityList)
	ix := int(g.lastNodeIndex % uint64(len(maxScores)))
	g.lastNodeIndex++

	return priorityList[maxScores[ix]].Host, nil
}
```

### 总结
---
* kube-scheduler通过连接ApiServer的client用来watch pod、node等一系列调度所需的资源并且调用api server bind接口完成node和pod的Bind操作。
* kube-scheduler中维护了一个优先级队列类型的PodQueue cache，新创建的Pod都会被ConfigFactory watch到，被添加到该PodQueue中，每次调度都从该PodQueue中getNextPod作为即将调度的Pod。
* 获取到待调度的Pod后，就执行AlgorithmProvider配置Algorithm的Schedule方法进行调度，整个调度过程分两个关键步骤：Predicates和Priorities，最终选出一个最适合的Node返回。
* 更新SchedulerCache中Pod的状态(AssumePod)，标志该Pod为scheduled。
* 向apiserver发送&api.Binding对象，表示绑定成功。

