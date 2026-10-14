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

| Option or command | Why |
|:--|:--|
| `docker pull nginx:alpine-perl` | Downloads the exact nginx variant requested by the task instead of the default `latest` tag. |
| `-d` | Runs nginx in detached mode so the container remains in the background. |
| `--name apps` | Assigns the exact container name required by the ticket. |
| `-p 8084:80` | Publishes host port `8084` and forwards its traffic to nginx on container port `80`. The order is `HOST:CONTAINER`. |
| `docker ps` | Confirms that `apps` is still running and displays its published port. |
| `curl` | Proves that nginx is reachable through the host-side port mapping. |

## Notes

- `EXPOSE 80` in the nginx image documents its listening port, but does not publish it. The `-p` option creates the host mapping.
- Omitting `-d` attaches nginx to the terminal. Exiting with `Ctrl+C` would stop the container and fail the running-state requirement.
- Port `8084` must be free on the host. If another process already owns it, Docker reports `address already in use` and does not start `apps`.
