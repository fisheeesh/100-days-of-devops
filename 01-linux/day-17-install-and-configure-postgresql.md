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
| `sudo su - postgres` | Local connections use `peer` authentication: PostgreSQL asks the OS who you are and only lets you in as the role with the same name. The `postgres` OS user maps to the `postgres` superuser, which is allowed to create roles and databases. |
| `CREATE USER ... WITH PASSWORD` | `CREATE USER` is `CREATE ROLE` with `LOGIN` switched on, which is why it answers `CREATE ROLE`. The password is a string, so it goes in single quotes. |
| `CREATE DATABASE` | Creates the database, owned by whoever ran it. Here that is `postgres`, not the new user. |
| `GRANT ALL PRIVILEGES ON DATABASE` | "All" on a database is exactly three privileges: `CONNECT`, `CREATE` (new schemas) and `TEMPORARY`. That is the `CTc` you see in `\l`. |
| `\du` / `\l` | Confirms the role exists and the grant landed, without having to log in as the new user. |

## Notes

- `\lu` is not a psql command; it errors with `invalid command \lu`. Roles are `\du`.
- On PostgreSQL 15 and newer, `ALL ON DATABASE` does **not** let the user create tables. The `public` schema stopped allowing `CREATE` for everyone in 15, so `CREATE TABLE` as `kodekloud_aim` fails with `permission denied for schema public`. If the app needs to create its own tables, make the user the owner: `ALTER DATABASE kodekloud_db4 OWNER TO kodekloud_aim;`, or `CREATE DATABASE kodekloud_db4 OWNER kodekloud_aim;` from the start. On 14 and older the grant alone is enough. `SELECT version();` tells you which you are on.
- Every SQL statement needs a `;`. Without one psql waits for more input, and the prompt changes from `postgres=#` to `postgres-#`.
- Unquoted names are folded to lowercase: `CREATE USER Kodekloud_Aim` creates `kodekloud_aim`. Only double quotes keep the case.
- None of this needs a restart, which is why the task can forbid one. Roles and grants apply immediately. Even editing `pg_hba.conf` only needs a reload.
- `sudo -u postgres psql` gets you in with one command, and `sudo -u postgres psql -c "..."` runs a single statement without an interactive session.
- `psql -U kodekloud_aim` from your own shell fails with `Peer authentication failed`, because your OS user is not `kodekloud_aim`. The auth method is refusing you, not the password.
