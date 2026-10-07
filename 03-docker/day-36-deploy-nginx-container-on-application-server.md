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
| `docker ps` first | Shows what is already running. If a container called `nginx_1` exists, `run` fails with a name conflict, and knowing that up front saves confusion. |
| `-d` | Detached. Without it the container runs in the foreground and nginx holds your terminal until you kill it, which also kills the container. |
| `--name nginx_1` | The task names the container, and the check looks for that exact name. Without `--name` docker invents one like `vibrant_hopper`. |
| `nginx:alpine` | `nginx` is the image, `alpine` is the tag. Leaving the tag off means `nginx:latest`, a different and much larger image, which would fail the "alpine tag" requirement. |
| `docker ps` again | The requirement is "in a running state", and this is the only thing that proves it. A container can be created and immediately exit. |

## Notes

- `docker ps` lists running containers only. `docker ps -a` includes stopped ones, which is what you need when a container is missing from `docker ps` because it died on startup.
- If it is not running, `docker logs nginx_1` says why. The container keeps its logs after exiting.
- You do not need to pull first. `docker run` pulls the image automatically when it is not already local.
- No ports are published here. The task only asks for a running container, so nothing is reachable from outside the host yet. That would need `-p 8080:80`.
- A container runs for exactly as long as its main process. nginx stays in the foreground by design, so the container stays up. An image whose command exits immediately (like `ubuntu` with no command) goes straight to `Exited`.
- To start over: `docker rm -f nginx_1` removes it even while running, then `run` again.
- `alpine` tags are the small ones, built on Alpine Linux rather than Debian. Same nginx, a fraction of the size, which is why tasks and real deployments reach for them.
