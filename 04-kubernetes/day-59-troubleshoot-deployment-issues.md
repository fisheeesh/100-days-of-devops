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
| `kubectl get pods` | The STATUS column names the category of failure before you read a single log line. |
| `kubectl describe pod <name>` | The `Events` carry the actual error. `describe deployment` only says the rollout is stuck, not why. |
| `kubectl get configmap` | A missing ConfigMap and a misspelled reference look identical from the pod's side. |
| `kubectl edit deployment` | Changing the template triggers a fresh rollout. Deleting pods does nothing: the ReplicaSet rebuilds them from the same bad template. |

## Reading the status

| Status | What it means | Where to look |
|:--|:--|:--|
| `ImagePullBackOff` / `ErrImagePull` | Image name or tag wrong, or registry unreachable | `describe pod` Events |
| `ContainerCreating` (stuck) | A volume cannot mount: missing ConfigMap, Secret or PVC | `describe pod` Events |
| `CrashLoopBackOff` | Container starts, then exits | `kubectl logs <pod>` |
| `Pending` | Cannot be scheduled: resources, nodeSelector, taints | `describe pod`, `describe node` |
| `Running` but `0/1` | Readiness probe failing | `describe pod` Events, `kubectl logs` |

## Notes

- `kubectl logs` is useless for `ImagePullBackOff` or a stuck `ContainerCreating`: the container never started, so there is nothing to log. Events are the only source.
- This pod had two faults at once. Fix the image and it moves to a stuck `ContainerCreating` on the volume. Re-read the events after each fix.
- `kubectl get events --sort-by=.metadata.creationTimestamp` gives a namespace-wide timeline when you do not know which object is broken.
