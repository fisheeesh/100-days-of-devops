# Day 60: Persistent Volumes in Kubernetes

## Question

The Nautilus DevOps team is working on a Kubernetes template to deploy a web application on the
cluster. There are some requirements to create/use persistent volumes to store the application
code, and the template needs to be designed accordingly. Please find more details below:

1. Create a `PersistentVolume` named as `pv-datacenter`. Configure the `spec` as storage class
   should be `manual`, set capacity to `4Gi`, set access mode to `ReadWriteOnce`, volume type
   should be `hostPath` and set path to `/mnt/dba` (this directory is already created, you might
   not be able to access it directly, so you need not to worry about it).
2. Create a `PersistentVolumeClaim` named as `pvc-datacenter`. Configure the `spec` as storage
   class should be `manual`, request `2Gi` of the storage, set access mode to `ReadWriteOnce`.
3. Create a `pod` named as `pod-datacenter`, mount the persistent volume you created with claim
   name `pvc-datacenter` at document root of the web server, the container within the pod should
   be named as `container-datacenter` using image `nginx` with `latest` tag only (remember to
   mention the tag i.e `nginx:latest`).
4. Create a node port type service named `web-datacenter` using node port `30008` to expose the
   web server running within the pod.

> `Note:` The `kubectl` utility on the `jump-host` has been configured to work with the Kubernetes
> cluster.

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
