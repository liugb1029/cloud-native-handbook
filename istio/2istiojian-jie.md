#### 为什么Istio会占据C位

* 出击及时（2017 年 5 月发布 0.1版本）

* 三巨头光环加身

* 第二代 Service Mesh

* Envoy 的加入让 Istio 如虎添翼

* 功能强大

* 各大平台、厂商的支持

* 过度营销

#### 为什么使用 Istio？

* 优势

  * 轻松构建服务网格

  * 应用代码无需更改

* 功能强大

#### Istio核心功能

![](/image/Istio/istio-function.png)

* 流量控制

  * 路由  灰度发布  蓝绿发布  A/B测试

  * 流量转移

  * 弹性  超时 熔断

  * 测试   故障注入  流量镜像

* 安全

  * 认证

  * 授权

* 策略   1.5版本已废弃\(Mixer\)

  * 限流

  * 黑白名单

* 可观测性

  * 指标

  * 日志

  * 分布式追踪

#### Istio两次重大变更

![](/image/Istio/istio-version.png)

##### Istio 1.0

![](/image/Istio/istio-1.0.png)

* Pilot   可以对接不同的服务注册中心，解耦了Pilot与底层平台的不同实现；服务发现，配置分发的组件，把运维人员配置的路由信息转变成数据平面可以识别的配置项，然后分发给所有的sidecar代理，通过标准的xDS协议发送给Envoy，在通信上，Envoy通过gRPC流式；配置故障恢复功能，如超时、重试、熔断。

* Citadel  安全 授权和认证 流量加密

* Mixer  遥测-从数据平面进行指标数据收集  设置一些策略，比如限流、黑白名单。Adapter内置其中，修改后需要重启Mixer。

##### 

##### Istio1.1![](/image/Istio/istio-1.1.png)

Adapter: 进程外的组件

Galley：  配置验证，提取处理和分发的组件，分担了Pilot的功能

性能问题： Mixer不仅需要与各个proxy通信，还要与Adapter通信，增加了本身的压力

易用性：  各个组件分散，需要单独部署，给运维和升级带来了一定的复杂性。

**过于解耦 过于复杂**

##### Istio1.5![](/image/Istio/istio-1.5.png)



