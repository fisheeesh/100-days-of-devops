# Day 62: Manage Secrets in Kubernetes

## Question

The Nautilus DevOps team is working to deploy some tools in Kubernetes cluster. Some of the tools
are licence based so that licence information needs to be stored securely within Kubernetes
cluster. Therefore, the team wants to utilize Kubernetes secrets to store those secrets. Below you
can find more details about the requirements:

1. We already have a secret key file `ecommerce.txt` under the `/opt/` directory. Create a
   `generic secret` named `ecommerce`, it should contain the password/license-number present in
   `ecommerce.txt` file.
2. Also create a `pod` named `secret-xfusion`.
3. Configure pod's `spec` as container name should be `secret-container-xfusion`, image should be
   `ubuntu` with `latest` tag (remember to mention the tag with image). Use `sleep` command for
   container so that it remains in running state. Consume the created secret and mount it under
   `/opt/demo` within the container.
4. To verify you can exec into the container `secret-container-xfusion`, to check the secret key
   under the mounted path `/opt/demo`. Before hitting the `Check` button please make sure
   pod/pods are in running state, also validation can take some time to complete so keep patience.

> `Note:` The `kubectl` utility on the `jump-host` has been configured to work with the Kubernetes
> cluster.

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
