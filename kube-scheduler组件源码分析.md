## kube-scheduler原理
kube-scheduler是k8s中的调度模块，负责调度Pod到具体的node节点上。 其工作原理是kube-scheduler需要对未被调度的Pod进行Watch，同时也需要对node进行watch，因为pod需要绑定到具体的Node上，当kube-scheduler监测到未被调度的pod，它会取出这个pod，然后依照内部设定的调度算法，选择合适的node，然后通过apiserver写回到etcd，至此该pod就绑定到该node上，后续kubelet会读取到该信息，然后在node上把pod给拉起来。
### 调度过程
![scheduler调度过程](/assets/kube-scheduler00.png)
kube-scheduler将PodSpec.NodeName字段为空的Pods逐个进行评分，经过预选(Predicates)和优选(Priorities)两个步骤，挑选最合适的Node作为该Pod的Destination。 
