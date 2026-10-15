# Day 44: Write a Docker Compose File

## Question

The Nautilus application development team needs to host existing static website content with an
`httpd` container. Complete the following task on `App Server 2` in `Stratos DC`:

1. Create `/opt/docker/docker-compose.yml` using that exact filename.
2. Use the `httpd:latest` image and name the container `httpd`. The service name can be anything.
3. Map host port `8088` to container port `80`.
4. Bind mount the existing host directory `/opt/sysops` at
   `/usr/local/apache2/htdocs` inside the container. Do not modify the existing content.

## Solution

Connect to the server and create the Compose file without changing directory ownership:

```bash
ssh steve@stapp02

sudo vi /opt/docker/docker-compose.yml
```

Add the following content:

```yaml
services:
  httpd:
    image: httpd:latest
    container_name: httpd
    ports:
      - "8088:80"
    volumes:
      - "/opt/sysops:/usr/local/apache2/htdocs"
```

Validate the file, then start the service in detached mode:

```bash
docker compose -f /opt/docker/docker-compose.yml config

docker compose -f /opt/docker/docker-compose.yml up -d
```

Verify the container, port, mount, and website:

```bash
docker compose -f /opt/docker/docker-compose.yml ps

docker port httpd
# 80/tcp -> 0.0.0.0:8088

docker inspect httpd --format \
  '{{range .Mounts}}{{.Source}} -> {{.Destination}}{{end}}'
# /opt/sysops -> /usr/local/apache2/htdocs

curl -I http://localhost:8088
# HTTP/1.1 200 OK
```

## Why

| Compose field or command | Why |
|:--|:--|
| `services` | Top-level map containing the containers managed by this Compose application. |
| `image: httpd:latest` | Uses the requested Apache HTTP Server image and tag. No image build is needed. |
| `container_name: httpd` | Overrides Compose's generated name and creates the exact container name required by the task. |
| `"8088:80"` | Publishes host port `8088` to the web server on container port `80`. The order is `HOST:CONTAINER`. |
| `/opt/sysops:/usr/local/apache2/htdocs` | Bind mounts the existing website directory over the image's document root, so Apache serves the supplied host content. |
| `docker compose ... config` | Parses and renders the Compose model, catching YAML or field errors before container creation. |
| `docker compose ... up -d` | Creates and starts the service in the background. |

## Notes

- SSH syntax is `ssh steve@stapp02`, not `ssh steve:stapp02`.
- Do not run `chown -R` on `/opt/docker`; it changes existing ownership beyond the task. `sudo vi` creates the required file without altering the directory or unrelated files.
- `--build` is unnecessary because this Compose file has an `image` field and no `build` section.
- The bind mount does not copy `/opt/sysops`. It presents that same host directory at Apache's document root, so edits on either side would affect the shared content.
- Modern Docker Compose does not require a top-level `version` field.
