# Day 43: Docker Ports Mapping

## Question

The Nautilus DevOps team plans to host an application in an nginx-based container. Complete the
following task on `Application Server 2` in `Stratos Datacenter`:

1. Pull the `nginx:alpine-perl` image.
2. Create a container named `apps` from that image.
3. Map host port `8084` to container port `80` and leave the container running.

## Solution

```bash
ssh steve@stapp02

docker pull nginx:alpine-perl

docker run -d \
  --name apps \
  -p 8084:80 \
  nginx:alpine-perl
```

Verify the container and port mapping:

```bash
docker ps --filter name=apps
# apps is Up with 0.0.0.0:8084->80/tcp

docker port apps
# 80/tcp -> 0.0.0.0:8084

curl -I http://localhost:8084
# HTTP/1.1 200 OK
```

## Why

| Option | Why |
|:--|:--|
| `nginx:alpine-perl` | The exact variant requested, not the default `latest`. |
| `-p 8084:80` | Publishes host 8084 to nginx on 80. The order is `HOST:CONTAINER`. |
| `docker ps` | Confirms it is still running and shows the published port. |

## Notes

- `EXPOSE 80` in the image is documentation. `-p` is what actually publishes the port.
- Without `-d`, nginx holds the terminal and Ctrl-C stops the container.
- Port 8084 must be free on the host, or docker refuses with `address already in use`.
