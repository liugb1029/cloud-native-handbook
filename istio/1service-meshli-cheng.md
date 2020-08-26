### Service Mesh的演进历程

#### 第一阶段  逻辑耦合

![](/image/Istio/ServiceMesh-1.png)

* 控制逻辑和业务逻辑耦合

#### 第二阶段  公共库

* 解耦

* 消除重复

* 成本  人力和时间成本

* 语言绑定  比如： spring cloud 是针对java

* 仍有侵入![](/image/Istio/ServiceMesh-2.png)

#### 第三阶段  代理

* 功能简陋 比如nginx  Apache

* 思路正确

![](/image/Istio/ServiceMesh-3.png)

#### 第四阶段  边车模式 Sidecar![](/image/Istio/ServiceMesh-4-sidecar.png)

#### 第五阶段  Service Mesh![](/image/Istio/ServiceMesh-5-sidecar.png)

#### 主要功能

1、流量控制

```
路由  流量转移  超时重试 熔断  故障注入  流量镜像
```

2、策略

```
流量限制、黑白名单
```

3、网络安全

```
 授权及身份验证
```

4、可观测性

```
 指标收集和展示、日志收集、分布式跟踪
```

#### Service Mesh与Kubernetes的关系

##### Kubernetes：

* 解决容器编排与调度问题
* 本质上是管理应用生命周期\(调度器\)
* 给予Service Mesh支持和帮助

##### Service Mesh:

* 解决服务间网络通信问题
* 本质上是管理服务通信\(代理\)

#### Service Mesh技术标准

* UDPA \(Universal Data Plane API\)  数据面
* SMI \(Service Mesh Interface\)    控制面

![](/image/Istio/SMI-and-UDPA.png)

#### Service Mesh 产品发展历史![](/image/Istio/service-mesh-devlop-history.png)

* Linkerd![](/image/Istio/linkerd.png)

* Envoy![](/image/Istio/envoy.png)

* Istio

![](/image/Istio/Istio.png)

* App Mesh
* Sofa Mesh



