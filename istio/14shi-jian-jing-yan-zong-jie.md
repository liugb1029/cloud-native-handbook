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



