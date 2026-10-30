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

| Field | Why |
|:--|:--|
| `apiVersion: apps/v1` | Deployments live in `apps/v1`, not core `v1`. |
| `selector.matchLabels` | Tells the Deployment which pods are its own. Must match the template labels or the API rejects it. |
| `kubectl rollout status` | Confirms the Deployment actually produced a ready pod. |

## Notes

- A Deployment self-heals: delete its pod and it makes another. That is the difference from Day 48.
- Leave out `template.metadata.name`. Pods are named from the Deployment and ReplicaSet hash.
- `kubectl create deployment httpd --image=httpd:latest` is the shortest valid answer; the manifest is worth keeping if you want it reusable.
