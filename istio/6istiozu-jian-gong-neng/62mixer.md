Mixer与Envoy sidecar数据平面进行网络通信，pilot控制平面配置Mixer check和report功能需要的配置信息。Mixer主要有两大核心功能：**check（策略： 黑白名单和流量限制）**和**report（遥测收集）**，这两个功能具体实现则由Mixer后端**众多适配器**操作，下面会介绍下Mixer这两个功能是如何工作的。

### Mixer与服务之间调用交互流程

每个经过Envoy sidecar的请求都会调用Mixer。如下图1所示，展示了Service A调用Service B的过程，Pod A请求到达Pod B的Envoy时，进行一次check操作，Mixer处理完后响应数据，Envoy B接收到通过或者拒绝，如果通过，请求继续，则Envoy B会继续请求Service B,Service B处理完业务向Envoy B 响应，Envoy B 接收到响应再次向Mixer发送一次report请求。Mixer组件提供了一组后端抽象服务，并且为这些服务提供了对应的 API，每次请求操作都会根据预先定义好的配置文件，调用相关Adapter的API来执行具体的check和report的操作。

![](/image/Istio/Mixer.png)

Istio 控制面部署了两个Mixer 组件： istio-telemetry 和istio-policy ，分别处理遥测数据的收集和策略的执行。查看两个组件的Pod 镜像会发现，容器的镜像是相同的，都是"/istio/mixer"。Mixer 是Istio 独有的一种设计, 不同于Pilot ，在其他平台上总能找到类似功能的服务组件。从调用时机上来说,Pilot 管理的是配置数据，在配置改变时和数据面交互即可；然而，对于Mixer 来说，在服务间交互时Envoy 都会对Mixer 进行一次调用，因此这是一种实时管理。当然，在实现上通过在Mixer 和Proxy 上使用缓存机制，可保证不用每次进行数据面请求时都和Mixer 交互。

istio-policy主要负责接收来自Envoy sidecar的check请求，拒绝或通过，只有通过check校验的请求，才允许继续请求目标服务，istio-telemetry则负责接收来自Envoy sidecar的report请求，该请求只做日志记录操作并响应于请求端。

1. **istio-telemetry**
     istio-telemetry是专门用于收集遥测数据的Mixer服务组件;如下图所示 所示，当网格中的两个服务间有调用发生时，服务的代理Envoy 就会上报遥测数据给istio-telemetry服务组件，istio-telemetry 服务组件则根据配置将生成访问Metric等数据分发给后端的遥测服务。数据面代理通过Report 接口上报数据时访问数据会被批量上报。
   ![](/image/Istio/istio-telemetry.png)
2. **istio-policy**
     istio-policy 是另外一个Mixer 服务，和istio-telemetry 基本上是完全相同的机制和流程。
     如图下图所示，数据面在转发服务的请求前调用istio-policy 的Check接口检查是否允许访问， Mixer 根据配置将请求转发到对应的Adapter 做对应检查，给代理返回允许访问还是拒绝。可以对接如配额、授权、黑白名单等不同的控制后端，对服务间的访问进行可扩展的控制。
   ![](/image/Istio/istio-policy.png)

Mixer支持的后端适配器有很多种，这些适配器能处理和完成不同的功能，根据check功能型和report功能型，将分类列出一些常用的适配器。

**Check功能型适配器：**

* Denier/list适配器：权限控制/黑白名单检测功能
* memory quota/redis quota适配器：速率限制功能

**Report功能型适配器：**

* prometheus适配器：收集指标、获取TCP服务指标功能
* Stdio适配器：日志输入输出
* Fluentd适配器：将日志发送到Fluentd 守护进程
* Statsd适配器：向istio-statsd-prom-bridge后端服务发送指标数据

### Mixer配置模型

Mixer配置模型分为三种：Handler、Instance、Rule。Handler对应了后端具体的某个Adapter实例，Instance对应着具体Handler需要的一组属性数据，Rule则配置匹配规则，指定满足什么样的条件才允许对相关后端适配器进行report或check操作。

