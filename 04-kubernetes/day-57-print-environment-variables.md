# Day 57: Print Environment Variables

## Question

Create a one-time Pod that prints a greeting assembled from environment variables:

1. Name the Pod `print-envars-greeting` and its container `print-env-container`.
2. Use the `bash` image.
3. Set `GREETING=Welcome to`, `COMPANY=xFusionCorp`, and `GROUP=Industries`.
4. Use the exact command `['/bin/sh', '-c', 'echo "$(GREETING) $(COMPANY) $(GROUP)"']`.
5. Set the Pod restart policy to `Never` and read the output through Pod logs.

> `kubectl` on the jump host is already configured to communicate with the Kubernetes cluster.

## Solution

Create a manifest on the jump host:

```bash
vi pod.yaml
```

Add the following content:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: print-envars-greeting
spec:
  restartPolicy: Never
  containers:
    - name: print-env-container
      image: bash
      env:
        - name: GREETING
          value: Welcome to
        - name: COMPANY
          value: xFusionCorp
        - name: GROUP
          value: Industries
      command: ["/bin/sh", "-c", 'echo "$(GREETING) $(COMPANY) $(GROUP)"']
```

Create the Pod and follow its output:

```bash
kubectl apply -f pod.yaml
# pod/print-envars-greeting created

kubectl logs -f print-envars-greeting
# Welcome to xFusionCorp Industries
```

The command finishes once, so the Pod should complete rather than restart:

```bash
kubectl get pod print-envars-greeting
# NAME                    READY   STATUS      ...
# print-envars-greeting   0/1     Completed   ...
```

## Why

| Field | Why |
|:--|:--|
| `$(GREETING)` syntax | Kubernetes expands these in `command` and `args` before the container starts. Not shell `$NAME`. |
| `/bin/sh -c` | Gives the container a shell to run the `echo` in. |
| `restartPolicy: Never` | Stops Kubernetes restarting a command that already completed. |
| `kubectl logs -f` | Streams the output from the container. |

## Notes

- `restartPolicy` belongs under the Pod's `spec`, not under a container.
- A completed Pod keeps its logs, so plain `kubectl logs print-envars-greeting` works afterwards too.
