# Egress TLS Origination

[控制 Egress 流量](https://istio.io/latest/zh/docs/tasks/traffic-management/egress/)的任务向我们展示了位于服务网格内部的应用应如何访问外部（即服务网格之外）的 HTTP 和 HTTPS 服务。 正如该任务所述，[`ServiceEntry`](https://istio.io/latest/zh/docs/reference/config/networking/service-entry/)用于配置 Istio 以受控的方式访问外部服务。 本示例将演示如何通过配置 Istio 去实现对发往外部服务的流量的TLS origination。 若此时原始的流量为 HTTP，则 Istio 会将其转换为 HTTPS 连接。

## 案例

假设有一个遗留应用正在使用 HTTP 和外部服务进行通信。而运行该应用的组织却收到了一个新的需求，该需求要求必须对所有外部的流量进行加密。 此时，使用 Istio 便可通过修改配置实现此需求，而无需更改应用中的任何代码。 该应用可以发送未加密的 HTTP 请求，然后 Istio 将为应用加密请求。

从应用源头发送未加密的 HTTP 请求并让 Istio 执行 TSL 升级的另一个好处是可以产生更好的遥测并为未加密的请求提供更多的路由控制。

## 开始之前

启动[sleep](https://github.com/istio/istio/tree/release-1.7/samples/sleep)示例，该示例将用作外部调用的测试源

```
kubectl apply -f samples/sleep/sleep.yaml
```

## 配置对外部服务的访问

首先，使用与[Egress 流量控制](https://istio.io/latest/zh/docs/tasks/traffic-management/egress/)任务中的相同的技术，配置对外部服务`edition.cnn.com`的访问。 但这一次我们将使用单个`ServiceEntry`来启用对服务的 HTTP 和 HTTPS 访问。

1. 创建一个`ServiceEntry`和`VirtualService`以启用对`edition.cnn.com`的访问：

```bash
[root@master istio-1.4.10]# kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: servicemesher-com
spec:
  hosts:
  - www.servicemesher.com
  ports:
  - number: 80
    name: http-port
    protocol: HTTP
  - number: 443
    name: https-port
    protocol: HTTPS
  resolution: DNS
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: servicemesher-com
spec:
  hosts:
  - www.servicemesher.com
  tls:
  - match:
    - port: 443
      sni_hosts:
      - www.servicemesher.com
    route:
    - destination:
        host: www.servicemesher.com
        port:
          number: 443
      weight: 100
EOF
```

1. 向外部的 HTTP 服务发送请求：

```
[root@master istio-1.4.10]# kubectl exec -it $SOURCE_POD -- curl -sL -o /dev/null -D - http://www.servicemesher.com/istio-handbook/
Defaulting container name to sleep.
Use 'kubectl describe pod/sleep-8f795f47d-xgmlc -n default' to see all of the containers in this pod.
HTTP/1.1 301 Moved Permanently
server: envoy
date: Mon, 24 Aug 2020 21:25:14 GMT
content-type: text/html
content-length: 169
location: https://www.servicemesher.com/istio-handbook/
x-envoy-upstream-service-time: 17

HTTP/1.1 200 OK
Server: nginx/1.16.1
Date: Wed, 26 Aug 2020 08:16:23 GMT
Content-Type: text/html
Content-Length: 124949
Last-Modified: Thu, 13 Aug 2020 03:27:22 GMT
Connection: keep-alive
ETag: "5f34b31a-1e815"
Accept-Ranges: bytes

[root@master istio-1.4.10]#
```

请注意_curl_的`-L`标志，该标志指示_curl_将遵循重定向。 在这种情况下，服务器将对到`http://www.servicemesher.com/istio-handbook`的 HTTP 请求返回重定向响应 \([301 Moved Permanently](https://tools.ietf.org/html/rfc2616#section-10.3.2)\)。 而重定向响应将指示客户端使用 HTTPS 向`https://www.servicemesher.com/istio-handbook`重新发送请求。 对于第二个请求，服务器则返回了请求的内容和_200 OK_状态码。

尽管_curl_命令简明地处理了重定向，但是这里有两个问题。 第一个问题是请求冗余，它使获取`http://www.servicemesher.com/istio-handbook`内容的延迟加倍。 第二个问题是 URL 中的路径（在本例中为_politics_）被以明文的形式发送。 如果有人嗅探您的应用与`www.servicemesher.com/`之间的通信，他将会知晓该应用获取了此网站中哪些特定的内容。 而出于隐私的原因，您可能希望阻止这些内容被披露。

通过配置`Istio`执行`TLS`发起，则可以解决这两个问题。

## 用于 egress 流量的 TLS 发起

1. 重新定义上一节的`ServiceEntry`和`VirtualService`以重写 HTTP 请求端口，并添加一个`DestinationRule`以执行 TLS 发起。

```
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: servicemesher-com
spec:
  hosts:
  - www.servicemesher.com
  ports:
  - number: 80
    name: http-port
    protocol: HTTP
  - number: 443
    name: https-port
    protocol: HTTPS
  resolution: DNS
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: servicemesher-com
spec:
  hosts:
  - www.servicemesher.com
  http:
  - match:
    - port: 80
    route:
    - destination:
        host: www.servicemesher.com
        subset: tls-origination
        port:
          number: 443
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: servicemesher-com
spec:
  host: www.servicemesher.com
  subsets:
  - name: tls-origination
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
      portLevelSettings:
      - port:
          number: 443
        tls:
          mode: SIMPLE # initiates HTTPS when accessing www.servicemesher.com
EOF
```

如您所见`VirtualService`将 80 端口的请求重定向到 443 端口，并在相应的`DestinationRule`执行 TSL 发起。

1. 如上一节一样，向`http://www.servicemesher.com/istio-handbook`发送 HTTP 请求：

```
[root@master istio-1.4.10]# kubectl exec -it $SOURCE_POD -- curl -sL -o /dev/null -D - http://www.servicemesher.com/istio-handbook/
Defaulting container name to sleep.
Use 'kubectl describe pod/sleep-8f795f47d-xgmlc -n default' to see all of the containers in this pod.
HTTP/1.1 200 OK
server: envoy
date: Mon, 24 Aug 2020 21:29:38 GMT
content-type: text/html
content-length: 124949
last-modified: Thu, 13 Aug 2020 03:27:22 GMT
etag: "5f34b31a-1e815"
accept-ranges: bytes
x-envoy-upstream-service-time: 42

[root@master istio-1.4.10]#
```

这次将会收到唯一的_200 OK_响应。 因为 Istio 为_curl_执行了 TLS 发起，原始的 HTTP 被升级为 HTTPS 并转发到`www.servicemesher.com`。 服务器直接返回内容而无需重定向。 这消除了客户端与服务器之间的请求冗余，使网格保持加密状态，隐藏了您的应用获取`www.servicemesher.com`中istio-handbook的事实。

