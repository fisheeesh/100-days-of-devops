# Day 56: Deploy Nginx Web Server on Kubernetes Cluster

## Question

Some of the Nautilus team developers are developing a static website and they want to deploy it on
Kubernetes cluster. They want it to be highly available and scalable. Therefore, based on the
requirements, the DevOps team has decided to create a deployment for it with multiple replicas.
Below you can find more details about it:

1. Create a deployment using `nginx` image with `latest` tag only and remember to mention the tag
   i.e `nginx:latest`. Name it as `nginx-deployment`. The container should be named as
   `nginx-container`, also make sure replica counts are `3`.
2. Create a `NodePort` type service named `nginx-service`. The nodePort should be `30011`.

> `Note:` The `kubectl` utility on the `jump-host` has been configured to work with the Kubernetes
> cluster.

## Solution

Create the Deployment manifest:

```bash
vi deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-app
  template:
    metadata:
      labels:
        app: nginx-app
    spec:
      containers:
        - name: nginx-container
          image: nginx:latest
```

Create the Service manifest:

```bash
vi service.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx-app
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30011
```

Apply both manifests and confirm the rollout and Service:

```bash
kubectl apply -f deployment.yaml -f service.yaml
# deployment.apps/nginx-deployment created
# service/nginx-service created

kubectl rollout status deployment/nginx-deployment --timeout=60s
# deployment "nginx-deployment" successfully rolled out

kubectl get deployment nginx-deployment
kubectl get service nginx-service
kubectl get pods -l app=nginx-app
```

Verify the configured replica count, image, and NodePort:

```bash
kubectl get deployment nginx-deployment \
  -o jsonpath='{.spec.replicas}{"\n"}{.spec.template.spec.containers[0].image}{"\n"}'
# 3
# nginx:latest

kubectl get service nginx-service \
  -o jsonpath='{.spec.type}{"\n"}{.spec.ports[0].nodePort}{"\n"}'
# NodePort
# 30011
```

## Why

| Field | Why |
|:--|:--|
| Pod template label `app: nginx-app` | Must match both the Deployment selector and the Service selector. |
| `type: NodePort` | Opens a port on every node, which is what makes it reachable from outside. |
| `port` and `targetPort` | Service port 80 forwards to nginx on 80 in the matched pods. |
| `nodePort: 30011` | The exact external port the task fixes. |

## Notes

- A Service only routes to pods whose labels match its selector. If `kubectl get endpoints nginx-service` is empty, check the labels first.
- Reach a NodePort at `http://<node-ip>:30011`. `localhost` only works from a node.
- Leave out `template.metadata.name`; pod names are generated.
