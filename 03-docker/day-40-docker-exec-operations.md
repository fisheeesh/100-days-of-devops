# Day 40: Docker EXEC Operations

## Question

One of the Nautilus DevOps team members was configuring services in a `kkloud` container running
on `App Server 3` in `Stratos Datacenter`. Complete the remaining work:

1. Install `apache2` in the `kkloud` container using `apt`.
2. Configure Apache to listen on port `3001` instead of the default HTTP port. It must listen on
   all interfaces rather than a specific IP address or hostname.
3. Ensure Apache and the container are both running at the end.

## Solution

Connect to the app server and confirm that the container is running:

```bash
ssh banner@stapp03

docker ps --filter name=kkloud
```

Install Apache inside the container:

```bash
docker exec kkloud apt update
docker exec kkloud apt install -y apache2
```

Change both the listening port and the default virtual host port:

```bash
docker exec kkloud sed -i \
  's/^Listen 80$/Listen 3001/' \
  /etc/apache2/ports.conf

docker exec kkloud sed -i \
  's/<VirtualHost \*:80>/<VirtualHost *:3001>/' \
  /etc/apache2/sites-available/000-default.conf
```

Verify the edits and test the configuration before restarting Apache:

```bash
docker exec kkloud grep '^Listen' /etc/apache2/ports.conf
# Listen 3001

docker exec kkloud grep '<VirtualHost' \
  /etc/apache2/sites-enabled/000-default.conf
# <VirtualHost *:3001>

docker exec kkloud apache2ctl configtest
# Syntax OK
```

Restart Apache and confirm that it is running:

```bash
docker exec kkloud service apache2 restart
docker exec kkloud service apache2 status
# * apache2 is running
```

Test port `3001` through the container's IP, then confirm the container is still running:

```bash
CONTAINER_IP=$(docker inspect \
  --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' kkloud)

curl -I "http://${CONTAINER_IP}:3001"
# HTTP/1.1 200 OK

docker ps --filter name=kkloud
# kkloud remains Up
```

## Why

| Command | Why |
|:--|:--|
| `docker exec kkloud <command>` | Runs a command inside the already-running `kkloud` container without replacing its main process. The container name is required before the command. |
| `apt update` | Refreshes the container's package index so `apt` can locate the current `apache2` package. |
| `Listen 3001` | Makes Apache accept connections on port `3001`. With no IP before the port, Apache listens on all available interfaces. |
| `<VirtualHost *:3001>` | Moves the default site to the same port. The `*` matches every local interface, including loopback and the container IP. |
| `apache2ctl configtest` | Checks the configuration for syntax errors before the running service is restarted. |
| `service apache2 restart` | Applies the new port configuration and leaves Apache running. |
| `curl` through the container IP | Proves Apache is reachable on `3001` through a non-loopback interface, satisfying the all-interface requirement. |
| Final `docker ps` | Confirms that configuring the service did not stop the `kkloud` container. |

## Notes

- `docker exec -it sh` is incomplete. The correct interactive form is `docker exec -it kkloud sh`; the solution uses one-shot commands, so `-it` is unnecessary.
- Editing only `/etc/apache2/ports.conf` is not enough. The default site's `<VirtualHost>` declaration must use the same port.
- `Listen 127.0.0.1:3001` would bind only to loopback and fail the requirement. `Listen 3001` listens on all container interfaces.
- These changes live in the container's writable layer. Removing and recreating the container from its original image would discard them.
