# Day 61: Init Containers in Kubernetes

## Question

The Nautilus DevOps team needs to use an init container to prepare data before an application
container starts. Create a Deployment with these requirements:

1. Name the Deployment `ic-deploy-devops`, set replicas to `1`, and use label `app: ic-devops`
   for the Deployment, selector, and Pod template.
2. Create init container `ic-msg-devops` using `ubuntu:latest`. It must run
   `/bin/bash -c 'echo Init Done - Welcome to xFusionCorp Industries > /ic/ecommerce'` and mount
   volume `ic-volume-devops` at `/ic`.
3. Create main container `ic-main-devops` using `ubuntu:latest`. It must run
   `/bin/bash -c 'while true; do cat /ic/ecommerce; sleep 5; done'` and mount the same volume at
   `/ic`.
4. Create `ic-volume-devops` as an `emptyDir` volume.

> `kubectl` on the jump host is already configured to communicate with the Kubernetes cluster.

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
