# Day 63: Deploy Iron Gallery App on Kubernetes

## Question

Deploy the Iron Gallery application and its database with the following requirements:

1. Create namespace `iron-namespace-devops`.
2. Create Deployment `iron-gallery-deployment-devops` in that namespace with one replica, label
   `run: iron-gallery`, container `iron-gallery-container-devops`, and image
   `kodekloud/irongallery:2.0`. Set limits to `100Mi` memory and `50m` CPU. Mount `emptyDir`
   volumes `config` and `images` at `/usr/share/nginx/html/data` and
   `/usr/share/nginx/html/uploads`.
3. Create Deployment `iron-db-deployment-devops` in the same namespace with one replica, label
   `db: mariadb`, container `iron-db-container-devops`, and image `kodekloud/irondb:2.0`. Set
   `MYSQL_DATABASE=database_blog`, a non-root `MYSQL_USER`, and complex values for
   `MYSQL_ROOT_PASSWORD` and `MYSQL_PASSWORD`. Mount `emptyDir` volume `db` at `/var/lib/mysql`.
4. Create ClusterIP Service `iron-db-service-devops` on TCP port `3306`, selecting
   `db: mariadb`.
5. Create NodePort Service `iron-gallery-service-devops` on TCP port `80`, target port `80`, and
   node port `32678`, selecting `run: iron-gallery`.

> The application and database do not need to be connected for this task. `kubectl` on the jump
> host is already configured to communicate with the Kubernetes cluster.

## Solution

`namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: iron-namespace-devops
```

`gallery-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: iron-gallery-deployment-devops
  namespace: iron-namespace-devops
  labels:
    run: iron-gallery
spec:
  replicas: 1
  selector:
    matchLabels:
      run: iron-gallery
  template:
    metadata:
      labels:
        run: iron-gallery
    spec:
      containers:
        - name: iron-gallery-container-devops
          image: kodekloud/irongallery:2.0
          resources:
            limits:
              memory: 100Mi
              cpu: 50m
          volumeMounts:
            - name: config
              mountPath: /usr/share/nginx/html/data
            - name: images
              mountPath: /usr/share/nginx/html/uploads
      volumes:
        - name: config
          emptyDir: {}
        - name: images
          emptyDir: {}
```

`db-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: iron-db-deployment-devops
  namespace: iron-namespace-devops
  labels:
    db: mariadb
spec:
  replicas: 1
  selector:
    matchLabels:
      db: mariadb
  template:
    metadata:
      labels:
        db: mariadb
    spec:
      containers:
        - name: iron-db-container-devops
          image: kodekloud/irondb:2.0
          env:
            - name: MYSQL_DATABASE
              value: database_blog
            - name: MYSQL_ROOT_PASSWORD
              value: "R00t!Iron_2026"
            - name: MYSQL_USER
              value: ironuser
            - name: MYSQL_PASSWORD
              value: "Db!Iron_2026"
          volumeMounts:
            - name: db
              mountPath: /var/lib/mysql
      volumes:
        - name: db
          emptyDir: {}
```

`services.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: iron-db-service-devops
  namespace: iron-namespace-devops
spec:
  type: ClusterIP
  selector:
    db: mariadb
  ports:
    - protocol: TCP
      port: 3306
      targetPort: 3306
---
apiVersion: v1
kind: Service
metadata:
  name: iron-gallery-service-devops
  namespace: iron-namespace-devops
spec:
  type: NodePort
  selector:
    run: iron-gallery
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 32678
```

Apply and verify:

```bash
kubectl apply -f namespace.yaml
kubectl apply -f gallery-deployment.yaml -f db-deployment.yaml -f services.yaml
kubectl get all -n iron-namespace-devops
```

## Notes

- The DB Service selector is `db: mariadb`; `run: mariadb` would not select the database Pod.
- The `emptyDir` volumes are temporary and are deleted when their Pods are deleted.
