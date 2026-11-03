# Day 46: Deploy an App on Docker Containers

## Question

The Nautilus Application development team recently finished development of one of the apps that
they want to deploy on a containerized platform. The Nautilus Application development and DevOps
teams met to discuss some of the basic pre-requisites and requirements to complete the deployment.
The team wants to test the deployment on one of the app servers before going live and set up a
complete containerized stack using a docker compose file. Below are the details of the task:

1. On `App Server 2` in `Stratos Datacenter` create a docker compose file
   `/opt/finance/docker-compose.yml` (should be named exactly).
2. The compose should deploy two services (web and DB), and each service should deploy a container
   as per details below:

`For web service:`

a. Container name must be `php_blog`.

b. Use image `php` with any `apache` tag.

c. Map `php_blog` container's port `80` with host port `6000`.

d. Map `php_blog` container's `/var/www/html` volume with host volume `/var/www/html`.

`For DB service:`

a. Container name must be `mysql_blog`.

b. Use image `mariadb` with any tag (preferably `latest`).

c. Map `mysql_blog` container's port `3306` with host port `3306`.

d. Map `mysql_blog` container's `/var/lib/mysql` volume with host volume `/var/lib/mysql`.

e. Set MYSQL_DATABASE=`database_blog` and use any custom user ( except root ) with some complex
password for DB connections.

3. After running docker-compose up you can access the app with curl command
   `curl <server-ip or hostname>:6000/`

> `Note:` Once you click on `FINISH` button, all currently running/stopped containers will be
> destroyed and stack will be deployed again using your compose file.

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
