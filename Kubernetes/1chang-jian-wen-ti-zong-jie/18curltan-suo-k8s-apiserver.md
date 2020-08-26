# CURL探索Kubernetes API Server

在Kubernetes集群中，API Server是集群管理API的入口，由运行在Master节点上的一个名为kube-apiserver的进程提供的服务。 用户进入API可以通过kubectl、客户端库或者http rest，User 或者 Service Account可以被授权进入API。当一个请求到达API时， 往往要经过几个阶段的安全控制，在一个典型的应用集群中，API Server通常会使用自签名的证书提供HTTPS服务，同时开启认证与授权等安全机制。



通常，在Kubernetes集群搭建之后，除了使用官方的kubectl工具与API Server进行交互，我们还可以使用Postman或者curl了，有些时候直接使用curl功能更强大， 与API Server交互通常需要首先创建一个有正确权限的ServiceAccount，这个ServiceAccount通过ClusterRole/Role、ClusterRoleBinding/RoleBinding等给其赋予相关资源的操作权限， 而Service Account对应的Token则用于API Server进行基本的认证。与API Server的交互是基于TLS，所以请求的时候还需要自签名的证书，当然也可以非安全方式连接API Server， 但是不推荐。

**创建ServiceAccount**

```
# kubectl create serviceaccount sunjinfu
serviceaccount "sunjinfu" created
# kubectl get sa sunjinfu -o json
{
    "apiVersion": "v1",
    "kind": "ServiceAccount",
    "metadata": {
        "creationTimestamp": "2019-05-22T12:26:17Z",
        "name": "sunjinfu",
        "namespace": "default",
        "resourceVersion": "26547033",
        "selfLink": "/api/v1/namespaces/default/serviceaccounts/sunjinfu",
        "uid": "cc00a8f4-7c8c-11e9-9cef-fa163e45366f"
    },
    "secrets": [
        {
            "name": "sunjinfu-token-2wlsj"
        }
    ]
}
```

**创建ClusterRole、RoleBinding**

给这个角色sunjinfuread赋予pods、services的相关查看权限

```
# vim sunjinfu-role.yaml

kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: sunjinfuread
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods", "services", "pods/log"]
  verbs: ["get", "watch", "list"]
```

```
# kubectl apply -f sunjinfu-role.yaml
clusterrole "sunjinfuread" created
# kubectl get clusterrole sunjinfuread
NAME           AGE
sunjinfuread   25s
```

ClusterRole创建成功之后，然后将Service与ClusterRole通过RoleBinding进行绑定

```
# kubectl create rolebinding sunjinfu:read --clusterrole sunjinfuread --serviceaccount default:sunjinfu
rolebinding "sunjinfu:read" created
# kubectl get rolebinding sunjinfu:read
NAME            AGE
sunjinfu:read   1m
```

这样，sunjinfu这个Service Account已经具备了在命名空间default下查看pod或者service的权限了。

**获取Bearer Token、Certificate、API Server URL**

在操作的过程中，需要对处理json字符串的命令行工具jq有基本的了解，请自行学习。

```
SERVICE_ACCOUNT=sunjinfu

# 获取Service Account token secret名字
SECRET=$(kubectl get serviceaccount ${SERVICE_ACCOUNT} -o json \
| jq -Mr '.secrets[].name | select(contains("token"))')

# 从secret中提取Token
TOKEN=$(kubectl get secret ${SECRET} -o json | jq -Mr '.data.token' | base64 -d)

# 从secret中提取证书文件
kubectl get secret ${SECRET} -o json | jq -Mr '.data["ca.crt"]' \
| base64 -d 
>
 /tmp/ca.crt

# 获取API Server URL，如果API Server部署在多台Master上，只需访问其中一台即可。
APISERVER=https://$(kubectl -n default get endpoints kubernetes --no-headers \
| awk '{ print $2 }' | cut -d "," -f 1)
```

可以将以上做成一个脚本，通过`echo $APISERVER`输出已获取相关内容，访问APIServer的相关参数均已获取，用以下curl命令请求即可。

```
curl -s $APISERVER/api/v1/namespaces/default/pods/ \
--header "Authorization: Bearer $TOKEN" --cacert /tmp/ca.crt
```

通过`jq -Mr`提取所有的Pod名字

```
curl -s $APISERVER/api/v1/namespaces/default/pods/ --header "Authorization: Bearer $TOKEN" \
--cacert /tmp/ca.crt  | jq -rM '.items[].metadata.name'
```

注意：Kubernetes在每个命名空间下都有一个默认的service account，如果要创建的Pod没有指定任何service account，Kubernetes会将默认的 service account挂载到Pod的指定目录下。

```
# kubectl exec -it container-deployment-8696749cc4-mthwc sh
# ls /var/run/secrets/kubernetes.io/serviceaccount
ca.crt   namespace  token
```



