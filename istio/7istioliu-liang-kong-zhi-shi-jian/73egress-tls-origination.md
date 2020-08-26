

# Egress TLS Origination {#title}



[控制 Egress 流量](https://istio.io/latest/zh/docs/tasks/traffic-management/egress/)的任务向我们展示了位于服务网格内部的应用应如何访问外部（即服务网格之外）的 HTTP 和 HTTPS 服务。 正如该任务所述，[`ServiceEntry`](https://istio.io/latest/zh/docs/reference/config/networking/service-entry/)用于配置 Istio 以受控的方式访问外部服务。 本示例将演示如何通过配置 Istio 去实现对发往外部服务的流量的TLS origination。 若此时原始的流量为 HTTP，则 Istio 会将其转换为 HTTPS 连接。

## 案例 {#use-case}

假设有一个遗留应用正在使用 HTTP 和外部服务进行通信。而运行该应用的组织却收到了一个新的需求，该需求要求必须对所有外部的流量进行加密。 此时，使用 Istio 便可通过修改配置实现此需求，而无需更改应用中的任何代码。 该应用可以发送未加密的 HTTP 请求，然后 Istio 将为应用加密请求。

从应用源头发送未加密的 HTTP 请求并让 Istio 执行 TSL 升级的另一个好处是可以产生更好的遥测并为未加密的请求提供更多的路由控制。

## 开始之前

启动[sleep](https://github.com/istio/istio/tree/release-1.7/samples/sleep)示例，该示例将用作外部调用的测试源

```
kubectl apply -f samples/sleep/sleep.yaml
```

## 配置对外部服务的访问 {#configuring-access-to-an-external-service}

首先，使用与[Egress 流量控制](https://istio.io/latest/zh/docs/tasks/traffic-management/egress/)任务中的相同的技术，配置对外部服务`edition.cnn.com`的访问。 但这一次我们将使用单个`ServiceEntry`来启用对服务的 HTTP 和 HTTPS 访问。

1. 创建一个`ServiceEntry`和`VirtualService`以启用对`edition.cnn.com`的访问：

```
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



