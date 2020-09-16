## Bookinfo 端到端调用分析 {#bookinfo-端到端调用分析}

通过前面对[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)配置文件的分析，我们对[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)上生成的各种配置数据的结构，包括 listener、[cluster](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#cluster)、route 和 endpoint 有了一定的了解。那么这些配置是如何有机地结合在一起，以对经过网格中的流量进行路由的呢？

下面我们通过 bookinfo 示例程序中一个端到端的调用请求把这些相关的配置串连起来，使用该完整的调用流程来帮助理解[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio)控制平面的流量控制能力是如何在数据平面的[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)上实现的。

下图描述了 bookinfo 示例程序中 productpage 服务调用 reviews 服务的请求流程：

![](/image/Istio/envoy-traffic-route.jpg)

Productpage 发起对 reviews 服务的调用：

Productpage 发起对 reviews 服务的调用：

1. `http://reviews:9080/reviews/0`
   。
2. 请求被 productpage
   [Pod](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pod)
   的 iptable 规则拦截，重定向到本地的 15001 端口。
3. 在 15001 端口上监听的 VirtualOutbound listener 收到了该请求。
4. 请求被 VirtualOutbound listener 根据原目标 IP（通配）和端口（9080）转发到`0.0.0.0_9080`这个 outbound listener。

   \`\`\`  
   {  
   "name"  
   :  
   "virtualOutbound"  
   ,  
   "active\_state"  
   :  
   {  
   "version\_info"  
   :  
   "2020-03-11T08:13:39Z/22"  
   ,  
   "listener"  
   :  
   {  
   "@type"  
   :  
   "type.googleapis.com/envoy.api.v2.Listener"  
   ,  
   "name"  
   :  
   "virtualOutbound"  
   ,  
   "address"  
   :  
   {  
   "socket\_address"  
   :  
   {  
   "address"  
   :  
   "0.0.0.0"  
   ,  
   "port\_value"  
   :  
   15001  
   }  
   }  
   ,

   ......

"use\_original\_dst"  
   :  
   true  
   ,  
   "traffic\_direction"  
   :  
   "OUTBOUND"  
   }  
   ,  
   "last\_updated"  
   :  
   "2020-03-11T08:14:04.929Z"  
   }

    5. 根据
       `0.0.0.0_9080`
       listener 的
       `http_connection_manager`
       filter 配置，该请求采用 9080 route 进行分发。

{  
   "name"  
   :  
   "0.0.0.0\_9080"  
   ,  
   "active\_state"  
   :  
   {  
   "version\_info"  
   :  
   "2020-03-11T08:13:39Z/22"  
   ,  
   "listener"  
   :  
   {  
   "@type"  
   :  
   "type.googleapis.com/envoy.api.v2.Listener"  
   ,  
   "name"  
   :  
   "0.0.0.0\_9080"  
   ,  
   "address"  
   :  
   {  
   "socket\_address"  
   :  
   {  
   "address"  
   :  
   "0.0.0.0"  
   ,  
   "port\_value"  
   :  
   9080  
   }  
   }  
   ,  
   "filter\_chains"  
   :  
   \[

```
   ......
```

{  
   "filters"  
   :  
   \[  
   {  
   "name"  
   :  
   "envoy.http\_connection\_manager"  
   ,  
   "typed\_config"  
   :  
   {  
   "@type"  
   :  
   "type.googleapis.com/envoy.config.filter.network.http\_connection\_manager.v2.HttpConnectionManager"  
   ,  
   "stat\_prefix"  
   :  
   "outbound\_0.0.0.0\_9080"  
   ,  
   "rds"  
   :  
   {  
   "config\_source"  
   :  
   {  
   "ads"  
   :  
   {  
   }  
   }  
   ,  
   "route\_config\_name"  
   :  
   "9080"  
   }  
   ,  
   "http\_filters"  
   :  
   \[  
   {  
   "name"  
   :  
   "envoy.filters.http.wasm"  
   ,

```
          ......
```

}  
   ,  
   {  
   "name"  
   :  
   "istio.alpn"  
   ,

```
          ......
```

}  
   ,  
   {  
   "name"  
   :  
   "envoy.cors"  
   }  
   ,  
   {  
   "name"  
   :  
   "envoy.fault"  
   }  
   ,  
   {  
   "name"  
   :  
   "envoy.filters.http.wasm"  
   ,

```
          ......
```

}  
   ,  
   {  
   "name"  
   :  
   "envoy.router"  
   }  
   \]  
   ,  
   "tracing"  
   :  
   {  
   "client\_sampling"  
   :  
   {  
   "value"  
   :  
   100  
   }  
   ,  
   "random\_sampling"  
   :  
   {  
   "value"  
   :  
   100  
   }  
   ,  
   "overall\_sampling"  
   :  
   {  
   "value"  
   :  
   100  
   }  
   }  
   ,

```
        ......           
```

}  
   }  
   \]  
   }  
   \]  
   ,  
   "deprecated\_v1"  
   :  
   {  
   "bind\_to\_port"  
   :  
   false  
   }  
   ,  
   "traffic\_direction"  
   :  
   "OUTBOUND"  
   }  
   ,  
   "last\_updated"  
   :  
   "2020-03-11T08:14:04.927Z"  
   }  
   }  
   ,

    6. 9080 这个 route 的配置中，host name 为
       `reviews:9080`
       的请求对应的
       [cluster](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#cluster)
       为
       `outbound|9080||reviews.default.svc.cluster.local`
       。

{  
   "version\_info"  
   :  
   "2020-03-11T08:13:39Z/22"  
   ,  
   "route\_config"  
   :  
   {  
   "@type"  
   :  
   "type.googleapis.com/envoy.api.v2.RouteConfiguration"  
   ,  
   "name"  
   :  
   "9080"  
   ,  
   "virtual\_hosts"  
   :  
   \[

......

"name"  
   :  
   "ratings.default.svc.cluster.local:9080"  
   ,  
   "domains"  
   :  
   \[  
   "ratings.default.svc.cluster.local"  
   ,  
   "ratings.default.svc.cluster.local:9080"  
   ,  
   "ratings"  
   ,  
   "ratings:9080"  
   ,  
   "ratings.default.svc.cluster"  
   ,  
   "ratings.default.svc.cluster:9080"  
   ,  
   "ratings.default.svc"  
   ,  
   "ratings.default.svc:9080"  
   ,  
   "ratings.default"  
   ,  
   "ratings.default:9080"  
   ,  
   "10.102.90.243"  
   ,  
   "10.102.90.243:9080"  
   \]  
   ,  
   "routes"  
   :  
   \[  
   {  
   "match"  
   :  
   {  
   "prefix"  
   :  
   "/"  
   }  
   ,  
   "route"  
   :  
   {  
   "cluster"  
   :  
   "outbound\|9080\|\|ratings.default.svc.cluster.local"  
   ,  
   "timeout"  
   :  
   "0s"  
   ,  
   "retry\_policy"  
   :  
   {  
   "retry\_on"  
   :  
   "connect-failure,refused-stream,unavailable,cancelled,resource-exhausted,retriable-status-codes"  
   ,  
   "num\_retries"  
   :  
   2  
   ,  
   "retry\_host\_predicate"  
   :  
   \[  
   {  
   "name"  
   :  
   "envoy.retry\_host\_predicates.previous\_hosts"  
   }  
   \]  
   ,  
   "host\_selection\_retry\_max\_attempts"  
   :  
   "5"  
   ,  
   "retriable\_status\_codes"  
   :  
   \[  
   503  
   \]  
   }  
   ,  
   "max\_grpc\_timeout"  
   :  
   "0s"  
   }  
   ,  
   "decorator"  
   :  
   {  
   "operation"  
   :  
   "ratings.default.svc.cluster.local:9080/\*"  
   }  
   ,  
   "name"  
   :  
   "default"  
   }  
   \]  
   }  
   ,  
   {

```
"name
```

": "  
   reviews.default.svc.cluster.local  
   :  
   9080  
   "  
   ,

```
"domains"
```

:  
   \[  
   "reviews.default.svc.cluster.local"  
   ,  
   "reviews.default.svc.cluster.local:9080"  
   ,  
   "reviews"  
   ,  
   "reviews:9080"  
   ,  
   "reviews.default.svc.cluster"  
   ,  
   "reviews.default.svc.cluster:9080"  
   ,  
   "reviews.default.svc"  
   ,  
   "reviews.default.svc:9080"  
   ,  
   "reviews.default"  
   ,  
   "reviews.default:9080"  
   ,  
   "10.107.156.4"  
   ,  
   "10.107.156.4:9080"  
   \]  
   ,

```
"routes"
```

:  
   \[  
   {

```
  "match"
```

:  
   {

```
   "prefix
```

": "  
   /"

}  
   ,

```
  "route"
```

:  
   {

```
   "cluster
```

": "  
   outbound\|  
   9080  
   \|\|reviews.default.svc.cluster.local"  
   ,

```
   "timeout
```

": "  
   0  
   s"  
   ,

```
   "retry_policy"
```

:  
   {

```
    "retry_on
```

": "  
   connect-failure  
   ,  
   refused-stream  
   ,  
   unavailable  
   ,  
   cancelled  
   ,  
   resource-exhausted  
   ,  
   retriable-status-codes"  
   ,

```
    "num_retries"
```

:  
   2  
   ,

```
    "retry_host_predicate"
```

:  
   \[  
   {

```
      "name
```

": "  
   envoy.retry\_host\_predicates.previous\_hosts"

}  
   \]  
   ,

```
    "host_selection_retry_max_attempts
```

": "  
   5  
   "  
   ,

```
    "retriable_status_codes"
```

:  
   \[  
   503  
   \]  
   }  
   ,

```
   "max_grpc_timeout
```

": "  
   0  
   s"

}  
   ,

```
  "decorator"
```

:  
   {

```
   "operation
```

": "  
   reviews.default.svc.cluster.local  
   :  
   9080  
   /\*"  
      },  
      "name": "default"  
     }  
    \]  
   }  
   \],  
   "validate\_clusters": false  
   },  
   "last\_updated": "2020-03-11T08:14:04.971Z"  
   }

    7. `outbound|9080||reviews.default.svc.cluster.local cluster`
       为动态资源，通过 EDS 查询得到该
       [cluster](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#cluster)
       中有3个 endpoint。

{  
   "clusterName"  
   :  
   "outbound\|9080\|\|reviews.default.svc.cluster.local"  
   ,  
   "endpoints"  
   :  
   \[  
   {  
   "lbEndpoints"  
   :  
   \[  
   {  
   "endpoint"  
   :  
   {  
   "address"  
   :  
   {  
   "socketAddress"  
   :  
   {  
   "address"  
   :  
   "10.40.0.15"  
   ,  
   "portValue"  
   :  
   9080  
   }  
   }  
   }  
   ,  
   "metadata"  
   :  
   {  
   }  
   ,  
   "loadBalancingWeight"  
   :  
   1  
   }  
   ,  
   {  
   "endpoint"  
   :  
   {  
   "address"  
   :  
   {  
   "socketAddress"  
   :  
   {  
   "address"  
   :  
   "10.40.0.16"  
   ,  
   "portValue"  
   :  
   9080  
   }  
   }  
   }  
   ,  
   "metadata"  
   :  
   {  
   }  
   ,  
   "loadBalancingWeight"  
   :  
   1  
   }  
   ,  
   {  
   "endpoint"  
   :  
   {  
   "address"  
   :  
   {  
   "socketAddress"  
   :  
   {  
   "address"  
   :  
   "10.40.0.17"  
   ,  
   "portValue"  
   :  
   9080  
   }  
   }  
   }  
   ,  
   "metadata"  
   :  
   {  
   }  
   ,  
   "loadBalancingWeight"  
   :  
   1  
   }  
   \]  
   ,  
   "loadBalancingWeight"  
   :  
   3  
   }  
   \]  
   }

    8. 请求被转发到其中一个 endpoint
       `10.40.0.15`
       ，即
       `reviews-v1`
       所在的
       [Pod](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pod)
       。
    9. 然后该请求被 iptable 规则拦截，重定向到本地的 15006 端口。
    10. 在 15006 端口上监听的 VirtualInbound listener 收到了该请求。
    11. 根据匹配条件，请求被 VirtualInbound listener 内部配置的 Http connection manager filter 处理，该 filter 设置的路由配置为将其发送给
        `inbound|9080|http|reviews.default.svc.cluster.local`
        这个 inbound
        [cluster](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#cluster)
        。

    {
    "name"
    :
    "virtualInbound"
    ,
    "active_state"
    :
    {
    "version_info"
    :
    "2020-03-11T08:13:14Z/21"
    ,
    "listener"
    :
    {
    "@type"
    :
    "type.googleapis.com/envoy.api.v2.Listener"
    ,
    "name"
    :
    "virtualInbound"
    ,
    "address"
    :
    {
    "socket_address"
    :
    {
    "address"
    :
    "0.0.0.0"
    ,
    "port_value"
    :
    15006
    }
    }
    ,
    "filter_chains"
    :
    [
    {
    "filter_chain_match"
    :
    {
    "prefix_ranges"
    :
    [
    {
    "address_prefix"
    :
    "10.40.0.15"
    ,
    "prefix_len"
    :
    32
    }
    ]
    ,
    "destination_port"
    :
    9080
    ,
    "application_protocols"
    :
    [
    "istio-peer-exchange"
    ,
    "istio"
    ,
    "istio-http/1.0"
    ,
    "istio-http/1.1"
    ,
    "istio-h2"
    ]
    }
    ,
    "filters"
    :
    [
    {
    "name"
    :
    "envoy.filters.network.metadata_exchange"
    ,
    "config"
    :
    {
    "protocol"
    :
    "istio-peer-exchange"
    }
    }
    ,
    {
    "name"
    :
    "envoy.http_connection_manager"
    ,
    "typed_config"
    :
    {
    "@type"
    :
    "type.googleapis.com/envoy.config.filter.network.http_connection_manager.v2.HttpConnectionManager"
    ,
    "stat_prefix"
    :
    "inbound_10.40.0.15_9080"
    ,
    "route_config"
    :
    {
    "name"
    :
    "inbound|9080|http|reviews.default.svc.cluster.local"
    ,
    "virtual_hosts"
    :
    [
    {
    "name"
    :
    "inbound|http|9080"
    ,
    "domains"
    :
    [
    "*"
    ]
    ,
    "routes"
    :
    [
    {
    "match"
    :
    {
    "prefix"
    :
    "/"
    }
    ,
    "route"
    :
    {
    "cluster"
    :
    "inbound|9080|http|reviews.default.svc.cluster.local"
    ,
    "timeout"
    :
    "0s"
    ,
    "max_grpc_timeout"
    :
    "0s"
    }
    ,
    "decorator"
    :
    {
    "operation"
    :
    "reviews.default.svc.cluster.local:9080/*"
    }
    ,
    "name"
    :
    "default"
    }
    ]
    }
    ]
    ,

         "validate_clusters"
    :
    false
    }
    ,

        "http_filters"
    :
    [
    {

          "name
    ": "
    envoy.filters.http.wasm"
    ,

          ......

    }
    ,
    {

          "name
    ": "
    istio_authn"
    ,

          ......

    }
    ,
    {

          "name
    ": "
    envoy.cors"

    }
    ,
    {

          "name
    ": "
    envoy.fault"

    }
    ,
    {

          "name
    ": "
    envoy.filters.http.wasm"
    ,

          ......

    }
    ,
    {

          "name
    ": "
    envoy.router"

    }
    ]
    ,

        ......

    }
    }
    ]
    ,

     "metadata"
    :
    {
    ...
    }
    ,

     "transport_socket"
    :
    {
    ...
    }
    ]
    ,

    ......

    }
    }
    ```

1. `inbound|9080|http|reviews.default.svc.cluster.local cluster`
   配置的 host 为
   `127.0.0.1:9080`
   。
   ```
   {
   "version_info"
   :
   "2020-03-11T08:13:14Z/21"
   ,
   "cluster"
   :
   {
   "@type"
   :
   "type.googleapis.com/envoy.api.v2.Cluster"
   ,
   "name"
   :
   "inbound|9080|http|reviews.default.svc.cluster.local"
   ,
   "type"
   :
   "STATIC"
   ,
   "connect_timeout"
   :
   "1s"
   ,
   "circuit_breakers"
   :
   {
   "thresholds"
   :
   [
   {
   "max_connections"
   :
   4294967295
   ,
   "max_pending_requests"
   :
   4294967295
   ,
   "max_requests"
   :
   4294967295
   ,
   "max_retries"
   :
   4294967295
   }
   ]
   }
   ,
   "load_assignment"
   :
   {
   "cluster_name"
   :
   "inbound|9080|http|reviews.default.svc.cluster.local"
   ,
   "endpoints"
   :
   [
   {
   "lb_endpoints"
   :
   [
   {
   "endpoint"
   :
   {
   "address"
   :
   {
   "socket_address"
   :
   {
   "address"
   :
   "127.0.0.1"
   ,
   "port_value"
   :
   9080
   }
   }
   }
   }
   ]
   }
   ]
   }
   }
   ,
   "last_updated"
   :
   "2020-03-11T08:13:39.118Z"
   }
   ```
2. 请求被转发到
   `127.0.0.1:9080`
   ，即 reviews 服务进行业务处理。



