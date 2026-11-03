# Day 57: Print Environment Variables

## Question

The Nautilus DevOps team is working on to setup some pre-requisites for an application that will
send the greetings to different users. There is a sample deployment, that needs to be tested.
Below is a scenario which needs to be configured on Kubernetes cluster. Please find below more
details about it.

1. Create a `pod` named `print-envars-greeting`.
2. Configure spec as, the container name should be `print-env-container` and use `bash` image.
3. Create three environment variables:

   a. `GREETING` and its value should be `Welcome to`

   b. `COMPANY` and its value should be `xFusionCorp`

   c. `GROUP` and its value should be `Industries`
4. Use command `["/bin/sh", "-c", 'echo "$(GREETING) $(COMPANY) $(GROUP)"']` (please use this exact
   command), also set its `restartPolicy` policy to `Never` to avoid crash loop back.
5. You can check the output using `kubectl logs -f print-envars-greeting` command.

> `Note:` The `kubectl` utility on the `jump-host` has been configured to work with the Kubernetes
> cluster.

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
