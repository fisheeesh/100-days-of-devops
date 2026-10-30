# Day 56: Deploy Nginx Web Server on Kubernetes Cluster

## Question

Deploy a highly available Nginx website on Kubernetes:

1. Create Deployment `nginx-deployment` with three replicas of `nginx:latest`.
2. Name the container `nginx-container`.
3. Create NodePort Service `nginx-service` using node port `30011`.

> `kubectl` on the jump host is already configured to communicate with the Kubernetes cluster.

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
