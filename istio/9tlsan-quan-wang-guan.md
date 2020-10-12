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

### 案例---配置TLS安全网关

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

#### ![](/image/Istio/istio-mTLS握手.png) {#permissive-mode}

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

![](/image/Istio/istio-authn.png)Istio 将两种类型的身份验证以及凭证中的其他声明（如果适用）输出到下一层：[授权](https://istio.io/latest/zh/docs/concepts/security/#authorization)。

### 认证策略 {#authentication-policies}

本节中提供了更多 Istio 认证策略方面的细节。正如[认证架构](https://istio.io/latest/zh/docs/concepts/security/#authentication-architecture)中所说的，认证策略是对服务收到的请求生效的。要在双向 TLS 中指定客户端认证策略，需要在`DetinationRule`中设置`TLSSettings`。[TLS 设置参考文档](https://istio.io/latest/zh/docs/reference/config/networking/destination-rule/#TLSSettings)中有更多这方面的信息。

和其他的 Istio 配置一样，可以用`.yaml`文件的形式来编写认证策略。部署策略使用`kubectl`。 下面例子中的认证策略要求：与带有`app:reviews`标签的工作负载的传输层认证，必须使用双向 TLS：

```
apiVersion: "security.istio.io/v1beta1"
kind: "PeerAuthentication"
metadata:
  name: "example-peer-policy"
  namespace: "foo"
spec:
  selector:
    matchLabels:
      app: reviews
  mtls:
    mode: STRICT
```

#### 策略存储 {#policy-storage}

Istio 将网格范围的策略存储在根命名空间。这些策略使用一个空的 selector 适用于网格中的所有工作负载。具有名称空间范围的策略存储在相应的名称空间中。它们仅适用于其命名空间内的工作负载。如果你配置了`selector`字段，则认证策略仅适用于与您配置的条件匹配的工作负载。

Peer 和 request 认证策略用 kind 字段区分，分别是`PeerAuthentication`和`RequestAuthentication`。

#### Selector 字段 {#selector-field}

Peer 和 request 认证策略使用`selector`字段来指定该策略适用的工作负载的标签。以下示例显示适用于带有`app：product-page`标签的工作负载的策略的 selector 字段：

```
selector:
  matchLabels:
    app: product-page
```

如果您没有为`selector`字段提供值，则 Istio 会将策略与策略存储范围内的所有工作负载进行匹配。因此，`selector`字段可帮助您指定策略的范围：

* 网格范围策略：为根名称空间指定的策略，不带或带有空的`selector`字段。
* 命名空间范围的策略：为非root命名空间指定的策略，不带有或带有空的`selector`字段。
* 特定于工作负载的策略：在常规名称空间中定义的策略，带有非空`selector`字段。

Peer 和 request 认证策略对`selector`字段遵循相同的层次结构原则，但是 Istio 以略有不同的方式组合和应用它们。

只能有一个网格范围的 Peer 认证策略，每个命名空间也只能有一个命名空间范围的 Peer 认证策略。当您为同一网格或命名空间配置多个网格范围或命名空间范围的 Peer 认证策略时，Istio 会忽略较新的策略。当多个特定于工作负载的 Peer 认证策略匹配时，Istio 将选择最旧的策略。

Istio 按照以下顺序为每个工作负载应用最窄的匹配策略：

1. 特定于工作负载的
2. 命名空间范围
3. 网格范围

Istio 可以将所有匹配的 request 认证策略组合起来，就像它们来自单个 request 认证策略一样。因此，您可以在网格或名称空间中配置多个网格范围或命名空间范围的策略。但是，避免使用多个网格范围或命名空间范围的 request 认证策略仍然是一个好的实践。

#### Peer authentication {#peer-authentication}

Peer 认证策略指定 Istio 对目标工作负载实施的双向 TLS 模式。支持以下模式：

* PERMISSIVE：工作负载接受双向 TLS 和纯文本流量。此模式在迁移因为没有 sidecar 而无法使用双向 TLS 的工作负载的过程中非常有用。一旦工作负载完成 sidecar 注入的迁移，应将模式切换为 STRICT。
* STRICT： 工作负载仅接收双向 TLS 流量。
* DISABLE：禁用双向 TLS。 从安全角度来看，除非您提供自己的安全解决方案，否则请勿使用此模式。

如果未设置模式，将继承父作用域的模式。未设置模式的网格范围的 peer 认证策略默认使用`PERMISSIVE`模式。

下面的 peer 认证策略要求命名空间`foo`中的所有工作负载都使用双向 TLS：

```
apiVersion: "security.istio.io/v1beta1"
kind: "PeerAuthentication"
metadata:
  name: "example-policy"
  namespace: "foo"
spec:
  mtls:
    mode: STRICT
```

对于特定于工作负载的 peer 认证策略，可以为不同的端口指定不同的双向 TLS 模式。您只能将工作负载声明过的端口用于端口范围的双向 TLS 配置。以下示例为`app:example-app`工作负载禁用了端口80上的双向TLS，并对所有其他端口使用名称空间范围的 peer 认证策略的双向 TLS 设置：

```
apiVersion: "security.istio.io/v1beta1"
kind: "PeerAuthentication"
metadata:
  name: "example-workload-policy"
  namespace: "foo"
spec:
  selector:
     matchLabels:
       app: example-app
  portLevelMtls:
    80:
      mode: DISABLE
```

上面的 peer 认证策略仅在有如下 Service 定义时工作，将流向`example-service`服务的请求绑定到`example-app`工作负载的

`80`端口。

```
apiVersion: v1
kind: Service
metadata:
  name: example-service
  namespace: foo
spec:
  ports:
  - name: http
    port: 8000
    protocol: TCP
    targetPort: 80
  selector:
    app: example-app
```

#### Request authentication {#request-authentication}

Request 认证策略指定验证 JSON Web Token（JWT）所需的值。 这些值包括：

* token 在请求中的位置
* 请求的 issuer
* 公共 JSON Web Key Set（JWKS）

Istio 会根据 request 认证策略中的规则检查提供的令牌（如果已提供），并拒绝令牌无效的请求。当请求不带有令牌时，默认情况下将接受它们。要拒绝没有令牌的请求，请提供授权规则，该规则指定对特定操作（例如，路径或操作）的限制。

如果 Request 认证策略使用唯一的位置，则它们可以指定多个JWT。当多个策略与工作负载匹配时，Istio 会将所有规则组合起来，就好像它们被指定为单个策略一样。此行为对于开发接受来自不同 JWT 提供者的工作负载时很有用。但是，不支持具有多个有效 JWT 的请求，因为此类请求的输出主体未定义。

#### Principals {#principals}

使用 peer 认证策略和双向 TLS 时，Istio 将身份从 peer 认证提取到`source.principal`中。同样，当您使用 request 认证策略时，Istio 会将 JWT 中的身份赋值给`request.auth.principal`。使用这些 principals 设置授权策略和作为遥测的输出。

### 更新认证策略 {#updating-authentication-policies}

您可以随时更改认证策略，Istio 几乎实时将新策略推送到工作负载。但是，Istio 无法保证所有工作负载都同时收到新政策。以下建议有助于避免在更新认证策略时造成干扰：

* 将 peer 认证策略的模式从`DISABLE`更改为`STRICT`时，请使用`PERMISSIVE`模式来过渡，反之亦然。当所有工作负载成功切换到所需模式时，您可以将策略应用于最终模式。您可以使用 Istio 遥测技术来验证工作负载已成功切换。
* 将 request 认证策略从一个 JWT 迁移到另一个 JWT 时，将新 JWT 的规则添加到该策略中，而不删除旧规则。这样，工作负载接受两种类型的 JWT，当所有流量都切换到新的 JWT 时，您可以删除旧规则。但是，每个 JWT 必须使用不同的位置。

### 案例---开启mTLS



