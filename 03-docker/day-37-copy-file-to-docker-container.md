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
| `docker ps -a` | A stopped container does not appear in plain `docker ps`, and you would think it was missing. |
| `docker cp <src> <container>:<dest>` | Host to container. Whichever side carries the `container:` prefix is the container side. |
| `/usr/src` as destination | It already exists in the image. If it did not, docker refuses with `no such directory` rather than creating it. |
| `md5sum` on both sides | "Ensure the file is not modified" needs matching checksums. `ls` only shows that *a* file arrived. |

## Notes

- The copy is byte identical, confirmed by matching md5. `docker cp` streams a tar archive and never rewrites content, so binary files are safe.
- It works on a **stopped** container too, since it touches the filesystem on disk, not a running process.
- Ownership comes across as a numeric UID. A root-owned host file shows as `root root` inside.
- `-it` on `docker exec` is only for interactive shells, and it breaks in scripts with no terminal.
- Anything written into a container's own filesystem dies with the container. Use a volume to outlive it.
