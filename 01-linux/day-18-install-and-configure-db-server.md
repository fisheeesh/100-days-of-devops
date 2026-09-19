# Day 18: Install and Configure DB Server

## Question

We need to setup a database server on `Nautilus DB Server` in `Stratos Datacenter`. Please perform
the below given steps on DB Server:

a. Install/Configure MariaDB server.

b. Create a database named `kodekloud_db5`.

c. Create a user called `kodekloud_cap` and set its password to `YchZHRcLkL`.

d. Grant full permissions to user `kodekloud_cap` on database `kodekloud_db5`.

## Solution

```bash
ssh peter@stdb01

sudo yum install -y mariadb-server
sudo systemctl enable --now mariadb
sudo systemctl status mariadb
# active (running)

# no password needed: OS root is MariaDB root
sudo mariadb
```

```sql
CREATE USER 'kodekloud_cap'@'localhost' IDENTIFIED BY 'YchZHRcLkL';

CREATE DATABASE kodekloud_db5;

-- every table in kodekloud_db5, now and later
GRANT ALL PRIVILEGES ON kodekloud_db5.* TO 'kodekloud_cap'@'localhost';

-- not needed after GRANT, harmless
FLUSH PRIVILEGES;

-- check the grant actually landed
SHOW GRANTS FOR 'kodekloud_cap'@'localhost';
-- GRANT USAGE ON *.* TO `kodekloud_cap`@`localhost` IDENTIFIED BY PASSWORD '...'
-- GRANT ALL PRIVILEGES ON `kodekloud_db5`.* TO `kodekloud_cap`@`localhost`

EXIT;
```

Log in as the new user to prove it:

```bash
mariadb -u kodekloud_cap -p kodekloud_db5
# password: YchZHRcLkL
SHOW DATABASES;
# kodekloud_db5 is listed
```

## Why

| Command | Why |
|:--|:--|
| `sudo mariadb` | On MariaDB 10.4 and newer, `root@localhost` logs in with `unix_socket`: if you are root on the OS, you are root in the database. No password. Same idea as PostgreSQL's `peer` login on Day 17. |
| `'kodekloud_cap'@'localhost'` | A MariaDB account is a name **plus** the host it may connect from. `localhost` means this box only. The same name at a different host is a separate account with its own password and grants. |
| `ON kodekloud_db5.*` | `.*` is every table in that database, including ones created later. Unlike the Day 17 PostgreSQL grant, this really does cover creating, reading and dropping tables. Other databases stay off limits. |
| `SHOW GRANTS` | The only reliable check. A user holding nothing but `USAGE` can log in and do nothing else, which looks fine until the app tries to read a table. |

## Notes

- A `GRANT` to a user that does not exist fails. A typo like `'kodekloud_cp'` (missing the `a`) gives `ERROR 1133: Can't find any matching row in the user table`, because `NO_AUTO_CREATE_USER` is in 10.5's default `sql_mode`. The real user is left with `USAGE` only and gets `ERROR 1044: Access denied` on `kodekloud_db5`. When you paste several statements at once, that error scrolls past easily, so always finish with `SHOW GRANTS`.
- `FLUSH PRIVILEGES` is only needed when you edit the `mysql.*` tables by hand. `CREATE USER` and `GRANT` take effect immediately.
- `'localhost'` is enough for this check, but the application will connect from the app servers over the network, which does not match `localhost`. A real deployment also needs `'kodekloud_cap'@'%'` (or the app servers' host) with its own `CREATE USER` and `GRANT`.
- Creating only the `'%'` account has its own trap: if an anonymous `''@'localhost'` account exists, local logins as `kodekloud_cap` match that first and get `Access denied`. `SELECT user, host FROM mysql.user;` shows what exists.
- Quote the name and the host separately. `'kodekloud_cap'@'localhost'` is right. `'kodekloud_cap@localhost'` creates a user literally named `kodekloud_cap@localhost`.
- `-p` with nothing after it makes the client prompt for the password, so it stays out of your shell history.
- `mariadb` and `mysql` are the same client on 10.5.

| | PostgreSQL (Day 17) | MariaDB (Day 18) |
|:--|:--|:--|
| Admin shell | `sudo -u postgres psql` | `sudo mariadb` |
| A user is | a role name | `'name'@'host'` |
| List users | `\du` | `SELECT user, host FROM mysql.user;` |
| List databases | `\l` | `SHOW DATABASES;` |
| Switch database | `\c db` | `USE db;` |
| Quit | `\q` | `EXIT;` |
