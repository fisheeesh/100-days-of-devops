# Day 55: Kubernetes Sidecar Containers

## Question

Create a Pod named `webserver` that serves with Nginx and ships its access and error logs through a
native sidecar container:

1. Create an `emptyDir` volume named `shared-logs`.
2. Create container `nginx-container` from `nginx:latest`.
3. Create native sidecar init container `sidecar-container` from `ubuntu:latest`, using
   `restartPolicy: Always`.
4. Run the specified log-reading loop in the sidecar.
5. Mount `shared-logs` at `/var/log/nginx` in both containers.

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
  name: webserver
spec:
  initContainers:
    - name: sidecar-container
      image: ubuntu:latest
      restartPolicy: Always
      command:
        - sh
        - -c
        - while true; do cat /var/log/nginx/access.log /var/log/nginx/error.log; sleep 30; done
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/nginx
  containers:
    - name: nginx-container
      image: nginx:latest
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/nginx
  volumes:
    - name: shared-logs
      emptyDir: {}
```

Create the Pod and confirm both containers are running:

```bash
kubectl apply -f pod.yaml
# pod/webserver created

kubectl wait --for=condition=Ready pod/webserver --timeout=60s
# pod/webserver condition met

kubectl get pod webserver
# NAME        READY   STATUS    ...
# webserver   2/2     Running   ...
```

Read the sidecar's output. It prints the Nginx access and error logs every 30 seconds:

```bash
kubectl logs -f webserver -c sidecar-container
```

## Why

| Field | Why |
|:--|:--|
| `initContainers` + `restartPolicy: Always` | Kubernetes' native sidecar form. Without the restart policy it would finish before nginx starts. |
| `shared-logs` on `/var/log/nginx` | Replaces nginx's log directory with the shared volume so the sidecar can read it. |
| `while true; do ... sleep 30; done` | The 30-second interval the task asks for. |
| `kubectl logs -c sidecar-container` | Follows only the sidecar's output in a two-container Pod. |

## Notes

- A native sidecar goes under `initContainers`, not alongside the app container.
- The first loop may report the log files missing. nginx creates them at startup and the loop retries.
- `emptyDir` is temporary, which is exactly what "no persistent storage" asks for.
