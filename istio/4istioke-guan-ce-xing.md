# 可观察性 

面对复杂的应用环境和不断扩展的业务需求，即使再完备的测试也难以覆盖所有场景，无法保证服务不会出现故障。正因为如此，才需要“可观察性”来对服务的运行时状态进行监控、上报、分析，以提高服务可靠性。具有可观察性的系统，可以在服务出现故障时大大降低问题定位的难度，甚至可以在出现问题之前及时发现问题以降低风险。具体来说，可观察性可以：

* 及时反馈异常或者风险使得开发人员可以及时关注、修复和解决问题（告警）；
* 出现问题时，能够帮助快速定位问题根源并解决问题，以减少服务损失（减损）；
* 收集并分析数据，以帮助开发人员不断调整和改善服务（持续优化）。

而在微服务治理之中，随着服务数量大大增加，服务拓扑不断复杂化，可观察性更是至关重要。[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio)自然也不可能缺少对可观察性的支持。它会为所有的服务间通信生成详细的遥测数据，使得网格中每个服务请求都可以被观察和跟踪。开发人员可以凭此定位故障，维护和优化相关服务。而且，这一特性的引入无需侵入被观察的服务。

[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio)一共提供了三种不同类型的数据从不同的角度支撑起其可观察性：

* 指标（Metrics）：指标本质上是时间序列上的一系列具有特定名称的计数器的组合，不同计数器用于表征系统中的不同状态并将之数值化。通过数据聚合之后，指标可以用于查看一段时间范围内系统状态的变化情况甚至预测未来一段时间系统的行为。举一个简单的例子，系统可以使用一个计数器来对所有请求进行计数，并且周期性（周期越短，实时性越好，开销越大）的将该数值输出到时间序列数据库（比如 Prometheus）中，由此得到的一组数值通过数学处理之后，可以直观的展示系统中单位时间内的请求数及其变化趋势，可以用于实时监控系统中流量大小并预测未来流量趋势。而具体到[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio)中，它**基于四类不同的监控标识（响应延迟、流量大小、错误数量、饱和度）**生成了一系列观测不同服务的监控指标，用用于记录和展示网格中服务状态。除此以外，它还提供了一组默认的基于上述指标的网格监控仪表板，对指标数据进行聚合和可视化。借助指标，开发人员可以快速的了解当前网格中流量大小、是否频繁的出现异常响应、性能是否符合预期等等关键状态。但是，如前所述，指标本质上是计数器的组合和系统状态的数值化表示，所以往往缺失细节内容，它是从一个相对宏观的角度来展现整个网格或者系统状态随时间发生的变化及趋势。在一些情况下，指标也可以辅助定位问题。

* 日志（Access Logs）：日志是软件系统中记录软件执行状态及内部事件最为常用也最为有效的工具。而在可观测性的语境之下，日志是具有相对固定结构的一段文本或者二进制数据（区别于运行时日志），并且和系统中需要关注的事件一一对应。当系统中发生一个新的事件，指标只会有几个相关的计数器自增，而日志则会记录下该事件具体的上下文。因此，日志包含了系统状态更多的细节部分。在分布式系统中，日志是定位复杂问题的关键手段；同时，由于每个事件都会产生一条对应的日志，所以日志也往往被用于计费系统，作为数据源。其相对固定的结构，也提供了日志解析和快速搜索的可能，对接 ELK 等日志分析系统后，可以快速的筛选出具有特定特征的日志以分析系统中某些特定的或者需要关注的事件。在[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio)网格中，当请求流入到网格中任何一个服务时，[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio)都会生成该请求的完整记录，包括请求源和请求目标以及请求本身的元数据等等。日志使网格开发人员可以在单个服务实例级别观察和审计流经该实例的所有流量。

* 分布式追踪（Distributed Traces）：尽管日志记录了各个事件的细节，可在分布式系统中，日志仍旧存在不足之处。日志记录的事件是孤立的，但是在实际的分布式系统中，不同组件中发生的事件往往存在因果关系。举例来说，组件 A 接收外部请求之后，会调用组件 B，而组件 B 会继续调用组件 C。在组件 A B C 中，分别有一个事件发生并各产生一条日志。但是三条日志没有将三个事件的因果关系记录下来。而分布式追踪正是为了解决该问题而存在。分布式追踪通过额外数据（Span ID等特殊标记）记录不同组件中事件之间的关联，并由外部数据分析系统重新构造出事件的完整事件链路以及因果关系。在服务网格的一次请求之中，  
  [Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio)会为途径的所有服务生成分布式追踪数据并上报，通过 Zipkin 等追踪系统重构服务调用链，开发人员可以借此了解网格内服务的依赖和调用流程，构建整个网格的服务拓扑。在未发生故障时，可以借此分析网格性能瓶颈或热点服务；而在发生故障时，则可以通过分布式追踪快速定位故障点。

本节内容仅仅简单介绍了[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio)中可观察性的相关概念，而未深入具体的细节，希望都能能够凭此建立起对可观察性初步的印象。更加详细的相关内容将在本书的实践篇介绍。

## 指标

**指标（Metric）提供了一种以聚合的方式监控和理解行为的方法**。

为了监控服务行为，Istio 为服务网格中所有出入的服务流量都生成了指标。这些指标提供了关于行为的信息，例如总流量数、错误率和请求响应时间。

除了监控网格中服务的行为外，监控网格本身的行为也很重要。Istio 组件可以导出自身内部行为的指标，以提供对网格控制平面的功能和健康情况的洞察能力。

Istio 指标收集由运维人员配置来驱动。运维人员决定如何以及何时收集指标，以及指标本身的详细程度。这使得它能够灵活地调整指标收集来满足个性化需求。

### 代理级别指标

Istio 指标收集从 sidecar 代理（Envoy）开始。每个代理为通过它的所有流量（入站和出站）生成一组丰富的指标。代理还提供关于它本身管理功能的详细统计信息，包括配置信息和健康信息。

Envoy 生成的指标提供了资源（例如监听器和集群）粒度上的网格监控。因此，为了监控 Envoy 指标，需要了解网格服务和 Envoy 资源之间的连接。

Istio 允许运维人员在每个工作负载实例上选择生成和收集哪个 Envoy 指标。默认情况下，Istio 只支持 Envoy 生成的统计数据的一小部分，以避免依赖过多的后端服务，还可以减少与指标收集相关的 CPU 开销。然而，运维人员可以在需要时轻松地扩展收集到的代理指标集。这支持有针对性地调试网络行为，同时降低了跨网格监控的总体成本。

[Envoy 文档](https://www.envoyproxy.io/docs/envoy/latest/)包括了[Envoy 统计信息收集](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/observability/statistics.html?highlight=statistics)的详细说明。[Envoy 统计](https://istio.io/latest/zh/docs/ops/diagnostic-tools/proxy-cmd/)里的操作手册提供了有关控制代理级别指标生成的更多信息。

代理级别指标的例子：

```
envoy_cluster_internal_upstream_rq{response_code_class="2xx",cluster_name="xds-grpc"} 7163

envoy_cluster_upstream_rq_completed{cluster_name="xds-grpc"} 7164

envoy_cluster_ssl_connection_error{cluster_name="xds-grpc"} 0

envoy_cluster_lb_subsets_removed{cluster_name="xds-grpc"} 0

envoy_cluster_internal_upstream_rq{response_code="503",cluster_name="xds-grpc"} 1
```

### 服务级别指标

除了代理级别指标之外，Istio 还提供了一组用于监控服务通信的面向服务的指标。这些指标涵盖了四个基本的服务监控需求：延迟、流量、错误和饱和情况。Istio 带有一组默认的[仪表板](https://istio.io/latest/zh/docs/tasks/observability/metrics/using-istio-dashboard/)，用于监控基于这些指标的服务行为。

[默认的 Istio 指标](https://istio.io/latest/zh/docs/reference/config/policy-and-telemetry/metrics/)由 Istio 提供的配置集定义并默认导出到[Prometheus](https://istio.io/latest/zh/docs/reference/config/policy-and-telemetry/adapters/prometheus/)。运维人员可以自由地修改这些指标的形态和内容，更改它们的收集机制，以满足各自的监控需求。

[收集指标](https://istio.io/latest/zh/docs/tasks/observability/metrics/collecting-metrics/)任务为定制 Istio 指标生成提供了更详细的信息。

服务级别指标的使用完全是可选的。运维人员可以选择关闭指标的生成和收集来满足自身需要。

服务级别指标的例子：

```
istio_requests_total{
  connection_security_policy="mutual_tls",
  destination_app="details",
  destination_principal="cluster.local/ns/default/sa/default",
  destination_service="details.default.svc.cluster.local",
  destination_service_name="details",
  destination_service_namespace="default",
  destination_version="v1",
  destination_workload="details-v1",
  destination_workload_namespace="default",
  reporter="destination",
  request_protocol="http",
  response_code="200",
  response_flags="-",
  source_app="productpage",
  source_principal="cluster.local/ns/default/sa/default",
  source_version="v1",
  source_workload="productpage-v1",
  source_workload_namespace="default"
} 214
```

### 控制平面指标

每一个 Istio 的组件（Pilot、Galley、Mixer）都提供了对自身监控指标的集合。这些指标容许监控 Istio 自己的行为（这与网格内的服务有所不同）。

有关这些被维护指标的更多信息，请查看每个组件的参考文档：

* [Pilot](https://istio.io/latest/zh/docs/reference/commands/pilot-discovery/#metrics)
* [Galley](https://istio.io/latest/zh/docs/reference/commands/galley/#metrics)
* [Mixer](https://istio.io/latest/zh/docs/reference/commands/mixs/#metrics)
* [Citadel](https://istio.io/latest/zh/docs/reference/commands/istio_ca/#metrics)



