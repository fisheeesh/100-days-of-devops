# Day 60: Persistent Volumes in Kubernetes

## Question

The Nautilus DevOps team is preparing a Kubernetes template to deploy a web application that stores
its application code on persistent storage. Complete the following requirements:

1. Create a PersistentVolume named `pv-nautilus` with storage class `manual`, capacity `4Gi`,
   access mode `ReadWriteOnce`, and a `hostPath` volume at `/mnt/dba`.
2. Create a PersistentVolumeClaim named `pvc-nautilus` with storage class `manual`, a `2Gi`
   storage request, and access mode `ReadWriteOnce`.
3. Create a Pod named `pod-nautilus` with a container named `container-nautilus` using
   `nginx:latest`. Mount the claim `pvc-nautilus` at the Nginx document root.
4. Create a NodePort Service named `web-nautilus` that exposes the web server on node port
   `30008`.

> `kubectl` on the jump host is already configured to communicate with the Kubernetes cluster.

## Solution

Using the repository's `nautilus` naming convention, create these manifests on the jump host.

`pv.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nautilus
spec:
  storageClassName: manual
  capacity:
    storage: 4Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/dba
```

`pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-nautilus
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```

`pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-nautilus
  labels:
    app: pod-nautilus
spec:
  volumes:
    - name: nautilus-storage
      persistentVolumeClaim:
        claimName: pvc-nautilus
  containers:
    - name: container-nautilus
      image: nginx:latest
      volumeMounts:
        - name: nautilus-storage
          mountPath: /usr/share/nginx/html
```

`service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-nautilus
spec:
  type: NodePort
  selector:
    app: pod-nautilus
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30008
```

Apply and verify:

```bash
kubectl apply -f pv.yaml -f pvc.yaml -f pod.yaml -f service.yaml

kubectl get pv pv-nautilus
kubectl get pvc pvc-nautilus
kubectl get pod pod-nautilus
kubectl get service web-nautilus
```

## Notes

- The PVC requests `2Gi` and binds to the `4Gi` PV because both use storage class `manual` and
  access mode `ReadWriteOnce`.
- Mounting the claim at `/usr/share/nginx/html` replaces Nginx's default document root with the
  persistent volume.
