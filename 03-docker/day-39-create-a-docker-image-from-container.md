# Day 39: Create a Docker Image From Container

## Question

One of the Nautilus developer was working to test new changes on a container. He wants to keep a
backup of his changes to the container. A new request has been raised for the DevOps team to
create a new image from this container. Below are more details about it:

a. Create an image `beta:datacenter` on `Application Server 1` from a container `ubuntu_latest`
that is running on same server.

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
