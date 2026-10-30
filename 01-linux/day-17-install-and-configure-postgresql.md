# Day 17: Install and Configure PostgreSQL

## Question

The `Nautilus` application development team has shared that they are planning to deploy one newly
developed application on `Nautilus` infra in `Stratos DC`. The application uses PostgreSQL
database, so as a pre-requisite we need to set up PostgreSQL database server as per requirements
shared below:

PostgreSQL database server is already installed on the `Nautilus` database server.

a. Create a database user `kodekloud_aim` and set its password to `dCV3szSGNA`.

b. Create a database `kodekloud_db4` and grant full permissions to user `kodekloud_aim` on this
database.

> **Note:** Please do not try to restart PostgreSQL server service.

## Solution

```bash
ssh peter@stdb01

# become the postgres OS user, then open the psql shell
sudo su - postgres
psql
```

```sql
CREATE USER kodekloud_aim WITH PASSWORD 'dCV3szSGNA';
-- CREATE ROLE

CREATE DATABASE kodekloud_db4;
-- CREATE DATABASE

GRANT ALL PRIVILEGES ON DATABASE kodekloud_db4 TO kodekloud_aim;
-- GRANT
```

Verify and get around:

```text
\du                 list roles
\l                  list databases; the Access privileges column shows kodekloud_aim=CTc/postgres
\c kodekloud_db4    connect to a database
\dt                 list tables in the current database
\q                  quit
```

## Why

| Command | Why |
|:--|:--|
| `sudo su - postgres` | Local logins use `peer` auth: PostgreSQL only lets you in as the role matching your OS user. |
| `CREATE USER ... WITH PASSWORD` | `CREATE USER` is `CREATE ROLE` plus `LOGIN`, which is why it answers `CREATE ROLE`. |
| `GRANT ALL PRIVILEGES ON DATABASE` | "All" on a database is three privileges: `CONNECT`, `CREATE` (schemas) and `TEMPORARY`. That is the `CTc` in `\l`. |
| `\du` / `\l` | Confirms the role exists and the grant landed, without logging in as the new user. |

## Notes

- `\lu` is not a command; it errors with `invalid command \lu`. Roles are `\du`.
- On PostgreSQL 15+, `ALL ON DATABASE` does **not** allow `CREATE TABLE`: the `public` schema stopped granting that. The app needs to own the database (`ALTER DATABASE ... OWNER TO ...`).
- Every statement needs a `;`. Without one the prompt changes from `postgres=#` to `postgres-#` and waits.
- Unquoted names fold to lowercase. Only double quotes keep the case.
