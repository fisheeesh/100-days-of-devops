# Day 63: Deploy Iron Gallery App on Kubernetes

## Question

There is an iron gallery app that the Nautilus DevOps team was developing. They have recently
customized the app and are going to deploy the same on the Kubernetes cluster. Below you can find
more details:

1. Create a namespace `iron-namespace-devops`
2. Create a deployment `iron-gallery-deployment-devops` for `iron gallery` under the same
   namespace you created.
   - Labels `run` should be `iron-gallery`.
   - Replicas count should be `1`.
   - Selector's matchLabels `run` should be `iron-gallery`.
   - Template labels `run` should be `iron-gallery` under metadata.
   - The container should be named as `iron-gallery-container-devops`, use
     `kodekloud/irongallery:2.0` image ( use exact image name / tag ).
   - Resources limits for memory should be `100Mi` and for CPU should be `50m`.
   - First volumeMount name should be `config`, its mountPath should be
     `/usr/share/nginx/html/data`.
   - Second volumeMount name should be `images`, its mountPath should be
     `/usr/share/nginx/html/uploads`.
   - First volume name should be `config` and give it `emptyDir` and second volume name should be
     `images`, also give it `emptyDir`.
3. Create a deployment `iron-db-deployment-devops` for `iron db` under the same namespace.
   - Labels `db` should be `mariadb`.
   - Replicas count should be `1`.
   - Selector's matchLabels `db` should be `mariadb`.
   - Template labels `db` should be `mariadb` under metadata.
   - The container name should be `iron-db-container-devops`, use `kodekloud/irondb:2.0` image
     ( use exact image name / tag ).
   - Define environment, set `MYSQL_DATABASE` its value should be `database_blog`, set
     `MYSQL_ROOT_PASSWORD` and `MYSQL_PASSWORD` value should be with some complex passwords for DB
     connections, and `MYSQL_USER` value should be any custom user ( except root ).
   - Volume mount name should be `db` and its mountPath should be `/var/lib/mysql`. Volume name
     should be `db` and give it an `emptyDir`.
4. Create a service for `iron db` which should be named `iron-db-service-devops` under the same
   namespace. Configure spec as selector's db should be `mariadb`. Protocol should be `TCP`, port
   and targetPort should be `3306` and its type should be `ClusterIP`.
5. Create a service for `iron gallery` which should be named `iron-gallery-service-devops` under
   the same namespace. Configure spec as selector's run should be `iron-gallery`. Protocol should
   be `TCP`, port and targetPort should be `80`, nodePort should be `32678` and its type should be
   `NodePort`.

> `Note:`
>
> 1. We don't need to make connection b/w database and front-end now, if the installation page is
>    coming up it should be enough for now.
> 2. The `kubectl` utility on the `jump-host` has been configured to work with the Kubernetes
>    cluster.

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
