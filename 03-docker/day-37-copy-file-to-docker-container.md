# Day 37: Copy File to Docker Container

## Question

The Nautilus DevOps team possesses confidential data on `App Server 2` in the
`Stratos Datacenter`. A container named `ubuntu_latest` is running on the same server.

Copy an encrypted file `/tmp/nautilus.txt.gpg` from the docker host to the `ubuntu_latest`
container located at `/usr/src/`. Ensure the file is not modified during this operation.

## Solution

```bash
ssh steve@stapp02

docker ps -a
# confirm ubuntu_latest is there

docker cp
# docker: 'docker cp' requires 2 arguments
# Usage:  docker cp [OPTIONS] CONTAINER:SRC_PATH DEST_PATH|-

docker cp /tmp/nautilus.txt.gpg ubuntu_latest:/usr/src

docker exec ubuntu_latest ls -lah /usr/src
# -rw-r--r-- 1 root root 105 Oct  8 06:52 nautilus.txt.gpg
```

Prove it came across unchanged:

```bash
md5sum /tmp/nautilus.txt.gpg
docker exec ubuntu_latest md5sum /usr/src/nautilus.txt.gpg
# same hash on both sides
```

## Why

| Command | Why |
|:--|:--|
| `docker ps -a` | Confirms `ubuntu_latest` exists and what state it is in. `-a` matters: a stopped container does not appear in plain `docker ps`, and you would think it was missing. |
| `docker cp` with no arguments | Prints the usage line. Worth knowing, because the direction is easy to get backwards. |
| `docker cp <src> <container>:<dest>` | Host to container. Swap the two sides to copy out. Whichever side carries the `container:` prefix is the container side. |
| `/usr/src` as the destination | That directory already exists in the image, so the file lands inside it. If it did not exist, docker refuses with `no such directory` rather than creating it. |
| `md5sum` on both sides | The task says "ensure the file is not modified". Matching checksums are the proof. `ls` only shows that *a* file arrived. |

## Notes

- The copy is byte identical, confirmed by matching md5 on host and container. `docker cp` streams the file as a tar archive and never rewrites content, so this holds for binary files such as a `.gpg` just as it does for text.
- It also works on a **stopped** container. `docker cp` reads and writes the container's filesystem on disk, so nothing needs to be running.
- Ownership carries over as a numeric UID/GID from the host. The lab's file is root-owned, so it shows as `root root` inside. A file owned by a normal user arrives with that user's UID and often no matching name in the container. `-a` (archive mode) preserves uid/gid explicitly.
- `-it` on `docker exec` is only for interactive shells. A one-shot `ls` or `md5sum` does not need it, and `-it` breaks in scripts where there is no terminal attached.
- Copying out is the same command reversed: `docker cp ubuntu_latest:/usr/src/nautilus.txt.gpg .`
- Anything written into a container's own filesystem dies with the container. `docker rm` and it is gone. Files that must outlive the container belong in a volume or a bind mount.
- `-L` makes docker follow a symlink in the source path. By default it copies the link itself.
