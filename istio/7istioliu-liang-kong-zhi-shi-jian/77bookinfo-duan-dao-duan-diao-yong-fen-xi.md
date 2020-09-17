## Bookinfo 端到端调用分析 {#bookinfo-端到端调用分析}

通过前面对[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)配置文件的分析，我们对[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)上生成的各种配置数据的结构，包括 listener、[cluster](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#cluster)、route 和 endpoint 有了一定的了解。那么这些配置是如何有机地结合在一起，以对经过网格中的流量进行路由的呢？

下面我们通过 bookinfo 示例程序中一个端到端的调用请求把这些相关的配置串连起来，使用该完整的调用流程来帮助理解[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio)控制平面的流量控制能力是如何在数据平面的[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)上实现的。

下图描述了 bookinfo 示例程序中 productpage 服务调用 reviews 服务的请求流程：

![](/image/Istio/envoy-traffic-route.jpg)

Productpage 发起对 reviews 服务的调用：

Productpage 发起对 reviews 服务的调用：

1. `http://reviews:9080/reviews/0`。
2. 请求被 productpage [Pod](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pod) 的 iptable 规则拦截，重定向到本地的 15001 端口。
3. 在 15001 端口上监听的 VirtualOutbound listener 收到了该请求。
4. 请求被 VirtualOutbound listener 根据原目标 IP（通配）和端口（9080）转发到`0.0.0.0_9080`这个 outbound listener。

```
[root@master envoy]# istioctl pc listener productpage-v1-7f9d9c48c8-thvxq --port 15001 -ojson
[
    {
        "name": "virtualOutbound",
        "address": {
            "socketAddress": {
                "address": "0.0.0.0",
                "portValue": 15001
            }
        },
        "filterChains": [
            {
                "filterChainMatch": {
                    "prefixRanges": [
                        {
                            "addressPrefix": "10.244.1.25",
                            "prefixLen": 32
                        }
                    ]
                },
                "filters": [
                    {
                        "name": "envoy.tcp_proxy",
                        "typedConfig": {
                            "@type": "type.googleapis.com/envoy.config.filter.network.tcp_proxy.v2.TcpProxy",
                            "statPrefix": "BlackHoleCluster",
                            "cluster": "BlackHoleCluster"
                        }
                    }
                ]
            },
            {
                "filters": [
                    {
                        "name": "envoy.tcp_proxy",
                        "typedConfig": {
                            "@type": "type.googleapis.com/envoy.config.filter.network.tcp_proxy.v2.TcpProxy",
                            "statPrefix": "PassthroughCluster",
                            "cluster": "PassthroughCluster",
                            "accessLog": [
                                {
                                    "name": "envoy.file_access_log",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/envoy.config.accesslog.v2.FileAccessLog",
                                        "path": "/dev/stdout",
                                        "format": "[%START_TIME%] \"%REQ(:METHOD)% %REQ(X-ENVOY-ORIGINAL-PATH?:PATH)% %PROTOCOL%\" %RESPONSE_CODE% %RESPONSE_FLAGS% \"%DYNAMIC_METADATA(istio.mixer:status)%\" \"%UPSTREAM_TRANSPORT_FAILURE_REASON%\" %BYTES_RECEIVED% %BYTES_SENT% %DURATION% %RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)% \"%REQ(X-FORWARDED-FOR)%\" \"%REQ(USER-AGENT)%\" \"%REQ(X-REQUEST-ID)%\" \"%REQ(:AUTHORITY)%\" \"%UPSTREAM_HOST%\" %UPSTREAM_CLUSTER% %UPSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_REMOTE_ADDRESS% %REQUESTED_SERVER_NAME% %ROUTE_NAME%\n"
                                    }
                                }
                            ]
                        }
                    }
                ]
            }
        ],
        "useOriginalDst": true
    }
]
```

5.根据`0.0.0.0_9080`listener 的 `http_connection_manager`filter 配置，该请求采用 9080 route 进行分发。

```
[root@master envoy]# istioctl pc listener productpage-v1-7f9d9c48c8-thvxq --address 0.0.0.0 --port 9080 -ojson
[
    {
        "name": "0.0.0.0_9080",
        "address": {
            "socketAddress": {
                "address": "0.0.0.0",
                "portValue": 9080
            }
        },
        "filterChains": [
            {
                "filterChainMatch": {
                    "prefixRanges": [
                        {
                            "addressPrefix": "10.244.1.25",
                            "prefixLen": 32
                        }
                    ]
                },
                "filters": [
                    {
                        "name": "envoy.tcp_proxy",
                        "typedConfig": {
                            "@type": "type.googleapis.com/envoy.config.filter.network.tcp_proxy.v2.TcpProxy",
                            "statPrefix": "BlackHoleCluster",
                            "cluster": "BlackHoleCluster"
                        }
                    }
                ]
            },
            {
                "filters": [
                    {
                        "name": "envoy.http_connection_manager",
                        "typedConfig": {
                            "@type": "type.googleapis.com/envoy.config.filter.network.http_connection_manager.v2.HttpConnectionManager",
                            "statPrefix": "outbound_0.0.0.0_9080",
                            "rds": {
                                "configSource": {
                                    "ads": {}
                                },
                                "routeConfigName": "9080"
```

6.9080 这个 route 的配置中，host name 为`reviews:9080`的请求对应的 [cluster](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#cluster) 为`outbound|9080||reviews.default.svc.cluster.local`。

```
[
    {
        "name": "9080",
.....
            {
                "name": "reviews.default.svc.cluster.local:9080",
                "domains": [
                    "reviews.default.svc.cluster.local",
                    "reviews.default.svc.cluster.local:9080",
                    "reviews",
                    "reviews:9080",
                    "reviews.default.svc.cluster",
                    "reviews.default.svc.cluster:9080",
                    "reviews.default.svc",
                    "reviews.default.svc:9080",
                    "reviews.default",
                    "reviews.default:9080",
                    "10.96.167.251",
                    "10.96.167.251:9080"
                ],
                "routes": [
                    {
                        "name": "default",
                        "match": {
                            "prefix": "/"
                        },
                        "route": {
                            "cluster": "outbound|9080||reviews.default.svc.cluster.local",
                            "timeout": "0s",
                            "retryPolicy": {
                                "retryOn": "connect-failure,refused-stream,unavailable,cancelled,resource-exhausted,retriable-status-codes",
                                "numRetries": 2,
                                "retryHostPredicate": [
                                    {
                                        "name": "envoy.retry_host_predicates.previous_hosts"
                                    }
                                ],
                                "hostSelectionRetryMaxAttempts": "5",
                                "retriableStatusCodes": [
                                    503
                                ]
                            },
                            "maxGrpcTimeout": "0s"
                        },
                        "decorator": {
                            "operation": "reviews.default.svc.cluster.local:9080/*"
                        },
...
        ],
        "validateClusters": false
    }
]
```

7.`outbound|9080||reviews.default.svc.cluster.local cluster`为动态资源，通过 EDS 查询得到该[cluster](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#cluster)

中有3个 endpoint。

8.请求被转发到其中一个 endpoint`10.40.0.15`，即`reviews-v1`所在的[Pod](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pod)。

1. 然后该请求被 iptable 规则拦截，重定向到本地的 15006 端口。

1. 在 15006 端口上监听的 VirtualInbound listener 收到了该请求。

```

```

1. 根据匹配条件，请求被 VirtualInbound listener 内部配置的 Http connection manager filter 处理，该 filter 设置的路由配置为将其发送给`inbound|9080|http|reviews.default.svc.cluster.local`这个 inbound[cluster](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#cluster)。

```

```

1. `inbound|9080|http|reviews.default.svc.cluster.local cluster`配置的 host 为`127.0.0.1:9080`。

```

```

1. 请求被转发到`127.0.0.1:9080`，即 reviews 服务进行业务处理。



