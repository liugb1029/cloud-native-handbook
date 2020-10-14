#### 调试工具和方法

* istioctl 命令行
* controlZ 自检工具
* Envoy admin 接口
* Pilot debug 接口

#### istioctl 命令行

```bash
[root@master envoy]# istioctl -h
Istio configuration command line utility for service operators to
debug and diagnose their Istio mesh.

Usage:
  istioctl [command]

Available Commands:
  analyze         Analyze Istio configuration and print validation messages
  authz           (authz is experimental. Use `istioctl experimental authz`)
  convert-ingress Convert Ingress configuration into Istio VirtualService configuration
  dashboard       Access to Istio web UIs
  deregister      De-registers a service instance
  experimental    Experimental commands that may be modified or deprecated
  help            Help about any command
  install         Applies an Istio manifest, installing or reconfiguring Istio on a cluster.
  kube-inject     Inject Envoy sidecar into Kubernetes pod resources
  manifest        Commands related to Istio manifests
  operator        Commands related to Istio operator controller.
  profile         Commands related to Istio configuration profiles
  proxy-config    Retrieve information about proxy configuration from Envoy [kube only]
  proxy-status    Retrieves the synchronization status of each Envoy in the mesh [kube only]
  register        Registers a service instance (e.g. VM) joining the mesh
  upgrade         Upgrade Istio control plane in-place
  validate        Validate Istio policy and rules files
  verify-install  Verifies Istio Installation Status
  version         Prints out build version information

Flags:
      --context string          The name of the kubeconfig context to use
  -h, --help                    help for istioctl
  -i, --istioNamespace string   Istio system namespace (default "istio-system")
  -c, --kubeconfig string       Kubernetes configuration file
  -n, --namespace string        Config namespace

Additional help topics:
  istioctl options         Displays istioctl global options

Use "istioctl [command] --help" for more information about a command.
```

##### 安装部署相关

* istioctl verify-install
* istioctl manifest \[apply / diff / generate / migrate / versions\]
* istioctl profile \[list / diff / dump\]
* istioctl kube-inject
* istioctl dashboard \[command\]
* controlz / envoy / Grafana / jaeger / kiali / Prometheus / zipkin

##### 网格配置状态检查

* 配置同步检查

  * istioctl ps（proxy-status）

  * 状态：SYNCED / NOT SENT / STALE

  * istioctl ps &lt;pod-name&gt;

* 配置详情

  * istioctl pc（proxy-config）

  * istioctl pc \[cluster/route/…\] &lt;pod-name.namespace&gt;

```bash
[root@master envoy]# istioctl ps
NAME                                                   CDS        LDS        EDS        RDS          ISTIOD                      VERSION
details-v1-558b8b4b76-jrfd4.default                    SYNCED     SYNCED     SYNCED     SYNCED       istiod-77df9b78f8-pkl2q     1.7.2
httpbin-6b7bd6467b-6l2rx.testjwt                       SYNCED     SYNCED     SYNCED     SYNCED       istiod-77df9b78f8-pkl2q     1.7.2
httpbin-6b7bd6467b-p98lg.demo                          SYNCED     SYNCED     SYNCED     SYNCED       istiod-77df9b78f8-pkl2q     1.7.2
httpbin-v2-56b746ddf7-54mnk.demo                       SYNCED     SYNCED     SYNCED     SYNCED       istiod-77df9b78f8-pkl2q     1.7.2
istio-egressgateway-fbb7dc4f4-wrw2s.istio-system       SYNCED     SYNCED     SYNCED     NOT SENT     istiod-77df9b78f8-pkl2q     1.7.2
istio-ingressgateway-5f84fcdd69-qmg7z.istio-system     SYNCED     SYNCED     SYNCED     SYNCED       istiod-77df9b78f8-pkl2q     1.7.2
productpage-v1-6987489c74-6fgz8.default                SYNCED     SYNCED     SYNCED     SYNCED       istiod-77df9b78f8-pkl2q     1.7.2
ratings-v1-7dc98c7588-x9gtt.default                    SYNCED     SYNCED     SYNCED     SYNCED       istiod-77df9b78f8-pkl2q     1.7.2
reviews-v1-7f99cc4496-gzfrb.default                    SYNCED     SYNCED     SYNCED     SYNCED       istiod-77df9b78f8-pkl2q     1.7.2
reviews-v2-7d79d5bd5d-wn2j8.default                    SYNCED     SYNCED     SYNCED     SYNCED       istiod-77df9b78f8-pkl2q     1.7.2
reviews-v3-7dbcdcbc56-p6gfg.default                    SYNCED     SYNCED     SYNCED     SYNCED       istiod-77df9b78f8-pkl2q     1.7.2
sleep-7584bc4cbd-sx6h5.testjwt                         SYNCED     SYNCED     SYNCED     SYNCED       istiod-77df9b78f8-pkl2q     1.7.2
sleep-8f795f47d-b4s69.demo                             SYNCED     SYNCED     SYNCED     SYNCED       istiod-77df9b78f8-pkl2q     1.7.2
sleep-8f795f47d-wpzpz.default                          SYNCED     SYNCED     SYNCED     SYNCED       istiod-77df9b78f8-pkl2q     1.7.2

[root@master envoy]# istioctl ps productpage-v1-6987489c74-6fgz8.default
Clusters Match
Listeners Match
Routes Match (RDS last loaded at Wed, 14 Oct 2020 14:27:17 CST)
[root@master envoy]#

[root@master envoy]# istioctl pc clusters ratings-v1-7dc98c7588-x9gtt.default
SERVICE FQDN                                                         PORT      SUBSET     DIRECTION     TYPE             DESTINATION RULE
BlackHoleCluster                                                     -         -          -             STATIC
InboundPassthroughClusterIpv4                                        -         -          -             ORIGINAL_DST
PassthroughCluster                                                   -         -          -             ORIGINAL_DST
agent                                                                -         -          -             STATIC
dashboard-metrics-scraper.kubernetes-dashboard.svc.cluster.local     8000      -          outbound      EDS
details.default.svc.cluster.local                                    9080      -          outbound      EDS              details.default
details.default.svc.cluster.local                                    9080      v1         outbound      EDS              details.default
fortio.default.svc.cluster.local                                     8080      -          outbound      EDS
grafana.istio-system.svc.cluster.local                               3000      -          outbound      EDS
httpbin-v2.demo.svc.cluster.local                                    8000      -          outbound      EDS
httpbin.demo.svc.cluster.local                                       8000      -          outbound      EDS
httpbin.testjwt.svc.cluster.local                                    8000      -          outbound      EDS
istio-egressgateway.istio-system.svc.cluster.local                   80        -          outbound      EDS
istio-egressgateway.istio-system.svc.cluster.local                   443       -          outbound      EDS
istio-egressgateway.istio-system.svc.cluster.local                   15443     -          outbound      EDS

[root@master envoy]# istioctl pc clusters ratings-v1-7dc98c7588-x9gtt.default --port 9080 --direction inbound
SERVICE FQDN                          PORT     SUBSET     DIRECTION     TYPE       DESTINATION RULE
ratings.default.svc.cluster.local     9080     http       inbound       STATIC
[root@master envoy]# istioctl pc clusters ratings-v1-7dc98c7588-x9gtt.default --port 9080 --direction inbound -ojson
[
    {
        "name": "inbound|9080|http|ratings.default.svc.cluster.local",
        "type": "STATIC",
        "connectTimeout": "10s",
        "loadAssignment": {
            "clusterName": "inbound|9080|http|ratings.default.svc.cluster.local",
            "endpoints": [
                {
                    "lbEndpoints": [
                        {
                            "endpoint": {
                                "address": {
                                    "socketAddress": {
                                        "address": "127.0.0.1",
                                        "portValue": 9080
                                    }
                                }
                            }
                        }
                    ]
                }
            ]
        },
        "circuitBreakers": {
            "thresholds": [
                {
                    "maxConnections": 4294967295,
                    "maxPendingRequests": 4294967295,
                    "maxRequests": 4294967295,
                    "maxRetries": 4294967295
                }
            ]
        }
    }
]
[root@master envoy]#
```

##### 查看 Pod 相关网格配置信息

`istioctl x（ experimental ）describe pod <pod-name>`

* 验证是否在网格内

* 验证 VirtualService

* 验证 DestinationRule

* 验证路由

```bash
[root@master envoy]# istioctl x describe pod productpage-v1-6987489c74-6fgz8
Pod: productpage-v1-6987489c74-6fgz8
   Pod Ports: 9080 (productpage), 15090 (istio-proxy)
--------------------
Service: productpage
   Port: http 9080/HTTP targets pod port 9080
DestinationRule: productpage for "productpage"
   Matching subsets: v1
   No Traffic Policy
VirtualService: productpage
   1 HTTP route(s)


Exposed on Ingress Gateway http://192.168.56.101
VirtualService: bookinfo
   /productpage, /static*, /login, /logout, /api/v1/products*

[root@master envoy]# istioctl x describe pod httpbin-6b7bd6467b-p98lg.demo
Pod: httpbin-6b7bd6467b-p98lg.demo
   Pod Ports: 80 (httpbin), 15090 (istio-proxy)
--------------------
Service: httpbin.demo
   Port: http 8000/HTTP targets pod port 80


Exposed on Ingress Gateway http://192.168.56.101
VirtualService: httpbin.demo
   1 HTTP route(s)

[root@master envoy]# istioctl x describe pod reviews-v1-7f99cc4496-gzfrb
Pod: reviews-v1-7f99cc4496-gzfrb
   Pod Ports: 9080 (reviews), 15090 (istio-proxy)
--------------------
Service: reviews
   Port: http 9080/HTTP targets pod port 9080
DestinationRule: reviews for "reviews"
   Matching subsets: v1
      (Non-matching subsets v2,v3)
   No Traffic Policy
VirtualService: reviews
   WARNING: No destinations match pod subsets (checked 1 HTTP routes)
      Route to non-matching subset v2 for (everything)
      Route to non-matching subset v3 for (everything)
```



