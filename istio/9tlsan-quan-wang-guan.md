#### Istio安全

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

#### 安全发现服务（SDS）

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

#### 配置TLS安全网关

```
1.为服务创建根证书和私钥：
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=example Inc./CN=example.com' -keyout example.com.key -out example.com.crt

2.为httpbin.example.com创建证书和私钥：
openssl req -out httpbin.example.com.csr -newkey rsa:2048 -nodes -keyout httpbin.example.com.key -subj "/CN=httpbin.example.com/O=httpbin organization"
openssl x509 -req -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 0 -in httpbin.example.com.csr -out httpbin.example.com.crt

3. 创建secret
kubectl create -n istio-system secret tls httpbin-credential --key=httpbin.example.com.key --cert=httpbin.example.com.crt

4.定义网关,vs，见yaml

5. 请求验证
curl -HHost:httpbin.example.com \
--resolve httpbin.example.com:443:127.0.0.1 \
--cacert example.com.crt "https://httpbin.example.com:443/status/418"

curl -v -HHost:httpbin.example.com --cacert example.com.crt https://httpbin.example.com:31264/status/418
```



