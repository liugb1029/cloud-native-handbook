## 背景信息 

灰度及蓝绿发布是为新版本创建一个与老版本完全一致的生产环境，在不影响老版本的前提下，按照一定的规则把部分流量切换到新版本，当新版本试运行一段时间没有问题后，将用户的全量流量从老版本迁移至新版本。

其中A/B测试就是一种灰度发布方式，一部分用户继续使用老版本的服务，将一部分用户的流量切换到新版本，如果新版本运行稳定，则逐步将所有用户迁移到新版本。

## 应用场景 

场景一

假设当前线上环境，您已经有一套服务Service A对外提供7层服务，此时上线了一些新的特性，需要发布上线一个新的版本Service A'，但又不想简单地直接替换掉Service A服务，而是希望将请求头中包含

`foo=bar`

或者cookie中包含

`foo=bar`

的客户端请求转发到Service A'服务中，待运行一段时间稳定后，可将所有的流量从Service A切换到Service A'服务中，再平滑地将Service A服务下线。

![](/image/kubernetes/ingress-cookie.png)

场景二

假设当前线上环境，您已经有一套服务Service B对外提供7层服务，此时修复了一些问题，需要发布上线一个新的版本Service B'，但又不想简单地将所有客户端流量切换到新版本Service B'中，而是希望将20%的流量切换到新版本Service B'中，待运行一段时间稳定后，可将所有的流量从Service B切换到Service B'服务中，再平滑地将Service B服务下线。

![](/image/kubernetes/ingress-weight.png)

针对以上多种不同的应用发布需求，阿里云容器服务Kubernetes的 Ingress 功能提供的4种流量切分方式：

灰度发布中的A/B 测试：

* 基于Request Header的流量切分
* 基于Cookie的流量切分
* 基于Query Param的流量切分

蓝绿发布：基于服务权重的流量切分

Ingress-Nginx 是一个K8S ingress工具，支持配置 Ingress Annotations 来实现不同场景下的灰度发布和测试。

[Nginx Annotations](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#canary)

支持以下 4 种 Canary 规则：

* `nginx.ingress.kubernetes.io/canary-by-header`：基于 Request Header 的流量切分，适用于灰度发布以及 A/B 测试。当 Request Header 设置为 `always`时，请求将会被一直发送到 Canary 版本；当 Request Header 设置为 `never`时，请求不会被发送到 Canary 入口；对于任何其他 Header 值，将忽略 Header，并通过优先级将请求与其他金丝雀规则进行优先级的比较。

* `nginx.ingress.kubernetes.io/canary-by-header-value`  
  ：要匹配的 Request Header 的值，用于通知 Ingress 将请求路由到 Canary Ingress 中指定的服务。当 Request Header 设置为此值时，它将被路由到 Canary 入口。该规则允许用户自定义 Request Header 的值，必须与上一个 annotation \(即：canary-by-header）一起使用。

* `nginx.ingress.kubernetes.io/canary-weight`  
  ：基于服务权重的流量切分，适用于蓝绿部署，权重范围 0 - 100 按百分比将请求路由到 Canary Ingress 中指定的服务。权重为 0 意味着该金丝雀规则不会向 Canary 入口的服务发送任何请求。权重为 100 意味着所有请求都将被发送到 Canary 入口。

* `nginx.ingress.kubernetes.io/canary-by-cookie`  
  ：基于 Cookie 的流量切分，适用于灰度发布与 A/B 测试。用于通知 Ingress 将请求路由到 Canary Ingress 中指定的服务的cookie。当 cookie 值设置为  
  `always`  
  时，它将被路由到 Canary 入口；当 cookie 值设置为  
  `never`  
  时，请求不会被发送到 Canary 入口；对于任何其他值，将忽略 cookie 并将请求与其他金丝雀规则进行优先级的比较。

注意：金丝雀规则按优先顺序进行如下排序：

> `canary-by-header - > canary-by-cookie - > canary-weight`

我们可以把以上的四个 annotation 规则可以总体划分为以下两类：

* 基于权重的 Canary 规则![](/image/kubernetes/canary-1.png)

* 基于用户请求的 Canary 规则![](/image/kubernetes/canary-2.png)

注意： Ingress-Nginx 实在0.21.0 版本 中，引入的Canary 功能，因此要确保ingress版本OK

## 测试

### 应用准备 

两个版本的服务，正常版本：

```
import static java.util.Collections.singletonMap;

@SpringBootApplication
@Controller
public class RestPrometheusApplication {

    @Autowired
    private MeterRegistry registry;

    @GetMapping(path = "/", produces = "application/json")
    @ResponseBody
    public Map<String, Object> landingPage() {
        Counter.builder("mymetric").tag("foo", "bar").register(registry).increment();
        return singletonMap("hello", "ambassador");
    }

    public static void main(String[] args) {
        SpringApplication.run(RestPrometheusApplication.class, args);
    }

}
```

访问会输出：

```
{"hello":"ambassador"}
```

灰度版本：

```
import static java.util.Collections.singletonMap;

@SpringBootApplication
@Controller
public class RestPrometheusApplication {

    @Autowired
    private MeterRegistry registry;

    @GetMapping(path = "/", produces = "application/json")
    @ResponseBody
    public Map<String, Object> landingPage() {
        Counter.builder("mymetric").tag("foo", "bar").register(registry).increment();
        return singletonMap("hello", "ambassador, this is a gray version");
    }

    public static void main(String[] args) {
        SpringApplication.run(RestPrometheusApplication.class, args);
    }

}
```

访问会输出：

```
{"hello":"ambassador, this is a gray version"}
```

### 基于Request Header ingress 配置

我们部署好两个服务，springboot-rest-demo是正常的服务，springboot-rest-demo-gray是灰度服务，我们来配置ingress，通过canary-by-header来实现：

正常服务的：

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: springboot-rest-demo
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: springboot-rest.jadepeng.com
    http:
      paths:
      - backend:
          serviceName: springboot-rest-demo
          servicePort: 80
```

canary 的：

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: springboot-rest-demo-gray
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-by-header: "canary"
    nginx.ingress.kubernetes.io/canary-by-header-value: "true"
spec:
  rules:
  - host: springboot-rest.jadepeng.com
    http:
      paths:
      - backend:
          serviceName: springboot-rest-demo-gray
          servicePort: 80
```

将上面的文件执行：

```
kubectl -n=default apply -f ingress-test.yml 
ingress.extensions/springboot-rest-demo created
ingress.extensions/springboot-rest-demo-gray created
```

执行测试，不添加header，访问的默认是正式版本：

```
# curl http://springboot-rest.jadepeng.com; echo
{"hello":"ambassador"}
# curl http://springboot-rest.jadepeng.com; echo
{"hello":"ambassador"}
```

添加header，可以看到，访问的已经是灰度版本了

```
# curl -H "canary: true" http://springboot-rest.jadepeng.com; echo
{"hello":"ambassador, this is a gray version"}
```

## 基于Weight 的 Ingress配置 

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: my-app-canary
  labels:
    app: my-app
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "30"
  namespace: default
spec:
  rules:
    - host: test.192.168.2.20.xip.io
      http:
        paths:
          - backend:
              serviceName: app-new
              servicePort: 80
            path: /
```

## 多实例Ingress controllers 

## 参考

* [https://kubesphere.com.cn/docs/v2.0/zh-CN/quick-start/ingress-canary/\#ingress-nginx-annotation-简介](https://kubesphere.com.cn/docs/v2.0/zh-CN/quick-start/ingress-canary/#ingress-nginx-annotation-简介)
* [https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/\#canary](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#canary)

## 路由规则`nginx.ingress.kubernetes.io/service-match` 

该Annotation用来配置新版本服务的路由匹配规则，格式如下：

```
nginx.ingress.kubernetes.io/service-match: | 
    <service-name>: <match-rule>
```

**参数解释**

service-name：服务名称，满足match-rule的请求会被路由到该服务中。

match-rule

：路由匹配规则。

* 匹配类型 ：
  * `header`
    ：基于请求头，支持正则匹配和完整匹配。
  * `cookie`
    ：基于cookie，支持正则匹配和完整匹配。
  * `query`
    ：基于请求参数，支持正则匹配和完整匹配。
* 匹配方式：
  * 正则匹配格式：
    `/{regular expression}/`
  * 完整匹配格式：
    `"{exact expression}"`

**配置示例**

```
# 请求头中满足foo正则匹配^bar$的请求被转发到新版本服务new-nginx中
new-nginx: header("foo", /^bar$/)

# 请求头中满足foo完整匹配bar的请求被转发到新版本服务new-nginx中
new-nginx: header("foo", "bar")

# cookie中满足foo正则匹配^sticky-.+$的请求被转发到新版本服务new-nginx中
new-nginx: cookie("foo", /^sticky-.+$/)

# query param中满足foo完整匹配bar的请求被转发到新版本服务new-nginx中
new-nginx: query("foo", "bar")
```

## 服务权重`nginx.ingress.kubernetes.io/service-weight`

该Annotation用来配置新老版本服务的流量权重，格式如下：

```
nginx.ingress.kubernetes.io/service-weight: | 
    <new-svc-name>:<new-svc-weight>, <old-svc-name>:<old-svc-weight>
```

**参数解释**

new-svc-name：新版本服务名称。

new-svc-weight：新版本服务权重。

old-svc-name：老版本服务名称。

old-svc-weight：老版本服务权重。

**配置示例**

```
nginx.ingress.kubernetes.io/service-weight: | 
    new-nginx: 20, old-nginx: 60
```

**说明**

* 服务权重采用相对值计算方式。如配置示例中的新版服务权重设置为20，旧版服务权重设置为60，则：新版服务的权重百分比为25%，旧版服务的权重百分比为75%。
* 一个服务组（同一个Ingress yaml中具有相同Host和Path的服务）中未明确设置权重的服务默认权重值为100。



