# Day 20: Configure Nginx + PHP-FPM Using Unix Sock

## Question

The `Nautilus` application development team is planning to launch a new PHP-based application,
which they want to deploy on `Nautilus` infra in `Stratos DC`. The development team had a meeting
with the production support team and they have shared some requirements regarding the
infrastructure. Below are the requirements they shared:

a. Install `nginx` on `app server 3`, configure it to use port `8096` and its document root should
be `/var/www/html`.

b. Install `php-fpm` version `8.2` on `app server 3`, it must use the unix socket
`/var/run/php-fpm/default.sock` (create the parent directories if don't exist).

c. Configure php-fpm and nginx to work together.

d. Once configured correctly, you can test the website using `curl http://stapp03:8096/index.php`
command from jump host.

> **NOTE:** We have copied two files, `index.php` and `info.php`, under `/var/www/html` as part of
> the `PHP-based application` setup. Please do not modify these files.

## Solution

**1. nginx on port 8096, document root `/var/www/html`:**

```bash
ssh banner@stapp03

sudo yum install -y nginx
sudo systemctl enable --now nginx
curl localhost                      # default page on 80

sudo vi /etc/nginx/nginx.conf
```

In the `server` block:

```nginx
server {
    listen       8096;
    listen       [::]:8096;
    server_name  _;
    root         /var/www/html;
    index        index.php index.html;
}
```

```bash
sudo nginx -t
sudo systemctl reload nginx

curl localhost                      # refused now, nothing on 80
curl http://stapp03:8096/index.php
# <?php
# echo "Welcome to xFusionCorp Industries!";
# ?>
# the raw source: nginx is serving the file, nobody is running it yet
```

**2. php-fpm 8.2 from Remi:**

```bash
sudo dnf install -y epel-release
sudo dnf install -y https://rpms.remirepo.net/enterprise/remi-release-9.rpm

sudo dnf module reset php -y
sudo dnf module enable php:remi-8.2 -y

sudo dnf install -y php-fpm php-common
php-fpm -v                          # PHP 8.2.x
```

**3. Point php-fpm at the required socket:**

```bash
sudo mkdir -p /var/run/php-fpm
sudo vi /etc/php-fpm.d/www.conf
# listen = /var/run/php-fpm/default.sock

sudo systemctl enable --now php-fpm
sudo systemctl restart php-fpm
ls -l /var/run/php-fpm/default.sock     # the socket only exists while php-fpm runs
```

**4. Hand `.php` requests to php-fpm:**

```bash
sudo vi /etc/nginx/nginx.conf
```

Inside the same `server` block:

```nginx
location ~ \.php$ {
    try_files $uri =404;
    include fastcgi.conf;
    fastcgi_pass unix:/var/run/php-fpm/default.sock;
}
```

```bash
sudo nginx -t
sudo systemctl reload nginx

curl http://stapp03:8096/index.php
# Welcome to xFusionCorp Industries!

exit
curl http://stapp03:8096/index.php
# Welcome to xFusionCorp Industries!
```

## Why

| Command | Why |
|:--|:--|
| `root /var/www/html;` | The stock nginx root is `/usr/share/nginx/html`, so without this the app's files are never found. |
| Raw PHP in the first `curl` | nginx alone serves `.php` as a plain file. Seeing the source is the expected halfway point. |
| `dnf module reset` then `enable php:remi-8.2` | The distro ships its own php stream. Without the reset the enable is refused. |
| `include fastcgi.conf;` | Sets `SCRIPT_FILENAME` for you. `fastcgi_params` does not, and php-fpm answers `File not found`. |

## Notes

- `mkdir -p /var/run/php-fpm` first: php-fpm will not create the parent directory for its socket, it fails to start.
- `502 Bad Gateway` means nginx could not reach php-fpm. Check in order: php-fpm running, socket path matches `fastcgi_pass`, nginx can open the socket. `/var/log/nginx/error.log` says which.
- On a permission error set `listen.owner`/`listen.group` to `nginx` in `www.conf`. RHEL's stock file usually ships `listen.acl_users = apache,nginx` already.
- `try_files $uri =404;` keeps non-existent `.php` paths from reaching php-fpm at all.
