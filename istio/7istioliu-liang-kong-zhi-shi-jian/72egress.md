### 什么是服务入口（ServiceEntry）

* 添加外部服务到网格内
* 管理到外部服务的请求
* 扩展网格

由于默认情况下，来自 Istio-enable Pod 的所有出站流量都会重定向到其 Sidecar 代理，群集外部 URL 的可访问性取决于代理的配置。默认情况下，Istio 将 Envoy 代理配置为允许传递未知服务的请求。尽管这为入门 Istio 带来了方便，但是，通常情况下，配置更严格的控制是更可取的。

这个任务向你展示了三种访问外部服务的方法：

1. 允许 Envoy 代理将请求传递到未在网格内配置过的服务。
2. 配置[service entries](https://istio.io/latest/zh/docs/reference/config/networking/service-entry/)以提供对外部服务的受控访问。
3. 对于特定范围的 IP，完全绕过 Envoy 代理。![](/image/Istio/ServiceEntry.png)

[https://istio.io/latest/zh/docs/tasks/traffic-management/egress/egress-control/](https://istio.io/latest/zh/docs/tasks/traffic-management/egress/egress-control/)

#### 1、部署sleep服务\(带curl\)

```
[root@master istio-1.4.10]# kubectl apply -f samples/sleep/sleep.yaml
[root@master istio-1.4.10]# kubectl get pod
NAME                              READY   STATUS    RESTARTS   AGE
details-v1-6c9f8bcbcb-fgv29       2/2     Running   0          6h48m
productpage-v1-7f9d9c48c8-xxq6f   2/2     Running   0          6h47m
ratings-v1-65cff55fb8-8wnbs       2/2     Running   0          6h47m
reviews-v1-d5b6b667f-t2hc7        2/2     Running   0          6h47m
reviews-v2-784495d9bc-g94kz       2/2     Running   0          6h47m
reviews-v3-57fcb844b7-p9shm       2/2     Running   0          6h48m
sleep-8f795f47d-djmjn             2/2     Running   0          2m9s
[root@master istio-1.4.10]# export SOURCE_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})
```

#### 2、测试访问外部

使用外部[http://httpbin.org来测试http请求](http://httpbin.org来测试http请求)

```
# 由于Istio默认是允许访问外部服务的，所以下面测试访问是正常的。
[root@master istio-1.4.10]# kubectl exec -it sleep-8f795f47d-djmjn -- sh
/ # curl http://httpbin.org/headers
{
  "headers": {
    "Accept": "*/*",
    "Content-Length": "0",
    "Host": "httpbin.org",
    "User-Agent": "curl/7.69.1",
    "X-Amzn-Trace-Id": "Root=1-5f44ba10-f57e981043cf3a355e933560",
    "X-B3-Sampled": "1",
    "X-B3-Spanid": "89d9ab0cf76ad263",
    "X-B3-Traceid": "bc397a11008e6de989d9ab0cf76ad263",
    "X-Envoy-Expected-Rq-Timeout-Ms": "15000",
    "X-Istio-Attributes": "CjAKGGRlc3RpbmF0aW9uLnNlcnZpY2UubmFtZRIUEhJQYXNzdGhyb3VnaENsdXN0ZXIKOgoKc291cmNlLnVpZBIsEiprdWJlcm5ldGVzOi8vc2xlZXAtOGY3OTVmNDdkLWRqbWpuLmRlZmF1bHQ="
  }
}
/ #
```

#### 3、关闭访问外部策略

Istio 有一个[安装选项](https://istio.io/latest/zh/docs/reference/config/installation-options/)，`global.outboundTrafficPolicy.mode`，它配置 sidecar 对外部服务（那些没有在 Istio 的内部服务注册中定义的服务）的处理方式。如果这个选项设置为`ALLOW_ANY`，Istio 代理允许调用未知的服务。如果这个选项设置为`REGISTRY_ONLY`，那么 Istio 代理会阻止任何没有在网格中定义的 HTTP 服务或 service entry 的主机。`ALLOW_ANY`是默认值，不控制对外部服务的访问，方便你快速地评估 Istio。你可以稍后再[配置对外部服务的访问](https://istio.io/latest/zh/docs/tasks/traffic-management/egress/egress-control/#controlled-access-to-external-services)。

要查看这种方法的实际效果，你需要确保 Istio 的安装配置了`global.outboundTrafficPolicy.mode`选项为`ALLOW_ANY`。它在默认情况下是开启的，除非你在安装 Istio 时显式地将它设置为`REGISTRY_ONLY`。

运行以下命令以确认配置是正确的：

```
$ kubectl get configmap istio -n istio-system -o yaml | grep -o "mode: ALLOW_ANY"
mode: ALLOW_ANY
```

如果它开启了，那么输出应该会出现`mode: ALLOW_ANY`。

如果你显式地设置了`REGISTRY_ONLY`模式，可以用以下的命令来改变它：

```
kubectl get configmap istio -n istio-system -o yaml | sed 's/mode: REGISTRY_ONLY/mode: ALLOW_ANY/g' | kubectl replace -n istio-system -f -
```

参考链接 [https://archive.istio.io/v1.4/docs/tasks/traffic-management/egress/egress-control/](https://archive.istio.io/v1.4/docs/tasks/traffic-management/egress/egress-control/)

```
[root@master istio-1.4.10]# kubectl get configmap istio -n istio-system -o yaml | sed 's/mode: ALLOW_ANY/mode: REGISTRY_ONLY/g' | kubectl replace -n istio-system -f -

# 关闭后访问外部服务返回502
[root@master istio-1.4.10]# kubectl exec -it sleep-8f795f47d-djmjn -- sh
Defaulting container name to sleep.
Use 'kubectl describe pod/sleep-8f795f47d-djmjn -n default' to see all of the containers in this pod.
/ #
/ # curl -I http://httpbin.org/headers
HTTP/1.1 502 Bad Gateway
location: http://httpbin.org/headers
date: Mon, 24 Aug 2020 14:42:56 GMT
server: envoy
transfer-encoding: chunked

/ #
```

#### 4、配置ServiceEntry---访问一个外部的HTTP服务

```
[root@master istio-1.4.10]# kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: httpbin-ext
spec:
  hosts:
  - httpbin.org
  ports:
  - number: 80
    name: http
    protocol: HTTP
  resolution: DNS
  location: MESH_EXTERNAL
EOF
```

#### 5、配置ServiceEntry---访问一个外部的HTTPS服务

```
[root@master istio-1.4.10]# kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: google
spec:
  hosts:
  - www.google.com
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  resolution: DNS
  location: MESH_EXTERNAL
EOF
```

#### 6. 测试连通性

```
[root@master istio-1.4.10]# kubectl exec -it sleep-8f795f47d-djmjn -- sh
Defaulting container name to sleep.
Use 'kubectl describe pod/sleep-8f795f47d-djmjn -n default' to see all of the containers in this pod.
/ #
/ #
/ # curl http://httpbin.org/headers
{
  "headers": {
    "Accept": "*/*",
    "Content-Length": "0",
    "Host": "httpbin.org",
    "User-Agent": "curl/7.69.1",
    "X-Amzn-Trace-Id": "Root=1-5f44c0bc-e0fb975a8b9f1b62bfffddb4",
    "X-B3-Sampled": "1",
    "X-B3-Spanid": "06eb85967ce87c12",
    "X-B3-Traceid": "150400b6814a1a5c06eb85967ce87c12",
    "X-Envoy-Decorator-Operation": "httpbin.org:80/*",
    "X-Istio-Attributes": "CikKGGRlc3RpbmF0aW9uLnNlcnZpY2UuaG9zdBINEgtodHRwYmluLm9yZwopChhkZXN0aW5hdGlvbi5zZXJ2aWNlLm5hbWUSDRILaHR0cGJpbi5vcmcKKgodZGVzdGluYXRpb24uc2VydmljZS5uYW1lc3BhY2USCRIHZGVmYXVsdAo6Cgpzb3VyY2UudWlkEiwSKmt1YmVybmV0ZXM6Ly9zbGVlcC04Zjc5NWY0N2QtZGptam4uZGVmYXVsdA=="
  }
}
```

注意由 Istio sidecar 代理添加的 headers:`X-Envoy-Decorator-Operation`。

