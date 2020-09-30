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

