下面我们以`Bookinfo`为例对 Istio 中的流量管理实现机制，以及控制平面和数据平面的交互进行进一步分析。

### Bookinfo 程序结构 {#bookinfo-程序结构}

下图显示了 Bookinfo 示例程序中各个组件的 IP 地址，端口和调用关系，以用于后续的分析。

![](https://hugo-picture.oss-cn-beijing.aliyuncs.com/images/fONobF.jpg)

### xDS 接口调试方法 {#xds-接口调试方法}

首先我们看看如何对 xDS 接口的相关数据进行查看和分析。Envoy v2 接口采用了`gRPC`，由于 gRPC 是基于二进制的 RPC 协议，无法像 V1 的`REST`接口一样通过 curl 和浏览器进行进行分析。但我们还是可以通过 Pilot 和 Envoy 的调试接口查看 xDS 接口的相关数据。

#### Pilot 调试方法 {#pilot-调试方法}

Pilot 在15014端口提供了下述[调试接口](https://github.com/istio/istio/tree/master/pilot/pkg/proxy/envoy/v2)下述方法查看 xDS 接口相关数据，同时15014也提供metrics接口

```
PILOT=istio-pilot.istio-system:15014

# What is sent to envoy
# Listeners and routes
curl  $PILOT/debug/adsz

# Endpoints

curl  $PILOT/debug/edsz


# Clusters
curl  $PILOT/debug/cdsz

#10.244.2.86为istio-pilot的地址
http://10.244.2.86:15014/metrics
```

#### Envoy 调试方法 {#envoy-调试方法}

Envoy 提供了管理接口，缺省为 localhost 的`15000`端口，可以获取 listener，cluster 以及完整的配置数据导出功能。

```
$ kubectl exec productpage-v1-54b8b9f55-bx2dq -c istio-proxy curl http://127.0.0.1:15000/help
admin commands are:
  /: Admin home page
  /certs: print certs on machine
  /clusters: upstream cluster status
  /config_dump: dump current Envoy configs (experimental)
  /contention: dump current Envoy mutex contention stats (if enabled)
  /cpuprofiler: enable/disable the CPU profiler
  /drain_listeners: drain listeners
  /healthcheck/fail: cause the server to fail health checks
  /healthcheck/ok: cause the server to pass health checks
  /heapprofiler: enable/disable the heap profiler
  /help: print out list of admin commands
  /hot_restart_version: print the hot restart compatibility version
  /listeners: print listener info
  /logging: query/change logging levels
  /memory: print current allocation/heap usage
  /quitquitquit: exit the server
  /ready: print server state, return 200 if LIVE, otherwise return 503
  /reset_counters: reset all counters to zero
  /runtime: print runtime values
  /runtime_modify: modify runtime values
  /server_info: print server version/status information
  /stats: print server stats
  /stats/prometheus: print server stats in prometheus format
  /stats/recentlookups: Show recent stat-name lookups
  /stats/recentlookups/clear: clear list of stat-name lookups and counter
  /stats/recentlookups/disable: disable recording of reset stat-name lookup names
  /stats/recentlookups/enable: enable recording of reset stat-name lookup names
```

进入 productpage pod 中的 istio-proxy\(Envoy\) container，可以看到有下面的监听端口：

* `9080`: productpage 进程对外提供的服务端口
* `15001`: Envoy 的出口监听器，监控出站流量\(outbound\)，iptable 会将 pod 的流量导入该端口中由 Envoy 进行处理----**针对pod出去访问的流量重定向**
* `15000`: Envoy 管理端口，该端口绑定在本地环回地址上，只能在 Pod 内访问。
* `15090: Envoy 监控指标端口，提供给Prometheus进行采集  http://x.x.x.x:15090/stats/prometheus`
* `15020`: pilot-agent端口，用于与pilot-discover通信。
* `15006`: Envoy 的入口监听器，监控入站流量\(inbound\)，iptable 会将访问 pod 的流量导入该端口中由 Envoy 进行处理----**针对访问pod业务的流量重定向。**

```
[root@master ~]# curl http://127.0.0.1:15090/stats/prometheus
# TYPE envoy_http_mixer_filter_total_remote_quota_calls counter
envoy_http_mixer_filter_total_remote_quota_calls{} 0
# TYPE envoy_tcp_mixer_filter_total_remote_call_other_errors counter
envoy_tcp_mixer_filter_total_remote_call_other_errors{} 0
# TYPE envoy_tcp_mixer_filter_total_quota_calls counter
envoy_tcp_mixer_filter_total_quota_calls{} 0
# TYPE envoy_tcp_mixer_filter_total_check_cache_misses counter
envoy_tcp_mixer_filter_total_check_cache_misses{} 0
# TYPE envoy_tcp_mixer_filter_total_remote_call_timeouts counter
envoy_tcp_mixer_filter_total_remote_call_timeouts{} 0
# TYPE envoy_http_mixer_filter_total_remote_report_send_errors counter
envoy_http_mixer_filter_total_remote_report_send_errors{} 0
# TYPE envoy_http_mixer_filter_total_remote_report_calls counter
envoy_http_mixer_filter_total_remote_report_calls{} 0
# TYPE envoy_tcp_mixer_filter_total_remote_report_successes

[root@master ~]# ss -ntulp
Netid State      Recv-Q Send-Q                        Local Address:Port                                       Peer Address:Port
tcp   LISTEN     0      128                                       *:9080                                                  *:*                   users:(("java",pid=27706,fd=180))
tcp   LISTEN     0      128                               127.0.0.1:15000                                                 *:*                   users:(("envoy",pid=27857,fd=14))
tcp   LISTEN     0      128                                       *:15001                                                 *:*                   users:(("envoy",pid=27857,fd=69))
tcp   LISTEN     0      128                                       *:15006                                                 *:*                   users:(("envoy",pid=27857,fd=70))
tcp   LISTEN     0      128                                       *:15090                                                 *:*                   users:(("envoy",pid=27857,fd=24))
tcp   LISTEN     0      50                                127.0.0.1:54550                                                 *:*                   users:(("java",pid=27706,fd=86))
tcp   LISTEN     0      128                                      :::15020                                                :::*                   users:(("pilot-agent",pid=27815,fd=3))
```

### Envoy 配置介绍

[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)是一个四层/七层代理，其架构非常灵活，采用了插件式的机制来实现各种功能，可以通过配置的方式对其功能进行定制。[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)提供了两种配置的方式：通过配置文件向[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)提供静态配置，或者通过 xDS 接口向[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)下发动态配置。在[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio)中同时采用了这两种方式对[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)的功能进行设置。本文假设读者对[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)已有基本的了解，如需要了解[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)的更多内容，请参考本书[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)章节部分的介绍。

#### Envoy 初始化配置文件

在[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio)中，[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)的大部分配置都来自于控制平面通过 xDS 接口下发的动态配置，包括网格中服务相关的[service](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#service)[cluster](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#cluster), listener, route 规则等。但[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)是如何知道 xDS server 的地址呢？这就是在[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)初始化配置文件中以静态资源的方式配置的。[Sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)容器中有一个[pilot](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pilot)-agent 进程，该进程根据启动参数生成[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)的初始配置文件，并采用该配置文件来启动[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)进程。

可以使用下面的命令将productpage[pod](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pod)中该文件导出来查看其中的内容：

```
kubectl exec -it productpage-v1-7f9d9c48c8-xxq6f -c istio-proxy cat /etc/istio/proxy/envoy-rev0.json > envoy-rev0.json
```

该初始化配置文件的结构如下所示：

```
{
    "node": {...},
    "stats_config": {...},
    "admin": {...},
    "dynamic_resources": {...},
    "static_resources": {...},
    "tracing": {...}
}
```

该配置文件中包含了下面的内容：

* node： 包含了[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)所在节点的相关信息，如节点的 id，节点所属的 Kubernetes 集群，节点的 IP 地址，等等。![](/image/Istio/envoy-node.png)

* admin：[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)的日志路径以及管理端口。

![](/image/Istio/envoy-admin.png)

* dynamic\_resources： 动态资源,即来自 xDS 服务器下发的配置。
* static\_resources： 静态资源，包括预置的一些 listener 和[cluster](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#cluster)，例如调用跟踪和指标统计使用到的 listener 和[cluster](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#cluster)。![](/image/Istio/envoy-static-config.png)

* [tracing](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#tracing)： 分布式调用追踪的相关配置。

![](/image/Istio/envoy-tracing.png)

#### Envoy 完整配置

从[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)初始化配置文件中，我们可以看出[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio)中[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)[sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)真正的配置实际上是由两部分组成的。[Pilot](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pilot)-agent 在启动[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)时将 xDS server 信息通过静态资源的方式配置到[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)的初始化配置文件中，[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)启动后再通过 xDS server 获取网格中的服务信息、路由规则等动态资源。

[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)完整配置的生成流程如下图所示：

![](/image/Istio/envoy-config-init.png)

```
/usr/local/bin/pilot-agent proxy sidecar --domain default.svc.cluster.local --configPath /etc/istio/proxy --binaryPath /usr/local/bin/envoy --serviceCluster reviews.default --drainDuration 45s --parentShutdownDuration 1m0s --discoveryAddress istio-pilot.istio-system:15010 --zipkinAddress zipkin.istio-system:9411 --proxyLogLevel=warning --proxyComponentLogLevel=misc:error --connectTimeout 10s --proxyAdminPort 15000 --concurrency 2 --controlPlaneAuthPolicy NONE --dnsRefreshRate 300s --statusPort 15020 --applicationPorts 9080 --trust-domain=cluster.local
```

1. Pilot-agent 根据启动参数生成 Envoy 的初始配置文件 envoy-rev0.json，该文件告诉 Envoy 从指定的 xDS server 中获取动态配置信息，并配置了 xDS server 的地址信息，即控制平面的 Pilot 服务器地址。
2. Pilot-agent 使用 envoy-rev0.json 启动 Envoy 进程。
3. Envoy 根据初始配置获得 Pilot 地址，采用 xDS 接口从 Pilot 获取到 listener，cluster，route 等动态配置信息。
4. Envoy 根据获取到的动态配置启动 Listener，并根据 listener 的配置，结合 route 和 cluster 对拦截到的流量进行处理。

可以看到，[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)中实际生效的配置是由初始化配置文件中的静态配置和从[Pilot](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pilot)获取的动态配置一起组成的。因此只对[envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)-rev0 .json 进行分析并不能看到网络中流量管理的全貌。那么有没有办法可以看到[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)中实际生效的完整配置呢？[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)提供了相应的管理接口，我们可以采用下面的命令导出 productpage-v1 服务[sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)的完整配置。

```
kubectl exec -it productpage-v1-7f9d9c48c8-xxq6f -c istio-proxy curl http://127.0.0.1:15000/config_dump > config_dump.json
```

该配置文件的内容如下：

![](/image/Istio/envoy-config-dump.png)

从导出的文件中可以看到[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)中主要由以下几部分内容组成：

* BootstrapConfigDump： 初始化配置，来自于初始化配置文件中配置的内容。
* ClustersConfigDump： 集群配置，包括对应于外部服务的 outbound [cluster](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#cluster)和 自身所在节点服务的 inbound [cluster](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#cluster)。
* ListenersConfigDump： 监听器配置，包括用于处理对外业务请求的 outbound listener，处理入向业务请求的 inbound listener，以及作为流量处理入口的 virtual listener。
* RoutesConfigDump： 路由配置，用于 HTTP 请求的路由处理。
* SecretsConfigDump： TLS 双向认证相关的配置，包括自身的证书以及用于验证请求方的 CA 根证书。

下面我们对该配置文件中和流量路由相关的配置一一进行详细分析。

#### Bootstrap {#bootstrap}

从名字可以看出这是[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)的初始化配置，打开该节点，可以看到其中的内容和[envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)-rev0.json 是一致的，这里不再赘述。 需要注意的是在 bootstrap 部分配置的一些内容也会被用于其他部分，例如 clusters 部分就包含了 bootstrap 中定义的一些静态[cluster](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#cluster)资源。

![](/image/Istio/envoy-bootstrap.png)

#### Clusters {#clusters}

这部分配置定义了[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)中所有的[cluster](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#cluster)，即服务集群，[cluster](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#cluster)中包含一个到多个 endpoint，每个 endpoint 都可以提供服务，[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)根据负载均衡算法将请求发送到这些 endpoint 中。

从配置文件结构中可以看到，在 productpage 的 clusters 配置中包含 **static\_clusters **和 **dynamic\_active\_clusters **两部分，其中 **static\_clusters **是来自于[envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)-rev0.json 的初始化配置中的 **prometheus\_stats、xDS server、zipkin server **信息。dynamic\_active\_clusters 是[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)通过 xDS 接口从[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio)控制平面获取的服务信息。

其中 dynamic [cluster](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#cluster)又分为以下几类：

##### Outbound Cluster {#outbound-cluster}

这部分的[cluster](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#cluster)占了绝大多数，该类[cluster](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#cluster)对应于[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)所在节点的外部服务。以 reviews 为例，对于 productpage 来说,reviews 是一个外部服务，因此其[cluster](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#cluster)名称中包含 outbound 字样。

从 reviews 服务对应的[cluster](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#cluster)配置中可以看到，其类型为 EDS，即表示该[cluster](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#cluster)的 endpoint 来自于动态发现，动态发现中 eds\_config 则指向了ads，最终指向 static resource 中配置的 xds-grpc[cluster](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#cluster)，即[Pilot](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pilot)的地址。

```
    {
        "name": "outbound|9080||reviews.default.svc.cluster.local",
        "type": "EDS",
        "edsClusterConfig": {
            "edsConfig": {
                "ads": {}
            },
            "serviceName": "outbound|9080||reviews.default.svc.cluster.local"
        },
        "connectTimeout": "1s",
        "circuitBreakers": {
            "thresholds": [
                {
                    "maxConnections": 4294967295,
                    "maxPendingRequests": 4294967295,
                    "maxRequests": 4294967295,
                    "maxRetries": 4294967295
                }
            ]
        },
        "metadata": {
            "filterMetadata": {
                "istio": {
                    "config": "/apis/networking/v1alpha3/namespaces/default/destination-rule/reviews"
                }
            }
        }
    }
```

可以通过Pilot的调试接口来获取该cluster的endpoint：

```
curl http://10.244.2.14:15014/debug/edsz > pilot_eds_dump
```

从导出的文件内容可以看到，reviews cluster配置类3个endpoint地址，是reviews的pod ip。

```
{
  "clusterName": "outbound|9080||reviews.default.svc.cluster.local",
  "endpoints": [
    {
      "lbEndpoints": [
        {
          "endpoint": {
            "address": {
              "socketAddress": {
                "address": "10.244.0.63",
                "portValue": 9080
              }
            }
          },
          "metadata": {
            "filterMetadata": {
              "envoy.transport_socket_match": {
                  "tlsMode": "istio"
                },
              "istio": {
                  "uid": "kubernetes://reviews-v3-57fcb844b7-qd44m.default"
                }
            }
          },
          "loadBalancingWeight": 1
        },
        {
          "endpoint": {
            "address": {
              "socketAddress": {
                "address": "10.244.1.24",
                "portValue": 9080
              }
            }
          },
          "metadata": {
            "filterMetadata": {
              "envoy.transport_socket_match": {
                  "tlsMode": "istio"
                },
              "istio": {
                  "uid": "kubernetes://reviews-v1-d5b6b667f-tkfhw.default"
                }
            }
          },
          "loadBalancingWeight": 1
        },
        {
          "endpoint": {
            "address": {
              "socketAddress": {
                "address": "10.244.2.24",
                "portValue": 9080
              }
            }
          },
          "metadata": {
            "filterMetadata": {
              "envoy.transport_socket_match": {
                  "tlsMode": "istio"
                },
              "istio": {
                  "uid": "kubernetes://reviews-v2-784495d9bc-9lpkb.default"
                }
            }
          },
          "loadBalancingWeight": 1
        }
      ],
      "loadBalancingWeight": 3
    }
  ]
}]
```

##### Inbound Cluster

对于 Envoy 来说，inbound cluster 对应于入向请求的 upstream 集群， 即 Envoy 自身所在节点的服务。对于 productpage Pod 上的 Envoy，其对应的 Inbound cluster 只有一个，即 productpage。该 cluster 对应的 host 为127.0.0.1，即回环地址上 productpage 的监听端口。由于 iptable 规则中排除了127.0.0.1，入站请求通过该 Inbound cluster 处理后将跳过 Envoy，直接发送给 productpage 进程处理。

```
[root@master envoy]# istioctl pc cluster productpage-v1-7f9d9c48c8-thvxq --direction inbound --fqdn productpage.default.svc.cluster.local -ojson
[
    {
        "name": "inbound|9080|http|productpage.default.svc.cluster.local",
        "type": "STATIC",
        "connectTimeout": "1s",
        "loadAssignment": {
            "clusterName": "inbound|9080|http|productpage.default.svc.cluster.local",
            "endpoints": [
                {
                    "lbEndpoints": [
                        {
                            "endpoint": {
                                "address": {
                                    "socketAddress": {
                                        "address": "127.0.0.1",
                                        "portValue": 9080
                                    }
                                }
                            }
                        }
                    ]
                }
            ]
        },
        "circuitBreakers": {
            "thresholds": [
                {
                    "maxConnections": 4294967295,
                    "maxPendingRequests": 4294967295,
                    "maxRequests": 4294967295,
                    "maxRetries": 4294967295
                }
            ]
        }
    }
]
```

#### BlackHoleCluster

这是一个特殊的 cluster ，其中并没有配置后端处理请求的 host。如其名字所表明的一样，请求进入该 cluster 后如同进入了一个黑洞，将被丢弃掉，而不是发向一个 upstream host。

```
{
     "version_info": "2020-03-11T08:13:39Z/22",
     "cluster": {
      "@type": "type.googleapis.com/envoy.api.v2.Cluster",
      "name": "BlackHoleCluster",
      "type": "STATIC",
      "connect_timeout": "1s",
      "filters": []
     },
     "last_updated": "2020-03-11T08:14:04.665Z"
```

#### PassthroughCluster

该 cluster 的 type 被设置为 ORIGINAL\_DST 类型， 表明任何发向该 cluster 的请求都会被直接发送到其请求中的原始目地的，Envoy 不会对请求进行重新路由。

```
{
 "version_info": "2020-03-11T08:13:39Z/22",
 "cluster": {
  "@type": "type.googleapis.com/envoy.api.v2.Cluster",
  "name": "PassthroughCluster",
  "type": "ORIGINAL_DST",
  "connect_timeout": "1s",
  "lb_policy": "CLUSTER_PROVIDED",
  "circuit_breakers": {
   "thresholds": []
  },
  "filters": []
 },
 "last_updated": "2020-03-11T08:14:04.666Z"
}
```

#### Listeners

Envoy 采用 listener 来接收并处理 downstream 发过来的请求，listener 采用了插件式的架构，可以通过配置不同的 filter 在 listener 中插入不同的处理逻辑。

Listener 可以绑定到 IP Socket 或者 Unix Domain Socket 上，以接收来自客户端的请求；也可以不绑定，而是接收从其他 listener 转发来的数据。Istio 利用了 Envoy listener 的这一特点，通过 VirtualOutboundListener 在一个端口接收所有出向请求，然后再按照请求的端口分别转发给不同的 listener 分别处理。

##### VirtualOutbound Listener

Istio 在 Envoy 中配置了一个在 15001 端口监听的虚拟入口监听器。Iptable 规则将 Envoy 所在 pod 的对外请求拦截后发向本地的 15001 端口，该监听器接收后并不进行业务处理，而是根据请求的目的端口分发给其他监听器处理。这就是该监听器取名为 "virtual”（虚拟）监听器的原因。

Envoy 是如何做到按请求的目的端口进行分发的呢？ 从下面 VirtualOutbound listener 的配置中可以看到 use_original\_dest 属性被设置为 true, 这表示该监听器在接收到来自 downstream 的请求后，会将请求转交给匹配该请求原目的地址的 listener（即名字格式为 0.0.0.0_ 请求目的端口 的 listener）进行处理。

如果在 Enovy 的配置中找不到匹配请求目的端口的 listener，则将会根据 Istio 的 outboundTrafficPolicy 全局配置选项进行处理。存在两种情况：

* 如果 outboundTrafficPolicy 设置为 ALLOW\_ANY：这表明网格允许发向任何外部服务的请求，无论该服务是否在 Pilot 的服务注册表中。在该策略下，Pilot 将会在下发给 Envoy 的 VirtualOutbound listener 加入一个 upstream cluster 为 PassthroughCluster 的 TCP proxy filter，找不到匹配端口 listener 的请求会被该 TCP proxy filter 处理，请求将会被发送到其 IP 头中的原始目的地地址。
* 如果 outboundTrafficPolicy 设置为 REGISTRY\_ONLY：只允许发向 Pilot 服务注册表中存在的服务的对外请求。在该策略下，Pilot 将会在下发给 Enovy 的 VirtualOutbound listener 加入一个 upstream cluster 为 BlackHoleCluster 的 TCP proxy filter，找不到匹配端口 listener 的请求会被该 TCP proxy filter 处理，由于 BlackHoleCluster 中没有配置 upstteam host，请求实际上会被丢弃。

下图是 bookinfo 例子中 productpage 服务中 Enovy Proxy 的 Virutal Outbound Listener 配置。由于 outboundTrafficPolicy 的默认配置为`ALLOW_ANY`，因此 listener 的 filterchain 中第二个 filter chain 中是一个 upstream[cluster](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#cluster)为 PassthroughCluster 的 TCP proxy filter。注意该 filter 没有 filter\_chain\_match 匹配条件，因此如果进入该 listener 的请求在配置中找不到匹配其目的端口的 listener，就会缺省进入该 filter 进行处理。

filterchain 中的第一个 filter chain 中是一个 upstream [cluster](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#cluster)为 BlackHoleCluster 的 TCP proxy filter，该 filter 设置了 filter\_chain\_match 匹配条件，只有发向 10.244.1.25 这个 IP 的出向请求才会进入该 filter 处理。10.244.1.25 是 productpage 服务自身的IP地址。该 filter 的目的是为了防止服务向自己发送请求可能导致的死循环。

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
                        "name": "mixer",
                        "typedConfig": {
                            "@type": "type.googleapis.com/istio.mixer.v1.config.client.TcpClientConfig",
                            "transport": {
                                "networkFailPolicy": {
                                    "policy": "FAIL_CLOSE",
                                    "baseRetryWait": "0.080s",
                                    "maxRetryWait": "1s"
                                },
                                "checkCluster": "outbound|9091||istio-policy.istio-system.svc.cluster.local",
                                "reportCluster": "outbound|9091||istio-telemetry.istio-system.svc.cluster.local",
                                "reportBatchMaxEntries": 100,
                                "reportBatchMaxTime": "1s"
                            },
                            "mixerAttributes": {
                                "attributes": {
                                    "context.proxy_version": {
                                        "stringValue": "1.4.10"
                                    },
                                    "context.reporter.kind": {
                                        "stringValue": "outbound"
                                    },
                                    "context.reporter.uid": {
                                        "stringValue": "kubernetes://productpage-v1-7f9d9c48c8-thvxq.default"
                                    },
                                    "destination.service.host": {
                                        "stringValue": "PassthroughCluster"
                                    },
                                    "destination.service.name": {
                                        "stringValue": "PassthroughCluster"
                                    },
                                    "source.namespace": {
                                        "stringValue": "default"
                                    },
                                    "source.uid": {
                                        "stringValue": "kubernetes://productpage-v1-7f9d9c48c8-thvxq.default"
                                    }
                                }
                            },
                            "disableCheckCalls": true
                        }
                    },
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

##### Outbound Listener {#outbound-listener}

[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)为网格中的外部服务按端口创建多个 Outbound listener，以用于处理出向请求。bookinfo 示例程序中使用了9080作为微服务的业务端口，因此我们这里主要分析9080这个业务端口的 listener。和其他所有 Outbound listener 一样，该 listener 配置了"bind\_to\_port”: false 属性，因此该 listener 没有被绑定到 tcp 端口上，其接收到的所有请求都转发自15001端口的 Virtual listener。

该listener 的名称为0.0.0.0\_9080,因此会匹配发向任意 IP 的9080端口的请求，bookinfo 程序中的 productpage,revirews,ratings,details 四个服务都使用了9080端口，那么[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)如何区别处理这四个服务呢？

> 备注： 根据业务逻辑，实际上 productpage 并不会调用 ratings 服务，但[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio)并不知道各个业务之间会如何调用，因此将所有的服务信息都下发到了[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)中。这样做对[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)的内存占用和效率有一定影响，如果希望去掉[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)配置中的无用数据，可以通过[sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)[CRD](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#crd)对[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)的 ingress 和 egress[service](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#service)配置进行调整。

首先，iptables 拦截到 productpage 向外发出的 HTTP 请求，并转发到同一[pod](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pod)中的[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)[sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)监听的 15001 virtualOutbound listener 进行处理。[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)根据目的端口匹配到`0.0.0.0_9080`这个 Outbound listener，并转交给该 listener。

如下面的配置所示，当`0.0.0.0_9080`接收到出向请求后，并不会直接发送到一个 downstream[cluster](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#cluster)，而是配置了一个路由规则 9080，在该路由规则中会根据不同的请求目的地对请求进行处理。

```
[root@master envoy]# istioctl pc listener productpage-v1-7f9d9c48c8-thvxq --port 9080
ADDRESS         PORT     TYPE
10.244.1.25     9080     HTTP
0.0.0.0         9080     TCP
[root@master envoy]#
[root@master envoy]# istioctl pc listener productpage-v1-7f9d9c48c8-thvxq --port 9080 --type tcp  -ojson
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
                            },

[root@master envoy]# istioctl pc route productpage-v1-7f9d9c48c8-thvxq --name 9080
NOTE: This output only contains routes loaded via RDS.
NAME     VIRTUAL HOSTS
9080     5
```

##### VirtualInbound Listener {#virtualinbound-listener}

在较早的版本中，[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio)采用同一个 VirtualListener 在端口 15001 上同时处理入向和出向的请求。该方案存在一些潜在的问题，例如可能会导致出现死循环，参见[这个 PR](https://github.com/istio/istio/pull/15713)。在 1.4 版本之后，[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio)为[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)单独创建了 一个 VirtualInboundListener，在 15006 端口监听入向请求，原来的 15001 端口只用于处理出向请求。

另外一个变化是当 VirtualInboundListener 接收到请求后，将直接在 VirtualInboundListener 采用一系列 filterChain 对入向请求进行处理，而不是像 VirtualOutboundListener 一样分发给其它独立的 listener 进行处理。

这样修改后，[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)配置中入向和出向的请求处理流程被完全拆分开，请求处理流程更为清晰，可以避免由于配置导致的一些潜在错误。

#### Routes {#routes}

这部分配置是[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)的 HTTP 路由规则。在前面 listener 的分析中，我们看到 Outbound listener 是以端口为最小粒度来进行处理的，而不同的服务可能采用了相同的端口，因此需要通过 Route 来进一步对发向同一目的端口的不同服务的请求进行区分和处理。[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio)在下发给[sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)的缺省路由规则中为每个端口设置了一个路由规则，然后再根据 host 来对请求进行路由分发。

```
[root@master envoy]# istioctl pc route productpage-v1-7f9d9c48c8-thvxq --name 9080
NOTE: This output only contains routes loaded via RDS.
NAME     VIRTUAL HOSTS
9080     5
```

### Istio 中的 sidecar 注入

[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio)中提供了以下两种[sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)注入方式：

* 使用`istioctl`手动注入。
* 基于 Kubernetes 的[突变 webhook 入驻控制器（mutating webhook addmission controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)的自动[sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)注入方式。

不论是手动注入还是自动注入，[sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)的注入过程都需要遵循如下步骤：

1. Kubernetes 需要了解待注入的[sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)所连接的[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio)集群及其配置；
2. Kubernetes 需要了解待注入的[sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)容器本身的配置，如镜像地址、启动参数等；
3. Kubernetes 根据[sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)注入模板和以上配置填充[sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)的配置参数，将以上配置注入到应用容器的一侧；

使用下面的命令可以手动注入[sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)。

```
istioctl kube-inject -f ${YAML_FILE}|kuebectl apply -f -
```

该命令会使用[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio)内置的[sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)配置来注入，下面使用[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio)详细配置请参考[Istio 官网](https://istio.io/docs/setup/additional-setup/sidecar-injection/#manual-sidecar-injection)。

注入完成后您将看到[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio)为原有[pod](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pod)template 注入了`initContainer`及[sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)proxy相关的配置。

#### Init 容器

Init 容器是一种专用容器，它在应用程序容器启动之前运行，用来包含一些应用镜像中不存在的实用工具或安装脚本。

一个[Pod](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pod)中可以指定多个 Init 容器，如果指定了多个，那么 Init 容器将会按顺序依次运行。只有当前面的 Init 容器必须运行成功后，才可以运行下一个 Init 容器。当所有的 Init 容器运行完成后，Kubernetes 才初始化[Pod](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pod)和运行应用容器。

Init 容器使用 Linux Namespace，所以相对应用程序容器来说具有不同的文件系统视图。因此，它们能够具有访问 Secret 的权限，而应用程序容器则不能。

在[Pod](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pod)启动过程中，Init 容器会按顺序在网络和数据卷初始化之后启动。每个容器必须在下一个容器启动之前成功退出。如果由于运行时或失败退出，将导致容器启动失败，它会根据[Pod](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pod)的`restartPolicy`指定的策略进行重试。然而，如果[Pod](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pod)的`restartPolicy`设置为 Always，Init 容器失败时会使用`RestartPolicy`策略。

在所有的 Init 容器没有成功之前，[Pod](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pod)将不会变成`Ready`状态。Init 容器的端口将不会在[Service](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#service)中进行聚集。 正在初始化中的[Pod](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pod)处于`Pending`状态，但应该会将`Initializing`状态设置为 true。Init 容器运行完成以后就会自动终止。

关于 Init 容器的详细信息请参考[Init 容器 - Kubernetes 中文指南/云原生应用架构实践手册](https://jimmysong.io/kubernetes-handbook/concepts/init-containers.html)。

#### Sidecar 注入示例分析

以[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio)官方提供的`bookinfo`中`productpage`的 YAML 为例，关于`bookinfo`应用的详细 YAML 配置请参考[bookinfo.yaml](https://github.com/istio/istio/blob/master/samples/bookinfo/platform/kube/bookinfo.yaml)。

下文将从以下几个方面讲解：

* [Sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)
  容器的注入
* iptables 规则的创建
* 路由的详细过程

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: productpage
    version: v1
  name: productpage-v1
  namespace: default
  resourceVersion: "921447"
  selfLink: /apis/apps/v1/namespaces/default/deployments/productpage-v1
  uid: 5fd33e5d-cd93-4449-8d9c-7b1ada675b7b
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: productpage
      version: v1
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: productpage
        version: v1
    spec:
      containers:
      - image: docker.io/istio/examples-bookinfo-productpage-v1:1.15.1
        imagePullPolicy: IfNotPresent
        name: productpage
        ports:
        - containerPort: 9080
          protocol: TCP
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      serviceAccount: bookinfo-productpage
      serviceAccountName: bookinfo-productpage
      terminationGracePeriodSeconds: 30
```

再查看下`productpage`容器的[Dockerfile](https://github.com/istio/istio/blob/master/samples/bookinfo/src/productpage/Dockerfile)。

```
FROM python:3.7.4-slim

COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

COPY test-requirements.txt ./
RUN pip install --no-cache-dir -r test-requirements.txt

COPY productpage.py /opt/microservices/
COPY tests/unit/* /opt/microservices/
COPY templates /opt/microservices/templates
COPY static /opt/microservices/static
COPY requirements.txt /opt/microservices/

ARG flood_factor
ENV FLOOD_FACTOR ${flood_factor:-0}

EXPOSE 9080
WORKDIR /opt/microservices
RUN python -m unittest discover

USER 1

CMD ["python", "productpage.py", "9080"]
```

我们看到`Dockerfile`中没有配置`ENTRYPOINT`，所以`CMD`的配置`python productpage.py 9080`将作为默认的`ENTRYPOINT`，记住这一点，再看下注入[sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)之后的配置。

```
$ istioctl kube-inject -f samples/bookinfo/platform/kube/bookinfo.yaml
```

我们只截取其中与`productpage`相关的`Deployment`配置中的部分 YAML 配置。

```
  containers:
  - image: docker.io/istio/examples-bookinfo-productpage-v1:1.15.1  # 业务容器
    imagePullPolicy: IfNotPresent
    name: productpage
    ports:
    - containerPort: 9080
      protocol: TCP
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: bookinfo-productpage-token-mmc9m
      readOnly: true
  - args:
    - proxy
    - sidecar
    - --domain
    - $(POD_NAMESPACE).svc.cluster.local
    - --configPath
    - /etc/istio/proxy
    - --binaryPath
    - /usr/local/bin/envoy
    - --serviceCluster
    - productpage.$(POD_NAMESPACE)
    - --drainDuration
    - 45s
    - --parentShutdownDuration
    - 1m0s
    - --discoveryAddress
    - istio-pilot.istio-system:15010
    - --zipkinAddress
    - zipkin.istio-system:9411
    - --proxyLogLevel=warning
    - --proxyComponentLogLevel=misc:error
    - --connectTimeout
    - 10s
    - --proxyAdminPort
    - "15000"
    - --concurrency
    - "2"
    - --controlPlaneAuthPolicy
    - NONE
    - --dnsRefreshRate
    - 300s
    - --statusPort
    - "15020"
    - --applicationPorts
    - "9080"
    - --trust-domain=cluster.local
    image: docker.io/istio/proxyv2:1.4.10  # sidecar
    imagePullPolicy: IfNotPresent
    name: istio-proxy
    ports:
    - containerPort: 15090
      name: http-envoy-prom
      protocol: TCP
    readinessProbe:
      failureThreshold: 30
      httpGet:
        path: /healthz/ready
        port: 15020
        scheme: HTTP
      initialDelaySeconds: 1
      periodSeconds: 2
      successThreshold: 1
      timeoutSeconds: 1
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  initContainers:
  - command:
    - istio-iptables
    - -p
    - "15001"
    - -z
    - "15006"
    - -u
    - "1337"
    - -m
    - REDIRECT
    - -i
    - '*'
    - -x
    - ""
    - -b
    - '*'
    - -d
    - "15020"
    image: docker.io/istio/proxyv2:1.4.10  # init容器
    imagePullPolicy: IfNotPresent
    name: istio-init
```

[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio)给应用[Pod](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pod)注入的配置主要包括：

* Init 容器`istio-init`：用于[pod](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pod)中设置 iptables 端口转发
* [Sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)容器`istio-proxy`：运行[sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)代理，如[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)或[MOSN](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#mosn)

接下来将分别解析下这两个容器

### Init 容器解析

[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio)在[pod](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pod)中注入的 Init 容器名为`istio-init`，我们在上面[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio)注入完成后的 YAML 文件中看到了该容器的启动命令是：

```
istio-iptables -p 15001 -z 15006 -u 1337 -m REDIRECT -i '*' -x "" -b '*' -d 15020
```

我们再检查下该容器的[Dockerfile](https://github.com/istio/istio/blob/master/pilot/docker/Dockerfile.proxyv2)看看`ENTRYPOINT`是怎么确定启动时执行的命令。

```
# 前面的内容省略
# The pilot-agent will bootstrap Envoy.
ENTRYPOINT ["/usr/local/bin/pilot-agent"]
```

我们看到`istio-init`容器的入口是`/usr/local/bin/istio-iptables`命令行，该命令行工具的代码的位置在[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio)源码仓库的[tools/istio-iptables](https://github.com/istio/istio/tree/master/tools/istio-iptables)目录。

注意：在[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio)1.1 版本时还是使用`isito-iptables.sh`命令行来操作 IPtables。

#### Init 容器启动入口

Init 容器的启动入口是`istio-iptables`命令行，该命令行工具的用法如下：

```
$ istio-iptables [flags]
  -p: 指定重定向所有 TCP 流量的 sidecar 端口（默认为 $ENVOY_PORT = 15001）
  -m: 指定入站连接重定向到 sidecar 的模式，“REDIRECT” 或 “TPROXY”（默认为 $ISTIO_INBOUND_INTERCEPTION_MODE)
  -b: 逗号分隔的入站端口列表，其流量将重定向到 Envoy（可选）。使用通配符 “*” 表示重定向所有端口。为空时表示禁用所有入站重定向（默认为 $ISTIO_INBOUND_PORTS）
  -d: 指定要从重定向到 sidecar 中排除的入站端口列表（可选），以逗号格式分隔。使用通配符“*” 表示重定向所有入站流量（默认为 $ISTIO_LOCAL_EXCLUDE_PORTS）
  -o：逗号分隔的出站端口列表，不包括重定向到 Envoy 的端口。
  -i: 指定重定向到 sidecar 的 IP 地址范围（可选），以逗号分隔的 CIDR 格式列表。使用通配符 “*” 表示重定向所有出站流量。空列表将禁用所有出站重定向（默认为 $ISTIO_SERVICE_CIDR）
  -x: 指定将从重定向中排除的 IP 地址范围，以逗号分隔的 CIDR 格式列表。使用通配符 “*” 表示重定向所有出站流量（默认为 $ISTIO_SERVICE_EXCLUDE_CIDR）。
  -k：逗号分隔的虚拟接口列表，其入站流量（来自虚拟机的）将被视为出站流量。
  -g：指定不应用重定向的用户的 GID。(默认值与 -u param 相同)
  -u：指定不应用重定向的用户的 UID。通常情况下，这是代理容器的 UID（默认值是 1337，即 istio-proxy 的 UID）。
  -z: 所有进入 pod/VM 的 TCP 流量应被重定向到的端口（默认 $INBOUND_CAPTURE_PORT = 15006）。
```

以上传入的参数都会重新组装成[`iptables`](https://wangchujiang.com/linux-command/c/iptables.html)规则，关于该命令的详细用法请访问[tools/istio-iptables/pkg/cmd/root.go](https://github.com/istio/istio/blob/master/tools/istio-iptables/pkg/cmd/root.go)。

该容器存在的意义就是让[sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)代理可以拦截所有的进出[pod](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pod)的流量，15090 端口（[Mixer](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#mixer)使用）和 15092 端口（Ingress Gateway）除外的所有入站（inbound）流量重定向到 15006 端口（[sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)），再拦截应用容器的出站（outbound）流量经过[sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)处理（通过 15001 端口监听）后再出站。关于[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio)中端口用途请参考[Istio 官方文档](https://istio.io/zh/docs/ops/deployment/requirements/)。

#### **命令解析**

这条启动命令的作用是：

* 将应用容器的所有流量都转发到[sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)的 15006 端口。
* 使用`istio-proxy`用户身份运行， UID 为 1337，即 [sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar) 所处的用户空间，这也是`istio-proxy`容器默认使用的用户，见 YAML 配置中的`runAsUser`字段。
* 使用默认的`REDIRECT`模式来重定向流量。
* 将所有出站流量都重定向到 [sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar) 代理（通过 15001 端口）。

因为 Init 容器初始化完毕后就会自动终止，因为我们无法登陆到容器中查看 iptables 信息，但是 Init 容器初始化结果会保留到应用容器和[sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)容器中。

为了查看 iptables 配置，我们需要登陆到[sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)容器中使用 root 用户来查看，因为`kubectl`无法使用特权模式来远程操作 docker 容器，所以我们需要登陆到`productpage`[pod](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pod)所在的主机上使用`docker`命令登陆容器中查看。

查看 iptables 配置，列出 NAT（网络地址转换）表的所有规则，因为在 Init 容器启动的时候选择给`istio-iptables`传递的参数中指定将入站流量重定向到[sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar)的模式为`REDIRECT`，因此在 iptables 中将只有 NAT 表的规格配置，如果选择`TPROXY`还会有`mangle`表配置。`iptables`命令的详细用法请参考[iptables](https://wangchujiang.com/linux-command/c/iptables.html)，规则配置请参考[iptables 规则配置](http://www.zsythink.net/archives/1517)。

我们仅查看与`productpage`有关的 iptables 规则如下。

### istio-proxy内部iptables规则

```
[root@master ~]# iptables -L -t nat
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination
ISTIO_INBOUND  tcp  --  anywhere             anywhere

Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
ISTIO_OUTPUT  tcp  --  anywhere             anywhere

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination

Chain ISTIO_INBOUND (1 references)
target     prot opt source               destination
RETURN     tcp  --  anywhere             anywhere             tcp dpt:ssh
RETURN     tcp  --  anywhere             anywhere             tcp dpt:15020
ISTIO_IN_REDIRECT  tcp  --  anywhere             anywhere

Chain ISTIO_IN_REDIRECT (2 references)
target     prot opt source               destination
REDIRECT   tcp  --  anywhere             anywhere             redir ports 15006

Chain ISTIO_OUTPUT (1 references)
target     prot opt source               destination
RETURN     all  --  127.0.0.6            anywhere
ISTIO_IN_REDIRECT  all  --  anywhere            !localhost
RETURN     all  --  anywhere             anywhere             owner UID match 1337
RETURN     all  --  anywhere             anywhere             owner GID match 1337
RETURN     all  --  anywhere             localhost
ISTIO_REDIRECT  all  --  anywhere             anywhere

Chain ISTIO_REDIRECT (1 references)
target     prot opt source               destination
REDIRECT   tcp  --  anywhere             anywhere             redir ports 15001


[root@master ~]# iptables-save
# Generated by iptables-save v1.4.21 on Thu Aug 27 17:12:34 2020
*mangle
:PREROUTING ACCEPT [684109:69798245]
:INPUT ACCEPT [684109:69798245]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [683815:79235938]
:POSTROUTING ACCEPT [683815:79235938]
COMMIT
# Completed on Thu Aug 27 17:12:34 2020
# Generated by iptables-save v1.4.21 on Thu Aug 27 17:12:34 2020
*nat
:PREROUTING ACCEPT [2522:151320]
:INPUT ACCEPT [2523:151380]
:OUTPUT ACCEPT [73537:4425502]
:POSTROUTING ACCEPT [73537:4425502]
:ISTIO_INBOUND - [0:0]
:ISTIO_IN_REDIRECT - [0:0]
:ISTIO_OUTPUT - [0:0]
:ISTIO_REDIRECT - [0:0]
-A PREROUTING -p tcp -j ISTIO_INBOUND        # PREROUTING全部转发到INBOUND,PREROUTING发生在流入的数据包进入路由表之前
-A OUTPUT -p tcp -j ISTIO_OUTPUT                          # 由本机产生的数据向外转发的
-A ISTIO_INBOUND -p tcp -m tcp --dport 22 -j RETURN       # 22 15020的不转发到ISTIO_REDIRECT 
-A ISTIO_INBOUND -p tcp -m tcp --dport 15020 -j RETURN
-A ISTIO_INBOUND -p tcp -j ISTIO_IN_REDIRECT              # 剩余的进来的流量都转发到ISTIO_IN_REDIRECT
-A ISTIO_IN_REDIRECT -p tcp -j REDIRECT --to-ports 15006  # 转发到15006
-A ISTIO_OUTPUT -s 127.0.0.6/32 -o lo -j RETURN           # 127.0.0.6是InboundPassthroughBindIpv4，代表原地址是passthrough的流量都直接跳过,不劫持
-A ISTIO_OUTPUT ! -d 127.0.0.1/32 -o lo -j ISTIO_IN_REDIRECT #lo网卡出流量，目标地址不是localhost的，且为同用户的流量进入ISTIO_IN_REDIRECT
-A ISTIO_OUTPUT -m owner --uid-owner 1337 -j RETURN       # 剩下的同用户的都跳过
-A ISTIO_OUTPUT -m owner --gid-owner 1337 -j RETURN       # 剩下的同用户的都跳过
-A ISTIO_OUTPUT -d 127.0.0.1/32 -j RETURN                 # 剩余的目标地址为127的不劫持
-A ISTIO_OUTPUT -j ISTIO_REDIRECT                         # 剩下的都进入 ISTIO_REDIRECT
-A ISTIO_REDIRECT -p tcp -j REDIRECT --to-ports 15001     # 其他的转达到15001 outbond
COMMIT
# Completed on Thu Aug 27 17:12:34 2020
# Generated by iptables-save v1.4.21 on Thu Aug 27 17:12:34 2020
*filter
:INPUT ACCEPT [684109:69798245]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [683815:79235938]
COMMIT
# Completed on Thu Aug 27 17:12:34 2020
```

> ISTIO\_INBOUND 链：所有进入Pod但非指定端口\(如22\)的流量全部重定向至15006端口\(envoy入口流量端口\)进行拦截处理。
>
> ISTIO\_OUTPUT 链：所有流出Pod由 istio-proxy 用户空间发出且目的地不为localhost的流量全部重定向至15001端口（envoy出口流量端口），其他流量全部直接放行至下一个POSTROUTING链，不用被envoy拦截处理。

![](/image/Istio/istio-sidecar.png)

1. productpage 服务对reviews 服务发送 TCP 连接请求
2. 请求进入reviews服务所在Pod内核空间，被netfilter拦截入口流量，经过PREROUTING链然后转发至ISTIO\_INBOUND链
3. 在被ISTIO\_INBOUND链被这个规则-A ISTIO\_INBOUND -p tcp -j ISTIO\_IN\_REDIRECT拦截再次转发至ISTIO\_IN\_REDIRECT链
4. ISTIO\_IN\_REDIRECT链直接重定向至 envoy监听的 15006 入口流量端口
5. 在 envoy 内部经过一系列入口流量治理动作后，发出TCP连接请求 reviews 服务，这一步对envoy来说属于出口流量，会被netfilter拦截转发至出口流量OUTPUT链
6. OUTPUT链转发流量至ISTIO\_OUTPUT链
7. 目的地为localhost，不能匹配到转发规则链，直接RETURN到下一个链，即POSTROUTING链
8. sidecar发出的请求到达reviews服务9080端口
9. reviews服务处理完业务逻辑后响应sidecar，这一步对reviews服务来说属于出口流量，再次被netfilter拦截转发至出口流量OUTPUT链
10. OUTPUT链转发流量至ISTIO\_OUTPUT链
11. 发送非localhost请求且为istio-proxy用户空间的流量被转发至ISTIO\_REDIRECT链
12. ISTIO\_REDIRECT链直接重定向至 envoy监听的 15001 出口流量端口
13. 在 envoy 内部经过一系列出口流量治理动作后继续发送响应数据，响应时又会被netfilter拦截转发至出口流量OUTPUT链
14. OUTPUT链转发流量至ISTIO\_OUTPUT链
15. 流量直接RETURN到下一个链，即POSTROUTING链



