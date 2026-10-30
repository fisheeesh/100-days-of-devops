# Day 59: Troubleshoot Deployment issues in Kubernetes

## Question

Last week, the Nautilus DevOps team deployed a redis app on the Kubernetes cluster, which was
working fine so far. This morning one of the team members was making some changes in this existing
setup, but he made some mistakes and the app went down. We need to fix this as soon as possible.
Please take a look.

The deployment name is `redis-deployment`. The pods are not in running state right now, so please
look into the issue and fix the same.

> **Note:** The `kubectl` utility on the `jump-host` has been configured to work with the
> Kubernetes cluster.

## Solution

Start with the pod status, because the status names the category of failure:

```bash
kubectl get all
kubectl get pods
# redis-deployment-xxxx   0/1   ImagePullBackOff   or   ContainerCreating
```

The real error is in the pod's Events, not the deployment's:

```bash
kubectl describe pod <pod-name>
# Events:
#   Failed to pull image "redis:alpin": not found
#   MountVolume.SetUp failed for volume "config": configmap "redis-cofig" not found
```

Two typos. Confirm what the ConfigMap is really called, then fix both in the deployment:

```bash
kubectl get configmap
# redis-config    <- the deployment is asking for redis-cofig

kubectl edit deployment redis-deployment
#   image: redis:alpin        ->  redis:alpine
#   configMap name: redis-cofig  ->  redis-config
```

Verify:

```bash
kubectl rollout status deployment/redis-deployment
kubectl get pods
# 1/1   Running
```

## Why

| Command | Why |
|:--|:--|
| `kubectl get pods` | The STATUS column tells you which kind of problem you have before you read a single log line. |
| `kubectl describe pod <name>` | The `Events` at the bottom carry the actual error text. `describe deployment` only shows replica counts and conditions, so it says the rollout is stuck without saying why. |
| `kubectl get configmap` | Checks the name the deployment asks for against what exists. A missing ConfigMap and a misspelled reference look identical from the pod's side. |
| `kubectl edit deployment` | Editing the deployment updates the pod template, which triggers a fresh rollout automatically. Deleting the broken pods would not help, since the ReplicaSet recreates them from the same bad template. |
| `kubectl rollout status` | Blocks until the new pods are actually ready, rather than catching them mid-restart. |

## Reading the status

The failure mode is usually written on the pod:

| Status | What it means | Where to look |
|:--|:--|:--|
| `ImagePullBackOff` / `ErrImagePull` | Image name or tag is wrong, or the registry is unreachable | `describe pod` Events |
| `ContainerCreating` (stuck) | A volume cannot be mounted: missing ConfigMap, Secret or PVC | `describe pod` Events |
| `CrashLoopBackOff` | The container starts, then exits | `kubectl logs <pod>` |
| `Pending` | Nothing could schedule it: resources, nodeSelector, taints | `describe pod`, then `kubectl describe node` |
| `Running` but `0/1` READY | Readiness probe failing | `describe pod` Events and `kubectl logs` |

## Notes

- Both bugs here are one character each, `alpin` for `alpine` and `cofig` for `config`. Kubernetes cannot tell a typo from a thing that genuinely does not exist, so the error is always "not found" rather than "did you mean".
- `kubectl logs` is no use for `ImagePullBackOff` or a stuck `ContainerCreating`, because the container never started. There is nothing to log yet. Events are the only source.
- `kubectl get events --sort-by=.metadata.creationTimestamp` gives a whole-namespace timeline, useful when you do not yet know which object is broken.
- A pod can hit more than one problem at once, as here. Fix the image and the pod moves from `ImagePullBackOff` to a stuck `ContainerCreating` on the volume. Re-read the events after each fix instead of assuming you are done.
- `kubectl set image deployment/redis-deployment <container>=redis:alpine` fixes the image without an editor, but the ConfigMap reference still needs `edit` (or a patch), so one `edit` for both is simpler here.
- Deleting the pods achieves nothing. The ReplicaSet rebuilds them from the same broken template. The template is what has to change.
