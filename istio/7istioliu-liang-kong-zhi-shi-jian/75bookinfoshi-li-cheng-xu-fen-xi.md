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
* `15006`: Envoy 的入口监听器，监控入站流量\(inbound\)，iptable 会将访问 pod 的流量导入该端口中由 Envoy 进行处理----**针对访问pod业务的流量重定向**
  * 

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

istio-proxy内部iptables规则

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







