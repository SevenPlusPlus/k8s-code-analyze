# Apiserver综述

## 综述

- API Server作为整个Kubernetes集群的核心组件，让所有资源可被描述和配置；这里的资源包括了类似网络、存储、Pod这样的基础资源也包括了replication controller、deployment这样的管理对象。

- API Server某种程度上来说更像是包含了一定逻辑的对象数据库；接口上更加丰富、自带GC、支持对象间的复杂逻辑；当然API Server本身是无状态的,数据都是存储在etcd当中。

- API Server提供集群管理的REST API接口，支持增删改查和patch、监听的操作，其他组件通过和API Server的接口获取资源配置和状态，以实现各种资源处理逻辑。

- 只有API Server才直接操作etcd

## 架构图
![](/assets/apiserver-00.jpeg)

- Scheme：定义了资源序列化和反序列化的方法以及资源类型和版本的对应关系；这里我们可以理解成一张映射表。

- Storage：是对资源的完整封装，实现了资源创建、删除、watch等所有操作。

- APIGroupInfo：是在同一个Group下的所有资源的集合。

## 资源版本

一个资源对应着两个版本: 一个版本是用户访问的接口对象（yaml或者json通过接口传递的格式），称之为external version;

另一个版本则是核心对象，实现了资源创建和删除等，并且直接参与持久化，对应了在etcd中存储，称之为internal version。

这两个版本的资源是需要相互转化的，并且转换方法需要事先注册在Scheme中。

版本化的API旨在保持稳定，而internal version是为了最好地反映Kubernetes代码本身的需要。

这也就是说如果要自己新定义一个资源，需要声明两个版本！

同时应该还要在Scheme中注册版本转换函数。

etcd中存储的是带版本的，这也是Apiserver实现多版本转换的核心。

多个external version版本之间的资源进行相互转换，都是需要通过internal version进行中转。

一个对象在internal version和external version中的定义可以一样，也可以不一样。


## 一个Restful请求需要经过的流程

Authentication-->Authorization-->Admission Control

![一个请求需要经过的流程](/assets/access-control-overview.jpg)
