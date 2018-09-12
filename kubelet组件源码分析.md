## kubelet组件源码分析
---

kubelet是Node节点上最重要的核心组件，负责Kubernetes集群具体的计算任务，管理Pod及Pod中的容器，每个kubelet进程会在ApiServer上注册自身节点信息，定期向master节点汇报节点的资源使用情况，并通过cAdvisor监控节点和容器的资源，具体功能包括：

* 监听Scheduler组件的任务分配
* 挂载POD所需Volume
* 下载POD所需Secrets
* 通过与docker daemon的交互运行docker容器
* 定期执行容器健康检查
* 监控、报告POD状态到kube-controller-manager组件
* 监控、报告Node状态到kube-controller-manager组件
### kubelet工作原理
如下 kubelet 内部组件结构图所示，Kubelet 由许多内部组件构成：

* Kubelet API，包括 10250 端口的认证 API、4194 端口的 cAdvisor API、10255 端口的只读 API 以及 10248 端口的健康检查 API
* syncLoop：从 API 或者 manifest 目录接收 Pod 更新，发送到 podWorkers 处理，大量使用 channel 处理来处理异步请求
* 辅助的 manager，如 cAdvisor、PLEG、Volume Manager 等，处理 syncLoop 以外的其他工作
* CRI：容器执行引擎接口，负责与 container runtime shim 通信
* 容器执行引擎，如 dockershim、rkt 等
* 网络插件，目前支持 CNI 和 kubenet

![kubelet内部组件结构图](/assets/kubelet.png)

### Pod启动流程
![pod启动流程](/assets/pod-start.png)

### 参考资料
[Kubernetes指南](https://kubernetes.feisky.xyz/zh/components/kubelet.html)







