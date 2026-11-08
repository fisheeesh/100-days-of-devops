# Day 67: Deploy Guest Book App on Kubernetes

## Question

The Nautilus Application development team has finished development of one of the applications and
it is ready for deployment. It is a guestbook application that will be used to manage entries for
guests/visitors. As per discussion with the DevOps team, they have finalized the infrastructure
that will be deployed on Kubernetes cluster. Below you can find more details about it.

`BACK-END TIER`

1. Create a deployment named `redis-master` for Redis master.

   a.) Replicas count should be `1`.

   b.) Container name should be `master-redis-nautilus` and it should use image `redis`.

   c.) Request resources as `CPU` should be `100m` and `Memory` should be `100Mi`.

   d.) Container port should be redis default port i.e `6379`.
2. Create a service named `redis-master` for Redis master. Port and targetPort should be Redis
   default port i.e `6379`.
3. Create another deployment named `redis-slave` for Redis slave.

   a.) Replicas count should be `2`.

   b.) Container name should be `slave-redis-nautilus` and it should use
   `gcr.io/google_samples/gb-redisslave:v3` image.

   c.) Requests resources as `CPU` should be `100m` and `Memory` should be `100Mi`.

   d.) Define an environment variable named `GET_HOSTS_FROM` and its value should be `dns`.

   e.) Container port should be Redis default port i.e `6379`.
4. Create another service named `redis-slave`. It should use Redis default port i.e `6379`.
5. Create another service named `redis-follower`. Port and targetPort should be Redis default port
   i.e `6379`. Its selector `app` should be `redis-slave`.

`FRONT END TIER`

1. Create a deployment named `frontend`.

   a.) Replicas count should be `3`.

   b.) Container name should be `php-redis-nautilus` and it should use
   `gcr.io/google-samples/gb-frontend@sha256:a908df8486ff66f2c4daa0d3d8a2fa09846a1fc8efd65649c0109695c7c5cbff`
   image.

   c.) Request resources as `CPU` should be `100m` and `Memory` should be `100Mi`.

   d.) Define an environment variable named as `GET_HOSTS_FROM` and its value should be `dns`.

   e.) Container port should be `80`.
2. Create a service named `frontend`. Its `type` should be `NodePort`, port should be `80` and its
   `nodePort` should be `30009`.

Finally, you can check the `guestbook app` by clicking on `App` button.

`You can use any labels as per your choice.`

> `Note:` The `kubectl` utility on the `jump-host` has been configured to work with the Kubernetes
> cluster.

## Solution

Back end, `redis-master-deploy.yaml` and `redis-master-svc.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-master
  labels:
    app: redis-master
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis-master
  template:
    metadata:
      labels:
        app: redis-master
    spec:
      containers:
      - name: master-redis-nautilus
        image: redis
        resources:
          requests:
            memory: "100Mi"
            cpu: "100m"
        ports:
        - containerPort: 6379
---
kind: Service
apiVersion: v1
metadata:
  name: redis-master
spec:
  selector:
    app: redis-master
  ports:
  - name: redis-master-port
    port: 6379
    targetPort: 6379
```

`redis-slave-deploy.yaml` plus both of its services. The pods are labelled `app: redis-slave` so
that **both** services select them:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-slave
  labels:
    app: redis-slave
spec:
  replicas: 2
  selector:
    matchLabels:
      app: redis-slave
  template:
    metadata:
      labels:
        app: redis-slave
    spec:
      containers:
      - name: slave-redis-nautilus
        image: gcr.io/google_samples/gb-redisslave:v3
        resources:
          requests:
            memory: "100Mi"
            cpu: "100m"
        env:
        - name: GET_HOSTS_FROM
          value: dns
        ports:
        - containerPort: 6379
---
kind: Service
apiVersion: v1
metadata:
  name: redis-slave
spec:
  selector:
    app: redis-slave
  ports:
  - name: redis-slave-port
    port: 6379
    targetPort: 6379
---
kind: Service
apiVersion: v1
metadata:
  name: redis-follower
spec:
  selector:
    app: redis-slave
  ports:
  - name: redis-follower-port
    port: 6379
    targetPort: 6379
```

Front end, `frontend-deploy.yaml` and `frontend-svc.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  labels:
    app: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: php-redis-nautilus
        image: gcr.io/google-samples/gb-frontend@sha256:a908df8486ff66f2c4daa0d3d8a2fa09846a1fc8efd65649c0109695c7c5cbff
        resources:
          requests:
            memory: "100Mi"
            cpu: "100m"
        env:
        - name: GET_HOSTS_FROM
          value: dns
        ports:
        - containerPort: 80
---
kind: Service
apiVersion: v1
metadata:
  name: frontend
spec:
  selector:
    app: frontend
  type: NodePort
  ports:
  - name: frontend-port
    port: 80
    nodePort: 30009
    targetPort: 80
```

```bash
k apply -f .
k get all
k get endpoints redis-master redis-slave redis-follower frontend
```

## Why

| Piece | Why |
|:--|:--|
| Slave pods labelled `app: redis-slave` | Requirement 5 pins the `redis-follower` selector to `redis-slave`. Label the pods anything else and that service selects nothing. |
| Two services on one deployment | `redis-slave` and `redis-follower` are different names for the same pods. The app looks one of them up by DNS. |
| `GET_HOSTS_FROM: dns` | Tells the app to resolve `redis-master` and its follower by service name instead of baked-in IPs. That is what makes the Service names load bearing. |
| Image pinned by `@sha256:` | A digest, not a tag. It resolves to one exact image and ignores any `:latest` drift. |

## Notes

- **The selector trap:** if the slave pods carry `app: backend-1` while `redis-follower` selects `app: redis-slave`, that service ends up with zero endpoints. The checker may still pass on the selector field alone, but the app's read path is broken. `k get endpoints redis-follower` showing `<none>` is the tell.
- `redis-master` uses `google_samples` with an underscore, `gb-frontend` uses `google-samples` with a hyphen. Both are as the task states, and they are easy to mistype.
- The services need no `type`, so they default to `ClusterIP`. Only `frontend` is reachable from outside.
- Three frontend replicas at `100m` each is `300m` of CPU requested before anything else. On a small cluster some pods can sit `Pending` with `Insufficient cpu`.
