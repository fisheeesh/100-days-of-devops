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

| Field | Why |
|:--|:--|
| `metadata.labels.app` | Labels belong under **pod** metadata, not inside the container definition. |
| `containers[].name` | The container name is set separately from the pod name. The task fixes both. |
| `kubectl wait` | Confirms the container actually started, not just that the API accepted the YAML. |
| JSONPath check | Reads the three required values straight out of the live object. |

## Notes

- `kubectl run pod-httpd --image=httpd:latest --labels=app=httpd_app` gets the pod and label right but names the container `pod-httpd`, missing the required `httpd-container`.
- A standalone pod has no controller. Delete it and nothing replaces it.
- `kubectl apply` is safe to re-run; it updates instead of failing on a duplicate name.
