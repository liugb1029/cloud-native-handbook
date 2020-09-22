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



