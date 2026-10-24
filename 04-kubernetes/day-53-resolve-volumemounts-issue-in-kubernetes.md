# Day 53: Resolve VolumeMounts Issue in Kubernetes

## Question

The Nginx and PHP-FPM application in pod `nginx-phpfpm` is not working. Diagnose and correct the
mount configuration using the `nginx-config` ConfigMap. Then copy `/home/thor/index.php` from the
jump host into the Nginx document root in container `nginx-container`.

> `kubectl` on the jump host is already configured to communicate with the Kubernetes cluster.

## Diagnosis

Inspect the ConfigMap to find the Nginx document root, then inspect the pod's mounted paths:

```bash
kubectl get configmap nginx-config -o yaml
# ...
# root /var/www/html;

kubectl get pod nginx-phpfpm -o yaml
```

The `shared-files` volume must be mounted at the same path in both containers. In this case, Nginx
uses `/var/www/html`, but `php-fpm-container` incorrectly mounts it at
`/usr/share/nginx/html`. PHP-FPM therefore cannot see the PHP files that Nginx receives.

## Solution

Save a copy of the current Pod manifest:

```bash
kubectl get pod nginx-phpfpm -o yaml > /tmp/nginx-phpfpm.yaml
vi /tmp/nginx-phpfpm.yaml
```

In the `php-fpm-container` section, change only the `shared-files` volume mount to this path:

```yaml
volumeMounts:
  - name: shared-files
    mountPath: /var/www/html
```

Before recreating the Pod, remove the generated fields from the saved manifest:

```yaml
metadata:
  creationTimestamp: ...
  resourceVersion: ...
  uid: ...
  # remove managedFields if present
status: ...
```

`volumeMounts` cannot be changed on an existing Pod. Replace the Pod from the corrected manifest:

```bash
kubectl replace --force -f /tmp/nginx-phpfpm.yaml
# pod "nginx-phpfpm" deleted
# pod/nginx-phpfpm replaced

kubectl wait --for=condition=Ready pod/nginx-phpfpm --timeout=60s
# pod/nginx-phpfpm condition met
```

Copy the PHP file into Nginx's document root and verify that both containers see it through the
shared volume:

```bash
kubectl cp /home/thor/index.php nginx-phpfpm:/var/www/html/index.php -c nginx-container

kubectl exec nginx-phpfpm -c nginx-container -- ls -l /var/www/html/index.php
kubectl exec nginx-phpfpm -c php-fpm-container -- ls -l /var/www/html/index.php
```

Open the **Website** button to confirm the PHP application responds.

## Why

| Command or setting | Why |
|:--|:--|
| `nginx-config` | The ConfigMap shows Nginx's configured document root: `/var/www/html`. |
| `shared-files` | An `emptyDir`-style shared volume only shares files when containers mount the same volume. |
| `/var/www/html` in both containers | Gives Nginx and PHP-FPM the same view of the PHP files. |
| `kubectl replace --force` | Deletes and recreates the Pod because its volume mounts are immutable. |
| `kubectl wait` | Ensures the replacement Pod is ready before copying the application file. |
| `kubectl cp ... -c nginx-container` | Targets the required Nginx container and its actual document root. |

## Notes

- The pod name is exactly `nginx-phpfpm`—not `nginx-php-fpm`.
- Do not alter the `nginx-config` ConfigMap for this fix. It correctly points Nginx at `/var/www/html`; it is the PHP-FPM mount path that is wrong.
- Removing generated metadata and `status` is necessary when turning output from `kubectl get -o yaml` into a createable manifest.
- `kubectl replace --force` briefly removes the Pod. For production workloads, use a Deployment or another controller so replacement is managed and availability can be maintained.
