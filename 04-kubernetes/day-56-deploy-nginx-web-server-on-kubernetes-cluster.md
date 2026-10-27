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

| Field or command | Why |
|:--|:--|
| `kind: Deployment` | Maintains the requested number of Nginx pods and replaces failed ones. |
| `replicas: 3` | Provides the three replicas required for availability and scaling. |
| `selector.matchLabels` | Defines the pods managed by the Deployment. |
| Pod template label `app: nginx-app` | Must match both the Deployment selector and the Service selector. |
| `nginx:latest` | Uses the exact Nginx image and explicitly required tag. |
| `type: NodePort` | Exposes the Service on a port on each Kubernetes node. |
| `port` and `targetPort` | Forward Service traffic on port 80 to Nginx port 80 in the selected pods. |
| `nodePort: 30011` | Reserves the exact externally reachable node port requested. |

## Notes

- `template.metadata.name` is not needed. The Deployment and its ReplicaSet generate unique Pod
  names automatically.
- A NodePort is normally reached at `http://<node-address>:30011`; `localhost:30011` only works
  when the command runs on a Kubernetes node and the network setup permits it.
- A Service routes only to Pods whose labels match its selector. If `kubectl get endpoints
  nginx-service` shows no addresses, check the `app: nginx-app` labels first.
- `nginx:latest` is required for this exercise. In production, prefer an immutable version or
  digest to make deployments reproducible.
