# Day 38: Pull Docker Image

## Question

Nautilus project developers are planning to start testing on a new project. As per their meeting
with the DevOps team, they want to test containerized environment application features. Complete
the following task:

Pull the `busybox:musl` image on `App Server 3` in `Stratos DC` and re-tag it as `busybox:blog`.

## Solution

```bash
ssh banner@stapp03

docker image ls busybox

docker pull busybox:musl

docker tag busybox:musl busybox:blog

docker image ls busybox
# REPOSITORY   TAG    IMAGE ID       CREATED       SIZE
# busybox      blog   <same-id>      ...           ...
# busybox      musl   <same-id>      ...           ...
```

Verify that both tags point to the same local image:

```bash
docker image inspect --format '{{.Id}}' busybox:musl busybox:blog
# both lines contain the same sha256 image ID
```

## Why

| Command | Why |
|:--|:--|
| `docker pull busybox:musl` | Without the tag you get `busybox:latest`, not the requested image. |
| `docker tag busybox:musl busybox:blog` | Adds a second local name for the image that is already there. |
| `docker image inspect` | Confirms both names resolve to the same image ID. |

## Notes

- Re-tagging copies nothing. Both tags point at the same image ID and the same layers.
- `busybox:blog` is local to this server. Creating a tag pushes nothing to a registry.
