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
| `kubectl rollout history deployment/...` | Lists the revisions Kubernetes kept. You need to see there **is** a previous revision before trying to go back to one. |
| `--revision=1` | Prints that revision's pod template, so you can confirm which image you are about to return to instead of trusting the revision number. |
| `kubectl rollout undo deployment/...` | Rolls back to the immediately previous revision. `--to-revision=N` targets a specific one when "previous" is not what you want. |
| `kubectl rollout status` | A rollback is itself a rolling update, so it is not instant. This waits for it to finish. |
| `kubectl get pods` and the `jsonpath` | Confirms the pods are `Running` and the image is genuinely the old one. The history table alone does not prove the pods came up. |

## Notes

- The command is `kubectl rollout undo` (easy to typo as `roullout`), and the deployment name is `nginx-deployment`. A typo'd name fails with `Error from server (NotFound)`, which at least tells you clearly.
- **Rolling back does not delete the bad revision, it adds a new one.** That is why revision 1 vanished from the list and revision 3 appeared in its place: Kubernetes re-applied the old pod template as the newest revision. Revision numbers only ever go up.
- A rollback is a normal rolling update under the hood. Kubernetes still has the old ReplicaSet, so it scales that back up and drains the current one, respecting `maxSurge` and `maxUnavailable` just like Day 51.
- `CHANGE-CAUSE` is `<none>` unless something recorded it. `--record` is deprecated; the current way is to set the annotation yourself: `kubectl annotate deployment/nginx-deployment kubernetes.io/change-cause="roll back to nginx:1.18"`.
- How far back you can go is limited by `revisionHistoryLimit` (default 10). Older ReplicaSets beyond that are garbage collected and those revisions are gone.
- `kubectl rollout undo --dry-run=server deployment/nginx-deployment` shows what the rollback would produce without applying it.
- Running `undo` twice in a row just flips between the two most recent revisions, rather than walking further back through history.
