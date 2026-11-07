# Day 66: Deploy MySQL on Kubernetes

## Question

A new MySQL server needs to be deployed on Kubernetes cluster. The Nautilus DevOps team was
working on to gather the requirements. Recently they were able to finalize the requirements and
shared them with the team members to start working on it. Below you can find the details:

1.) Create a PersistentVolume `mysql-pv`, its capacity should be `250Mi`, set other parameters as
per your preference.

- .) Create a PersistentVolumeClaim to request this PersistentVolume storage. Name it as
  `mysql-pv-claim` and request a `50Mi` of storage. Set other parameters as per your preference.

3.) Create a deployment named `mysql-deployment`, use any mysql image as per your preference.
Mount the PersistentVolume at mount path `/var/lib/mysql`.

4.) Create a `NodePort` type service named `mysql` and set nodePort to `30007`.

5.) Create a secret named `mysql-root-pass` having a key pair value, where key is `password` and
its value is `YUIidhb667`, create another secret named `mysql-user-pass` having some key pair
values, where first key is `username` and its value is `kodekloud_cap`, second key is `password`
and value is `BruCStnMT5`, create one more secret named `mysql-db-url`, key name is `database` and
value is `kodekloud_db6`

6.) Define some environment variables within the container:

a.) `name: MYSQL_ROOT_PASSWORD`, should pick value from secretKeyRef `name: mysql-root-pass` and
`key: password`

b.) `name: MYSQL_DATABASE`, should pick value from secretKeyRef `name: mysql-db-url` and
`key: database`

c.) `name: MYSQL_USER`, should pick value from secretKeyRef `name: mysql-user-pass` key
`key: username`

d.) `name: MYSQL_PASSWORD`, should pick value from secretKeyRef `name: mysql-user-pass` and
`key: password`

> `Note:` The `kubectl` utility on the `jump-host` has been configured to work with the Kubernetes
> cluster.

## Solution

The three secrets:

```bash
k create secret generic mysql-root-pass --from-literal=password=YUIidhb667

k create secret generic mysql-user-pass --from-literal=username=kodekloud_cap \
                                        --from-literal=password=BruCStnMT5

k create secret generic mysql-db-url --from-literal=database=kodekloud_db6
```

`pv.yaml` and `pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
  labels:
    app: mysql-app
spec:
  capacity:
    storage: 250Mi
  accessModes:
  - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
  labels:
    app: mysql-app
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 50Mi
```

`deploy.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-deployment
  labels:
    app: mysql-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql-app
  template:
    metadata:
      labels:
        app: mysql-app
    spec:
      volumes:
      - name: mysql-pv
        persistentVolumeClaim:
          claimName: mysql-pv-claim
      containers:
      - name: mysql-container
        image: mysql:latest
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-root-pass
              key: password
        - name: MYSQL_DATABASE
          valueFrom:
            secretKeyRef:
              name: mysql-db-url
              key: database
        - name: MYSQL_USER
          valueFrom:
            secretKeyRef:
              name: mysql-user-pass
              key: username
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-user-pass
              key: password
        volumeMounts:
        - mountPath: /var/lib/mysql
          name: mysql-pv
```

`svc.yaml`:

```yaml
kind: Service
apiVersion: v1
metadata:
  name: mysql
spec:
  selector:
    app: mysql-app
  type: NodePort
  ports:
  - name: mysql-port
    port: 3306
    nodePort: 30007
    targetPort: 3306
```

```bash
k apply -f .
k get all
k get pv,pvc
kubectl rollout status deployment/mysql-deployment
```

## Why

| Piece | Why |
|:--|:--|
| PVC requests `50Mi`, PV offers `250Mi` | A claim is a **minimum**, not an exact size. A bigger PV satisfies a smaller request, which is why these bind. |
| `secretKeyRef` rather than plain values | Requirement 6 names the secret and key for each variable. The pod reads them at start, so the password never appears in the Deployment. |
| `volumes[].name: mysql-pv` | A pod-local name. It happens to match the PV's name, but what ties it to the storage is `claimName: mysql-pv-claim`. |
| `nodePort: 30007` with `targetPort: 3306` | 3306 is where MySQL listens; 30007 is the fixed external port the task asks for. |

## Notes

- Secrets are **base64, not encryption**. `YUIidhb667` is stored as `WVVJaWRoYjY2Nw==`, and `base64 -d` reads it straight back. They keep credentials out of manifests, not out of reach.
- If the PVC sits `Pending`, the usual cause is a default StorageClass getting injected into the claim while the PV has none, so they never match. `kubectl describe pvc mysql-pv-claim` says so.
- `MYSQL_USER` cannot be `root`. The image creates that user itself, and passing root here makes it fail at startup.
- The `MYSQL_*` variables only apply on a **first** initialisation. If `/var/lib/mysql` already holds a database, changing them does nothing.
