# Day 64: Fix Python App Deployed on Kubernetes Cluster

## Question

One of the DevOps engineers was trying to deploy a python app on Kubernetes cluster.
Unfortunately, due to some mis-configuration, the application is not coming up. Please take a look
into it and fix the issues. Application should be accessible on the specified nodePort.

1. The deployment name is `python-deployment-nautilus`, its using `poroko/flask-demo-app` image.
   The deployment and service of this app is already deployed.
2. nodePort should be `32345` and targetPort should be python flask app's default port.

> `Note:` The `kubectl` utility on the `jump-host` has been configured to work with the Kubernetes
> cluster.

## Solution

```bash
k get all
# pods are not ready

k describe deployment python-deployment-nautilus
# Events show the image pull failing on a misspelled image name

k edit deployment python-deployment-nautilus
# fix the image to:  poroko/flask-demo-app

k get all
# pods Running now, but the app is still not reachable
```

Second fault, in the Service:

```bash
k describe service python-deployment-nautilus
# targetPort is not the port flask listens on

k edit service python-deployment-nautilus
#   port: 5000
#   targetPort: 5000
#   nodePort: 32345

k get all
kubectl get nodes -o wide
curl http://<node-ip>:32345
```

## Why

| Command | Why |
|:--|:--|
| `k describe deployment` | The Events name the exact image string it tried to pull, which is where the typo shows. |
| `k edit deployment` | Changing the pod template triggers a fresh rollout. Deleting pods would just recreate them from the same bad template. |
| `k describe service` | Pods Running but nothing served points at the Service, not the app. |
| `targetPort: 5000` | Flask's default port. `targetPort` has to match what the container actually listens on, or every request is refused. |

## Notes

- Two separate faults in one task. Fixing the image gets the pods Running and the app is still unreachable, so re-check after each fix instead of assuming one was the whole story. Same shape as Day 59.
- `port` is the Service's own port inside the cluster, `targetPort` is the container's. They need not match each other, but `targetPort` must match the app.
- `kubectl get endpoints python-deployment-nautilus` empty means the selector matches no pods. Populated but still failing means `targetPort` is wrong.
- A NodePort answers on the node's IP, not on `localhost`, unless you are on a node.
