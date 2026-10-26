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

| Field or command | Why |
|:--|:--|
| `initContainers` | Holds the sidecar definition; an init container with `restartPolicy: Always` is Kubernetes' native sidecar form. |
| `restartPolicy: Always` | Keeps the sidecar running alongside the main application container instead of allowing it to finish before Nginx starts. |
| `shared-logs` | Makes the same temporary log directory available to both containers. |
| `/var/log/nginx` | Replaces Nginx's log directory with the shared volume, so the sidecar can read the log files. |
| `while true; do ... sleep 30; done` | Repeatedly reads both log files on the required 30-second interval. |
| `kubectl logs -f ... -c sidecar-container` | Follows only the sidecar container's standard output in a multi-container Pod. |

## Notes

- This is a native sidecar, not a regular application container: it belongs under
  `initContainers` and needs `restartPolicy: Always`.
- The first iteration can report that one or both log files do not yet exist; Nginx creates them
  when it starts, and the loop continues to retry every 30 seconds.
- `emptyDir` is temporary storage. The log files disappear when the Pod is deleted, which matches
  the requirement to avoid persistent storage.
- The sidecar currently writes its output to Kubernetes container logs. A production log agent
  would normally forward that output to the chosen log-aggregation service.
