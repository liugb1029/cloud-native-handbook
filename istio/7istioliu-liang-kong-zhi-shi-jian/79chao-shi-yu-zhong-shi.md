### 超时与重试： 提升系统的健壮性和可用性

#### ![](/image/Istio/bookinfo-delay-retry.png)

#### 延迟与超时

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

查看ratings envoy的日志，有两次重试的记录。![](/image/Istio/bookinfo-retry.png)

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
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
    outlierDetection:
      consecutiveErrors: 1
      interval: 1s
      baseEjectionTime: 3m
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



