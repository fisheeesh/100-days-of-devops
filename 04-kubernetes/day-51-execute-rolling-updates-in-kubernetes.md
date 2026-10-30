# Day 51: Execute Rolling Updates in Kubernetes

## Question

An application currently running on the Kubernetes cluster employs the nginx web server. The
Nautilus application development team has introduced some recent changes that need deployment.
They've crafted an image `nginx:1.18` with the latest updates.

Execute a rolling update for this application, integrating the `nginx:1.18` image. The deployment
is named `nginx-deployment`.

Ensure all pods are operational post-update.

> **Note:** The `kubectl` utility on the `jump-host` has been configured to work with the
> Kubernetes cluster.

## Solution

```bash
kubectl get all
kubectl describe deployment nginx-deployment
# note the container NAME, not just the image: you need it for the next command
```

Trigger the rolling update:

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.18
# deployment.apps/nginx-deployment image updated
```

`kubectl edit deployment nginx-deployment` and changing the image by hand does the same thing.

Then confirm it finished:

```bash
kubectl rollout status deployment/nginx-deployment
# deployment "nginx-deployment" successfully rolled out

kubectl get pods
# all Running, all READY 1/1

kubectl get deployment nginx-deployment -o jsonpath='{.spec.template.spec.containers[*].image}'
# nginx:1.18
```

## Why

| Command | Why |
|:--|:--|
| `kubectl describe deployment nginx-deployment` | Gives you the **container name**. `set image` addresses containers by name, often but not always `nginx`. |
| `kubectl set image deployment/... <container>=nginx:1.18` | Patches the pod template directly: no editor, and it works in a script. |
| Changing the image at all | A Deployment rolls out whenever its **pod template** changes. You never tell it to roll. |
| `kubectl rollout status` | Blocks until the rollout finishes. That is the "ensure all pods are operational" requirement. |

## Notes

- A new ReplicaSet is created and pods shift across gradually. `kubectl get rs` shows the old one draining to 0.
- Defaults `maxSurge: 25%` and `maxUnavailable: 25%` are what keep the app serving throughout.
- Stuck in `ErrImagePull`? The tag is wrong. The old pods keep serving because the rollout never completes.
- `kubectl rollout restart` recycles pods without changing the image, for when a ConfigMap changed underneath.
