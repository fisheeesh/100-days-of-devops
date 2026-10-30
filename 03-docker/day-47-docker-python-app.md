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

| Instruction | Why |
|:--|:--|
| `COPY requirements.txt` before the rest | Docker caches that layer, so changing application code does not reinstall dependencies. |
| `pip install --no-cache-dir` | Keeps pip's download cache out of the image. |
| `COPY src/ .` | Without it the runtime command cannot find `server.py`. |
| `-p 8094:3000` | Publishes host 8094 to the app on 3000. |

## Notes

- `CMD ["python", "server.py"]` is JSON, so the strings need double quotes. Single quotes break it.
- `EXPOSE 3000` documents the port; `-p` is what publishes it.
- The app must listen on `0.0.0.0:3000`. Bound to `127.0.0.1` it is unreachable through the port mapping.
- Missing from `docker ps`? Check `docker ps -a` and `docker logs pythonapp_nautilus`.
