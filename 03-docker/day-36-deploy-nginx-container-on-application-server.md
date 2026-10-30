# Day 36: Deploy Nginx Container on Application Server

## Question

The Nautilus DevOps team is conducting application deployment tests on selected application
servers. They require a nginx container deployment on `Application Server 1`. Complete the task
with the following instructions:

1. On `Application Server 1` create a container named `nginx_1` using the `nginx` image with the
   `alpine` tag. Ensure container is in a `running` state.

## Solution

```bash
ssh tony@stapp01

docker ps
# nothing running yet

docker run -d --name nginx_1 nginx:alpine

docker ps
# CONTAINER ID   IMAGE          COMMAND                  STATUS         NAMES
# a1b2c3d4e5f6   nginx:alpine   "/docker-entrypoint.…"   Up 3 seconds   nginx_1
```

## Why

| Command | Why |
|:--|:--|
| `-d` | Detached. Without it nginx holds your terminal, and Ctrl-C kills the container. |
| `--name nginx_1` | The check looks for that exact name. Without it docker invents one like `vibrant_hopper`. |
| `nginx:alpine` | Drop the tag and you get `nginx:latest`, a different image, which fails the requirement. |
| `docker ps` again | "In a running state" is a requirement, and a container can be created and exit immediately. |

## Notes

- A container lives exactly as long as its main process. nginx stays in the foreground, so it stays up.
- `docker ps` shows running only. `docker ps -a` includes the ones that died, then `docker logs nginx_1` says why.
- No ports published here, so nothing is reachable from outside the host. That would need `-p 8080:80`.
- Start over with `docker rm -f nginx_1`, which removes it even while running.
