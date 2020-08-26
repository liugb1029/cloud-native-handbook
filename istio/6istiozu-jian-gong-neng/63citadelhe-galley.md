### istio-citadel {#istio-citadel}

服务列表中的istio-citadel 是Istio 的核心安全组件，提供了**自动生成、分发、轮换与撤销密钥和证书功能**；**强大的身份验证功能、强大的策略、透明的TLS加密**。Citadel 一直监听Kube-apiserver ，以Secret 的形式为每个服务都生成证书密钥，并在Pod 创建时挂载到Pod 上，代理容器使用这些文件来做服务身份认证，进而代理两端服务实现双向TLS认证、通道加密、访问授权等安全功能，这样用户就不用在代码里面维护证书密钥了。如下图 所示，frontend 服务对forecast 服务的访问用到了HTTP 方式，通过配置即可对服务增加认证功能， 双方的Envoy 会建立双向认证的TLS 通道，从而在服务间启用双向认证的HTTPS 。

![](/image/Istio/istio-citadel.png)

### istio-galley {#istio-galley}

istio-galley 并不直接向数据面提供业务能力，而是在控制面上向其他组件提供支持。**Galley 作为负责配置管理的组件，验证配置信息的格式和内容的正确性，**并将这些配置信息提供给管理面的Pilot和Mixer服务使用，**这样其他管理面组件只用和Galley 打交道**，从而与底层平台解耦。在新的版本中Galley的作用越来越核心。

### istio-sidecar-injector {#istio-sidecar-injector}

istio-sidecar-inj ector 是负责向动注入的组件，只要开启了自动注入，在Pod 创建时就会自动调用istio-sidecar-injector 向Pod 中注入Sidecar 容器。  
在Kubernetes环境下，根据自动注入配置， Kube-apiserver 在拦截到Pod 创建的请求时，会调用自动注入服务istio-sidecar-injector生成Sidecar 容器的描述并将其插入原Pod的定义中，这样，在创建的Pod 内除了包括业务容器，还包括Sidecar 容器。这个注入过程对用户透明，用户使用原方式创建工作负载。

### istio-ingressgateway {#istio-ingressgateway}

istio-ingressgateway 就是入口处的Gateway ，从网格外访问网格内的服务就是通过这个Gateway 进行的。istio-ingressgateway 比较特别， 是一个Loadbalancer 类型的Service,不同于其他服务组件只有一两个端口,istio-ingressgateway 开放了一组端口，这些就是网格内服务的外部访问端口.如下图 所示，网格入口网关istio-ingressgateway 的负载和网格内的Sidecar 是同样的执行体，也和网格内的其他Sidecar 一样从Pilot处接收流量规则并执行。 。Istio 通过一个特有的资源对象Gateway 来配置对外的协议、端口等。

![](/image/Istio/istio-ingressgateway.png)

