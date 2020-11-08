## curl访问Kubernetes API Server实践

### Deployment

#### Create

curl

```yaml
$ curl -X POST -H 'Content-Type: application/yaml' --data '
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-example
spec:
  replicas: 3
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14
        ports:
        - containerPort: 80
' http://127.0.0.1:8080/apis/apps/v1/namespaces/default/deployments
```



