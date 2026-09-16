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
| `mkdir /etc/nginx/ssl` + `mv` | `/tmp` is world-writable and gets cleaned out. Anything nginx needs at startup has to live somewhere stable. |
| `chmod 600` on the key only | The private key is the secret. The certificate is public by design and gets sent to every client during the handshake, so `644` on it is correct, not sloppy. |
| `conf.d/nautilus.conf` | `nginx.conf` has `include /etc/nginx/conf.d/*.conf;` inside its `http` block, so any `.conf` file dropped there is picked up. You never edit the main file. |
| `ssl_certificate` + `ssl_certificate_key` | Both are required. The cert proves who you are, the key does the handshake maths. One without the other fails the config test. |
| `nginx -t` | Parses the config without applying it. Reloading a broken config is how you take a live site down. |
| `systemctl reload nginx` | Reload re-reads the config and keeps existing connections. Restart drops them. `nginx -s reload` does the same job, so running reload *and* restart is one step too many. |
| `curl -k` | The cert is self-signed, so no CA vouches for it and curl refuses by default with error 60. `-k` skips that check. `-I` sends a HEAD request, so you get status and headers only. |

## Notes

- Error 60 is not a broken config, it is curl doing its job. Self-signed means no trusted CA signed the cert. That is exactly why the task's own test command is `curl -Ik`.
- `mv` keeps the SELinux label from `/tmp`. If SELinux is enforcing, nginx can be denied the key even with the modes right. `ls -Z` shows the label, `sudo restorecon -Rv /etc/nginx/ssl` fixes it.
- nginx 1.25.1 deprecated `listen ... http2`. On newer builds you write `http2 on;` as its own directive instead. The `listen` form above is right for the version on this box, which is why `nginx -t` passed clean.
- `echo 'Welcome!' | sudo tee /usr/share/nginx/html/index.html` replaces the file in one line. The vi route (`gg` then `dG` then insert) does the same thing.
- If `curl -k https://localhost` works on the box but the jump host hangs, it is the firewall on 443, not nginx. Same shape as Day 12 and Day 13.
- Check a cert and key are actually a pair before debugging anything else. Matching hashes mean they belong together:
  `sudo openssl x509 -noout -modulus -in /etc/nginx/ssl/nautilus.crt | openssl md5`
  `sudo openssl rsa  -noout -modulus -in /etc/nginx/ssl/nautilus.key | openssl md5`
- The document root in the server block (`/usr/share/nginx/html`) has to match where you write `index.html`. Change one and you change both.
