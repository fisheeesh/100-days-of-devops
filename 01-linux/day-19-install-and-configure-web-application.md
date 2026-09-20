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
| `scp -r` | `-r` recurses into the directory. Without it scp refuses and says the source is a directory. |
| `Listen 8083` in `httpd.conf` | The listening port lives in the config and is only read at startup, so the edit does nothing until the service restarts. |
| `apachectl configtest` | Parses the config without applying it. Same habit as `nginx -t` on Day 15 and 16: a broken config plus a restart takes the site down. |
| `grep DocumentRoot` | Read the document root instead of assuming it. Every URL path is resolved relative to it, so `/var/www/html/beta` is what serves `/beta/`. |
| `sudo mv` | `/var/www/html` is owned by root. Without sudo the move fails with `Permission denied`. |
| Trailing slash in `/beta/` | It is a directory, so Apache looks for `index.html` inside it via `DirectoryIndex`. |

## Notes

- `curl localhost` failing after the restart is the proof the port change took effect: nothing listens on 80 any more. That is expected, not a mistake.
- **403 Forbidden instead of the page, with SELinux enforcing:** `mv` keeps the SELinux label the files had in the home directory, so httpd is refused. `ls -Z /var/www/html` shows it and `sudo restorecon -Rv /var/www/html` relabels. `cp` would have picked up the destination's label instead. Same trap as the cert in Day 15.
- **httpd refuses to start after the port change, with SELinux enforcing:** a non-standard port needs registering first, `sudo semanage port -a -t http_port_t -p tcp 8083`.
- The files have to be readable by the `apache` user. If the backup arrived with odd modes, `sudo chmod -R a+rX /var/www/html/beta` fixes it (`X` sets execute on directories only).
- Part d tests from the app server itself, so the firewall is not involved. Reaching it from the jump host would also need port 8083 open (Day 13).
- `apachectl configtest` and `httpd -t` are the same check.
