# Day 54: Kubernetes Shared Volumes

## Question

We are working on an application that will be deployed on multiple containers within a pod on
Kubernetes cluster. There is a requirement to share a volume among the containers to save some
temporary data. The Nautilus DevOps team is developing a similar template to replicate the
scenario. Below you can find more details about it.

1. Create a pod named `volume-share-devops`.
2. For the first container, use image `debian` with `latest` tag only and remember to mention the
   tag i.e `debian:latest`, container should be named as `volume-container-devops-1`, and run a
   `sleep` command for it so that it remains in running state. Volume `volume-share` should be
   mounted at path `/tmp/official`.
3. For the second container, use image `debian` with the `latest` tag only and remember to mention
   the tag i.e `debian:latest`, container should be named as `volume-container-devops-2`, and
   again run a `sleep` command for it so that it remains in running state. Volume `volume-share`
   should be mounted at path `/tmp/games`.
4. Volume name should be `volume-share` of type `emptyDir`.
5. After creating the pod, exec into the first container i.e `volume-container-devops-1`, and just
   for testing create a file `official.txt` with the content `Welcome to xFusionCorp Industries`
   under the mounted path of first container i.e `/tmp/official`.
6. The file `official.txt` should be present under the mounted path `/tmp/games` on the second
   container `volume-container-devops-2` as well, since they are using a shared volume.

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
