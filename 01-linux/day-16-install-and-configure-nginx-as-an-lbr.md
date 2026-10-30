# Day 16: Install and Configure Nginx as an LBR

## Question

Day by day traffic is increasing on one of the websites managed by the `Nautilus` production
support team. Therefore, the team has observed a degradation in website performance. Following
discussions about this issue, the team has decided to deploy this application on a high
availability stack i.e on `Nautilus` infra in `Stratos DC`. They started the migration last month
and it is almost done, as only the LBR server configuration is pending. Configure LBR server as per
the information given below:

a. Install `nginx` on the `LBR` (load balancer) server if it is not already installed.

b. Configure load-balancing with the `http` context making use of all `App Servers`. Ensure that
you update only the main Nginx configuration file located at `/etc/nginx/nginx.conf`.

c. Make sure you do not update the apache port that is already defined in the apache configuration
on all app servers, also make sure apache service is up and running on all the app servers.

d. Once done, you can access the website by running `curl http://stlb01:80` in the terminal.

## Solution

**1. Find the Apache port on each app server.** Read it, do not assume it, and do not change it.

```bash
ssh tony@stapp01

sudo systemctl status httpd
# active (running)

sudo ss -tulnp | grep httpd
# tcp  LISTEN 0 511  *:8087  *:*  users:(("httpd",pid=32840,fd=4),("httpd",pid=32839,fd=4),...)

curl localhost:8087
# Welcome to xFusionCorp Industries!

exit
```

Repeat on `steve@stapp02` and `banner@stapp03`. Then from the jump host, confirm each backend is
reachable from outside its own box:

```bash
curl http://stapp01:8087
curl http://stapp02:8087
curl http://stapp03:8087
# Welcome to xFusionCorp Industries!
```

**2. Install nginx on the LBR.**

```bash
ssh loki@stlb01

sudo yum install -y nginx
sudo systemctl enable --now nginx
sudo systemctl status nginx
```

**3. Add the upstream pool and point the default server at it.**

```bash
sudo vi /etc/nginx/nginx.conf
```

Inside the existing `http { }` block:

```nginx
http {
    # ... existing settings ...

    upstream stapp {
        server stapp01:8087;
        server stapp02:8087;
        server stapp03:8087;
    }

    server {
        listen       80;
        listen       [::]:80;
        # ... existing lines ...

        location / {
            proxy_pass http://stapp;
        }
    }
}
```

```bash
sudo nginx -t
sudo systemctl reload nginx
exit
```

**4. Test from the jump host.**

```bash
curl http://stlb01:80
# Welcome to xFusionCorp Industries!
```

## Why

| Command | Why |
|:--|:--|
| `ss -tulnp \| grep httpd` | The Apache port changes between runs and part c forbids changing it, so read it off the box. |
| `upstream stapp { ... }` | Names a pool of backends. It lives inside `http { }`, never inside a `server` block. |
| `proxy_pass http://stapp;` | `stapp` is the upstream name, not a hostname. |
| `nginx.conf`, not `conf.d/` | Part b says the main file only. The same config in `conf.d/` works and still fails the check. |

## Notes

- `curl https://stlb01:80` fails with an SSL error. Port 80 is plain HTTP; the scheme and port have to agree.
- Balancing is round-robin by default. Watch `/var/log/httpd/access_log` on the app servers to see it spread.
- A dead backend is marked failed and skipped (`max_fails=1`, `fail_timeout=10s`). That is the high availability the task is about.
- `502 Bad Gateway` means nginx is fine and cannot reach a backend: httpd down, wrong port, app-server firewall, then SELinux (`setsebool -P httpd_can_network_connect 1`).
