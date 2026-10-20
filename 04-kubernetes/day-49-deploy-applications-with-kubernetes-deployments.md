# Day 49: Deploy Applications with Kubernetes Deployments

## Question

The Nautilus DevOps team is using Kubernetes for application management. Create a Deployment with
the following configuration:

1. Name the Deployment `httpd`.
2. Deploy the `httpd` application using the exact image `httpd:latest`.

> `kubectl` on the jump host is already configured to communicate with the Kubernetes cluster.

## Solution

Create a manifest on the jump host:

```bash
vi deployment.yaml
```

Add the following content:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpd
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpd
  template:
    metadata:
      labels:
        app: httpd
    spec:
      containers:
        - name: httpd
          image: httpd:latest
```

Create the Deployment:

```bash
kubectl apply -f deployment.yaml
# deployment.apps/httpd created
```

Wait for the rollout to finish, then inspect the Deployment and its pod:

```bash
kubectl rollout status deployment/httpd --timeout=60s
# deployment "httpd" successfully rolled out

kubectl get deployment httpd
kubectl get pods -l app=httpd
```

Verify the Deployment name and exact container image stored by the API server:

```bash
kubectl get deployment httpd \
  -o jsonpath='{.metadata.name}{"\n"}{.spec.template.spec.containers[0].image}{"\n"}'
# httpd
# httpd:latest
```

The same task can also be completed without writing a manifest:

```bash
kubectl create deployment httpd --image=httpd:latest
kubectl rollout status deployment/httpd --timeout=60s
```

## Why

| Field or command | Why |
|:--|:--|
| `apiVersion: apps/v1` | Deployments belong to the stable `apps/v1` API. |
| `kind: Deployment` | Creates a controller that maintains the requested pods. |
| `metadata.name: httpd` | Sets the exact Deployment name required by the task. |
| `replicas: 1` | Runs one application pod; three replicas are unnecessary because the task does not request them. |
| `selector.matchLabels` | Tells the Deployment which pods it manages. |
| `template.metadata.labels` | Labels the created pods. These labels must match the Deployment selector. |
| `image: httpd:latest` | Uses the explicitly required image and tag. |
| `kubectl rollout status` | Confirms the Deployment successfully created a ready pod. |

## Notes

- Run all commands on the jump host; its kubeconfig is already configured for the cluster.
- A Deployment provides self-healing behavior. If its pod is deleted, the Deployment creates a replacement.
- `template.metadata.name` should be omitted. A Deployment generates unique pod names from its own name and ReplicaSet hash.
- Extra Deployment labels such as `type: frontend` are valid but are not required for this task.
- `kubectl apply` is useful when keeping a reusable manifest; `kubectl create deployment` is the shortest valid solution for this requirement.
