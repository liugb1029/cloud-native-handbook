### 什么是网关（Gateway）

* 一个运行在网格边缘的负载均衡器
* 接收外部请求，转发给网格内的服务
* 配置对外的端口、协议与内部服务的映射关系

#### Gateway 的应用场景

* 暴露网格内服务给外界访问
* 访问安全（HTTPS、mTLS 等）
* 统一应用入口，API 聚合

![](/image/Istio/istio-ingress.png)![](/image/Istio/gateway-config.png)

#### 1、事先部署好bookinfo demo

```
[root@master ~]# kubectl get pod
NAME                              READY   STATUS    RESTARTS   AGE
details-v1-6c9f8bcbcb-fgv29       2/2     Running   0          4h1m
productpage-v1-7f9d9c48c8-xxq6f   2/2     Running   0          4h1m
ratings-v1-65cff55fb8-8wnbs       2/2     Running   0          4h1m
reviews-v1-d5b6b667f-t2hc7        2/2     Running   0          4h1m
reviews-v2-784495d9bc-g94kz       2/2     Running   0          4h1m
reviews-v3-57fcb844b7-p9shm       2/2     Running   0          4h1m

# 一般来说针对一个系统，只暴露一个入口，这里将detail服务也暴露出来，用于演示Gateway
[root@master ~]# cat detaile-gateway.yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: detail-gateway
  namespace: default
spec:
  selector:
    istio: ingressgateway
  servers:
  - hosts:
    - '*'
    port:
      name: http
      number: 80
      protocol: HTTP

---

apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: detail-gateway
  namespace: default
spec:
  gateways:
  - detail-gateway
  hosts:
  - '*'
  http:
  - match:
    - uri:
        exact: /health
    - uri:
        prefix: /details
    route:
    - destination:
        host: details
        port:
          number: 9080

# VirtualService配置
[root@master ~]# kubectl get virtualservices.networking.istio.io detail-gateway -oyaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: detail-gateway
  namespace: default
spec:
  gateways:
  - detail-gateway
  hosts:
  - '*'
  http:
  - match:
    - uri:
        exact: /health
    - uri:
        prefix: /details
    route:
    - destination:
        host: details
        port:
          number: 9080
```

#### 2、测试

```bash
# 获取ingress Http NodePort端口
[root@master ~]# kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}'
30175

[root@master ~]# curl http://192.168.56.100:30175/health
{"status":"Details is healthy"}

[root@master ~]# curl http://192.168.56.100:30175/details/0
{"id":0,"author":"William Shakespeare","year":1595,"type":"paperback","pages":200,"publisher":"PublisherA","language":"English","ISBN-10":"1234567890","ISBN-13":"123-1234567890"}
```

#### 3、实现仅允许域名方式访问detail

hosts中设置具体的域名

```
[root@master ~]# cat details-gateway.yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: detail-gateway
  namespace: default
spec:
  selector:
    istio: ingressgateway
  servers:
  - hosts:
    - 'abc.k8s.com'
    port:
      name: http
      number: 80
      protocol: HTTP

---

apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: detail-gateway
  namespace: default
spec:
  gateways:
  - detail-gateway
  hosts:
  - 'abc.k8s.com'
  http:
  - match:
    - uri:
        exact: /health
    - uri:
        prefix: /details
    route:
    - destination:
        host: details
        port:
          number: 9080
```

#### 4、测试

```
# 不带Host: abc.k8s.com 访问报404
[root@master ~]# curl -I -s http://192.168.56.100:30175/health
HTTP/1.1 404 Not Found
date: Mon, 24 Aug 2020 11:05:59 GMT
server: istio-envoy
transfer-encoding: chunked

# 带Host: abc.k8s.com 访问返回200
[root@master ~]# curl -I -s -H "Host:abc.k8s.com" http://192.168.56.100:30175/health
HTTP/1.1 200 OK
content-type: application/json
server: istio-envoy
date: Mon, 24 Aug 2020 11:07:48 GMT
content-length: 31
x-envoy-upstream-service-time: 3


# 配置修改前后的istio-ingress gateway日志
[2020-08-24T10:55:13.341Z] "GET /health HTTP/1.1" 200 - "-" "-" 0 31 4 2 "10.244.0.0" "curl/7.29.0" "93639236-4969-9a7f-8de1-a2bfc123c62d" "192.168.56.100:30175" "10.244.2.81:9080" outbound|9080||details.default.svc.cluster.local - 10.244.1.55:80 10.244.0.0:14162 - -
[2020-08-24T10:57:15.683Z] "GET /details/0 HTTP/1.1" 200 - "-" "-" 0 178 4 3 "10.244.0.0" "curl/7.29.0" "f7372ae8-b2b5-9c69-864e-0e43a0ae0fcc" "192.168.56.100:30175" "10.244.2.81:9080" outbound|9080||details.default.svc.cluster.local - 10.244.1.55:80 10.244.0.0:51378 - -
[2020-08-24T11:04:23.808Z] "GET /details/0 HTTP/1.1" 404 NR "-" "-" 0 0 1 - "10.244.0.0" "curl/7.29.0" "4eb92e14-cb2f-9db7-8625-8b2d08a4f8b3" "192.168.56.100:30175" "-" - - 10.244.1.55:80 10.244.0.0:12594 - -
[2020-08-24T11:04:33.840Z] "GET /health HTTP/1.1" 404 NR "-" "-" 0 0 0 - "10.244.0.0" "curl/7.29.0" "0b19d6a9-cc51-9046-86e7-1f3bc2cefa74" "192.168.56.100:30175" "-" - - 10.244.1.55:80 10.244.0.0:27537 - -
[2020-08-24T11:05:10.067Z] "HEAD /health HTTP/1.1" 404 NR "-" "-" 0 0 0 - "10.244.0.0" "curl/7.29.0" "da3f4843-e35b-909f-84df-4717c72906df" "192.168.56.100:30175" "-" - - 10.244.1.55:80 10.244.0.0:8550 - -
[2020-08-24T11:05:47.794Z] "HEAD /health HTTP/1.1" 404 NR "-" "-" 0 0 0 - "10.244.0.0" "curl/7.29.0" "08396bff-4644-938f-b7b2-7f5aef1c6276" "192.168.56.100:30175" "-" - - 10.244.1.55:80 10.244.0.0:1479 - -
[2020-08-24T11:05:59.980Z] "HEAD /health HTTP/1.1" 404 NR "-" "-" 0 0 0 - "10.244.0.0" "curl/7.29.0" "b9e4d30e-61fa-9e34-9586-968018d628e2" "192.168.56.100:30175" "-" - - 10.244.1.55:80 10.244.0.0:9292 - -
[2020-08-24T11:07:49.208Z] "HEAD /health HTTP/1.1" 200 - "-" "-" 0 0 4 3 "10.244.0.0" "curl/7.29.0" "21453292-1e0d-900e-a8a3-1a304a2a2c2c" "abc.k8s.com" "10.244.2.81:9080" outbound|9080||details.default.svc.cluster.local - 10.244.1.55:80 10.244.0.0:8394 - -
```

