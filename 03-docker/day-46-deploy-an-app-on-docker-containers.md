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

| Compose field or command | Why |
|:--|:--|
| `php:8.3-apache` | Provides PHP together with Apache listening on container port `80`. |
| `container_name` | Creates the exact `php_blog` and `mysql_blog` names required by the task. |
| `"6000:80"` | Publishes the PHP/Apache site on host port `6000`. |
| `/var/www/html:/var/www/html` | Makes the existing host application files available at the PHP Apache image's document root. |
| `mariadb:latest` | Uses the requested MariaDB image. |
| `"3306:3306"` | Publishes MariaDB's container port on the same host port. |
| `MYSQL_DATABASE` | Creates `database_blog` during first-time database initialization. |
| `MYSQL_USER` and `MYSQL_PASSWORD` | Create the required non-root application account and grant it access to `database_blog`. |
| `MYSQL_ROOT_PASSWORD` | Supplies the mandatory root credential needed by the MariaDB image during initial startup. Without a root-password option, initialization fails. |
| `/var/lib/mysql:/var/lib/mysql` | Stores database files in the required host directory so they persist beyond container replacement. |
| `docker compose ... config` | Catches YAML, interpolation, and Compose-model errors before deployment. |

## Notes

- The required filename is `docker-compose.yml`. The spelling `docker-comopse.yml` is incorrect and will not be found when the platform redeploys the stack.
- Compose automatically places both services on the same project network. The web container can address the database by its service name, `db`, on port `3306`; it should not use `localhost` for the database connection.
- The MariaDB initialization variables apply only when `/var/lib/mysql` is empty. If that host directory already contains an initialized database, changing these variables does not recreate its users or databases.
- The passwords are literal strings because they are quoted. Without quotes, `#` begins a YAML comment.
- Plaintext Compose credentials are acceptable for this lab only. A production deployment should use Docker secrets or another secret manager.
- The bind mounts point at existing host data. Do not delete or rewrite `/var/www/html` or `/var/lib/mysql` while preparing the Compose file.
