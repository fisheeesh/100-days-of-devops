# Day 39: Create a Docker Image From Container

## Question

One of the Nautilus developers was testing new changes in a container and wants to keep a backup
of those changes. The DevOps team has received the following request:

Create an image named `beta:datacenter` on `Application Server 1` from the running container
`ubuntu_latest` on the same server.

## Solution

```bash
ssh tony@stapp01

docker ps --filter name=ubuntu_latest
# confirm ubuntu_latest is running

docker commit ubuntu_latest beta:datacenter
# sha256:<new-image-id>

docker image ls beta:datacenter
# REPOSITORY   TAG          IMAGE ID       CREATED          SIZE
# beta         datacenter   <new-id>       ...              ...
```

## Why

| Command | Why |
|:--|:--|
| `docker commit ubuntu_latest beta:datacenter` | Captures the container's filesystem changes as a new image with the required name and tag. |
| `docker image ls beta:datacenter` | Confirms the exact repository and tag the task asked for. |

## Notes

- The container is `ubuntu_latest`. `ubuntn_latest` fails with `No such container`.
- `commit` saves the container's writable layer. Data in mounted volumes is **not** included.
- Docker pauses the container briefly during the commit to avoid capturing a half-written state.
