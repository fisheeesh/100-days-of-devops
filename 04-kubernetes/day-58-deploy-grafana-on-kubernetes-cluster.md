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

| Piece | Why |
|:--|:--|
| `selector.matchLabels` matching the template labels | The Deployment finds its own pods by label. If the two disagree, the API server rejects the Deployment outright. |
| Service `selector: app: grafana-app` | A Service targets **pod labels**, not the deployment name. This is the link between the two files. |
| `targetPort: 3000` | Grafana listens on 3000 inside the container. Get this wrong and the Service still exists, but every request is refused. |
| `port: 3000` | The port the Service answers on inside the cluster. It does not have to equal `targetPort`, it just happens to here. |
| `nodePort: 32000` | The port opened on every node, which is what makes it reachable from outside the cluster. The task fixes this value, so it has to be written out. |
| `kubectl rollout status` | Confirms the pods are actually up before you curl and wonder why nothing answers. |

## Notes

- `kubectl expose` **cannot set a nodePort.** There is no `--node-port` flag, and the YAML it generates has no `nodePort` line at all, so it would take a random one from the range. Your tip of generating the file then editing it is the right way round:

  ```bash
  kubectl create deploy grafana-deployment-xfusion --image=grafana/grafana --dry-run=client -o yaml > deployment.yaml
  kubectl expose deploy grafana-deployment-xfusion --port=3000 --type=NodePort --dry-run=client -o yaml > service.yaml
  # then add  nodePort: 32000  to service.yaml by hand
  ```

- nodePort has to fall in `30000-32767` by default. `32000` fits; something like `8080` would be rejected.
- `kubectl apply -f .` applies **every** manifest in the directory. Fine in a scratch folder, but it will happily pick up an unrelated `job.yaml` sitting next to it. Naming the files is safer.
- The `name:` under `template.metadata` is ignored. Pods are named by their ReplicaSet. Only the labels there matter.
- `replicas: 2` passes the task, but two Grafana pods each keep their own SQLite database, so they do not share users or dashboards. Real deployments run one replica, or point several at an external database.
- The login page being reachable is the whole requirement. Default credentials are `admin` / `admin`.
- `curl localhost:32000` only works if you are on a node. From the jump host, use the node IP.
