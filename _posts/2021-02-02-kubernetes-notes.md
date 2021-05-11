---
title:  Kubernetes (K8S) 笔记
classes: wide
categories:
  - 2021-02
tags:
  - kubernetes
type: note
---

K8S 是分布式应用的编排器（orchestrator），无论是部署应用到云上还是内部（on-premise）机房，都需要打包应用，分发软件包，保证应用高可用，负载均衡流量等一系列复杂环节。K8S 提供了一层抽象，屏蔽了这些复杂且容易出错的环节，让开发者能专心开发功能，只需要改动几个参数，就能让应用运行在几万台机器上。

[这个视频](https://www.youtube.com/watch?v=q1PcAawa4Bg&list=PLLasX02E8BPCrIhFrc_ZiINhbRkYMKdPT&index=1)是 Brendan Burns 对 K8S 的介绍。

未来的开发者们不需要关心应用是跑在云上还是内部（on-premise）环境，只需要适配 K8S，最后应用就可以在各种云环境或内部环境下运行，从而有效得避免了 Vendor lock-in。从这点看，K8S 有点类似云时代的 Linux，帮开发者屏蔽了底层的 CPU 架构。

K8S 系统比较复杂，涉及到的组件很多，在这里记录下学习和使用过程中的一些心得，供以后参考。


## Components & Terms

### API-Server

**职责：资源 CRUD**


### Scheduler

**职责：资源调度（resource scheduling）**

### Controller

**职责：协调资源（reconcile resource)**

类似自动化里的控制回路（control loop），调节系统的状态。

一个controller 跟踪至少一类资源的状态，保证当前状态和期望状态一致。

[https://kubernetes.io/docs/concepts/architecture/controller/](https://kubernetes.io/docs/concepts/architecture/controller/)

---

使用水平触发（level trigger）方式使状态达到期望状态。

分布式计算存在固有的问题（网络临时不可用等），使用 level trigger 比 edge trigger 更合适，最后出来的架构更简单，系统能够自愈（self-healing）。

[https://hackernoon.com/level-triggering-and-reconciliation-in-kubernetes-1f17fe30333d](https://hackernoon.com/level-triggering-and-reconciliation-in-kubernetes-1f17fe30333d)

---

Each component concentrates on its responsibilities. = There is no Orchestra conductor who controls the whole and gives instructions

Kubernetes is not an Orchestration. As jazz improv, by players(Controllers) concentrating on each plays(Control Loop), the whole is consisted of.

K8S 没有中心化的指挥调度，依靠各组件各自的控制器形成整体。

Kubernetes = Edge-driven Trigger + Level-driven Trigger

[Inside of Kubernetes Controller.pdf](/assets/slides/Inside_of_Kubernetes_Controller.pdf)

### Custom Resource

K8S 内建 API 对象的扩展，能动态注册到集群里，系统更可扩展和模块化。

两种方式创建自定义资源:

- CRDs are simple and can be created without any programming.

- API Aggregation requires programming, but allows more control over API behaviors like how data is stored and conversion between API versions


The CustomResourceDefinition API resource allows you to define custom resources. Defining a CRD object creates a new custom resource with a name and schema that you specify. The Kubernetes API serves and handles the storage of your custom resource. The name of a CRD object must be a valid DNS subdomain name.


CRD 使用起来简单，但灵活性不如 API Aggregation.

### Operator Pattern

某些需要人操作的事情（比如，创建deployment，创建PV/PVC等），可以通过 operator 来自动化完成，提高运维效率，减少出错几率。


> Kubernetes' controllers concept lets you extend the cluster's behaviour without modifying the code of Kubernetes itself. Operators are clients of the Kubernetes API that act as controllers for a Custom Resource.

[https://kubernetes.io/docs/concepts/extend-kubernetes/operator/](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)

Operator 作为 K8S API 的客户端，充当 Custom Resource 的控制器的角色。

---

## Architecture

![k8s high-level arch](/assets/images/2021/02/k8s_overview.jpeg)

[https://sysadvent.blogspot.com/2017/12/day-24-on-premise-kubernetes-with.html](https://sysadvent.blogspot.com/2017/12/day-24-on-premise-kubernetes-with.html)


## 资源列表

- [https://speakerdeck.com/govargo/inside-of-kubernetes-controller?slide=16](https://speakerdeck.com/govargo/inside-of-kubernetes-controller?slide=16)

- [https://azure.microsoft.com/mediahandler/files/resourcefiles/kubernetes-learning-path/Kubernetes%20Learning%20Path_Version%202.0.pdf](https://azure.microsoft.com/mediahandler/files/resourcefiles/kubernetes-learning-path/Kubernetes%20Learning%20Path_Version%202.0.pdf)

- [https://www.redhat.com/cms/managed-files/cm-oreilly-kubernetes-patterns-ebook-f19824-201910-en.pdf](https://www.redhat.com/cms/managed-files/cm-oreilly-kubernetes-patterns-ebook-f19824-201910-en.pdf)

- [https://docs.google.com/spreadsheets/d/10NltoF_6y3mBwUzQ4bcQLQfCE1BWSgUDcJXy-Qp2JEU/htmlview](https://docs.google.com/spreadsheets/d/10NltoF_6y3mBwUzQ4bcQLQfCE1BWSgUDcJXy-Qp2JEU/htmlview)

- [https://github.com/caicloud/kube-ladder](https://github.com/caicloud/kube-ladder)

