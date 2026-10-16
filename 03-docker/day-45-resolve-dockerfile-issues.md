# Day 45: Resolve Dockerfile Issues

## Question

The Nautilus DevOps team is creating new images from requirements supplied by the development
team. A Docker build is failing on `App Server 1` in `Stratos DC`. Complete the following task:

1. Inspect the `Dockerfile` under `/opt/docker`.
2. Fix it so Docker can build the image successfully.
3. Do not change the base image, other valid Dockerfile configuration, or supplied data such as
   `index.html`.

> When **FINISH** is clicked, the existing containers are destroyed and the image is rebuilt from
> this Dockerfile.

## Solution

Connect to the server and inspect the build context before editing anything:

```bash
ssh tony@stapp01

sudo -i
cd /opt/docker

find . -maxdepth 2 -type f -print
# ./Dockerfile
# ./html/index.html
# ./certs/server.crt
# ./certs/server.key

nl -ba Dockerfile
```

Run the failing build once so the error identifies the broken instruction and source path:

```bash
docker build -t mylab:latest .
# build fails because the COPY source files are not at the paths used in Dockerfile
```

Edit only the invalid file-copy instructions:

```bash
vi Dockerfile
```

The corrected lines must use paths relative to `/opt/docker`, which is the build context:

```dockerfile
COPY certs/server.crt /usr/local/apache2/conf/server.crt
COPY certs/server.key /usr/local/apache2/conf/server.key
COPY html/index.html /usr/local/apache2/htdocs/
```

Leave the existing `FROM` instruction and all valid Apache configuration unchanged. Build again
from `/opt/docker` and verify the resulting image:

```bash
docker build -t mylab:latest .
# Successfully built ...
# Successfully tagged mylab:latest

docker image ls mylab:latest
```

## Why

| Command or instruction | Why |
|:--|:--|
| `find . -maxdepth 2 -type f` | Shows the real paths and spellings of every supplied file before the Dockerfile is changed. |
| `docker build ... .` | Reproduces the failure and uses `/opt/docker` as the build context because `.` is evaluated from that directory. |
| `COPY certs/server.crt ...` | Reads the certificate from `/opt/docker/certs/server.crt` and places it at the existing destination inside the image. |
| `COPY certs/server.key ...` | Copies the supplied private key from its actual build-context location. |
| `COPY html/index.html ...` | Copies the existing website file into Apache's document root without changing its contents. |
| Second `docker build` | A successful clean completion is the required proof that the Dockerfile issues are fixed. |

## Notes

- `COPY` source paths are resolved relative to the build context, not relative to arbitrary host filesystem paths. With `docker build .` from `/opt/docker`, the correct sources are `certs/...` and `html/...`.
- Do not move or edit `index.html`, the certificate, or the key just to match incorrect Dockerfile paths. Fix the Dockerfile references instead.
- Do not replace or upgrade the base image. The task explicitly requires all already-valid configuration to remain intact.
- If this lab variant uses `RUN cp certs/...`, replace those file-transfer lines with `COPY`. A `RUN` command executes inside an image layer and cannot read host build-context files until they have been copied into the image.
- Starting a container is not required to validate this task. The platform destroys existing containers and performs its own image build after **FINISH**.
