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



