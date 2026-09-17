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
| `ss -tulnp \| grep httpd` | The Apache port changes between lab runs and part c forbids touching it, so read it off the box. That number is what goes into the upstream. |
| `curl http://stapp0N:8087` from the jump host | Proves every backend answers from outside its own box before nginx is involved. If one fails here, the LBR will fail too. |
| `upstream stapp { ... }` | Names a pool of backends. It lives inside `http { }`, alongside the `server` blocks, never inside one. |
| `proxy_pass http://stapp;` | `stapp` here is the upstream name, not a hostname. Every request under `/` gets forwarded to that pool. |
| `nginx.conf`, not `conf.d/` | Part b says the main file only. Day 15 used `conf.d/` on purpose; here the same config in `conf.d/` would work identically and still fail the check. |
| `nginx -t` | Also resolves every upstream hostname. A typo like `stapp4` fails here with `host not found in upstream` instead of breaking the live site. |
| `systemctl reload nginx` | Re-reads the config without dropping connections. `nginx -s reload` is the same thing. |

## Notes

- `curl https://stlb01:80` does not work. nginx on port 80 speaks plain HTTP, so curl expecting TLS fails with an SSL error (exit code 35). The scheme and the port have to agree.
- Default balancing is round-robin: each request goes to the next server in turn. Every backend serves the same page here, so curl output looks identical. Watch `/var/log/httpd/access_log` on the app servers to see requests spread across all three.
- If a backend dies, nginx marks it failed and skips it for a while (defaults `max_fails=1`, `fail_timeout=10s`), so the site stays up on the other two. That failover is the "high availability" the task is about.
- `502 Bad Gateway` means nginx is fine and it cannot reach a backend. Check in this order: httpd down on an app server, wrong port in the upstream, a firewall on the app servers not letting the LBR in (Day 13), SELinux.
- SELinux enforcing on the LBR blocks nginx from opening connections to backends. `/var/log/nginx/error.log` shows `Permission denied` while connecting to upstream. Fix with `sudo setsebool -P httpd_can_network_connect 1`.
- By default nginx sends the upstream name (`stapp`) as the `Host` header, not `stlb01`. Apache here has no virtual hosts so it does not matter. An app that routes on `Host` needs `proxy_set_header Host $host;`.
