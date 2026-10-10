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
| `docker ps --filter name=ubuntu_latest` | Confirms that the source container exists and is running before creating the image. |
| `docker commit ubuntu_latest beta:datacenter` | Captures the container's current filesystem changes as a new image with repository `beta` and tag `datacenter`. |
| `docker image ls beta:datacenter` | Verifies that the image was created with the exact required name and tag. |

## Notes

- The source container name is `ubuntu_latest`. The spelling `ubuntn_latest` would fail with `No such container`.
- `docker commit` saves changes in the container's writable layer. Data stored in mounted volumes is not included in the image.
- Docker pauses the container briefly during the commit by default to reduce the chance of capturing inconsistent data.
