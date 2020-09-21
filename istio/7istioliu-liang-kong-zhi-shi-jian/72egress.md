### Egress---管理出网络的流量

一般访问外部服务的方法：

* 配置global.outboundTrafficPolicy.mode=ALLOW\_ANY
* 使用服务入口\(ServiceEntry\)
* 配置sidecar让流量绕过代理
* 配置egree网关

#### 配置流程

1. 查看egress gateway组件是否正常
2. 为外部服务定义ServiceEntry
3. 定义Egress gateway
4. 定义路由，将流量引导到egressgateway
5. 查看日志验证

#### 1、查看组件书否正常

```bash
[root@master egress]# kubectl get pod -n istio-system |grep egress
istio-egressgateway-68988594d6-h5lh7         1/1     Running     1          4d7h
[root@master egress]#
```

#### 2、配置ServiceEntry

```
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: dr-for-egressgateway
spec:
  host: istio-egressgateway.istio-system.svc.cluster.local
  subsets:
  - name: httpbin
```

#### 3、定义Egress gateway

```
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: httpbin-egressgateway
spec:
  selector:
    istio: egressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "httpbin.org"
```

#### 4、定义路由，将流量引导到gateway

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: vs-for-egressgateway
spec:
  hosts:
  - httpbin.org
  gateways:
  - httpbin-egressgateway
  - mesh
  http:
  - match:
    - gateways:
      - mesh
      port: 80
    route:
    - destination:
        host: istio-egressgateway.istio-system.svc.cluster.local
        subset: httpbin
        port:
          number: 80
      weight: 100
  - match:
    - gateways:
      - httpbin-egressgateway
      port: 80
    route:
    - destination:
        host: httpbin.org
        port:
          number: 80
      weight: 100
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: dr-for-egressgateway
spec:
  host: istio-egressgateway.istio-system.svc.cluster.local
  subsets:
  - name: httpbin
```

#### 5、查看日志验证

```bash
# isto-egressgateway日志

[2020-09-20T09:49:07.378Z] "GET /ip HTTP/2" 200 - "-" "-" 0 47 606 590 "10.244.1.36" "curl/7.69.1" "8b5de16e-690a-94c8-91d0-34ddea75003a" "httpbin.org" "34.194.129.11:80" outbound|80||httpbin.org - 10.244.2.35:80 10.244.1.36:44294 - -
[2020-09-20T09:49:11.551Z] "GET /ip HTTP/2" 200 - "-" "-" 0 47 2604 2602 "10.244.1.36" "curl/7.69.1" "6d1015c6-42bb-96ce-a5cb-984a6de60927" "httpbin.org" "35.172.162.144:80" outbound|80||httpbin.org - 10.244.2.35:80 10.244.1.36:44320 - -
[2020-09-20T09:49:16.873Z] "GET /ip HTTP/2" 200 - "-" "-" 0 47 537 536 "10.244.1.36" "curl/7.69.1" "07fbee79-9105-973e-93f2-2f9a2aa07106" "httpbin.org" "3.221.81.55:80" outbound|80||httpbin.org - 10.244.2.35:80 10.244.1.36:44294 - -
```

#### 6、流程分析



### 什么是服务入口（ServiceEntry）

* 添加外部服务到网格内
* 管理到外部服务的请求
* 扩展网格

由于默认情况下，来自 Istio-enable Pod 的所有出站流量都会重定向到其 Sidecar 代理，群集外部 URL 的可访问性取决于代理的配置。默认情况下，Istio 将 Envoy 代理配置为允许传递未知服务的请求。尽管这为入门 Istio 带来了方便，但是，通常情况下，配置更严格的控制是更可取的。

这个任务向你展示了三种访问外部服务的方法：

* 允许 Envoy 代理将请求传递到未在网格内配置过的服务。

* 配置[service entries](https://istio.io/latest/zh/docs/reference/config/networking/service-entry/)以提供对外部服务的受控访问。

* 对于特定范围的 IP，完全绕过 Envoy 代理。![](/image/Istio/ServiceEntry.png)

[https://istio.io/latest/zh/docs/tasks/traffic-management/egress/egress-control/](https://istio.io/latest/zh/docs/tasks/traffic-management/egress/egress-control/)

#### 1、部署sleep服务\(带curl\)

```bash
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

```bash
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

```bash
$ kubectl get configmap istio -n istio-system -o yaml | grep -o "mode: ALLOW_ANY"
mode: ALLOW_ANY
```

如果它开启了，那么输出应该会出现`mode: ALLOW_ANY`。

如果你显式地设置了`REGISTRY_ONLY`模式，可以用以下的命令来改变它：

```bash
kubectl get configmap istio -n istio-system -o yaml | sed 's/mode: REGISTRY_ONLY/mode: ALLOW_ANY/g' | kubectl replace -n istio-system -f -
```

参考链接 [https://archive.istio.io/v1.4/docs/tasks/traffic-management/egress/egress-control/](https://archive.istio.io/v1.4/docs/tasks/traffic-management/egress/egress-control/)

```bash
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

```bash
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

```bash
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

### 管理到外部服务的流量

与集群内的请求相似，也可以为使用`ServiceEntry`配置访问的外部服务设置[Istio 路由规则](https://istio.io/latest/zh/docs/concepts/traffic-management/#routing-rules)。在本示例中，你将设置对`httpbin.org`服务访问的超时规则。

1. 从用作测试源的 pod 内部，向外部服务`httpbin.org`的`/delay`endpoint 发出_curl_请求：

2. ```bash
   kubectl exec -it $SOURCE_POD -c sleep sh
   time curl -o /dev/null -s -w "%{http_code}\n" http://httpbin.org/delay/5
   ```

   这个请求大约在 5 秒内返回 200 \(OK\)。

3. 退出测试源 pod，使用`kubectl`设置调用外部服务`httpbin.org`的超时时间为 3 秒。

   ```
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: VirtualService
   metadata:
     name: httpbin-ext
   spec:
     hosts:
       - httpbin.org
     http:
     - timeout: 3s
       route:
         - destination:
             host: httpbin.org
           weight: 100
   EOF
   ```

4. 几秒后，重新发出_curl_请求：

   ```bash
   $ kubectl exec -it $SOURCE_POD -c sleep sh
   $ time curl -o /dev/null -s -w "%{http_code}\n" http://httpbin.org/delay/5
   504

   real    0m3.149s
   user    0m0.004s
   sys     0m0.004s
   ```

   这一次，在 3 秒后出现了 504 \(Gateway Timeout\)。Istio 在 3 秒后切断了响应时间为 5 秒的`httpbin.org`服务。

### 直接访问外部服务

如果要让特定范围的 ​​IP 完全绕过 Istio，则可以配置 Envoy sidecars 以防止它们[拦截](https://istio.io/latest/zh/docs/concepts/traffic-management/)外部请求。要设置绕过 Istio，请更改`global.proxy.includeIPRanges`或`global.proxy.excludeIPRanges`配置选项，并使用`kubectl apply`命令更新`istio-sidecar-injector`的[配置](https://istio.io/latest/zh/docs/reference/config/installation-options/)。`istio-sidecar-injector`配置的更新，影响的是新部署应用的 pod。

> 与[Envoy 转发流量到外部服务](https://istio.io/latest/zh/docs/tasks/traffic-management/egress/egress-control/#envoy-passthrough-to-external-services)不同，后者使用`ALLOW_ANY`流量策略来让 Istio sidecar 代理将调用传递给未知服务， 该方法完全绕过了 sidecar，从而实质上禁用了指定 IP 的所有 Istio 功能。你不能像`ALLOW_ANY`方法那样为特定的目标增量添加 service entries。 因此，仅当出于性能或其他原因无法使用边车配置外部访问时，才建议使用此配置方法。

排除所有外部 IP 重定向到 Sidecar 代理的一种简单方法是将`global.proxy.includeIPRanges`配置选项设置为内部集群服务使用的 IP 范围。这些 IP 范围值取决于集群所在的平台。

#### 配置代理绕行

删除本指南中先前部署的 service entry 和 virtual service。

使用平台的 IP 范围更新`istio-sidecar-injector`的配置。比如，如果 IP 范围是 10.0.0.1/24，则使用一下命令：

```bash
istioctl manifest apply <the flags you used to install Istio> --set values.global.proxy.includeIPRanges="10.0.0.1/24"
```

在[安装 Istio](https://istio.io/latest/zh/docs/setup/install/istioctl)命令的基础上增加`--set values.global.proxy.includeIPRanges="10.0.0.1/24"`

#### 访问外部服务

由于绕行配置仅影响新的部署，因此您需要按照[开始之前](https://istio.io/latest/zh/docs/tasks/traffic-management/egress/egress-control/#before-you-begin)部分中的说明重新部署`sleep`程序。

在更新`istio-sidecar-injector`configmap 和重新部署`sleep`程序后，Istio sidecar 将仅拦截和管理集群中的内部请求。 任何外部请求都会绕过 Sidecar，并直接到达其预期的目的地。举个例子：

```bash
istioctl manifest apply --set profile=demo --set values.global.proxy.includeIPRanges="10.96.0.0/12"
```

修改后无需重启sidecar-injector，并且只针对新部署的服务有效。

与通过 HTTP 和 HTTPS 访问外部服务不同，你不会看到任何与** Istio sidecar 有关的请求头**， 并且发送到外部服务的请求既不会出现在 Sidecar 的日志中，也不会出现在 Mixer 日志中。 绕过 Istio sidecar 意味着你不能再监视对外部服务的访问。

```bash
[root@master istio-1.4.10]# export SOURCE_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})
[root@master istio-1.4.10]# kubectl exec -it $SOURCE_POD -- sh
/ # curl http://httpbin.org/headers
{
  "headers": {
    "Accept": "*/*",
    "Host": "httpbin.org",
    "User-Agent": "curl/7.69.1",
    "X-Amzn-Trace-Id": "Root=1-5f460aec-6cfe0a88b970720b344997c7"
  }
}
```

### 理解原理

在此任务中，您研究了从 Istio 网格调用外部服务的三种方法：

1. 配置 Envoy 以允许访问任何外部服务。

2. 使用 service entry 将一个可访问的外部服务注册到网格中。这是推荐的方法。

3. 配置 Istio sidecar 以从其重新映射的 IP 表中排除外部 IP。

第一种方法通过 Istio sidecar 代理来引导流量，包括对网格内部未知服务的调用。使用这种方法时，你将无法监控对外部服务的访问或无法利用 Istio 的流量控制功能。 要轻松为特定的服务切换到第二种方法，只需为那些外部服务创建 service entry 即可。 此过程使你可以先访问任何外部服务，然后再根据需要决定是否启用控制访问、流量监控、流量控制等功能。

第二种方法可以让你使用 Istio 服务网格所有的功能区调用集群内或集群外的服务。 在此任务中，你学习了如何监控对外部服务的访问并设置对外部服务的调用的超时规则。

第三种方法绕过了 Istio Sidecar 代理，使你的服务可以直接访问任意的外部服务。 但是，以这种方式配置代理需要了解集群提供商相关知识和配置。 与第一种方法类似，你也将失去对外部服务访问的监控，并且无法将 Istio 功能应用于外部服务的流量。

### 安全说明

> 请注意，此任务中的配置示例 **没有启用安全的出口流量控制**恶意程序可以绕过 Istio Sidecar 代理并在没有 Istio 控制的情况下访问任何外部服务。

为了以更安全的方式实施出口流量控制，你必须[通过 egress gateway 引导出口流量](https://istio.io/latest/zh/docs/tasks/traffic-management/egress/egress-gateway/)， 并查看[其他安全注意事项](https://istio.io/latest/zh/docs/tasks/traffic-management/egress/egress-gateway/#additional-security-considerations)部分中描述的安全问题。

#### 将出站流量策略模式设置为所需的值

1. 检查现在的值:

   ```
   $ kubectl get configmap istio -n istio-system -o yaml | grep -o "mode: ALLOW_ANY" | uniq
   $ kubectl get configmap istio -n istio-system -o yaml | grep -o "mode: REGISTRY_ONLY" | uniq
   mode: ALLOW_ANY
   ```

   输出将会是`mode: ALLOW_ANY`或`mode: REGISTRY_ONLY`。

2. 如果你想改变这个模式，执行以下命令：

```
kubectl get configmap istio -n istio-system -o yaml | sed 's/mode: ALLOW_ANY/mode: REGISTRY_ONLY/g' | kubectl replace -n istio-system -f -
```

```
kubectl get configmap istio -n istio-system -o yaml | sed 's/mode: REGISTRY_ONLY/mode: ALLOW_ANY/g' | kubectl replace -n istio-system -f -
```



