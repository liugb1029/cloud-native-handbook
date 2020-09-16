## Bookinfo 端到端调用分析 {#bookinfo-端到端调用分析}

通过前面对[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)配置文件的分析，我们对[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)上生成的各种配置数据的结构，包括 listener、[cluster](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#cluster)、route 和 endpoint 有了一定的了解。那么这些配置是如何有机地结合在一起，以对经过网格中的流量进行路由的呢？

下面我们通过 bookinfo 示例程序中一个端到端的调用请求把这些相关的配置串连起来，使用该完整的调用流程来帮助理解[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio)控制平面的流量控制能力是如何在数据平面的[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy)上实现的。

下图描述了 bookinfo 示例程序中 productpage 服务调用 reviews 服务的请求流程：

![](/image/Istio/envoy-traffic-route.jpg)

Productpage 发起对 reviews 服务的调用：







