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
| `docker pull busybox:musl` | Downloads the exact `musl` variant. Omitting the tag would pull `busybox:latest`, which is not the requested image. |
| `docker tag busybox:musl busybox:blog` | Adds `busybox:blog` as another local name for the image already identified by `busybox:musl`. |
| `docker image ls busybox` | Shows both repository-and-tag entries and their image IDs. |
| `docker image inspect` | Confirms that both names resolve to the same underlying image object. |

## Notes

- Re-tagging does not copy or rebuild the image. Both tags point to the same image ID and layers.
- The new `busybox:blog` tag is local to `App Server 3`; creating it does not push anything to a registry.
