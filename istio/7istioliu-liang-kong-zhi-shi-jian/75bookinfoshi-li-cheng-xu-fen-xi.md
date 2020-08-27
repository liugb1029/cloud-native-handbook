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
* `15001`: Envoy 的入口监听器，iptable 会将 pod 的流量导入该端口中由 Envoy 进行处理
* `15000`: Envoy 管理端口，该端口绑定在本地环回地址上，只能在 Pod 内访问。
* `15090: Envoy 监控指标端口，提供给Prometheus进行采集  http://x.x.x.x:15090/stats/prometheus`
* `15020`: pilot-agent端口，用于与pilot-discover通信。

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



