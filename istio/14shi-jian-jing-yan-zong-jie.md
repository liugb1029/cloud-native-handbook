### 常见问题

#### 503错误

* 可能的原因

  * 上游服务不可用（UH）；连接不上（UF）

  * 路由配置错误（NR）

  * 熔断导致（UO）

  * 配置下发问题，可自愈

* 解决方法：

  * 根据 Envoy 日志中 RESPONSE\_FLAGS 判断

  * 保证配置的可用性（先更新定义，后更新调用）

[Envoy Response\_Flags](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage#config-access-log-format-response-flags)![](/image/Istio/Envoy-RESPONSE_FLAGS.png)常见问题：

[https://istio.io/latest/docs/ops/common-problems/network-issues/](https://istio.io/latest/docs/ops/common-problems/network-issues/)

#### 请求中断分析

* 因为 Envoy 的引入，无法准确判断出问题的节点

* 解决方法

  * 根据 request id 串连上下游

  * 分析 Envoy 日志中的上下游元组信息

  * UPSTREAM\_CLUSTER/ DOWNSTREAM\_REMOTE\_ADDRESS/  
    DOWNSTREAM\_LOCAL\_ADDRESS/UPSTREAM\_LOCAL\_ADDRESS/  
    UPSTREAM\_HOST



