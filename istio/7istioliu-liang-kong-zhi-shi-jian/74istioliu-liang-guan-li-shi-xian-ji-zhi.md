## 前言 

---

Istio 作为一个 service mesh 开源项目,其中最重要的功能就是对网格中微服务之间的流量进行管理,包括服务发现,请求路由和服务间的可靠通信。Istio 实现了 service mesh 的控制平面，并整合 Envoy 开源项目作为数据平面的 sidecar，一起对流量进行控制。

Istio 体系中流量管理配置下发以及流量规则如何在数据平面生效的机制相对比较复杂，通过官方文档容易管中窥豹，难以了解其实现原理。本文尝试结合系统架构、配置文件和代码对 Istio 流量管理的架构和实现机制进行分析，以达到从整体上理解 Pilot 和 Envoy 的流量管理机制的目的。

## Pilot高层架构

---

Istio 控制平面中负责流量管理的组件为`Pilot`，Pilot 的高层架构如下图所示：

![](https://hugo-picture.oss-cn-beijing.aliyuncs.com/images/5Zywav.jpg)

Pilot Architecture（来自 \[Isio官网文档\]\([https://istio.io/docs/concepts/traffic-management/\)\](https://istio.io/docs/concepts/traffic-management/%29%29\)

根据上图,Pilot 主要实现了下述功能：

### 统一的服务模型 

Pilot 定义了网格中服务的标准模型，这个标准模型独立于各种底层平台。由于有了该标准模型，各个不同的平台可以通过适配器和 Pilot 对接，将自己特有的服务数据格式转换为标准格式，填充到 Pilot 的标准模型中。

例如 Pilot 中的 Kubernetes 适配器通过`Kubernetes API`服务器得到 kubernetes 中 service 和 pod 的相关信息，然后翻译为标准模型提供给 Pilot 使用。通过适配器模式，Pilot 还可以从`Mesos`,`Cloud Foundry`,`Consul`等平台中获取服务信息，还可以开发适配器将其他提供服务发现的组件集成到 Pilot 中。

### 标准数据平面 API 

Pilot 使用了一套起源于 Envoy 项目的标准数据平面 API 来将服务信息和流量规则下发到数据平面的`sidecar`中。

通过采用该标准 API，Istio 将控制平面和数据平面进行了解耦，为多种数据平面 sidecar 实现提供了可能性。事实上基于该标准 API 已经实现了多种 Sidecar 代理和 Istio 的集成，除 Istio 目前集成的 Envoy 外，还可以和`Linkerd`,`Nginmesh`等第三方通信代理进行集成，也可以基于该 API 自己编写 Sidecar 实现。

控制平面和数据平面解耦是 Istio 后来居上，风头超过 Service mesh 鼻祖`Linkerd`的一招妙棋。Istio 站在了控制平面的高度上，而 Linkerd 则成为了可选的一种 sidecar 实现，可谓**降维打击**的一个典型成功案例！

数据平面标准 API 也有利于生态圈的建立，开源、商业的各种 sidecar 以后可能百花齐放，用户也可以根据自己的业务场景选择不同的 sidecar 和控制平面集成，如高吞吐量的，低延迟的，高安全性的等等。有实力的大厂商可以根据该 API 定制自己的 sidecar，例如蚂蚁金服开源的 Golang 版本的 Sidecar`MOSN`\(Modular Observable Smart Netstub\)（`SOFAMesh`中 Golang 版本的 Sidecar\)；小厂商则可以考虑采用成熟的开源项目或者提供服务的商业 sidecar 实现。

Istio 和 Envoy 项目联合制定了`Envoy V2 API`,并采用该 API 作为 Istio 控制平面和数据平面流量管理的标准接口。

### 业务 DSL 语言 

Pilot 还定义了一套`DSL`（Domain Specific Language）语言，DSL 语言提供了面向业务的高层抽象，可以被运维人员理解和使用。运维人员使用该 DSL 定义流量规则并下发到 Pilot，这些规则被 Pilot 翻译成数据平面的配置，再通过标准 API 分发到 Envoy 实例，可以在运行期对微服务的流量进行控制和调整。

Pilot 的规则 DSL 是采用 K8S API Server 中的[Custom Resource \(CRD\)](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)实现的，因此和其他资源类型如 Service，Pod 和 Deployment 的创建和使用方法类似，都可以用`Kubectl`进行创建。

通过运用不同的流量规则，可以对网格中微服务进行精细化的流量控制，如按版本分流，断路器，故障注入，灰度发布等。

## Istio 流量管理相关组件

---

我们可以通过下图了解 Istio 流量管理涉及到的相关组件。虽然该图来自`Istio Github old pilot repo`, 但图中描述的组件及流程和目前 Pilot 的最新代码的架构基本是一致的。

![](https://hugo-picture.oss-cn-beijing.aliyuncs.com/images/dCSUXw.jpg)

Pilot Design Overview \(来自 \[Istio old\_pilot\_repo\]\([https://github.com/istio/old\_pilot\_repo/blob/master/doc/design.md\)\](https://github.com/istio/old_pilot_repo/blob/master/doc/design.md%29%29\)

图例说明：图中红色的线表示控制流，**黑色**的线表示数据流。蓝色部分为和Pilot相关的组件。

从上图可以看到，Istio 中和流量管理相关的有以下组件：

### 控制平面组件

#### Discovery Services

对应的 docker 镜像为`gcr.io/istio-release/pilot`,进程为`pilot-discovery`，该组件的功能包括：

* 从`Service provider`（如kubernetes或者consul）中获取服务信息
* 从 K8S API Server 中获取流量规则（K8S CRD Resource）
* 将服务信息和流量规则转化为数据平面可以理解的格式，通过标准的数据平面 API 下发到网格中的各个 sidecar 中

#### K8S API Server

提供 Pilot 相关的 CRD Resource 的增、删、改、查。和 Pilot 相关的`CRD`有以下几种：

* Virtualservice: 用于定义路由规则，如根据来源或 Header 制定规则，或在不同服务版本之间分拆流量。
* DestinationRule: 定义目的服务的配置策略以及可路由子集。策略包括断路器、负载均衡以及 TLS 等。
* ServiceEntry: 用[ServiceEntry](https://istio.io/docs/reference/config/istio.networking.v1alpha3/#ServiceEntry)可以向 Istio 中加入附加的服务条目，以使网格内可以向 Istio 服务网格之外的服务发出请求。
* Gateway: 为网格配置网关，以允许一个服务可以被网格外部访问。
* EnvoyFilter: 可以为 Envoy 配置过滤器。由于 Envoy 已经支持`Lua`过滤器，因此可以通过`EnvoyFilter`启用 Lua 过滤器，动态改变 Envoy 的过滤链行为。我之前一直在考虑如何才能动态扩展 Envoy 的能力，EnvoyFilter 提供了很灵活的扩展性。

### 数据平面组件 

在数据平面有两个进程`Pilot-agent`和`envoy`，这两个进程被放在一个 docker 容器`gcr.io/istio-release/proxyv2`中。

#### Pilot-agent

该进程根据 K8S API Server 中的配置信息生成 Envoy 的配置文件，并负责启动 Envoy 进程。注意 Envoy 的大部分配置信息都是通过`xDS`接口从 Pilot 中动态获取的，因此 Agent 生成的只是用于初始化 Envoy 的少量静态配置。在后面的章节中，本文将对 Agent 生成的 Envoy 配置文件进行进一步分析。

#### Envoy

Envoy 由`Pilot-agent`进程启动，启动后，Envoy 读取 Pilot-agent 为它生成的配置文件，然后根据该文件的配置获取到 Pilot 的地址，通过数据平面标准 API 的 xDS 接口从 pilot 拉取动态配置信息，包括路由（route），监听器（listener），服务集群（cluster）和服务端点（endpoint）。Envoy 初始化完成后，就根据这些配置信息对微服务间的通信进行寻址和路由。

### 命令行工具 

`kubectl`和`istioctl`，由于 Istio 的配置是基于 K8S 的`CRD`，因此可以直接采用 kubectl 对这些资源进行操作。Istioctl 则针对 Istio 对 CRD 的操作进行了一些封装。Istioctl 支持的功能参见该[表格](https://istio.io/docs/reference/commands/istioctl)。

## 数据平面标准 API 

---

前面讲到，Pilot 采用了一套标准的 API 来向数据平面 Sidecar 提供服务发现，负载均衡池和路由表等流量管理的配置信息。该标准 API 的文档参见[Envoy v2 API](https://www.envoyproxy.io/docs/envoy/latest/configuration/overview/v2_overview)。[Data Plane API Protocol Buffer Definition](https://github.com/envoyproxy/data-plane-api/tree/master/envoy/api/v2)给出了`v2 grpc`接口相关的数据结构和接口定义。

Istio 早期采用了 Envoy v1 API，目前的版本中则使用 V2 API，V1 已被废弃。

### 基本概念和术语

首先我们需要了解数据平面 API 中涉及到的一些基本概念：

* `Host`：能够进行网络通信的实体（如移动设备、服务器上的应用程序）。在此文档中，主机是逻辑网络应用程序。一块物理硬件上可能运行有多个主机，只要它们是可以独立寻址的。在 EDS 接口中，也使用`Endpoint`来表示一个应用实例，对应一个 IP+Port 的组合。
* `Downstream`: 下游主机连接到 Envoy，发送请求并接收响应。
* `Upstream`: 上游主机接收来自 Envoy 的连接和请求，并返回响应。
* `Listener`: 监听器是命名网地址（例如，端口、unix domain socket 等\)，可以被下游客户端连接。Envoy 暴露一个或者多个监听器给下游主机连接。在 Envoy 中，Listener 可以绑定到端口上直接对外服务，也可以不绑定到端口上，而是接收其他 listener 转发的请求。
* `Cluster`: 集群是指 Envoy 连接到的逻辑上相同的一组上游主机。Envoy 通过服务发现来发现集群的成员。可以选择通过主动健康检查来确定集群成员的健康状态。Envoy 通过负载均衡策略决定将请求路由到哪个集群成员。

### XDS 服务接口 

Istio 数据平面 API 定义了 xDS 服务接口，Pilot 通过该接口向数据平面 sidecar 下发动态配置信息，以对 Mesh 中的数据流量进行控制。xDS 中的 DS 表示`discovery service`，即发现服务，表示`xDS`接口使用动态发现的方式提供数据平面所需的配置数据。而 x 则是一个代词，表示有多种 discover service。这些发现服务及对应的数据结构如下：

* `LDS`\(Listener Discovery Service\) :[envoy.api.v2.Listener](https://github.com/envoyproxy/data-plane-api/blob/master/envoy/api/v2/lds.proto)
* `CDS`\(Cluster Discovery Service\) : [envoy.api.v2.RouteConfiguration](https://github.com/envoyproxy/data-plane-api/blob/master/envoy/api/v2/rds.proto)
* `EDS`\(Endpoint Discovery Service\) :[envoy.api.v2.Cluster](https://github.com/envoyproxy/data-plane-api/blob/master/envoy/api/v2/cds.proto)
* `RDS`\(Route Discovery Service\) : [envoy.api.v2.ClusterLoadAssignment](https://github.com/envoyproxy/data-plane-api/blob/master/envoy/api/v2/eds.proto)

### XDS 服务接口的最终一致性考虑 

xDS 的几个接口是相互独立的，接口下发的配置数据是最终一致的。但在配置更新过程中，可能暂时出现各个接口的数据不匹配的情况，从而导致部分流量在更新过程中丢失。

设想这种场景：在`CDS/EDS`只知道 cluster X 的情况下，`RDS`的一条路由配置将指向Cluster X 的流量调整到了 Cluster Y。在 CDS/EDS 向 Mesh 中 Envoy 提供 Cluster Y 的更新前，这部分导向 Cluster Y 的流量将会因为 Envoy 不知道 Cluster Y 的信息而被丢弃。

对于某些应用来说，短暂的部分流量丢失是可以接受的，例如客户端重试可以解决该问题，并不影响业务逻辑。对于另一些场景来说，这种情况可能无法容忍。可以通过调整 xDS 接口的更新逻辑来避免该问题，对上面的情况，可以先通过 CDS/EDS 更新 Y Cluster，然后再通过 RDS 将 X 的流量路由到Y。

一般来说，为了避免 Envoy 配置数据更新过程中出现流量丢失的情况，xDS 接口应采用下面的顺序：

1. `CDS`首先更新`Cluster`数据（如果有变化）
2. `EDS`更新相应 Cluster 的`Endpoint`信息（如果有变化）
3. `LDS`更新 CDS/EDS 相应的`Listener`
4. `RDS`最后更新新增 Listener 相关的`Route`配置
5. 删除不再使用的 CDS cluster 和 EDS endpoints

### ADS 聚合发现服务

保证控制平面下发数据一致性，避免流量在配置更新过程中丢失的另一个方式是使用 ADS\(Aggregated Discovery Services\)，即聚合的发现服务。`ADS`通过一个 gRPC 流来发布所有的配置更新，以保证各个 xDS 接口的调用顺序，避免由于 xDS 接口更新顺序导致的配置数据不一致问题。

关于 XDS 接口的详细介绍可参考[xDS REST and gRPC protocol](https://github.com/envoyproxy/data-plane-api/blob/master/XDS_PROTOCOL.md)

