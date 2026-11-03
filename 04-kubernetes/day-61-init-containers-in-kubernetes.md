# Day 61: Init Containers in Kubernetes

## Question

There are some applications that need to be deployed on Kubernetes cluster and these apps have
some pre-requisites where some configurations need to be changed before deploying the app
container. Some of these changes cannot be made inside the images so the DevOps team has come up
with a solution to use init containers to perform these tasks during deployment. Below is a sample
scenario that the team is going to test first.

1. Create a `Deployment` named as `ic-deploy-devops`.
2. Configure `spec` as replicas should be `1`, labels `app` should be `ic-devops`, template's
   metadata lables `app` should be the same `ic-devops`.
3. The `initContainers` should be named as `ic-msg-devops`, use image `ubuntu` with `latest` tag
   and use command `'/bin/bash'`, `'-c'` and
   `'echo Init Done - Welcome to xFusionCorp Industries > /ic/ecommerce'`. The volume mount should
   be named as `ic-volume-devops` and mount path should be `/ic`.
4. Main container should be named as `ic-main-devops`, use image `ubuntu` with `latest` tag and
   use command `'/bin/bash'`, `'-c'` and `'while true; do cat /ic/ecommerce; sleep 5; done'`. The
   volume mount should be named as `ic-volume-devops` and mount path should be `/ic`.
5. Volume to be named as `ic-volume-devops` and it should be an emptyDir type.

> `Note:` The `kubectl` utility on the `jump-host` has been configured to work with the Kubernetes
> cluster.

## Solution

`deploy.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ic-deploy-devops
  labels:
    app: ic-devops
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ic-devops
  template:
    metadata:
      labels:
        app: ic-devops
    spec:
      initContainers:
        - name: ic-msg-devops
          image: ubuntu:latest
          command: ["/bin/bash", "-c", "echo Init Done - Welcome to xFusionCorp Industries > /ic/ecommerce"]
          volumeMounts:
            - name: ic-volume-devops
              mountPath: /ic
      containers:
        - name: ic-main-devops
          image: ubuntu:latest
          command: ["/bin/bash", "-c", "while true; do cat /ic/ecommerce; sleep 5; done"]
          volumeMounts:
            - name: ic-volume-devops
              mountPath: /ic
      volumes:
        - name: ic-volume-devops
          emptyDir: {}
```

Apply and verify:

```bash
kubectl apply -f deploy.yaml
kubectl rollout status deployment/ic-deploy-devops
kubectl logs deployment/ic-deploy-devops -c ic-main-devops
# Init Done - Welcome to xFusionCorp Industries
```

## Notes

- Init containers run to completion before the main container starts.
- `emptyDir` lets the init and main containers share `/ic` for the lifetime of the Pod.
