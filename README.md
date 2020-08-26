## 1. 什么是云原生

### 1.1 CNCF组织

在讲云原生之前，我们先了解一下CNCF，即云原生计算基金会，2015年由谷歌牵头成立，基金会成员目前已有一百多企业与机构，包括亚马逊、微软。思科等巨头。

![](//upload-images.jianshu.io/upload_images/7891228-53fadc92a3bf7635.png?imageMogr2/auto-orient/strip|imageView2/2/w/159/format/webp)

cncf

目前CNCF所托管的应用已达14个，下图为其公布的Cloud Native Landscape，给出了云原生生态的参考体系。

  


![](//upload-images.jianshu.io/upload_images/7891228-b180408dca8e144e.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

Cloud Native Landscape

### 1.2 云原生

CNCF给出了云原生应用的三大特征：

* 容器化封装：以容器为基础，提高整体开发水平，形成代码和组件重用，简化云原生应用程序的维护。在容器中运行应用程序和进程，并作为应用程序部署的独立单元，实现高水平资源隔离。
* 动态管理：通过集中式的编排调度系统来动态的管理和调度。
* 面向微服务：明确服务间的依赖，互相解耦。

云原生包含了一组应用的模式，用于帮助企业快速，持续，可靠，规模化地交付业务软件。云原生由微服务架构，DevOps 和以容器为代表的敏捷基础架构组成。

这边引用网上关于云原生所需要的能力和特征总结，如下图。

![](//upload-images.jianshu.io/upload_images/7891228-b9f958627d0aa2fc.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

cca

### 1.3 The Twelve Factors

12-Factors经常被直译为12要素，也被称为12原则，12原则由公有云PaaS的先驱Heroku于2012年提出（[https://12factor.net/](https://link.jianshu.com?t=https%3A%2F%2F12factor.net%2F)），目的是告诉开发者如何利用云平台提供的便利来开发更具可靠性和扩展性、更加易于维护的云原生应用。具体如下：

* 基准代码
* 显式声明依赖关系
* 在环境中存储配置
* 把后端服务当作附加资源
* 严格分离构建、发布和运行
* 无状态进程
* 通过端口绑定提供服务
* 通过进程模型进行扩展
* 快速启动和优雅终止
* 开发环境与线上环境等价
* 日志作为事件流
* 管理进程

另外还有补充的三点：

* API声明管理
* 认证和授权
* 监控与告警

距离12原则的提出已有五年多，12原则的有些细节可能已经不那么跟得上时代，也有人批评12原则的提出从一开始就有过于依赖Heroku自身特性的倾向。不过不管怎么说，12原则依旧是业界最为系统的云原生应用开发指南。

## 2. 容器化封装

最近几年docker容器化技术很火，经常在各种场合能够听到关于docker的分享。Docker让开发工程师可以将他们的应用和依赖封装到一个可移植的容器中。Docker背后的想法是创建软件程序可移植的轻量容器，让其可以在任何安装了Docker的机器上运行，而不用关心底层操作系统。

Docker可以解决虚拟机能够解决的问题，同时也能够解决虚拟机由于资源要求过高而无法解决的问题。其优势包括：

* 隔离应用依赖
* 创建应用镜像并进行复制
* 创建容易分发的即启即用的应用
* 允许实例简单、快速地扩展
* 测试应用并随后销毁它们

自动化运维工具可以降低环境搭建的复杂度，但仍然不能从根本上解决环境的问题。在看似稳定而成熟的场景下，使用Docker的好处越来越多。

## 3. 服务编排

笔者看到[Jimmy Song](https://link.jianshu.com?t=https%3A%2F%2Fjimmysong.io%2Fposts%2Ffrom-kubernetes-to-cloud-native%2F)对云原生架构中运用服务编排的总结是：

> Kubernetes——让容器应用进入大规模工业生产。

这个总结确实很贴切。编排调度的开源组件还有：Kubernetes、Mesos和Docker swarm。

> Kubernetes是目前世界上关注度最高的开源项目，它是一个出色的容器编排系统。Kubernetes出身于互联网行业的巨头Google公司，它借鉴了由上百位工程师花费十多年时间打造Borg系统的理念，通过极其简易的安装，以及灵活的网络层对接方式，提供一站式的服务。

> Mesos则更善于构建一个可靠的平台，用以运行多任务关键工作负载，包括Docker容器、遗留应用程序\(例如Java\)和分布式数据服务\(例如Spark、Kafka、Cassandra、Elastic\)。Mesos采用两级调度的架构，开发人员可以很方便的结合公司业务场景自定制MesosFramework。

他们为云原生应用提供的强有力的编排和调度能力，它们是云平台上的分布式操作系统。在单机上运行容器，无法发挥它的最大效能，只有形成集群，才能最大程度发挥容器的良好隔离、资源分配与编排管理的优势，而对于容器的编排管理，Swarm、Mesos和Kubernetes的大战已经基本宣告结束，kubernetes成为了无可争议的赢家。

## 4. 微服务架构

传统的web开发方式，一般被称为单体架构（Monolithic）所有的功能打包在一个WAR包里，基本没有外部依赖（除了容器），部署在一个JEE容器（Tomcat，JBoss，WebLogic）里，包含了DO/DAO，Service，UI等所有逻辑。其架构如下图所示。

![](//upload-images.jianshu.io/upload_images/7891228-5541ee8965435c39.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/400/format/webp)

monolithic

单体架构进行演化升级之后，过渡到SOA架构，即面向服务架构。近几年微服务架构（Micro-Service Archeticture）是最流行的架构风格，旨在通过将功能模块分解到各个独立的子系统中以实现解耦，它并没有一成不变的规定，而是需要根据业务来做设计。微服务架构是对SOA的传承，是SOA的具体实践方法。微服务架构中，每个微服务模块只是对简单、独立、明确的任务进行处理，通过REST API返回处理结果给外部。在微服务推广实践角度来看，微服务将整个系统进行拆分，拆分成更小的粒度，保持这些服务独立运行，应用容器化技术将微服务独立运行在容器中。过去设计架构时，是在内存中以参数或对象的方式实现粒度细化。微服务使用各个子服务控制模块的思想代替总线。不同的业务要求，服务控制模块至少包含服务的发布、注册、路由、代理功能。

容器化的出现，一定程度上带动了微服务架构。架构演化从单体式应用到分布式，再从分布式架构到云原生架构，微服务在其中有着不可或缺的角色。微服务带给我们很多开发和部署上的灵活性和技术多样性，但是也增加了服务调用的开销、分布式系事务、调试与服务治理方面的难题。

![](//upload-images.jianshu.io/upload_images/7891228-771acdc181e27558.png?imageMogr2/auto-orient/strip|imageView2/2/w/867/format/webp)

cloud structure

从上图Spring Cloud组件的架构可以看出在微服务架构中所必须的组件，包括：服务发现与注册、熔断机制、路由、全局锁、中心配置管理、控制总线、决策竞选、分布式会话和集群状态管理等基础组件。

![](//upload-images.jianshu.io/upload_images/7891228-a52db4b2de995e0b.png?imageMogr2/auto-orient/strip|imageView2/2/w/998/format/webp)

scvsk8s

Spring Cloud和Kubernetes有很大的不同，Spring Cloud和Kubernetes处理了不同范围的微服务架构技术点，而且是用了不同的方法。Spring Cloud方法是试图解决在JVM中的微服务架构要点，而Kubernetes方法是试图让问题消失，为开发者在平台层解决。Spring Cloud在JVM中非常强大，Kubernetes管理那些JVM很强大。看起来各取所长，充分利用这两者的优势是自然而然的趋势了。

## 5. 总结

技术架构的演变非常快，各种新的名词也是层出不穷。本文主要是对云原生的概述。云原生应用的三大特征：容器化封装、动态管理、面向微服务。首先由CNCF组织介绍了云原生的概念，然后分别对这三个特征进行详述。云原生架构是当下很火的讨论话题，是不同思想的集合，集目前各种热门技术之大成。





