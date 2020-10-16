# Pilot

在应用从单体架构向微服务架构演进的过程中，微服务之间的服务发现、负载均衡、熔断、限流等服务治理需求是无法回避的问题。

在[Service Mesh](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#service-mesh)出现之前，通常的做法是将这些基础功能以 SDK 的形式嵌入业务代码中，但是这种强耦合的方案会增加开发的难度，增加维护成本，增加质量风险。比如 SDK 需要新增新特性，业务侧也很难配合 SDK 开发人员进行升级，所以很容易造成 SDK 的版本碎片化问题。如果再存在跨语言应用间的交互，对于多语言 SDK 的支持也非常的低效。一方面是相当于相同的代码以不同语言重复实现，实现这类代码既很难给开发人员带来成就感，团队稳定性难以保障；另一方面是如果实现这类基础框架时涉及到了语言特性，其他语言的开发者也很难直接翻译。

而[Service Mesh](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#service-mesh)的本质则是将此类通用的功能沉淀至[sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)中，由[sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)接管服务的流量并对其进行治理。在这个思路下，可以通过流量劫持的手段，做到代码零侵入性。这样可以让业务开发人员更关心业务功能。而底层功能由于对业务零侵入，也使得基础功能的升级和快速的更新迭代成为可能。

[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio)是近年来[Service Mesh](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#service-mesh)的代表作，而[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio)流量管理的核心组件就是是[Pilot](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pilot)。[Pilot](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pilot)主要功能就是管理和配置部署在特定[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio)服务网格中的所有[sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)代理实例。它管理[sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)代理之间的路由流量规则，并配置故障恢复功能，如超时、重试和熔断。

## Pilot 架构

服务列表中的istio-pilot是istio的控制中枢Pilot服务。如果把数据面的envoy 也看作一种agent， 则Pilot 类似传统C /S 架构中的服务端Master，下发指令控制客户端完成业务功能。和传统的微服务架构对比， Pilot 至少涵盖服务注册中心和Config Server等管理组件的功能。

* 统一的抽象模型（Abstract Model）

Pilot 定义了网格中服务的标准模型，这个标准模型独立于各种底层平台。由于有了该标准模型，各个不同的平台可以通过适配器和 Pilot 对接，将自己特有的服务数据格式转换为标准格式，填充到 Pilot 的标准模型中。

例如 Pilot 中的 Kubernetes 适配器通过`Kubernetes API`服务器得到 kubernetes 中 service 和 pod 的相关信息，然后翻译为标准模型提供给 Pilot 使用。通过适配器模式，Pilot 还可以从`Mesos`,`Cloud Foundry`,`Consul`等平台中获取服务信息，还可以开发适配器将其他提供服务发现的组件集成到 Pilot 中。

* 标准数据平面API

Pilot 使用了一套起源于 Envoy 项目的标准数据平面 API 来将服务信息和流量规则下发到数据平面的`sidecar`中。

通过采用该标准 API，Istio 将控制平面和数据平面进行了解耦，为多种数据平面 sidecar 实现提供了可能性。事实上基于该标准 API 已经实现了多种 Sidecar 代理和 Istio 的集成，除 Istio 目前集成的 Envoy 外，还可以和`Linkerd`,`Nginmesh`等第三方通信代理进行集成，也可以基于该 API 自己编写 Sidecar 实现。

控制平面和数据平面解耦是 Istio 后来居上，风头超过 Service mesh 鼻祖`Linkerd`的一招妙棋。Istio 站在了控制平面的高度上，而 Linkerd 则成为了可选的一种 sidecar 实现，可谓**降维打击**的一个典型成功案例！

数据平面标准 API 也有利于生态圈的建立，开源、商业的各种 sidecar 以后可能百花齐放，用户也可以根据自己的业务场景选择不同的 sidecar 和控制平面集成，如高吞吐量的，低延迟的，高安全性的等等。有实力的大厂商可以根据该 API 定制自己的 sidecar，例如蚂蚁金服开源的 Golang 版本的 Sidecar`MOSN`\(Modular Observable Smart Netstub\)（`SOFAMesh`中 Golang 版本的 Sidecar\)；小厂商则可以考虑采用成熟的开源项目或者提供服务的商业 sidecar 实现。

Istio 和 Envoy 项目联合制定了`Envoy V2 API`,并采用该 API 作为 Istio 控制平面和数据平面流量管理的标准接口。

* 规则DSL语言

Pilot 还定义了一套`DSL`（Domain Specific Language）语言，DSL 语言提供了面向业务的高层抽象，可以被运维人员理解和使用。运维人员使用该 DSL 定义流量规则并下发到 Pilot，这些规则被 Pilot 翻译成数据平面的配置，再通过标准 API 分发到 Envoy 实例，可以在运行期对微服务的流量进行控制和调整。

Pilot 的规则 DSL 是采用 K8S API Server 中的[Custom Resource \(CRD\)](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)实现的，因此和其他资源类型如 Service，Pod 和 Deployment 的创建和使用方法类似，都可以用`Kubectl`进行创建。

通过运用不同的流量规则，可以对网格中微服务进行精细化的流量控制，如按版本分流，断路器，故障注入，灰度发布等。

如下图所示：pilot直接从运行平台\(kubernetes,consul\)提取数据并将其构造和转换成istio的服务发现模型， 因此pilot只有服务发现功能，无须进行服务注册。这种抽象模型解耦Pilot 和底层平台的不同实现，可支持kubernetes，consul等平台

![](/image/Istio/pilot-discovery.png)

除了服务发现， Pilot 更重要的一个功能是向数据面下发规则，包括VirtualService 、DestinationRule 、Gateway 、ServiceEntry 等流量治理规则，也包括认证授权等安全规则。Pilot 负责将各种规则转换成Envoy 可识别的格式，通过标准的XDS 协议发送给Envoy,指导Envoy 完成功作。在通信上， Envoy 通过gRPC 流式订阅Pilot 的配置资源。如下图所示， Pilot 将VirtualService 表达的路由规则分发到Evnoy 上， Envoy 根据该路由规则进行流量转发。

![](/image/Istio/pilot-xiafa.png)

![](/image/Istio/pilot-arch.png)

根据上图，[Pilot](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pilot)几个关键的模块如下。

### 抽象模型 （Abstract Model）

为了实现对不同服务注册中心 （Kubernetes、consul） 的支持，[Pilot](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pilot)需要对不同的输入来源的数据有一个统一的存储格式，也就是抽象模型。

抽象模型中定义的关键成员包括 HostName（[service](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#service)名称）、Ports（[service](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#service)端口）、Address（[service](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#service)ClusterIP）、Resolution （负载均衡策略） 等。

### 平台适配器 （Platform adapters）

[Pilot](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pilot)的实现是基于平台适配器（Platform adapters） 的，借助平台适配器[Pilot](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pilot)可以实现服务注册中心数据到抽象模型之间的数据转换。

例如[Pilot](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pilot)中的 Kubernetes 适配器通过 Kubernetes API 服务器得到 Kubernetes 中[service](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#service)和[pod](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pod)的相关信息，然后翻译为抽象模型提供给[Pilot](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pilot)使用。

通过平台适配器模式，[Pilot](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pilot)还可以从 Consul 等平台中获取服务信息，还可以开发适配器将其他提供服务发现的组件集成到[Pilot](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pilot)中。

### xDS API

[Pilot](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pilot)使用了一套起源于[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)项目的标准数据面 API 来将服务信息和流量规则下发到数据面的[sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)中。这套标准数据面 API，也叫 xDS。

[Sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)通过 xDS API 可以动态获取 Listener （监听器）、Route （路由）、[Cluster](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#cluster)（集群）及 Endpoint （集群成员）配置：

* LDS，Listener 发现服务：Listener 监听器控制[sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)启动端口监听（目前只支持 TCP 协议），并配置 L3/L4 层过滤器，当网络连接达到后，配置好的网络过滤器堆栈开始处理后续事件。
* RDS，Router 发现服务：用于 HTTP 连接管理过滤器动态获取路由配置，路由配置包含 HTTP 头部修改（增加、删除 HTTP 头部键值），virtual hosts （虚拟主机），以及 virtual hosts 定义的各个路由条目。
* CDS，[Cluster](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#cluster)发现服务：用于动态获取[Cluster](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#cluster)信息。
* EDS，Endpoint 发现服务：用与动态维护端点信息，端点信息中还包括负载均衡权重、金丝雀状态等，基于这些信息，
  [sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)可以做出智能的负载均衡决策。

通过采用该标准 API，[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio)将控制面和数据面进行了解耦，为多种数据平面[sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)实现提供了可能性。例如蚂蚁金服开源的 Golang 版本的[Sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)[MOSN \(Modular Observable Smart Network\)](https://github.com/mosn/mosn)。

### 用户 API （User API）

[Pilot](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pilot)还定义了一套用户 API， 用户 API 提供了面向业务的高层抽象，可以被运维人员理解和使用。

运维人员使用该 API 定义流量规则并下发到[Pilot](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pilot)，这些规则被[Pilot](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pilot)翻译成数据面的配置，再通过标准数据面 API 分发到[sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)实例，可以在运行期对微服务的流量进行控制和调整。

通过运用不同的流量规则，可以对网格中微服务进行精细化的流量控制，如按版本分流、断路器、故障注入、灰度发布等。

## Pilot 实现

![](/image/istio/pilot.png)

图中实线连线表示控制流，虚线连线表示数据流。带`[pilot]`的组件表示为[Pilot](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pilot)组件，图中关键的组件如下：

* Discovery [service](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#service)：即[pilot](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pilot)-discovery，主要功能是从[Service](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#service) provider（如 kubernetes 或者 consul ）中获取服务信息，从 Kubernetes API Server 中获取流量规则（Kubernetes [CRD](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#crd) Resource），并将服务信息和流量规则转化为数据面可以理解的格式，通过标准的数据面 API 下发到网格中的各个 [sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)中。
* agent：即[pilot](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pilot)-agent 组件，该进程根据 Kubernetes API Server 中的配置信息生成[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)的配置文件，负责启动、监控[sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)进程。
* proxy：既[sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar) proxy，是所有服务的流量代理，直接连接[pilot](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pilot)-discovery ，间接地从 Kubernetes 等服务注册中心获取集群中微服务的注册情况。
* [service](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#service) A/B：使用了[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio)的应用，如[Service](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#service)A/B，的进出网络流量会被 proxy 接管。

下面介绍下[Pilot](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pilot)相关的组件[pilot](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pilot)-agent、[pilot](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pilot)-discovery 的关键实现。

### pilot-agent

[pilot](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pilot)-agent 负责的主要工作如下：

* 生成[sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)的配置
* [Sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)的启动与监控

### pilot-discovery

`pilot-discovery`扮演服务注册中心、[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio)控制平面到[sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)之间的桥梁作用。[pilot](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pilot)-discovery 的主要功能如下：

* 从`Service provider`（如kubernetes或者consul）中获取服务信息
* 从 K8S API Server 中获取流量规则（K8S CRD Resource）
* 将服务信息和流量规则转化为数据平面可以理解的格式，通过标准的数据平面 API 下发到网格中的各个 sidecar 中

* > * 监控服务注册中心（如 Kubernetes）的服务注册情况。在 Kubernetes 环境下，会监控`service`、`endpoint`、`pod`、`node`等资源信息。
  >
  > * 监控[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio)控制面信息变化，在 Kubernetes 环境下，会监控包括Destination`Rule`、`VirtualService、Gateway`、`EgressRule`、`ServiceEntry`等以 Kubernetes [CRD](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#crd)形式存在的[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio)控制面配置信息。
  >
  > * 将上述两类信息合并组合为[sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)可以理解的（遵循[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)  [data plane](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#data-plane)  api 的）配置信息，并将这些信息以 gRPC 协议提供给[sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)。



