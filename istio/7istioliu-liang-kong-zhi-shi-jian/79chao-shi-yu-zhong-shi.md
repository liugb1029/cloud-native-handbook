### 超时与重试、熔断、故障注入

#### ![](/image/Istio/bookinfo-delay-retry.png)

#### 延迟与超时：提升系统的健壮性和可用性

1、由于只有reviews-v2和v3才会调用ratings服务，所以我们先将流量引导到reviews-v2。

```bash
[root@master]# kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
  namespace: default
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v2
EOF
```

2、这时访问productpage页面应该是黑色星星。

3、配置rating延迟策略。

```
[root@master]# kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
  namespace: default
spec:
  hosts:
  - ratings
  http:
  - fault:
      delay:
        fixedDelay: 2s
        percent: 100
    route:
    - destination:
        host: ratings
        subset: v1
EOF
```

这时访问productpage页面需等待2s才会出现黑色星星。![](/image/Istio/bookinfo-delay-2s.png)注意如果rating延迟策略配置为5s，那就会报错，不会消失黑色星星。（reviews调用rating服务默认最长延迟为3s）

4、配置reviews超时策略

```
[root@master]# kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v2
    timeout: 1s
EOF
```

这时候访问页面直接报错![](/image/Istio/bookinfo-timeout-1s.png)

#### 重试

1、恢复reviews的配置\(去掉超时策略\)

```
[root@master]# kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v2
EOF
```

2、配置ratings重试策略

```
[root@master]# kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - fault:
      delay:
        percent: 100
        fixedDelay: 5s
    route:
    - destination:
        host: ratings
        subset: v1
    retries:
      attempts: 2
      perTryTimeout: 1s
EOF
```

查看ratings envoy的日志，有两次重试的记录。![](/image/Istio/bookinfo-retry.png)针对500错误才出发重试

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
  namespace: demo
spec:
  hosts:
  - "*"
  gateways:
  - httpbin-gateway
  http:
  - route:
    - destination:
        host: httpbin
        port:
          number: 8000
    retries:
      attempts: 3
      perTryTimeout: 1s
      retryOn: 5xx
    timeout: 8s
```

### 熔断

熔断

* 一种过载保护的手段
* 目的：避免服务的级联失败
* 关键点：三个状态；失效计数器；超时时钟![](/image/Istio/bookinfo-circut.png)

  以httpbin服务来实践

1、配置熔断策略

```
[root@master]# kubectl apply -f httpbin/httpbin.yaml
[root@master]# kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: httpbin
spec:
  host: httpbin
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1
      http:
        http1MaxPendingRequests: 1  # 最大被阻挡的请求数
        maxRequestsPerConnection: 1
    # 失败探测
    outlierDetection:
      consecutiveErrors: 1  # 失败的计数器
      interval: 1s   # 熔断间隔时间
      baseEjectionTime: 3m # 最小驱逐时间，默认30s
      maxEjectionPercent: 100
EOF
```

2、部署负载测试客户端fortio, 其可以控制连接数、并发数及发送 HTTP 请求的延迟。通过 Fortio 能够有效的触发前面 在

`DestinationRule`中设置的熔断策略。

```
[root@master samples]# kubectl apply -f httpbin/sample-client/fortio-deploy.yaml
```

3、登入客户端Pod并使用Fortio工具调用httpbin服务。-curl参数表明发送一次调用

```
[root@master samples]# FORTIO_POD=$(kubectl get pod | grep fortio | awk '{ print $1 }')
[root@master samples]# kubectl exec -it $FORTIO_POD  -c fortio -- /usr/bin/fortio load -curl  http://httpbin:8000/get
02:32:50 I fortio_main.go:167> Not using dynamic flag watching (use -config to set watch directory)
HTTP/1.1 200 OK
server: envoy
date: Tue, 22 Sep 2020 02:32:50 GMT
content-type: application/json
content-length: 371
access-control-allow-origin: *
access-control-allow-credentials: true
x-envoy-upstream-service-time: 12

{
  "args": {},
  "headers": {
    "Content-Length": "0",
    "Host": "httpbin:8000",
    "User-Agent": "fortio.org/fortio-1.6.8",
    "X-B3-Parentspanid": "bd04ce2f5376ab90",
    "X-B3-Sampled": "1",
    "X-B3-Spanid": "96386c689d12d226",
    "X-B3-Traceid": "bb214696a2ad71cbbd04ce2f5376ab90"
  },
  "origin": "127.0.0.1",
  "url": "http://httpbin:8000/get"
}
```

4、触发熔断器测试

```
# 发送并发数为2的连接(-c 2),请求20次(-n 20)
[root@master samples]# kubectl exec -it $FORTIO_POD  -c fortio -- /usr/bin/fortio load -c 2 -qps 0 -n 20 -loglevel Warning  http://httpbin:8000/get
02:36:35 I logger.go:115> Log level is now 3 Warning (was 2 Info)
Fortio 1.6.8 running at 0 queries per second, 2->2 procs, for 20 calls: http://httpbin:8000/get
Starting at max qps with 2 thread(s) [gomax 2] for exactly 20 calls (10 per thread + 0)
02:36:35 W http_client.go:698> Parsed non ok code 503 (HTTP/1.1 503)
Ended after 51.796ms : 20 calls. qps=386.13
Aggregated Function Time : count 20 avg 0.0050606452 +/- 0.001042 min 0.0015169 max 0.006568188 sum 0.101212904
# range, mid point, percentile, count
>= 0.0015169 <= 0.002 , 0.00175845 , 5.00, 1
> 0.004 <= 0.005 , 0.0045 , 60.00, 11
> 0.005 <= 0.006 , 0.0055 , 85.00, 5
> 0.006 <= 0.00656819 , 0.00628409 , 100.00, 3
# target 50% 0.00481818
# target 75% 0.0056
# target 90% 0.0061894
# target 99% 0.00653031
# target 99.9% 0.0065644
Sockets used: 3 (for perfect keepalive, would be 2)
Jitter: false
Code 200 : 19 (95.0 %)
Code 503 : 1 (5.0 %)
Response Header Sizes : count 20 avg 218.5 +/- 50.13 min 0 max 230 sum 4370
Response Body/Total Sizes : count 20 avg 583 +/- 78.46 min 241 max 601 sum 11660
All done 20 calls (plus 0 warmup) 5.061 ms avg, 386.1 qps


# 将并发连接数提高到 3 个：
[root@master samples]# kubectl exec -it $FORTIO_POD  -c fortio -- /usr/bin/fortio load -c 3 -qps 0 -n 30 -loglevel Warning  http://httpbin:8000/get
02:42:53 I logger.go:115> Log level is now 3 Warning (was 2 Info)
Fortio 1.6.8 running at 0 queries per second, 2->2 procs, for 30 calls: http://httpbin:8000/get
Starting at max qps with 3 thread(s) [gomax 2] for exactly 30 calls (10 per thread + 0)
02:42:53 W http_client.go:698> Parsed non ok code 503 (HTTP/1.1 503)
02:42:53 W http_client.go:698> Parsed non ok code 503 (HTTP/1.1 503)
02:42:53 W http_client.go:698> Parsed non ok code 503 (HTTP/1.1 503)
02:42:53 W http_client.go:698> Parsed non ok code 503 (HTTP/1.1 503)
02:42:53 W http_client.go:698> Parsed non ok code 503 (HTTP/1.1 503)
02:42:53 W http_client.go:698> Parsed non ok code 503 (HTTP/1.1 503)
02:42:53 W http_client.go:698> Parsed non ok code 503 (HTTP/1.1 503)
02:42:53 W http_client.go:698> Parsed non ok code 503 (HTTP/1.1 503)
02:42:53 W http_client.go:698> Parsed non ok code 503 (HTTP/1.1 503)
02:42:53 W http_client.go:698> Parsed non ok code 503 (HTTP/1.1 503)
02:42:53 W http_client.go:698> Parsed non ok code 503 (HTTP/1.1 503)
Ended after 74.086428ms : 30 calls. qps=404.93
Aggregated Function Time : count 30 avg 0.0050834761 +/- 0.004166 min 0.000303125 max 0.014342636 sum 0.152504284
# range, mid point, percentile, count
>= 0.000303125 <= 0.001 , 0.000651563 , 33.33, 10
> 0.001 <= 0.002 , 0.0015 , 36.67, 1
> 0.004 <= 0.005 , 0.0045 , 50.00, 4
> 0.005 <= 0.006 , 0.0055 , 66.67, 5
> 0.006 <= 0.007 , 0.0065 , 70.00, 1
> 0.007 <= 0.008 , 0.0075 , 76.67, 2
> 0.008 <= 0.009 , 0.0085 , 83.33, 2
> 0.01 <= 0.011 , 0.0105 , 90.00, 2
> 0.012 <= 0.014 , 0.013 , 96.67, 2
> 0.014 <= 0.0143426 , 0.0141713 , 100.00, 1
# target 50% 0.005
# target 75% 0.00775
# target 90% 0.011
# target 99% 0.0142398
# target 99.9% 0.0143324
Sockets used: 13 (for perfect keepalive, would be 3)
Jitter: false
Code 200 : 19 (63.3 %)
Code 503 : 11 (36.7 %)
Response Header Sizes : count 30 avg 145.76667 +/- 110.9 min 0 max 231 sum 4373
Response Body/Total Sizes : count 30 avg 469.1 +/- 173.6 min 241 max 602 sum 14073
All done 30 calls (plus 0 warmup) 5.083 ms avg, 404.9 qps
```

5、查询istio-proxy状态以了解更多熔断详情

```
[root@master samples]# kubectl exec -it fortio-deploy-6dc9b4d7d9-hmgnj -c istio-proxy -- pilot-agent request GET stats|grep httpbin.default|grep pending
cluster.outbound|8000||httpbin.default.svc.cluster.local.circuit_breakers.default.rq_pending_open: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.circuit_breakers.high.rq_pending_open: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_active: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_failure_eject: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_overflow: 15
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_total: 45
```

![](/image/Istio/circut配置分析.png)

#### 故障注入

1、准备工作

```bash
# productpage → reviews:v2 → ratings (针对 jason 用户)
# productpage → reviews:v1 (其他用户)

[root@master istio-1.4.10]# kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
[root@master istio-1.4.10]# kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml
[root@master istio-1.4.10]# kubectl get virtualservices.networking.istio.io reviews -oyaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
  namespace: default
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v1
[root@master istio-1.4.10]# kubectl get virtualservices.networking.istio.io ratings -oyaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
  namespace: default
spec:
  hosts:
  - ratings
  http:
  - route:
    - destination:
        host: ratings
        subset: v1
```

2、注入HTTP延迟故障

为了测试微服务应用程序 Bookinfo 的弹性，我们将为用户`jason`在`reviews:v2`和`ratings`服务之间注入一个 7 秒的延迟。 这个测试将会发现一个故意引入 Bookinfo 应用程序中的 bug。

注意`reviews:v2`服务对`ratings`服务的调用具有 10 秒的硬编码连接超时。 因此，尽管引入了 7 秒的延迟，我们仍然期望端到端的流程是没有任何错误的。

```bash
# 创建故障注入规则以延迟来自测试用户 jason 的流量：
[root@master istio-1.4.10]# kubectl apply -f samples/bookinfo/networking/virtual-service-ratings-test-delay.yaml
[root@master istio-1.4.10]# kubectl get virtualservices.networking.istio.io ratings -oyaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
  namespace: default
spec:
  hosts:
  - ratings
  http:
  - fault:
      delay:
        fixedDelay: 7s
        percentage:
          value: 100
    match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: ratings
        subset: v1
  - route:
    - destination:
        host: ratings
        subset: v1
```

3、测试延迟配置

通过浏览器打开[Bookinfo](https://istio.io/latest/zh/docs/examples/bookinfo)应用。使用用户`jason`登陆到`/productpage`页面。

你期望 Bookinfo 主页在大约 7 秒钟加载完成并且没有错误。 但是，出现了一个问题：Reviews 部分显示了错误消息：

```
Error fetching product reviews!
Sorry, product reviews are currently unavailable for this book.
```

![](/image/Istio/bookinfo-faultinject-delay.png)

4、注入HTTP abort故障

```
# 为用户 jason 创建一个发送 HTTP abort 的故障注入规则：
[root@master istio-1.4.10]# kubectl apply -f samples/bookinfo/networking/virtual-service-ratings-test-abort.yaml
[root@master istio-1.4.10]# kubectl get virtualservice ratings -o yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
  ...
spec:
  hosts:
  - ratings
  http:
  - fault:
      abort:
        httpStatus: 500
        percentage:
          value: 100
    match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: ratings
        subset: v1
  - route:
    - destination:
        host: ratings
        subset: v1
```

5、测试中止配置

用浏览器打开[Bookinfo](https://istio.io/latest/zh/docs/examples/bookinfo)应用。使用用户`jason`登陆到`/productpage`页面。

* 如果规则成功传播到所有的 pod，您应该能立即看到页面加载并看到`Ratings service is currently unavailable`消息。

* 如果您注销用户`jason`或在匿名窗口（或其他浏览器）中打开 Bookinfo 应用程序， 您将看到`/productpage`为除`jason`以外的其他用户调用了`reviews:v1`（完全不调用`ratings`）。 因此，您不会看到任何错误消息。

![](/image/Istio/fault-injection配置分析.png)

