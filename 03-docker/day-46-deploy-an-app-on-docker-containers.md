# Day 46: Deploy an App on Docker Containers

## Question

The Nautilus application development team wants to test a complete containerized application
stack. On `App Server 2` in `Stratos Datacenter`, create the exact file
`/opt/finance/docker-compose.yml` with two services:

### Web service

1. Name the container `php_blog`.
2. Use the `php` image with an Apache tag.
3. Map host port `6000` to container port `80`.
4. Bind mount host directory `/var/www/html` at `/var/www/html` in the container.

### Database service

1. Name the container `mysql_blog`.
2. Use a `mariadb` image, preferably `latest`.
3. Map host port `3306` to container port `3306`.
4. Bind mount host directory `/var/lib/mysql` at `/var/lib/mysql` in the container.
5. Create database `database_blog` and configure a non-root user with a complex password.

After starting the stack, the application must be reachable on port `6000`.

> When **FINISH** is clicked, all existing containers are destroyed and the stack is deployed
> again from this Compose file.

## Solution

Connect to the server and create the file with the exact required name:

```bash
ssh steve@stapp02

sudo vi /opt/finance/docker-compose.yml
```

Add the following content:

```yaml
services:
  web:
    image: php:8.3-apache
    container_name: php_blog
    ports:
      - "6000:80"
    volumes:
      - "/var/www/html:/var/www/html"

  db:
    image: mariadb:latest
    container_name: mysql_blog
    ports:
      - "3306:3306"
    environment:
      MYSQL_DATABASE: database_blog
      MYSQL_USER: user_blog
      MYSQL_PASSWORD: "Blog_DB@2026#Access"
      MYSQL_ROOT_PASSWORD: "Root_DB@2026#Admin"
    volumes:
      - "/var/lib/mysql:/var/lib/mysql"
```

Validate the Compose model, then start both services:

```bash
docker compose -f /opt/finance/docker-compose.yml config

docker compose -f /opt/finance/docker-compose.yml up -d
```

Verify both containers and the application:

```bash
docker compose -f /opt/finance/docker-compose.yml ps
# php_blog and mysql_blog are Up

curl http://localhost:6000/
# application response
```

Confirm that MariaDB initialized and accepts the configured application user:

```bash
docker exec mysql_blog sh -c \
  'mariadb-admin ping -h 127.0.0.1 -uroot -p"$MYSQL_ROOT_PASSWORD"'
# mysqld is alive

docker exec mysql_blog sh -c \
  'mariadb -u"$MYSQL_USER" -p"$MYSQL_PASSWORD" "$MYSQL_DATABASE" \
  -e "SELECT CURRENT_USER(), DATABASE();"'
# user_blog@%   database_blog
```

## Why

| Field | Why |
|:--|:--|
| `container_name` | Creates the exact `php_blog` and `mysql_blog` names the task checks for. |
| `MYSQL_ROOT_PASSWORD` | Mandatory for the MariaDB image. Without a root password option, initialisation fails outright. |
| `MYSQL_DATABASE` / `MYSQL_USER` / `MYSQL_PASSWORD` | Create the database and the non-root account on first start. |
| `/var/lib/mysql:/var/lib/mysql` | Keeps the database files on the host so they survive the container. |
| `docker compose config` | Catches YAML and interpolation errors before deploying. |

## Notes

- Compose puts both services on one network, so PHP reaches the database at the **service name** `db`, not `localhost`.
- The `MYSQL_*` variables only apply when `/var/lib/mysql` is empty. On an already initialised directory they do nothing.
- Quote the passwords. Unquoted, a `#` starts a YAML comment and silently truncates the value.
- The filename must be `docker-compose.yml`.
