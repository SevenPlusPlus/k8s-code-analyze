## kubelet工作原理
---

kubelet是Node节点上最重要的核心组件，负责Kubernetes集群具体的计算任务，具体功能包括：

* 监听Scheduler组件的任务分配
* 挂载POD所需Volume
* 下载POD所需Secrets
* 通过与docker daemon的交互运行docker容器
* 定期执行容器健康检查
* 监控、报告POD状态到kube-controller-manager组件
* 监控、报告Node状态到kube-controller-manager组件
