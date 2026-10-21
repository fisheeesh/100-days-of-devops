# Day 50: Set Resource Limits in Kubernetes Pods

## Question

The Nautilus DevOps team needs to limit resource use for an HTTPD pod. Create a pod with the
following configuration:

1. Name the pod `httpd-pod` and its container `httpd-container`.
2. Use the exact image `httpd:latest`.
3. Set requests to `100m` CPU and `15Mi` memory.
4. Set limits to `100m` CPU and `20Mi` memory.

> `kubectl` on the jump host is already configured to communicate with the Kubernetes cluster.

## Solution

Create the manifest on the jump host:

```bash
vi pod.yml
```

Add the following content:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: httpd-pod
spec:
  containers:
    - name: httpd-container
      image: httpd:latest
      resources:
        requests:
          cpu: 100m
          memory: 15Mi
        limits:
          cpu: 100m
          memory: 20Mi
```

Create the pod and wait until it is ready:

```bash
kubectl apply -f pod.yml
# pod/httpd-pod created

kubectl wait --for=condition=Ready pod/httpd-pod --timeout=60s
# pod/httpd-pod condition met
```

Confirm the resource configuration stored by the API server:

```bash
kubectl get pod httpd-pod \
  -o jsonpath='{.spec.containers[0].resources.requests.cpu}{"\n"}{.spec.containers[0].resources.requests.memory}{"\n"}{.spec.containers[0].resources.limits.cpu}{"\n"}{.spec.containers[0].resources.limits.memory}{"\n"}'
# 100m
# 15Mi
# 100m
# 20Mi
```

For a broader, human-readable view:

```bash
kubectl describe pod httpd-pod
```

## Why

| Field or command | Why |
|:--|:--|
| `apiVersion: v1` | Pods use Kubernetes' core `v1` API. |
| `metadata.name` | Sets the required pod name `httpd-pod`. |
| `containers[].name` | Sets the required container name separately from the pod name. |
| `image: httpd:latest` | Uses the exact HTTPD image and tag requested. |
| `requests.cpu: 100m` | Reserves one-tenth of a CPU core for scheduling. |
| `requests.memory: 15Mi` | Tells the scheduler the pod needs 15 mebibytes of memory. |
| `limits.cpu: 100m` | Caps CPU use at one-tenth of a core; CPU overuse is throttled. |
| `limits.memory: 20Mi` | Caps memory use at 20 mebibytes; exceeding it can cause an out-of-memory termination. |
| `kubectl wait` | Verifies the HTTPD container becomes ready. |

## Notes

- The task says `CPPU`; this is a typo. The Kubernetes resource key is `cpu`.
- `m` means millicpu: `100m` equals `0.1` CPU core. `Mi` means mebibytes.
- Requests affect scheduling; limits constrain how much the running container may consume.
- Do not put `resources` under `spec` directly. It belongs under the individual container definition.
- `kubectl apply -f pod.yml` can be rerun to update the manifest. Some Pod fields are immutable after creation; if Kubernetes rejects a later change, delete and recreate the pod only when that is safe for the workload.
