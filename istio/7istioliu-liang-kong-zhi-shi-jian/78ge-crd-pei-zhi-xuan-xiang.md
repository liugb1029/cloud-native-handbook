### 流量管理CRD配置选项

#### VirtualService和DestinationRule

![](/image/Istio/VirtualService配置选项.png)
![](/image/Istio/VirtualService-example.png)
![](/image/Istio/VirtualService-headers-example.png)
![](/image/Istio/VirtualService-灰度发布.png)

#### ServiceEntry

![](/image/Istio/ServiceEntry配置选项.png)
![](/image/Istio/ServiceEntry.png)
![](/image/Istio/ServiceEntry-example.png)

#### Ingress GateWay

Gateway与对应的服务的VirtualService绑定![](/image/Istio/Gateway配置选项.png)![](/image/Istio/Gateway-ingress-example.png)

详细配置：

```bash
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: bookinfo-gateway
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
  name: bookinfo
  namespace: default
spec:
  gateways:
  - bookinfo-gateway
  hosts:
  - '*'
  http:
  - match:
    - uri:
        exact: /productpage
    - uri:
        prefix: /static
    - uri:
        exact: /login
    - uri:
        exact: /logout
    - uri:
        prefix: /api/v1/products
    route:
    - destination:
        host: productpage
        port:
          number: 9080
```

![](/image/Istio/gateway-route-bookinfo.png)

```
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: httpbin-gateway
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
  name: httpbin
  namespace: default
spec:
  gateways:
  - httpbin-gateway
  hosts:
  - '*'
  http:
  - match:
    - uri:
        prefix: /status
    - uri:
        prefix: /delay
    route:
    - destination:
        host: httpbin
        port:
          number: 8000
```

![](/image/Istio/gateway-route-httpbin.png)

```
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
    - abc.k8s.com
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
  - abc.k8s.com
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

![](/image/Istio/gateway-route-details.png)

