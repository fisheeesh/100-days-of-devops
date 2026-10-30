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

| Field | Why |
|:--|:--|
| `emptyDir: {}` | An empty directory on the node that lives as long as the Pod. |
| One `volume-share` in both containers | Mounting the same named volume is what makes it shared. |
| Different `mountPath` values | Each container sees the same directory at whatever path it needs. |
| `kubectl exec -c ...` | Names the container, since the Pod has two. |

## Notes

- Both paths are views of one directory, so `/tmp/official/official.txt` appears as `/tmp/games/official.txt`.
- `emptyDir` survives a container restart but dies with the Pod.
- `command: ["sleep", "infinity"]` keeps otherwise idle containers alive for the test.
