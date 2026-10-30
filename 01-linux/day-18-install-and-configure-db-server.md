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
| `sudo mariadb` | On 10.4+ `root@localhost` uses `unix_socket`: OS root is database root, no password. |
| `'kodekloud_cap'@'localhost'` | An account is a name **plus** the host it may connect from. Same name, different host is a different account. |
| `ON kodekloud_db5.*` | Every table in that database, including future ones. Other databases stay off limits. |
| `SHOW GRANTS` | The only reliable check. A user with only `USAGE` can log in and do nothing. |

## Notes

- A `GRANT` to a user that does not exist fails with `ERROR 1133: Can't find any matching row in the user table`. A typo leaves the real user on `USAGE` only, so always finish with `SHOW GRANTS`.
- `FLUSH PRIVILEGES` is only for hand-edited `mysql.*` tables. `CREATE USER` and `GRANT` apply immediately.
- `'localhost'` passes the check, but the app connects over the network. A real deployment also needs `'kodekloud_cap'@'%'`.
- Quote name and host separately. `'kodekloud_cap@localhost'` creates a user literally called that.

| | PostgreSQL (Day 17) | MariaDB (Day 18) |
|:--|:--|:--|
| Admin shell | `sudo -u postgres psql` | `sudo mariadb` |
| A user is | a role name | `'name'@'host'` |
| List users | `\du` | `SELECT user, host FROM mysql.user;` |
| List databases | `\l` | `SHOW DATABASES;` |
| Switch database | `\c db` | `USE db;` |
| Quit | `\q` | `EXIT;` |
