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

| Instruction | Why |
|:--|:--|
| `apt-get install -y apache2` | `-y` because a build has nobody to answer the prompt. |
| `Listen 6200` + `<VirtualHost *:6200>` | Both have to change. The port in `ports.conf` alone leaves the default site on 80. |
| Removing `/var/lib/apt/lists/*` | Drops the package indexes from the layer, which are dead weight in the image. |
| `EXPOSE 6200` | Documentation only. It does not publish anything. |
| `CMD ... FOREGROUND` | Apache must be the container's foreground process, or the container exits immediately. |

## Notes

- `/opt/docker/Dockerfile` is a file. `cd` into `/opt/docker` and edit `Dockerfile` there.
- `EXPOSE` does not publish the port. That happens at run time with `-p 6200:6200`.
- `-e` sets environment variables and cannot publish ports. The test command needs lowercase `-p`.
