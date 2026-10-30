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

| Command | Why |
|:--|:--|
| `find . -maxdepth 2 -type f` | Shows the real paths and spellings before editing anything. |
| `docker build ... .` | The `.` is the build context, which is what `COPY` paths are relative to. |
| `COPY certs/server.crt ...` | The fix: point the Dockerfile at where the files actually are. |
| Second `docker build` | A clean build is the proof the task wanted. |

## Notes

- `COPY` sources are relative to the **build context**, not to arbitrary host paths. From `/opt/docker` with `docker build .`, that means `certs/...` and `html/...`.
- Fix the Dockerfile, do not move the supplied files to match it.
- `RUN cp certs/...` cannot work: `RUN` executes inside a layer and cannot see host files until they are copied in.
- Do not change the base image. The task says leave valid configuration intact.
