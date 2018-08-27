## kube-scheduler原理
kube-scheduler是k8s中的调度模块，负责调度Pod到具体的node节点上。 其工作原理是kube-scheduler需要对未被调度的Pod进行Watch，同时也需要对node进行watch，因为pod需要绑定到具体的Node上，当kube-scheduler监测到未被调度的pod，它会取出这个pod，然后依照内部设定的调度算法，选择合适的node，然后通过apiserver写回到etcd，至此该pod就绑定到该node上，后续kubelet会读取到该信息，然后在node上把pod给拉起来。
### 调度过程
![scheduler调度过程](/assets/kube-scheduler00.png)
kube-scheduler将PodSpec.NodeName字段为空的Pods逐个进行评分，经过预选(Predicates)和优选(Priorities)两个步骤，挑选最合适的Node作为该Pod的Destination。 

1. 预选
  * 根据配置的Predicates Policies（默认为DefaultProvider中定义的default predicates policies集合）过滤掉那些不满足这些Policies的的Nodes，
  * 剩下的Nodes就作为优选的输入。
2. 优选
  * 根据配置的Priorities Policies（默认为DefaultProvider中定义的default priorities policies集合）给预选后的Nodes进行打分排名，得分最高的Node即作为最适合的Node，该Pod就Bind到这个Node。
  * 如果经过优选将Nodes打分排名后，有多个Nodes并列得分最高，那么scheduler将随机从中选择一个Node作为目标Node。
