# Day 52: Revert Deployment to Previous Version in Kubernetes

## Question

Earlier today, the Nautilus DevOps team deployed a new release for an application. However, a
customer has reported a bug related to this recent release. Consequently, the team aims to revert
to the previous version.

There exists a deployment named `nginx-deployment`; initiate a rollback to the previous revision.

> **Note:** The `kubectl` utility on the `jump-host` has been configured to work with the
> Kubernetes cluster.

## Solution

```bash
kubectl get all

kubectl rollout history deployment/nginx-deployment
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         kubectl set image deployment nginx-deployment nginx-container=nginx:alpine-perl --record=true

# see what revision 1 actually contains before going back to it
kubectl rollout history deployment/nginx-deployment --revision=1
```

```bash
kubectl rollout undo deployment/nginx-deployment
# deployment.apps/nginx-deployment rolled back

kubectl rollout status deployment/nginx-deployment
# deployment "nginx-deployment" successfully rolled out

kubectl rollout history deployment/nginx-deployment
# REVISION  CHANGE-CAUSE
# 2         kubectl set image deployment nginx-deployment nginx-container=nginx:alpine-perl --record=true
# 3         <none>

kubectl get pods
kubectl get deployment nginx-deployment -o jsonpath='{.spec.template.spec.containers[*].image}'
# back to the old image
```

## Why

| Command | Why |
|:--|:--|
| `kubectl rollout history deployment/...` | Confirms there **is** a previous revision before you try to go back to one. |
| `--revision=1` | Prints that revision's pod template, so you can see which image you are returning to. |
| `kubectl rollout undo deployment/...` | Goes back one revision. `--to-revision=N` targets a specific one. |
| `kubectl rollout status` | A rollback is a rolling update too, so it is not instant. |

## Notes

- **Rolling back adds a revision, it does not delete one.** Revision 1 became revision 3, which is why it vanished from the top of the list. Numbers only go up.
- Running `undo` twice just flips between the two most recent revisions. Use `--to-revision=N` to go deeper.
- `revisionHistoryLimit` (default 10) caps how far back you can go.
- `CHANGE-CAUSE` is `<none>` unless recorded. `--record` is deprecated; use `kubectl annotate ... kubernetes.io/change-cause="..."`.
