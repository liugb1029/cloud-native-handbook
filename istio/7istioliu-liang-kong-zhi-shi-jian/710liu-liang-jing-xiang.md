# 流量镜像 {#title}

此任务演示了 Istio 的流量镜像功能。

流量镜像，也称为影子流量，是一个以尽可能低的风险为生产带来变化的强大的功能。镜像会将实时流量的副本发送到镜像服务。镜像流量发生在主服务的关键请求路径之外。

在此任务中，首先把流量全部路由到`v1`版本的测试服务。然后，执行规则将一部分流量镜像到`v2`版本。

## 开始之前 {#before-you-begin}

* 按照[安装指南](https://istio.io/latest/zh/docs/setup/)中的说明设置 Istio。

* 首先部署两个版本的[httpbin](https://github.com/istio/istio/tree/release-1.7/samples/httpbin)服务，[httpbin](https://github.com/istio/istio/tree/release-1.7/samples/httpbin)服务已开启访问日志：

  **httpbin-v1:**

  ```
  [root@master ~]# kubectl apply -f - <<EOF
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: httpbin-v1
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: httpbin
        version: v1
    template:
      metadata:
        labels:
          app: httpbin
          version: v1
      spec:
        containers:
        - image: docker.io/kennethreitz/httpbin
          imagePullPolicy: IfNotPresent
          name: httpbin
          command: ["gunicorn", "--access-logfile", "-", "-b", "0.0.0.0:80", "httpbin:app"]
          ports:
          - containerPort: 80
  EOF
  ```

  **httpbin-v2:**

  ```
  [root@master ~]# kubectl apply -f - <<EOF
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: httpbin-v2
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: httpbin
        version: v2
    template:
      metadata:
        labels:
          app: httpbin
          version: v2
      spec:
        containers:
        - image: docker.io/kennethreitz/httpbin
          imagePullPolicy: IfNotPresent
          name: httpbin
          command: ["gunicorn", "--access-logfile", "-", "-b", "0.0.0.0:80", "httpbin:app"]
          ports:
          - containerPort: 80
  EOF
  ```

  **httpbin Kubernetes service:**

  ```bash
  [root@master ~]# kubectl apply -f - <<EOF
  apiVersion: v1
  kind: Service
  metadata:
    name: httpbin
    labels:
      app: httpbin
  spec:
    ports:
    - name: http
      port: 8000
      targetPort: 80
    selector:
      app: httpbin

  EOF
  ```

* 启动`sleep`服务，这样就可以使用`curl`来提供负载了：

  **sleep service:**

  ```bash
  $ cat <<EOF | istioctl kube-inject -f - | kubectl create -f -
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: sleep
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: sleep
    template:
      metadata:
        labels:
          app: sleep
      spec:
        containers:
        - name: sleep
          image: tutum/curl
          command: ["/bin/sleep","infinity"]
          imagePullPolicy: IfNotPresent
  EOF
  ```

## 创建一个默认路由策略 {#creating-a-default-routing-policy}

默认情况下，Kubernetes 在`httpbin`服务的两个版本之间进行负载均衡。在此步骤中会更改该行为，把所有流量都路由到`v1`。

1. 创建一个默认路由规则，将所有流量路由到服务的`v1`：如果安装/配置 Istio 的时候开启了 TLS 认证，在应用  
   `DestinationRule`之前必须将 TLS 流量策略`mode: ISTIO_MUTUAL`添加到`DestinationRule`。否则，请求将发生 503 错误，如[设置目标规则后出现 503 错误](https://istio.io/latest/zh/docs/ops/common-problems/network-issues/#service-unavailable-errors-after-setting-destination-rule)所述。

   ```bash
   [root@master ~]# kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: VirtualService
   metadata:
     name: httpbin
   spec:
     hosts:
       - httpbin
     http:
     - route:
       - destination:
           host: httpbin
           subset: v1
         weight: 100
   ---
   apiVersion: networking.istio.io/v1alpha3
   kind: DestinationRule
   metadata:
     name: httpbin
   spec:
     host: httpbin
     subsets:
     - name: v1
       labels:
         version: v1
     - name: v2
       labels:
         version: v2
   EOF
   ```

   现在所有流量都转到`httpbin:v1`服务。

2. 向服务发送一下流量：

   ```bash
   [root@master ~]# export SLEEP_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})
   [root@master ~]# kubectl exec -it $SLEEP_POD -c sleep -- sh -c 'curl  http://httpbin:8000/headers'
   {
     "headers": {
       "Accept": "*/*",
       "Content-Length": "0",
       "Host": "httpbin:8000",
       "User-Agent": "curl/7.69.1",
       "X-B3-Parentspanid": "a032b7abde529568",
       "X-B3-Sampled": "1",
       "X-B3-Spanid": "12fcc5644aad052c",
       "X-B3-Traceid": "a02c7f6b40599516a032b7abde529568"
     }
   }
   ```

3. 分别查看`httpbin`服务`v1`和`v2`两个 pods 的日志，您可以看到访问日志进入`v1`，而`v2`中没有日志，显示为`<none>`：

   ```bash
   [root@master ~]# kubectl logs -f httpbin-v1-79d49d4dc6-4gxlq -c httpbin
   [2020-09-22 03:40:31 +0000] [1] [INFO] Starting gunicorn 19.9.0
   [2020-09-22 03:40:31 +0000] [1] [INFO] Listening at: http://0.0.0.0:80 (1)
   [2020-09-22 03:40:31 +0000] [1] [INFO] Using worker: sync
   [2020-09-22 03:40:31 +0000] [8] [INFO] Booting worker with pid: 8
   127.0.0.1 - - [22/Sep/2020:03:44:53 +0000] "GET /headers HTTP/1.1" 200 303 "-" "curl/7.69.1"
   ```

   ```bash
   [root@master ~]# kubectl logs -f httpbin-v2-878b798b5-6rzq9 -c httpbin
   [2020-09-22 03:41:04 +0000] [1] [INFO] Starting gunicorn 19.9.0
   [2020-09-22 03:41:04 +0000] [1] [INFO] Listening at: http://0.0.0.0:80 (1)
   [2020-09-22 03:41:04 +0000] [1] [INFO] Using worker: sync
   [2020-09-22 03:41:04 +0000] [9] [INFO] Booting worker with pid: 9
   ```

## 镜像流量到 v2 {#mirroring-traffic-to-v2}

1. 改变流量规则将流量镜像到 v2：

   ```bash
   [root@master ~]# kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: VirtualService
   metadata:
     name: httpbin
   spec:
     hosts:
       - httpbin
     http:
     - route:
       - destination:
           host: httpbin
           subset: v1
         weight: 100
       mirror:
         host: httpbin
         subset: v2
       mirror_percent: 100
   EOF
   [root@master ~]# kubectl get virtualservices.networking.istio.io httpbin -oyaml
   apiVersion: networking.istio.io/v1alpha3
   kind: VirtualService
   metadata:
     name: httpbin
     namespace: default
   spec:
     hosts:
     - httpbin
     http:
     - mirror:
         host: httpbin
         subset: v2
       mirror_percent: 100
       route:
       - destination:
           host: httpbin
           subset: v1
         weight: 100
   ```

   这个路由规则发送 100% 流量到`v1`。最后一段表示你将镜像流量到`httpbin:v2`服务。当流量被镜像时，请求将发送到镜像服务中，并在`headers`中的`Host/Authority`属性值上追加`-shadow`。例如`cluster-1`变为`cluster-1-shadow`。

   此外，重点注意这些被镜像的流量是『 即发即弃』 的，就是说镜像请求的响应会被丢弃。

   您可以使用`mirror_percent`属性来设置镜像流量的百分比，而不是镜像全部请求。为了兼容老版本，如果这个属性不存在，将镜像所有流量。

2. 发送流量：

```bash
[root@master ~]# kubectl exec -it $SLEEP_POD -c sleep -- sh -c 'curl  http://httpbin:8000/headers'
{
  "headers": {
    "Accept": "*/*",
    "Content-Length": "0",
    "Host": "httpbin:8000",
    "User-Agent": "curl/7.69.1",
    "X-B3-Parentspanid": "c217c8eaa24bd195",
    "X-B3-Sampled": "1",
    "X-B3-Spanid": "5717897858892847",
    "X-B3-Traceid": "d63cd704dd29f3a6c217c8eaa24bd195"
  }
}
```

      现在就可以看到`v1`和`v2`中都有了访问日志。v2 中的访问日志就是由镜像流量产生的，这些请求的实际目标是 v1。

```
[root@master ~]# kubectl logs -f httpbin-v1-79d49d4dc6-4gxlq -c httpbin
[2020-09-22 03:40:31 +0000] [1] [INFO] Starting gunicorn 19.9.0
[2020-09-22 03:40:31 +0000] [1] [INFO] Listening at: http://0.0.0.0:80 (1)
[2020-09-22 03:40:31 +0000] [1] [INFO] Using worker: sync
[2020-09-22 03:40:31 +0000] [8] [INFO] Booting worker with pid: 8
127.0.0.1 - - [22/Sep/2020:03:44:53 +0000] "GET /headers HTTP/1.1" 200 303 "-" "curl/7.69.1"
127.0.0.1 - - [22/Sep/2020:03:49:12 +0000] "GET /headers HTTP/1.1" 200 303 "-" "curl/7.69.1"



127.0.0.1 - - [22/Sep/2020:03:50:19 +0000] "GET /headers HTTP/1.1" 200 303 "-" "curl/7.69.1"

[root@master ~]# kubectl logs -f httpbin-v2-878b798b5-6rzq9 -c httpbin
[2020-09-22 03:41:04 +0000] [1] [INFO] Starting gunicorn 19.9.0
[2020-09-22 03:41:04 +0000] [1] [INFO] Listening at: http://0.0.0.0:80 (1)
[2020-09-22 03:41:04 +0000] [1] [INFO] Using worker: sync
[2020-09-22 03:41:04 +0000] [9] [INFO] Booting worker with pid: 9
127.0.0.1 - - [22/Sep/2020:03:49:10 +0000] "GET /headers HTTP/1.1" 200 343 "-" "curl/7.69.1"
127.0.0.1 - - [22/Sep/2020:03:49:11 +0000] "GET /headers HTTP/1.1" 200 343 "-" "curl/7.69.1"
127.0.0.1 - - [22/Sep/2020:03:49:12 +0000] "GET /headers HTTP/1.1" 200 343 "-" "curl/7.69.1"


127.0.0.1 - - [22/Sep/2020:03:50:19 +0000] "GET /headers HTTP/1.1" 200 343 "-" "curl/7.69.1"
127.0.0.1 - - [22/Sep/2020:03:50:21 +0000] "GET /headers HTTP/1.1" 200 343 "-" "curl/7.69.1"
```

![](/image/Istio/TrafficMirror配置分析.png)

