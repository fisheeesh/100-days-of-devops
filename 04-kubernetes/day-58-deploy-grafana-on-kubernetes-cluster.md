# Day 58: Deploy Grafana on Kubernetes Cluster

## Question

The Nautilus DevOps team is planning to set up a Grafana tool to collect and analyze analytics
from some applications. They are planning to deploy it on a Kubernetes cluster. Below you can find
more details.

1. Create a deployment named `grafana-deployment-xfusion` using any grafana image for the Grafana
   app. Set other parameters as per your choice.
2. Create a `NodePort` type service with nodePort `32000` to expose the app.

You do not need to make any configuration changes inside the Grafana app once deployed; just make
sure you can access the Grafana login page.

> **Note:** The `kubectl` utility on the `jump-host` has been configured to work with the
> Kubernetes cluster.

## Solution

`deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana-deployment-xfusion
  labels:
    app: grafana-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: grafana-app
  template:
    metadata:
      labels:
        app: grafana-app
    spec:
      containers:
      - name: grafana-container
        image: grafana/grafana:latest
```

`service.yaml`:

```yaml
kind: Service
apiVersion: v1
metadata:
  name: grafana-service
spec:
  selector:
    app: grafana-app
  type: NodePort
  ports:
  - name: grafana-port
    port: 3000
    nodePort: 32000
    targetPort: 3000
```

```bash
kubectl apply -f deployment.yaml -f service.yaml

kubectl rollout status deployment/grafana-deployment-xfusion
kubectl get all
kubectl get svc -o wide
```

Reach the login page. NodePort opens the port on the **nodes**, so you need a node's IP:

```bash
kubectl get nodes -o wide        # take INTERNAL-IP
curl http://<node-ip>:32000/login
```

## Why

| Field | Why |
|:--|:--|
| `selector.matchLabels` matching the template labels | The Deployment finds its pods by label. Disagree and the API rejects it. |
| Service `selector: app: grafana-app` | A Service targets **pod labels**, not the deployment name. |
| `targetPort: 3000` | Grafana listens on 3000. Wrong value and the Service exists but refuses every request. |
| `nodePort: 32000` | The port opened on every node. The task fixes the value, so it has to be written out. |

## Notes

- `kubectl expose` **cannot set a nodePort**: there is no `--node-port` flag and the generated YAML has no `nodePort` line. Generate the file, then add it by hand.
- nodePort must fall in `30000-32767`.
- `kubectl apply -f .` applies every manifest in the directory, including unrelated ones. Name the files.
- `curl localhost:32000` only works on a node. From the jump host use `kubectl get nodes -o wide` and the node IP.
- Two replicas each keep their own SQLite database, so they do not share users or dashboards.
