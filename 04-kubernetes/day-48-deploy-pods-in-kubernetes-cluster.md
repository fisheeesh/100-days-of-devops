# Day 48: Deploy Pods in Kubernetes Cluster

## Question

The Nautilus DevOps team is beginning to use Kubernetes for application management. Create a pod
with the following configuration:

1. Name the pod `pod-httpd`.
2. Use the exact image `httpd:latest`.
3. Set label `app=httpd_app`.
4. Name the container `httpd-container`.

> `kubectl` on the jump host is already configured to communicate with the Kubernetes cluster.

## Solution

Create a manifest on the jump host:

```bash
vi pod.yml
```

Add the following content:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-httpd
  labels:
    app: httpd_app
spec:
  containers:
    - name: httpd-container
      image: httpd:latest
```

Create the pod from the manifest:

```bash
kubectl apply -f pod.yml
# pod/pod-httpd created
```

Wait for it to become ready, then inspect it:

```bash
kubectl wait --for=condition=Ready pod/pod-httpd --timeout=60s
# pod/pod-httpd condition met

kubectl get pod pod-httpd --show-labels
# NAME        READY   STATUS    ...   LABELS
# pod-httpd   1/1     Running   ...   app=httpd_app
```

Verify the exact label, container name, and image recorded by the API server:

```bash
kubectl get pod pod-httpd \
  -o jsonpath='{.metadata.labels.app}{"\n"}{.spec.containers[0].name}{"\n"}{.spec.containers[0].image}{"\n"}'
# httpd_app
# httpd-container
# httpd:latest
```

## Why

| Field or command | Why |
|:--|:--|
| `apiVersion: v1` | Pods belong to Kubernetes' core `v1` API. |
| `kind: Pod` | Requests a standalone Pod rather than a Deployment or another controller. |
| `metadata.name` | Sets the exact pod name `pod-httpd`. |
| `metadata.labels.app` | Attaches the required `app=httpd_app` label to the pod. Labels belong under pod metadata, not inside the container definition. |
| `containers[].name` | Sets the required container name independently of the pod name. |
| `image: httpd:latest` | Uses the exact repository and tag required by the task. |
| `kubectl wait` | Confirms the container started successfully instead of only confirming that the API accepted the manifest. |
| JSONPath verification | Reads the three exact values from the live Kubernetes object without unrelated output. |

## Notes

- Run these commands on the jump host. Its kubeconfig is already prepared, so no SSH connection to a cluster node is required.
- `kubectl run pod-httpd --image=httpd:latest --labels=app=httpd_app` creates the correct pod and label, but names its container `pod-httpd`. It therefore misses the required `httpd-container` name unless additional overrides are used.
- `kubectl apply` is safe to run again with the same manifest; it updates the existing object instead of failing because the name already exists.
- A standalone pod is not managed by a Deployment. If it is deleted, Kubernetes does not create a replacement.
