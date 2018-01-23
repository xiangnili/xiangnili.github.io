## 一、什么是Kubernetes？
kubernetes是google开源的容器集群管理系统，提供应用部署、维护、扩展机制等功能，利用kubernetes能方便管理跨集群运行容器化的应用，简称：k8s（k与s之间有8个字母）。
## 二、使用K8s可以做什么？
- 自动化容器的部署和复制；
- 随时扩展或收缩容器规模；
- 将容器组织成组，并且提供容器间的负载均衡；
- 很容易地升级应用程序容器的新版本；
- 提供容器弹性，如果容器失效就替换它，等等...
## 三、K8s的使用到底有多简单？
实际上，使用Kubernetes只需一个部署文件，使用一条命令就可以部署多层容器(前端、后台等)的完整集群：
```
$ kubectl create -f single-config-file.yaml
```
## 四、关于K8s的一些基本概念
#### 1. Pod
若干相关容器的组合，Pod包含的容器运行在同一host上，这些容器使用相同的网络命令空间、IP地址和端口，相互之间能通过localhost来发现和通信。另外，这些容器还可共享一块存储卷空间。在k8s中创建，调度和管理的最小单位就是Pod，而非容器，Pod通过提供更高层次的抽象，提供了更加灵活的部署和管理模式；
- 若干相关容器的组合，Pod包含的容器运行在同一host上，这些容器使用相同的网络命令空间、IP地址和端口，相互之间能通过localhost来发现和通信。另外，这些容器还可共享一块存储卷空间。在k8s中创建，调度和管理的最小单位就是Pod，而非容器，Pod通过提供更高层次的抽象，提供了更加灵活的部署和管理模式；
- 同一pod包含的容器运行在同一host上，作为统一管理单元：同一pod 共享着相同的volumes， network命名空间， ip和port空间，这是通过Mapped Container做到的。
 + pid ns：处于同一pod中的应用可以看到彼此的进程；
 + network ns：处于同一pod中的应用可以访问一样的ip和port空间；
 + ipc ns：处于同一pod的应用可以用systemV ipc 或者posix消息队列进行通信；
 + UTC ns：处于同一pod应用共用一个主机名。
 
#### 2. ReplicationController （RC）
- RC是用来管理Pod的，每个RC由一个或多个Pod组成；在RC被创建之后，系统将会保持RC中的可用Pod的个数与创建RC时定义的Pod个数一致，如果Pod个数小于定义的个数，RC会启动新的Pod，反之则会杀死多余的Pod；
- RC通过定义的Pod模板被创建，创建后对象叫做Pods（也可以理解为RC），可以在线修改Pods的属性，以实现动态缩减、扩展Pods的规模；
- RC通过label关联对应的Pods，通过修改Pods的label可以删除对应的Pods在需要对Pods中的容器进行更新时，RC采用一个一个替换原则来更新整个Pods中的Pod；
- reschudeling: 维护pod副本，“多退少补”；即使是某些minion宕机；
- scaling：通过修改rc的副本数来水平扩展或者缩小运行的pods；
- Rolling updates:一个一个地替换pods来rolling updates服务； 
- multiple release tracks：如果需要在系统中运行multiple release 服务，replication controller使用labels来区分multiple release tracks；
#### 3. Label
- Label是用于区分Pod、Service、RC的key/value键值对；
- Pod、Service、RC可以有多个label，但是每个label的key只能对应一个value；
- 整个系统都是通过Label进行关联，得到真正需要操作的目标； 
#### 4. Service
- Service也是k8s的最小操作单元，是真实应用服务的抽象；
- Service通常用来将浮动的资源与后端真实提供服务的容器进行关联；
- Service对外表现为一个单一的访问接口，外部不需要了解后端的规模与机制； 
Service是定义在集群中一组运行Pod集合的抽象资源，它提供了所有相同的功能。当一个Service资源被创建后，将会分配一个唯一的IP（也叫做集群IP），这个IP地址将存在于Service的整个生命资源，Service一旦被创建，整个IP无法进行修改。
Pod可以通过Service进行通信，并且所有的通信将会通过Service自动负载均很到所有的Pod中的容器。
## 五、集群以及K8s架构
集群是一组节点，这些节点可以是物理服务器或者虚拟机。K8s部署在上面，用来管理这些集群。下图是一个典型的K8s架构图：
![](Screen Shot 2017-12-11 at 19.28.21.png)
[图1 Kubernetes架构图]
流程：通过Kubectl提交一个创建RC的请求，该请求通过API Server被写入etcd中，此时Controller Manager通过API Server的监听资源变化的接口监听到这个RC事件，分析之后，发现当前集群中还没有它所对应的Pod实例，于是根据RC里的Pod模板定义生成一个Pod对象，通过API Server写入etcd，接下来，此事件被Scheduler发现，它立即执行一个复杂的调度流程，为这个新Pod选定一个落户的Node，然后通过API Server讲这一结果写入到etcd中，随后，目标Node上运行的Kubelet进程通过API Server监测到这个“新生的”Pod，并按照它的定义，启动该Pod并任劳任怨地负责它的下半生，直到Pod的生命结束。
随后，我们通过Kubectl提交一个新的映射到该Pod的Service的创建请求，Controller Manager会通过Label标签查询到相关联的Pod实例，然后生成Service的Endpoints信息，并通过API Server写入到etcd中，接下来，所有Node上运行的Proxy进程通过API Server查询并监听Service对象与其对应的Endpoints信息，建立一个软件方式的负载均衡器来实现Service访问到后端Pod的流量转发功能。
- etcd ：用于持久化存储集群中所有的资源对象，如Node、Service、Pod、RC、Namespace等；API Server提供了操作etcd的封装接口API，这些API基本上都是集群中资源对象的增删改查及监听资源变化的接口。
- API Server ：提供了资源对象的唯一操作入口，其他所有组件都必须通过它提供的API来操作资源数据，通过对相关的资源数据“全量查询”+“变化监听”，这些组件可以很“实时”地完成相关的业务功能。
- Controller Manager ：集群内部的管理控制中心，其主要目的是实现Kubernetes集群的故障检测和恢复的自动化工作，比如根据RC的定义完成Pod的复制或移除，以确保Pod实例数符合RC副本的定义；根据Service与Pod的管理关系，完成服务的Endpoints对象的创建和更新；其他诸如Node的发现、管理和状态监控、死亡容器所占磁盘空间及本地缓存的镜像文件的清理等工作也是由Controller Manager完成的。
- Scheduler ：集群中的调度器，负责Pod在集群节点中的调度分配。
- Kubelet ：负责本Node节点上的Pod的创建、修改、监控、删除等全生命周期管理，同时Kubelet定时“上报”本Node的状态信息到API Server里。
- Proxy ：实现了Service的代理与软件模式的负载均衡器。
客户端通过Kubectl命令行工具或Kubectl Proxy来访问Kubernetes系统，在Kubernetes集群内部的客户端可以直接使用Kuberctl命令管理集群。Kubectl Proxy是API Server的一个反向代理，在Kubernetes集群外部的客户端可以通过Kubernetes Proxy来访问API Server。
API Server内部有一套完备的安全机制，包括认证、授权和准入控制等相关模块。