### Istio安全

Istio 安全功能提供强大的身份，强大的策略，透明的 TLS 加密，认证，授权和审计（AAA）工具来保护你的服务和数据。Istio 安全的目标是：

* 默认安全：应用程序代码和基础设施无需更改
* 深度防御：与现有安全系统集成以提供多层防御
* 零信任网络：在不受信任的网络上构建安全解决方案

Istio 中的安全性涉及多个组件：

* 用于密钥和证书管理的证书颁发机构（CA）
* 配置 API 服务器分发给代理：

  * [认证策略](https://istio.io/latest/zh/docs/concepts/security/#authentication-policies)
  * [授权策略](https://istio.io/latest/zh/docs/concepts/security/#authorization-policies)
  * [安全命名信息](https://istio.io/latest/zh/docs/concepts/security/#secure-naming)

* Sidecar 和边缘代理作为[Policy Enforcement Points](https://www.jerichosystems.com/technology/glossaryterms/policy_enforcement_point.html)\(PEPs\) 以保护客户端和服务器之间的通信安全.

* 一组 Envoy 代理扩展，用于管理遥测和审计

控制面处理来自 API server 的配置，并且在数据面中配置 PEPs。PEPs 用 Envoy 实现。下图显示了架构。![](/image/Istio/arch-sec.png)

### 安全发现服务（SDS）

* 身份和证书管理
* 实现安全配置自动化
* 中心化 SDS Server
* 优点：

  * 无需挂载 secret 卷

  * 动态更新证书，无需重启

  * 可监视多个证书密钥对

![](/image/Istio/TLS.png)

Istio PKI 使用 X.509 证书为每个工作负载都提供强大的身份标识。可以大规模进行自动化密钥和证书轮换，伴随每个 Envoy 代理都运行着一个`istio-agent`负责证书和密钥的供应。下图显示了这个机制的运行流程。

> 这里用`istio-agent`来表述，是因为下图及对图的相关解读中反复用到了 “Istio agent” 这个术语，这样的描述更容易理解。 另外，在实现层面，`istio-agent`是指 sidecar 容器中的`pilot-agent`进程，它有很多功能，这里不表，只特别提一下：它通过 Unix socket 的方式在本地提供 SDS 服务供 Envoy 使用，这个信息对了解 Envoy 与 SDS 之间的交互有意义。

Istio 供应身份是通过 secret discovery service（SDS）来实现的，具体流程如下：

1. CA 提供 gRPC 服务以接受[证书签名请求](https://en.wikipedia.org/wiki/Certificate_signing_request)（CSRs）。
2. Envoy 通过 Envoy 秘密发现服务（SDS）API 发送证书和密钥请求。
3. 在收到 SDS 请求后，`istio-agent`创建私钥和 CSR，然后将 CSR 及其凭据发送到 Istio CA 进行签名。
4. CA 验证 CSR 中携带的凭据并签署 CSR 以生成证书。
5. `Istio-agent`通过 Envoy SDS API 将私钥和从 Istio CA 收到的证书发送给 Envoy。
6. 上述 CSR 过程会周期性地重复，以处理证书和密钥轮换。

### 配置TLS安全网关

```
1.为服务创建根证书和私钥：
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=example Inc./CN=example.com' -keyout example.com.key -out example.com.crt

2.为httpbin.example.com创建证书和私钥：
openssl req -out httpbin.example.com.csr -newkey rsa:2048 -nodes -keyout httpbin.example.com.key -subj "/CN=httpbin.example.com/O=httpbin organization"
openssl x509 -req -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 0 -in httpbin.example.com.csr -out httpbin.example.com.crt

3. 创建secret
kubectl create -n istio-system secret tls httpbin-credential --key=httpbin.example.com.key --cert=httpbin.example.com.crt

4.定义网关,vs，见yaml
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
  selector:
    app: httpbin
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpbin
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
      - image: docker.io/citizenstig/httpbin
        imagePullPolicy: IfNotPresent
        name: httpbin
        ports:
        - containerPort: 8000
---
apiVersion:  networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
  - "httpbin.example.com"
  gateways:
  - mygateway
  http:
  - match:
    - uri:
        prefix: /status
    - uri:
        prefix: /delay
    route:
    - destination:
        port:
          number: 8000
        host: httpbin    
--- 
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: mygateway
spec:
  selector:
    istio: ingressgateway 
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: httpbin-credential
    hosts:
    - httpbin.example.com

5. 请求验证
curl -HHost:httpbin.example.com \
--resolve httpbin.example.com:443:127.0.0.1 \
--cacert example.com.crt "https://httpbin.example.com:443/status/418"

curl -v -HHost:httpbin.example.com --cacert example.com.crt https://httpbin.example.com:31264/status/418
```

### 认证

Istio 提供两种类型的认证：

* Peer authentication：用于服务到服务的认证，以验证进行连接的客户端。Istio 提供[双向 TLS](https://en.wikipedia.org/wiki/Mutual_authentication)作为传输认证的全栈解决方案，无需更改服务代码就可以启用它。这个解决方案：

  * 为每个服务提供强大的身份，表示其角色，以实现跨群集和云的互操作性。
  * 保护服务到服务的通信。
  * 提供密钥管理系统，以自动进行密钥和证书的生成，分发和轮换。

* Request authentication：用于最终用户认证，以验证附加到请求的凭据。 Istio 使用 JSON Web Token（JWT）验证启用请求级认证，并使用自定义认证实现或任何 OpenID Connect 的认证实现（例如下面列举的）来简化的开发人员体验。

  * [ORY Hydra](https://www.ory.sh/)
  * [Keycloak](https://www.keycloak.org/)
  * [Auth0](https://auth0.com/)
  * [Firebase Auth](https://firebase.google.com/docs/auth/)
  * [Google Auth](https://developers.google.com/identity/protocols/OpenIDConnect)

在所有情况下，Istio 都通过自定义 Kubernetes API 将认证策略存储在`Istio config store`。Istiod使每个代理保持最新状态，并在适当时提供密钥。此外，Istio 的认证机制支持宽容模式（permissive mode），以帮助您了解策略更改在实施之前如何影响您的安全状况。

### 双向 TLS 认证 {#mutual-TLS-authentication}

Istio 通过客户端和服务器端 PEPs 建立服务到服务的通信通道，PEPs 被实现为[Envoy 代理](https://envoyproxy.github.io/envoy/)。当一个工作负载使用双向 TLS 认证向另一个工作负载发送请求时，该请求的处理方式如下：

1. Istio 将出站流量从客户端重新路由到客户端的本地 sidecar Envoy。
2. 客户端 Envoy 与服务器端 Envoy 开始双向 TLS 握手。在握手期间，客户端 Envoy 还做了
   [安全命名](https://istio.io/latest/zh/docs/concepts/security/#secure-naming)
   检查，以验证服务器证书中显示的服务帐户是否被授权运行目标服务。
3. 客户端 Envoy 和服务器端 Envoy 建立了一个双向的 TLS 连接，Istio 将流量从客户端 Envoy 转发到服务器端 Envoy。
4. 授权后，服务器端 Envoy 通过本地 TCP 连接将流量转发到服务器服务。

#### 宽容模式 {#permissive-mode}

Istio 双向 TLS 具有一个宽容模式（permissive mode），允许服务同时接受纯文本流量和双向 TLS 流量。这个功能极大的提升了双向 TLS 的入门体验。

在运维人员希望将服务移植到启用了双向 TLS 的 Istio 上时，许多非 Istio 客户端和非 Istio 服务端通信时会产生问题。通常情况下，运维人员无法同时为所有客户端安装 Istio sidecar，甚至没有这样做的权限。即使在服务端上安装了 Istio sidecar，运维人员也无法在不中断现有连接的情况下启用双向 TLS。

启用宽容模式后，服务可以同时接受纯文本和双向 TLS 流量。这个模式为入门提供了极大的灵活性。服务中安装的 Istio sidecar 立即接受双向 TLS 流量而不会打断现有的纯文本流量。因此，运维人员可以逐步安装和配置客户端 Istio sidecar 发送双向 TLS 流量。一旦客户端配置完成，运维人员便可以将服务端配置为仅 TLS 模式。更多信息请访问[双向 TLS 迁移向导](https://istio.io/latest/zh/docs/tasks/security/authentication/mtls-migration)。



#### 安全命名 {#secure-naming}

服务器身份（Server identities）被编码在证书里，但服务名称（service names）通过服务发现或 DNS 被检索。安全命名信息将服务器身份映射到服务名称。身份`A`到服务名称`B`的映射表示“授权`A`运行服务`B`“。控制平面监视`apiserver`，生成安全命名映射，并将其安全地分发到 PEPs。 以下示例说明了为什么安全命名对身份验证至关重要。

假设运行服务`datastore`的合法服务器仅使用`infra-team`身份。恶意用户拥有`test-team`身份的证书和密钥。恶意用户打算模拟服务以检查从客户端发送的数据。恶意用户使用证书和`test-team`身份的密钥部署伪造服务器。假设恶意用户成功攻击了发现服务或 DNS，以将`datastore`服务名称映射到伪造服务器。

当客户端调用`datastore`服务时，它从服务器的证书中提取`test-team`身份，并用安全命名信息检查`test-team`是否被允许运行`datastore`。客户端检测到`test-team`不允许运行`datastore`服务，认证失败。



### 认证架构 {#authentication-architecture}

您可以使用 peer 和 request 认证策略为在 Istio 网格中接收请求的工作负载指定认证要求。网格运维人员使用`.yaml`文件来指定策略。部署后，策略将保存在 Istio 配置存储中。Istio 控制器监视配置存储。

一有任何的策略变更，新策略都会转换为适当的配置，告知 PEP 如何执行所需的认证机制。控制平面可以获取公共密钥，并将其附加到配置中以进行 JWT 验证。或者，Istiod 提供了 Istio 系统管理的密钥和证书的路径，并将它们安装到应用程序 pod 用于双向 TLS。您可以在[PKI 部分](https://istio.io/latest/zh/docs/concepts/security/#PKI)中找到更多信息。

Istio 异步发送配置到目标端点。代理收到配置后，新的认证要求会立即生效。

发送请求的客户端服务负责遵循必要的认证机制。对于 peer authentication，应用程序负责获取 JWT 凭证并将其附加到请求。对于双向 TLS，Istio 会自动将两个 PEPs 之间的所有流量升级为双向 TLS。如果认证策略禁用了双向 TLS 模式，则 Istio 将继续在 PEPs 之间使用纯文本。要覆盖此行为，请使用[destination rules](https://istio.io/latest/zh/docs/concepts/traffic-management/#destination-rules)显式禁用双向 TLS 模式。您可以在[双向 TLS 认证](https://istio.io/latest/zh/docs/concepts/security/#mutual-TLS-authentication)中找到有关双向 TLS 如何工作的更多信息。

![](/image/Istio/istio-authn.png)



