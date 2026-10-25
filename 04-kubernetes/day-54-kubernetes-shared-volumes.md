# Day 54: Kubernetes Shared Volumes

## Question

Create a Pod that shares temporary data between two containers:

1. Name the Pod `volume-share-devops`.
2. Use two `debian:latest` containers named `volume-container-devops-1` and
   `volume-container-devops-2`; keep both running with a sleep command.
3. Create an `emptyDir` volume named `volume-share`.
4. Mount it at `/tmp/official` in the first container and `/tmp/games` in the second.
5. Create `official.txt` containing `Welcome to xFusionCorp Industries` through the first mount,
   then confirm it is visible through the second mount.

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
  name: volume-share-devops
spec:
  containers:
    - name: volume-container-devops-1
      image: debian:latest
      command: ["sleep", "infinity"]
      volumeMounts:
        - name: volume-share
          mountPath: /tmp/official
    - name: volume-container-devops-2
      image: debian:latest
      command: ["sleep", "infinity"]
      volumeMounts:
        - name: volume-share
          mountPath: /tmp/games
  volumes:
    - name: volume-share
      emptyDir: {}
```

Create the Pod and wait for both containers to become ready:

```bash
kubectl apply -f pod.yaml
# pod/volume-share-devops created

kubectl wait --for=condition=Ready pod/volume-share-devops --timeout=60s
# pod/volume-share-devops condition met
```

Write the test file through the first container's mount, then read it through the second:

```bash
kubectl exec volume-share-devops -c volume-container-devops-1 -- \
  sh -c 'printf "%s\\n" "Welcome to xFusionCorp Industries" > /tmp/official/official.txt'

kubectl exec volume-share-devops -c volume-container-devops-1 -- \
  cat /tmp/official/official.txt
# Welcome to xFusionCorp Industries

kubectl exec volume-share-devops -c volume-container-devops-2 -- \
  cat /tmp/games/official.txt
# Welcome to xFusionCorp Industries
```

## Why

| Field or command | Why |
|:--|:--|
| `emptyDir: {}` | Creates an initially empty directory on the node for the lifetime of this Pod. |
| `volume-share` | The same named volume is referenced by both containers, so it is the shared storage. |
| Different `mountPath` values | Each container can expose the same volume at the path required by its application. |
| `debian:latest` | Uses the exact image and explicitly specified tag requested. |
| `command: ["sleep", "infinity"]` | Keeps the otherwise idle Debian containers running for the test. |
| `kubectl exec -c ...` | Selects the intended container because this Pod contains two containers. |

## Notes

- The two paths are different container-local views of the same `volume-share` directory. A file
  written to `/tmp/official/official.txt` is therefore available as `/tmp/games/official.txt`.
- `emptyDir` data remains while the Pod runs, including when an individual container restarts, but
  it is deleted when the Pod is removed or permanently moved to another node.
- `emptyDir` uses node storage by default. It can consume memory instead with
  `emptyDir: { medium: Memory }`, but that is not required here.
- `kubectl run ... --dry-run=client -o yaml` is useful for a starter manifest, but it cannot
  generate this complete multi-container volume configuration by itself.
