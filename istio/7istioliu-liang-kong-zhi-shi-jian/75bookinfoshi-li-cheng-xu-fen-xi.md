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

## Istio 中的 sidecar 注入 {#istio-中的-sidecar-注入}

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

### Init 容器 {#init-容器}

Init 容器是一种专用容器，它在应用程序容器启动之前运行，用来包含一些应用镜像中不存在的实用工具或安装脚本。

一个[Pod](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pod)中可以指定多个 Init 容器，如果指定了多个，那么 Init 容器将会按顺序依次运行。只有当前面的 Init 容器必须运行成功后，才可以运行下一个 Init 容器。当所有的 Init 容器运行完成后，Kubernetes 才初始化[Pod](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pod)和运行应用容器。

Init 容器使用 Linux Namespace，所以相对应用程序容器来说具有不同的文件系统视图。因此，它们能够具有访问 Secret 的权限，而应用程序容器则不能。

在[Pod](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pod)启动过程中，Init 容器会按顺序在网络和数据卷初始化之后启动。每个容器必须在下一个容器启动之前成功退出。如果由于运行时或失败退出，将导致容器启动失败，它会根据[Pod](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pod)的`restartPolicy`指定的策略进行重试。然而，如果[Pod](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pod)的`restartPolicy`设置为 Always，Init 容器失败时会使用`RestartPolicy`策略。

在所有的 Init 容器没有成功之前，[Pod](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pod)将不会变成`Ready`状态。Init 容器的端口将不会在[Service](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#service)中进行聚集。 正在初始化中的[Pod](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pod)处于`Pending`状态，但应该会将`Initializing`状态设置为 true。Init 容器运行完成以后就会自动终止。

关于 Init 容器的详细信息请参考[Init 容器 - Kubernetes 中文指南/云原生应用架构实践手册](https://jimmysong.io/kubernetes-handbook/concepts/init-containers.html)。

## Sidecar 注入示例分析 {#sidecar-注入示例分析}

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
        resources: {}
        terminationMessagePath: /dev/termination-log
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

#### Prxoyv2

#### Envoy初始配置文件

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



