# Day 47: Docker Python App

## Question

A Python application must be containerized and deployed on `App Server 3`. The application files,
including `requirements.txt` and `server.py`, are already under `/python_app/src/`. Complete the
following task:

1. Create `/python_app/Dockerfile` that:
   - Uses a Python base image.
   - Installs dependencies from `requirements.txt`.
   - Exposes port `3000`.
   - Runs `server.py` with `CMD`.
2. Build an image named `nautilus/python-app`.
3. Create a container named `pythonapp_nautilus` and map host port `8094` to container port `3000`.
4. Verify the application from `App Server 3` with `curl`.

## Solution

Connect to the server and create the Dockerfile in the build-context root:

```bash
ssh banner@stapp03

sudo -i
cd /python_app
vi Dockerfile
```

Add the following content:

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY src/requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY src/ .

EXPOSE 3000

CMD ["python", "server.py"]
```

Build the image from `/python_app`:

```bash
docker build -t nautilus/python-app .
```

Create and start the container with the required name and port mapping:

```bash
docker run -d \
  --name pythonapp_nautilus \
  -p 8094:3000 \
  nautilus/python-app
```

Verify the deployment:

```bash
docker ps --filter name=pythonapp_nautilus

docker port pythonapp_nautilus
# 3000/tcp -> 0.0.0.0:8094

curl http://localhost:8094/
# Welcome to xFusionCorp Industries!
```

## Why

| Instruction or command | Why |
|:--|:--|
| `FROM python:3.12-slim` | Supplies Python 3.12 in a smaller Debian-based image. |
| `WORKDIR /app` | Creates and selects the directory used by the remaining build steps and the runtime command. |
| First `COPY` | Copies only `requirements.txt` before dependency installation, allowing Docker to reuse the dependency layer when application code changes. |
| `pip install --no-cache-dir` | Installs the declared dependencies without retaining pip's download cache in the image. |
| `COPY src/ .` | Copies `server.py` and the rest of the application into `/app`. Without this step, the runtime command cannot find the script. |
| `EXPOSE 3000` | Documents the port on which the application listens inside the container. |
| JSON-form `CMD` | Starts `python server.py` as the container's main process without an extra shell. |
| `-p 8094:3000` | Publishes host port `8094` and forwards requests to the application on container port `3000`. |

## Notes

- `CMD ["python", "server.py"]` uses JSON syntax, so its strings require double quotes. Single quotes make the JSON form invalid.
- `EXPOSE 3000` does not publish the port. The `docker run -p` option creates the host-to-container mapping.
- The application must listen on `0.0.0.0:3000` inside the container. If `server.py` binds only to `127.0.0.1`, the host port mapping cannot reach it.
- If the container is absent from `docker ps`, inspect `docker ps -a` and `docker logs pythonapp_nautilus` to see why the Python process exited.
