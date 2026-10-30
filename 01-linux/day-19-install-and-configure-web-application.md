# Day 19: Install and Configure Web Application

## Question

xFusionCorp Industries is planning to host two static websites on their infra in
`Stratos Datacenter`. The development of these websites is still in-progress, but we want to get
the servers ready. Please perform the following steps to accomplish the task:

a. Install `httpd` package and dependencies on `app server 2`.

b. Apache should serve on port `8083`.

c. There are two website's backups `/home/thor/beta` and `/home/thor/apps` on `jump_host`. Set
them up on Apache in a way that `beta` should work on the link `http://localhost:8083/beta/` and
`apps` should work on link `http://localhost:8083/apps/` on the mentioned app server.

d. Once configured you should be able to access the website using `curl` command on the respective
app server, i.e `curl http://localhost:8083/beta/` and `curl http://localhost:8083/apps/`

## Solution

**1. Copy both site folders over,** from the jump host:

```bash
ls ~
# apps  beta

scp -r ~/beta ~/apps steve@stapp02:~/
```

**2. Install Apache and get it running:**

```bash
ssh steve@stapp02

sudo yum install -y httpd
sudo systemctl enable --now httpd

curl localhost
# default Apache page, still on port 80
```

**3. Move it to port 8083:**

```bash
sudo vi /etc/httpd/conf/httpd.conf
# search /Listen 80  and change the line to:
#   Listen 8083

sudo apachectl configtest
# Syntax OK

sudo systemctl restart httpd

curl localhost
# curl: (7) Failed to connect ... port 80: Connection refused   <- the change took

curl localhost:8083
# Apache test page
```

**4. Put the sites under the document root:**

```bash
grep DocumentRoot /etc/httpd/conf/httpd.conf
# DocumentRoot "/var/www/html"

sudo mv ~/beta ~/apps /var/www/html/
ls -l /var/www/html

curl http://localhost:8083/beta/
curl http://localhost:8083/apps/
```

## Why

| Command | Why |
|:--|:--|
| `scp -r` | Without `-r` scp refuses a directory. |
| `Listen 8083` in `httpd.conf` | Read only at startup, so the edit does nothing until the service restarts. |
| `grep DocumentRoot` | Every URL path resolves relative to it, so `/var/www/html/beta` is what serves `/beta/`. |
| `sudo mv` | `/var/www/html` is root-owned. Without sudo the move fails with `Permission denied`. |

## Notes

- `curl localhost` failing after the restart proves the port change took: nothing listens on 80 any more.
- **403 Forbidden with SELinux enforcing:** `mv` keeps the home-directory label, so httpd is refused. `restorecon -Rv /var/www/html` fixes it.
- **httpd will not start with SELinux enforcing:** a non-standard port needs `semanage port -a -t http_port_t -p tcp 8083`.
- `apachectl configtest` before the restart, same habit as `nginx -t`.
