# Day 41: Write a Dockerfile

## Question

The Nautilus application development team needs a custom image for one of its projects. Create
`/opt/docker/Dockerfile` on `App Server 2` in `Stratos DC` with the following requirements:

1. Use `ubuntu:24.04` as the base image.
2. Install `apache2` and configure it to use port `6200`. Do not change other Apache settings such
   as the document root.

> The filename must be exactly `Dockerfile` with a capital `D`.

## Solution

Connect to the server and open the required file:

```bash
ssh steve@stapp02

sudo -i
cd /opt/docker
vi Dockerfile
```

Add the following content:

```dockerfile
FROM ubuntu:24.04

RUN apt-get update \
    && DEBIAN_FRONTEND=noninteractive apt-get install -y apache2 \
    && sed -i 's/^Listen 80$/Listen 6200/' /etc/apache2/ports.conf \
    && sed -i 's/<VirtualHost \*:80>/<VirtualHost *:6200>/' \
        /etc/apache2/sites-available/000-default.conf \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

EXPOSE 6200

CMD ["apache2ctl", "-D", "FOREGROUND"]
```

Build the image from the directory containing `Dockerfile`:

```bash
docker build -t apache2:ubuntu .
```

Run a test container and publish host port `6200` to the same container port:

```bash
docker run -d --name apache_6200 -p 6200:6200 apache2:ubuntu

docker ps --filter name=apache_6200

curl -I http://localhost:6200
# HTTP/1.1 200 OK
```

## Why

| Instruction or command | Why |
|:--|:--|
| `FROM ubuntu:24.04` | Sets the exact base image and tag required by the task. |
| `apt-get update` | Downloads the package indexes needed to locate `apache2` during the image build. |
| `apt-get install -y apache2` | Installs Apache without waiting for interactive confirmation. |
| `Listen 6200` | Changes Apache's listening socket from port `80` to `6200` without binding it to a specific address. |
| `<VirtualHost *:6200>` | Moves the default virtual host to the same port while leaving its document root and other settings unchanged. |
| Removing `/var/lib/apt/lists/*` | Deletes package-index files that are no longer needed, reducing the final image layer size. |
| `EXPOSE 6200` | Documents that containers created from the image are expected to accept traffic on port `6200`. |
| `CMD ... FOREGROUND` | Keeps Apache in the foreground as the container's main process, so the container remains running. |
| `-p 6200:6200` | Publishes host port `6200` to container port `6200`, allowing the host-side `curl` test to reach Apache. |

## Notes

- `/opt/docker/Dockerfile` is a file, so `cd /opt/docker/Dockerfile` fails with `Not a directory`. Change into `/opt/docker`, then edit `Dockerfile`.
- `EXPOSE` is image metadata; it does not publish the port. Port publishing happens at runtime with `docker run -p 6200:6200`.
- `-e` sets environment variables and cannot publish ports. The supplied test command must use lowercase `-p`.
- Apache must run in the foreground. Starting it as a background service would let the container's main process exit, which stops the container.
