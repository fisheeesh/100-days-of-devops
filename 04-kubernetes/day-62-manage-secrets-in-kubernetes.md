# Day 62: Manage Secrets in Kubernetes

## Question

The Nautilus DevOps team needs to store license information securely in Kubernetes. Complete these
requirements:

1. Create generic Secret `ecommerce` from `/opt/ecommerce.txt` on the jump host.
2. Create Pod `secret-xfusion`.
3. Use container `secret-container-xfusion` with image `ubuntu:latest` and keep it running with a
   sleep command.
4. Consume Secret `ecommerce` and mount it at `/opt/demo`.
5. Verify the secret file from inside the running container.

> `kubectl` on the jump host is already configured to communicate with the Kubernetes cluster.

## Solution

Create the Secret. The file basename becomes the Secret key, so the mounted file is
`ecommerce.txt`:

```bash
kubectl create secret generic ecommerce --from-file=/opt/ecommerce.txt
kubectl get secret ecommerce
```

`pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-xfusion
spec:
  volumes:
    - name: secret-volume
      secret:
        secretName: ecommerce
  containers:
    - name: secret-container-xfusion
      image: ubuntu:latest
      command: ["/bin/sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: secret-volume
          mountPath: /opt/demo
```

Apply and verify:

```bash
kubectl apply -f pod.yaml
kubectl wait --for=condition=Ready pod/secret-xfusion --timeout=60s

kubectl exec secret-xfusion -c secret-container-xfusion -- \
  cat /opt/demo/ecommerce.txt
```

## Notes

- Each key in a Secret mounted as a volume becomes a file under the mount path.
- Do not commit the license value or `/opt/ecommerce.txt` to the repository.
