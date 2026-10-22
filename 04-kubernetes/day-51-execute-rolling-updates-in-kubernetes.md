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
| `kubectl describe deployment nginx-deployment` | Gives you the **container name** inside the pod template. `set image` addresses containers by name, and it is often but not always `nginx`. |
| `kubectl set image deployment/... <container>=nginx:1.18` | Patches the pod template directly. No editor, nothing to mistype, and it works the same in a script. |
| Changing the image at all | A Deployment rolls out whenever its **pod template** changes. You never tell it to "roll"; editing the template is the trigger. |
| `kubectl rollout status` | Waits until the rollout actually completes and then says so. This is the "ensure all pods are operational" requirement, and it blocks instead of making you re-run `get pods`. |
| `kubectl get pods` | Confirms every pod is `Running` and `READY 1/1`. A pod can be `Running` while its container is still failing readiness. |
| `jsonpath` on the image | Reads the image straight out of the live object, so there is no squinting at `describe` output. |

## Notes

- A rolling update is the default strategy. Kubernetes creates a **new ReplicaSet** for the new template and shifts pods across gradually, instead of killing everything at once. `kubectl get rs` shows it: the old ReplicaSet drains to 0 while the new one scales up.
- The defaults are `maxSurge: 25%` and `maxUnavailable: 25%`, which is what keeps the app serving throughout. That is the whole point compared with deleting and recreating pods.
- `kubectl edit` works, but it opens `vi` against the live object. A YAML mistake means it rejects the save and reopens the file. `set image` avoids the whole dance.
- Old ReplicaSets are kept on purpose (`revisionHistoryLimit`, default 10) so a rollback is possible. That is what the next task uses: `kubectl rollout undo deployment/nginx-deployment`, with `kubectl rollout history deployment/nginx-deployment` to see the revisions.
- If pods get stuck in `ErrImagePull` or `ImagePullBackOff`, the tag is wrong or unreachable. `kubectl describe pod <name>` shows the pull error in its Events, and the old pods keep serving because the rollout will not finish.
- `kubectl rollout restart deployment/nginx-deployment` recycles pods without changing the image, which is handy when a ConfigMap or Secret changed underneath.
- `kubectl get all` is fine for a quick look but prints everything in the namespace. `kubectl get deploy,rs,pods` is the narrower version.
