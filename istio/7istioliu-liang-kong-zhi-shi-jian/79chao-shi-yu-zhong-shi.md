### 超时与重试： 提升系统的健壮性和可用性

![](/image/Istio/bookinfo-delay-retry.png)1、由于只有reviews-v2和v3才会调用ratings服务，所以我们先将流量引导到reviews-v2

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

2、这时访问productpage页面应该是黑色星星





3、

