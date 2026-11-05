# Day 65: Deploy Redis Deployment on Kubernetes

## Question

The Nautilus application development team observed some performance issues with one of the
application that is deployed in Kubernetes cluster. After looking into number of factors, the team
has suggested to use some in-memory caching utility for DB service. After number of discussions,
they have decided to use Redis. Initially they would like to deploy Redis on kubernetes cluster
for testing and later they will move it to production. Please find below more details about the
task:

Create a redis deployment with following parameters:

1. Create a `config map` called `my-redis-config` having `maxmemory 2mb` in `redis-config`.
2. Name of the `deployment` should be `redis-deployment`, it should use `redis:alpine` image and
   container name should be `redis-container`. Also make sure it has only `1` replica.
3. The container should request for `1` CPU.
4. Mount volumes:

   a. An Empty directory volume called `data` at path `/redis-master-data`.

   b. A configmap volume called `redis-config` at path `/redis-master`.

   c. The container should expose the port `6379`.
5. Finally, `redis-deployment` should be up and running.

> `Note:` The `kubectl` utility on the `jump-host` has been configured to work with the Kubernetes
> cluster.

## Solution

The ConfigMap, either way:

```bash
kubectl create configmap my-redis-config --from-literal=redis-config="maxmemory 2mb"
```

or `configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-redis-config
  labels:
    app: my-redis
data:
  redis-config: |
    maxmemory 2mb
```

`deploy.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-deployment
  labels:
    app: my-redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-redis
  template:
    metadata:
      labels:
        app: my-redis
    spec:
      volumes:
      - name: data
        emptyDir: {}
      - name: redis-config
        configMap:
          name: my-redis-config
      containers:
      - name: redis-container
        image: redis:alpine
        resources:
          requests:
            cpu: "1"
        ports:
        - containerPort: 6379
        volumeMounts:
        - mountPath: /redis-master-data
          name: data
        - mountPath: /redis-master
          name: redis-config
```

```bash
k apply -f .
k get all
kubectl rollout status deployment/redis-deployment
```

## Why

| Field | Why |
|:--|:--|
| Volumes declared in the **pod spec** | A container can only mount a volume the pod already declares. `volumeMounts` references it by name. |
| Volume name `redis-config`, ConfigMap name `my-redis-config` | The task names both separately. The volume is what the container mounts; `configMap.name` is what it is filled from. |
| `cpu: "1"` quoted | `1` is whole cores. Quoting keeps YAML from reading it as a number where a quantity string is expected. |
| `containerPort: 6379` | Redis's default port, which requirement 4c asks to expose. |

## Notes

- `--from-literal` and the YAML block scalar are not byte identical: `|` keeps a trailing newline, `--from-literal` does not. Either passes here, but it matters for configs that are parsed strictly.
- Mounting the ConfigMap at `/redis-master` only places a file named `redis-config` there. Redis is not told to read it, since nothing passes it as a config file. The task asks for the mount, not for redis to honour it.
- Requesting a full CPU can leave the pod `Pending` if no node has a spare core. `kubectl describe pod` says `Insufficient cpu` when that happens.
- `emptyDir` lives and dies with the pod, which is fine for a cache.
