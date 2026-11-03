# Day 44: Write a Docker Compose File

## Question

The Nautilus application development team shared static website content that needs to be hosted on
the `httpd` web server using a containerised platform. The team has shared details with the DevOps
team, and we need to set up an environment according to those guidelines. Below are the details:

a. On `App Server 2` in `Stratos DC` create a container named `httpd` using a docker compose file
`/opt/docker/docker-compose.yml` (please use the exact name for file).

b. Use `httpd` (preferably `latest` tag) image for container and make sure container is named as
`httpd`; you can use any name for service.

c. Map `80` number port of container with port `8088` of docker host.

d. Map container's `/usr/local/apache2/htdocs` volume with `/opt/sysops` volume of docker host
which is already there. (please do not modify any data within these locations).

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

| Field | Why |
|:--|:--|
| `container_name: httpd` | Overrides Compose's generated name to the exact one the task wants. |
| `"8088:80"` | Host 8088 to the web server on 80. The order is `HOST:CONTAINER`. |
| `/opt/sysops:/usr/local/apache2/htdocs` | Bind mounts the existing site over the image's document root. |
| `docker compose config` | Catches YAML and field errors before anything is created. |

## Notes

- A bind mount does not copy. It presents the same host directory inside the container, so edits on either side are the same files.
- `--build` is pointless here: the file has `image:` and no `build:` section.
- Modern Compose does not need a top-level `version` field.
- SSH syntax is `ssh steve@stapp02`, not `ssh steve:stapp02`.
