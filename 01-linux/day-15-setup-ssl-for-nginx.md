# Day 15: Setup SSL for Nginx

## Question

The system admins team of `xFusionCorp Industries` needs to deploy a new application on
`App Server 3` in `Stratos Datacenter`. They have some pre-requites to get ready that server for
application deployment. Prepare the server as per requirements shared below:

1. Install and configure `nginx` on `App Server 3`.

2. On `App Server 3` there is a self signed SSL certificate and key present at location
   `/tmp/nautilus.crt` and `/tmp/nautilus.key`. Move them to some appropriate location and deploy
   the same in Nginx.

3. Create an `index.html` file with content `Welcome!` under Nginx document root.

4. For final testing try to access the `App Server 3` link (via hostname) from `jump host` using
   curl command. For example: `curl -Ik https://<app-server-name>/`.

## Solution

```bash
ssh banner@stapp03

sudo yum install -y nginx
sudo systemctl enable --now nginx
sudo systemctl status nginx

curl localhost
# port 80 works out of the box

curl https://localhost
# curl: (7) Failed to connect to localhost port 443: Connection refused
# nothing is listening on 443 yet
```

Move the cert and key somewhere permanent:

```bash
ls -lah /etc/nginx

sudo mkdir /etc/nginx/ssl
sudo mv /tmp/nautilus.crt /tmp/nautilus.key /etc/nginx/ssl/

# the key is the secret half, the cert is not
sudo chmod 600 /etc/nginx/ssl/nautilus.key

ls -lah /etc/nginx/ssl
# -rw-r--r-- 1 root root 2.2K nautilus.crt
# -rw------- 1 root root 3.2K nautilus.key
```

Add the TLS server block:

```bash
# the main config pulls in everything matching this glob
grep include /etc/nginx/nginx.conf
# include /etc/nginx/conf.d/*.conf;

sudo vi /etc/nginx/conf.d/nautilus.conf
```

```nginx
server {
    listen       443 ssl http2;
    listen       [::]:443 ssl http2;
    server_name  _;
    root         /usr/share/nginx/html;

    ssl_certificate     "/etc/nginx/ssl/nautilus.crt";
    ssl_certificate_key "/etc/nginx/ssl/nautilus.key";

    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

```bash
sudo nginx -t
# nginx: configuration file /etc/nginx/nginx.conf test is successful

sudo systemctl reload nginx

curl https://localhost
# curl: (60) SSL certificate problem: self-signed certificate

curl -k https://localhost
# serves the default nginx page, so TLS works
```

Put the required content in the document root:

```bash
echo 'Welcome!' | sudo tee /usr/share/nginx/html/index.html

curl -k https://localhost
# Welcome!
```

Final test from the jump host:

```bash
exit
curl -Ik https://stapp03/
```

## Why

| Command | Why |
|:--|:--|
| `chmod 600` on the key only | The key is the secret. The cert is sent to every client during the handshake, so `644` on it is correct. |
| `conf.d/nautilus.conf` | `nginx.conf` includes `conf.d/*.conf` inside its `http` block, so you never edit the main file. |
| `nginx -t` | Parses without applying. Reloading a broken config takes the site down. |
| `curl -k` | Self-signed means no CA vouches for it, so curl refuses with error 60. `-k` skips the check, `-I` sends HEAD. |

## Notes

- Error 60 is curl doing its job, not a broken config. That is why the task's own test is `curl -Ik`.
- `mv` keeps the SELinux label from `/tmp`. If SELinux is enforcing, nginx is denied the key even with the modes right: `restorecon -Rv /etc/nginx/ssl`.
- nginx 1.25.1 deprecated `listen ... http2` in favour of a separate `http2 on;` directive.
- Reload and restart do the same job here. Running both is one step too many.
