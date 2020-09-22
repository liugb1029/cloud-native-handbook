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

