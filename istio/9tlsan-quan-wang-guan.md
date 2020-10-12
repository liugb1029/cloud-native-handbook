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



