#### 微服务架构可视化的重要性

* 痛点

  ```
   服务间依赖关系错综复杂

   问题排查困难，扯皮甩锅时有发生
  ```

* 优势

  ```
     梳理服务的交互关系

     了解应用的行为与状态
  ```

### Kiali（望远镜）

#### 官方定义

```
   Istio的可观测性控制台

   通过服务拓扑帮助你理解服务网格的结构

   提供网格的健康状态视图

   具有服务网格配置功能
```

#### 功能![](/image/Istio/Kiali-Feature.png)架构

![](/image/Istio/Kiali-architecture.png)

### Prometheus

#### 简介![](/image/Istio/Prometheus-introduce.png)架构

#### ![](/image/Istio/Prometheus-architecture.png)

#### Istio遥测

![](/image/Istio/Istio-telemetry-v1.png)

### ![](/image/Istio/Istio-telemetry-v2.png)Grafana

#### 功能

![](/image/Istio/Grafana-introduce.png)

#### Istio Dashboard

* Mesh Dashboard：查看应用（服务）数据

  ```
      网格数据总览

      服务视图

      工作负载视图
  ```

* Performance Dashboard：查看 Istio 自身（各组件）数据

  ```
       Istio 系统总览

       各组件负载情况
  ```

![](/image/Istio/Grafana-istio-mesh-board.png)

### 日志

![](/image/Istio/ELK最终形态.png)

#### 查看Envoy日志

```
[root@master ~]# kubectl logs -f productpage-v1-6987489c74-6fgz8 istio-proxy
....
{"requested_server_name":"-","bytes_received":"0","istio_policy_status":"-","bytes_sent":"178","upstream_cluster":"outbound|9080|v1|details.default.svc.cluster.local","downstream_remote_address":"10.244.2.77:56892","authority":"details:9080","path":"/details/0","protocol":"HTTP/1.1","upstream_service_time":"1","upstream_local_address":"10.244.2.77:40778","duration":"1","upstream_transport_failure_reason":"-","route_name":"-","downstream_local_address":"10.96.64.99:9080","user_agent":"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/85.0.4183.121 Safari/537.36","response_code":"200","response_flags":"-","start_time":"2020-09-30T03:22:42.267Z","method":"GET","request_id":"9bab4f77-f271-9d8b-9cf7-127055663144","upstream_host":"10.244.2.75:9080","x_forwarded_for":"-"}
{"requested_server_name":"-","bytes_received":"0","istio_policy_status":"-","bytes_sent":"379","upstream_cluster":"outbound|9080|v2|reviews.default.svc.cluster.local","downstream_remote_address":"10.244.2.77:43326","authority":"reviews:9080","path":"/reviews/0","protocol":"HTTP/1.1","upstream_service_time":"13","upstream_local_address":"10.244.2.77:41504","duration":"14","upstream_transport_failure_reason":"-","route_name":"-","downstream_local_address":"10.96.11.46:9080","user_agent":"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/85.0.4183.121 Safari/537.36","response_code":"200","response_flags":"-","start_time":"2020-09-30T03:22:42.272Z","method":"GET","request_id":"9bab4f77-f271-9d8b-9cf7-127055663144","upstream_host":"10.244.0.81:9080","x_forwarded_for":"-"}
{"user_agent":"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/85.0.4183.121 Safari/537.36","response_code":"200","response_flags":"-","start_time":"2020-09-30T03:22:42.263Z","method":"GET","request_id":"9bab4f77-f271-9d8b-9cf7-127055663144","upstream_host":"127.0.0.1:9080","x_forwarded_for":"10.244.0.0","requested_server_name":"outbound_.9080_._.productpage.default.svc.cluster.local","bytes_received":"0","istio_policy_status":"-","bytes_sent":"5183","upstream_cluster":"inbound|9080|http|productpage.default.svc.cluster.local","downstream_remote_address":"10.244.0.0:0","authority":"192.168.56.100:30175","path":"/productpage","protocol":"HTTP/1.1","upstream_service_time":"25","upstream_local_address":"127.0.0.1:37784","duration":"26","upstream_transport_failure_reason":"-","route_name":"default","downstream_local_address":"10.244.2.77:9080"}
```

Envoy的日志相关配置是通过configmap istio。

```
[root@master ~]# kubectl -n istio-system describe cm istio
Name:         istio
Namespace:    istio-system
Labels:       install.operator.istio.io/owning-resource=installed-state
              install.operator.istio.io/owning-resource-namespace=istio-system
              istio.io/rev=default
              operator.istio.io/component=Pilot
              operator.istio.io/managed=Reconcile
              operator.istio.io/version=1.7.2
              release=istio
Annotations:
Data
====
mesh:
----
accessLogFile: /dev/stdout
accessLogEncoding: "JSON"
defaultConfig:
  discoveryAddress: istiod.istio-system.svc:15012
  proxyMetadata:
    DNS_AGENT: ""
  tracing:
    zipkin:
      address: zipkin.istio-system:9411
disableMixerHttpReports: true
enablePrometheusMerge: true
rootNamespace: istio-system
trustDomain: cluster.local
meshNetworks:
----
networks: {}
Events:  <none>
[root@master ~]#
```

日志格式配置文件，支持TEXT和JSON两种格式

[https://istio.io/latest/docs/reference/config/istio.mesh.v1alpha1/\#MeshConfig-AccessLogEncoding](https://istio.io/latest/docs/reference/config/istio.mesh.v1alpha1/#MeshConfig-AccessLogEncoding)![](/image/Istio/Envoy日志-JSON格式.png)

#### Envoy流量五元组

![](/image/Istio/Envoy流量五元组.png)

```
{"bytes_sent":"178","upstream_cluster":"outbound|9080|v1|details.default.svc.cluster.local","downstream_remote_address":"10.244.2.77:35116","authority":"details:9080","path":"/details/0","protocol":"HTTP/1.1","upstream_service_time":"1","upstream_local_address":"10.244.2.77:40778","duration":"2","upstream_transport_failure_reason":"-","route_name":"-","downstream_local_address":"10.96.64.99:9080","user_agent":"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/85.0.4183.121 Safari/537.36","response_code":"200","response_flags":"-","start_time":"2020-09-30T03:45:05.045Z","method":"GET","request_id":"47296862-4015-90da-8029-512668ee7746","upstream_host":"10.244.2.75:9080","x_forwarded_for":"-","requested_server_name":"-","bytes_received":"0","istio_policy_status":"-"}
{"bytes_sent":"375","upstream_cluster":"outbound|9080|v3|reviews.default.svc.cluster.local","downstream_remote_address":"10.244.2.77:49784","authority":"reviews:9080","path":"/reviews/0","protocol":"HTTP/1.1","upstream_service_time":"20","upstream_local_address":"10.244.2.77:55374","duration":"20","upstream_transport_failure_reason":"-","route_name":"-","downstream_local_address":"10.96.11.46:9080","user_agent":"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/85.0.4183.121 Safari/537.36","response_code":"200","response_flags":"-","start_time":"2020-09-30T03:45:05.049Z","method":"GET","request_id":"47296862-4015-90da-8029-512668ee7746","upstream_host":"10.244.2.74:9080","x_forwarded_for":"-","requested_server_name":"-","bytes_received":"0","istio_policy_status":"-"}
{"response_flags":"-","start_time":"2020-09-30T03:45:05.040Z","method":"GET","request_id":"47296862-4015-90da-8029-512668ee7746","upstream_host":"127.0.0.1:9080","x_forwarded_for":"10.244.0.0","requested_server_name":"outbound_.9080_._.productpage.default.svc.cluster.local","bytes_received":"0","istio_policy_status":"-","bytes_sent":"5179","upstream_cluster":"inbound|9080|http|productpage.default.svc.cluster.local","downstream_remote_address":"10.244.0.0:0","authority":"192.168.56.100:30175","path":"/productpage","protocol":"HTTP/1.1","upstream_service_time":"30","upstream_local_address":"127.0.0.1:44240","duration":"31","upstream_transport_failure_reason":"-","route_name":"default","downstream_local_address":"10.244.2.77:9080","user_agent":"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/85.0.4183.121 Safari/537.36","response_code":"200"}
```

#### 示例

productpage访问details------productpage pod istio-proxy查看日志

```
{"bytes_sent":"178","upstream_cluster":"outbound|9080|v1|details.default.svc.cluster.local","downstream_remote_address":"10.244.2.77:35116","authority":"details:9080","path":"/details/0","protocol":"HTTP/1.1","upstream_service_time":"1","upstream_local_address":"10.244.2.77:40778","duration":"2","upstream_transport_failure_reason":"-","route_name":"-","downstream_local_address":"10.96.64.99:9080","user_agent":"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/85.0.4183.121 Safari/537.36","response_code":"200","response_flags":"-","start_time":"2020-09-30T03:45:05.045Z","method":"GET","request_id":"47296862-4015-90da-8029-512668ee7746","upstream_host":"10.244.2.75:9080","x_forwarded_for":"-","requested_server_name":"-","bytes_received":"0","istio_policy_status":"-"}
```

![](/image/Istio/productpage访问details流量日志.png)productpage访问reviews------productpage pod istio-proxy查看日志

```
{"bytes_sent":"375","upstream_cluster":"outbound|9080|v3|reviews.default.svc.cluster.local","downstream_remote_address":"10.244.2.77:49784","authority":"reviews:9080","path":"/reviews/0","protocol":"HTTP/1.1","upstream_service_time":"20","upstream_local_address":"10.244.2.77:55374","duration":"20","upstream_transport_failure_reason":"-","route_name":"-","downstream_local_address":"10.96.11.46:9080","user_agent":"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/85.0.4183.121 Safari/537.36","response_code":"200","response_flags":"-","start_time":"2020-09-30T03:45:05.049Z","method":"GET","request_id":"47296862-4015-90da-8029-512668ee7746","upstream_host":"10.244.2.74:9080","x_forwarded_for":"-","requested_server_name":"-","bytes_received":"0","istio_policy_status":"-"}
```

![](/image/Istio/productpage访问reviews流量日志.png)

#### 调试关键字段---REPONSE\_FLAGS

* UH：upstream cluster 中没有健康的 host，503
* UF：upstream 连接失败，503
* UO：upstream overflow（熔断）
* NR：没有路由配置，404
* URX：请求被拒绝因为限流或最大连接次数

#### Envoy日志配置项

| 配置项 | 说明 |
| :--- | :--- |
| global.proxy.accessLogFile | 日志输出文件，空为关闭输出 |
| global.proxy.accessLogEncoding | 日志编码格式： JSON、TEXT |
| global.proxy.accessLogFormat | 配置显示在日志中的字段，空为默认格式 |
| global.proxy.logLevel | 日志级别，空为warning,可选trace\|debug\|info\|warning\|error\|critical\|off |

### 分布式追踪

#### 分布式追踪概念

* 分析和监控应用的监控方法
* 查找故障点、分析性能问题
* ##### 起源于Google的Dapper
* OpenTracing:   API规范、框架、库的组合

#### ![](/image/Istio/Distributed-Systems-Tracing-concept.jpeg)

#### 常见分布式追踪工具

Jaeger Zipkin Datadog  skywalking

#### Jaeger

* 开源、端到端的分布式追踪系统。
* 针对复杂的分布式系统，对业务链路进行监控和问题排查。

[https://www.jaegertracing.io/](https://www.jaegertracing.io/)

##### 术语

**Span：**

* 逻辑单元

* 有操作名、执行时间

* 嵌套、有序、因果关系

**Trace：**

* 数据/执行路径
* Span 的组合

![](/image/Istio/Span-and-Trace.png)

#### Jaeger架构

* Collectors are writing directly to storage

![](/image/Istio/Jaeger-architecture-v1.png)

* Collectors are writing to Kafka as a preliminary buffer

#### ![](/image/Istio/Jaeger-architecture-v2.png)Istio 分布式追踪实现原理

Istio 服务网格的核心是 Envoy，是一个高性能的开源 L7 代理和通信总线。在 Istio 中，每个微服务都被注入了 Envoy Sidecar，该实例负责处理所有传入和传出的网络流量。因此，每个 Envoy Sidecar 都可以监控所有的服务间 API 调用，并记录每次服务调用所需的时间以及是否成功完成。

每当微服务发起外部调用时，客户端 Envoy 会创建一个新的 span。一个 span 代表一组微服务之间的完整交互过程，从请求者（客户端）发出请求开始到接收到服务方的响应为止。

在服务交互过程中，客户端会记录请求的发起时间和响应的接收时间，服务器端 Envoy 会记录请求的接收时间和响应的返回时间。

每个 Envoy 都会将自己的 span 视图信息发布到分布式追踪系统。当一个微服务处理请求时，可能需要调用其他微服务，从而导致因果关联的 span 的创建，形成完整的 trace。这就需要由应用来从请求消息中收集和转发下列 Header。

* `x-request-id`
* `x-b3-traceid`
* `x-b3-spanid`
* `x-b3-parentspanid`
* `x-b3-sampled`
* `x-b3-flags`
* `x-ot-span-context`

在通信链路中的 Envoy，可以截取、处理、转发相应的 Header。

```
Client Tracer                                              Server Tracer
┌──────────────────┐                                       ┌──────────────────┐
│                  │                                       │                  │
│   TraceContext   │           Http Request Headers        │   TraceContext   │
│ ┌──────────────┐ │          ┌───────────────────┐        │ ┌──────────────┐ │
│ │ TraceId      │ │          │ X─B3─TraceId      │        │ │ TraceId      │ │
│ │              │ │          │                   │        │ │              │ │
│ │ ParentSpanId │ │ Extract  │ X─B3─ParentSpanId │ Inject │ │ ParentSpanId │ │
│ │              ├─┼─────────>│                   ├────────┼>│              │ │
│ │ SpanId       │ │          │ X─B3─SpanId       │        │ │ SpanId       │ │
│ │              │ │          │                   │        │ │              │ │
│ │ Sampled      │ │          │ X─B3─Sampled      │        │ │ Sampled      │ │
│ └──────────────┘ │          └───────────────────┘        │ └──────────────┘ │
│                  │                                       │                  │
└──────────────────┘                                       └──────────────────┘
```

## Envoy-Jaeger 架构 {#envoy-jaeger-架构}

`Envoy`原生支持`Jaeger`，追踪所需`x-b3`开头的 Header 和`x-request-id`在不同的服务之间由业务逻辑进行传递，并由`Envoy`上报给`Jaeger`，最终`Jaeger`生成完整的追踪信息。

在`Istio`中，`Envoy`和`Jaeger`的关系如下：

![](/image/Istio/envoy-jaeger.png)

图中 Front[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)指的是第一个接收到请求的[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)[Sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)，它会负责创建 Root Span 并追加到请求 Header 内，请求到达不同的服务时，[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)[Sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)会将追踪信息进行上报。

`Jaeger`的内部组件架构与 EFK 日志系统架构有一定相似性：

![](/image/Istio/jaeger-architecture.png)                                                                                   Jaeger 架构图

`Jaeger`主要由以下几个组件构成：

* Client：Jaeger 客户端，是[OpenTracing](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#opentracing)API 的具体语言实现，可以为各种开源框架提供分布式追踪工具。
* Agent：监听在 UDP 端口的守护进程，以 Daemonset 的方式部署在宿主机或以[sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)方式注入容器内，屏蔽了 Client 和 Collector 之间的细节以及服务发现。用于接收 Client 发送过来的追踪数据，并将数据批量发送至 Collector。
* Collector：用来接收 Agent 发送的数据，验证追踪数据，并建立索引，最后异步地写入后端存储，Collector 是无状态的。
* DataBase：后端存储组件，支持内存、Cassandra、Elasticsearch、Kafka 的存储方式。
* Query：用于接收查询请求，从数据库检索数据并通过 UI 展示。
* UI：使用 React 编写，用于 UI 界面展示。

在`Istio`提供“开箱即用”的追踪环境中，`Jaeger`的部署方式是`all-in-one`的方式。该模式下部署的[Pod](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pod)为`istio-tracing`，使用`jaegertracing/all-in-one`镜像，包含：`Jaeger-agent`、`Jaeger-collector`、`Jaeger-query(UI)`几个组件。

不同的是，`Bookinfo`的业务代码并没有集成`Jaeger-client`，而是由`Envoy`将追踪信息直接上报到`Jaeger-collector`，另外，存储方式默认为内存，随着[Pod](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pod)销毁，追踪数据将会被删除。

## 部署方式 {#部署方式}

`Jaeger`的部署方式主要有以下几种：

* `all-in-one`部署 - 适用于快速体验`Jaeger`，所有追踪数据存储在内存中，不适用于生产环境。
* `Kubernetes`部署 - 通过在集群独立部署`Jaeger`各组件 manifest 完成，定制化程度高，可使用已有的 Elasticsearch、Kafka 服务，适用于生产环境。
* `OpenTelemetry`部署 - 适用于使用`OpenTelemetry`API 的部署方式。
* `Windows`部署 - 适用于`Windows`环境的部署方式，通过运行 exe 可执行文件安装和配置。



