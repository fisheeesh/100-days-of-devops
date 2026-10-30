# Day 9: MariaDB Troubleshooting

## Question

There is a critical issue going on with the `Nautilus` application in `Stratos DC`. The
production support team identified that the application is unable to connect to the database.
After digging into the issue, the team found that mariadb service is down on the database
server.

Look into the issue and fix the same.

## Solution

```bash
ssh peter@stdb01

# confirm the symptom first
sudo mysql
# ERROR 2002 (HY000): Can't connect to local server through socket '/var/lib/mysql/mysql.sock'

sudo systemctl status mariadb
# Loaded: ... ; disabled
# Active: inactive (dead)

sudo systemctl start mariadb
# Job for mariadb.service failed because the control process exited with error code.
# See "systemctl status mariadb.service" and "journalctl -xeu mariadb.service" for details.
```

The start failed, so read the actual reason instead of guessing:

```bash
sudo journalctl -xeu mariadb.service
# ... Make sure the /var/lib/mysql is empty before running mariadb-prepare-db-dir
```

Check that claim rather than trusting it:

```bash
sudo ls -lah /var/lib/mysql
# ls: cannot access '/var/lib/mysql': No such file or directory
# the data directory is gone entirely
```

Fix and restart:

```bash
sudo mkdir /var/lib/mysql
sudo chown -R mysql:mysql /var/lib/mysql

sudo systemctl start mariadb
sudo systemctl status mariadb
# Active: active (running)

sudo mysql
show databases;
```

## Why

| Command | Why |
|:--|:--|
| `systemctl status mariadb` | `inactive (dead)` plus `disabled`: not running, and not coming back after a reboot. |
| `journalctl -xeu mariadb.service` | The real diagnosis. `-u` unit, `-e` end, `-x` explain. `status` truncates the log. |
| `mkdir /var/lib/mysql` | MariaDB's datadir was missing, so the pre-start step could not proceed. |
| `chown -R mysql:mysql` | The unit sets `User=mysql` and `ExecStartPre` inherits it, so the script cannot create or re-own a root-owned directory itself. |

## Notes

- "control process exited with error code" is systemd's generic wording for any failed start job, `ExecStart` included. The journal names the step, not that message.
- An empty but correctly owned datadir gets **initialised fresh**: system schemas only, no application data. Look for backups before recreating it.
- That log line normally means the datadir exists and is not empty. If the directory is really there, clearing it is the fix, not `mkdir`.
- If SELinux is enforcing, `restorecon -Rv /var/lib/mysql`. A hand-made directory under `/var/lib` gets `var_lib_t`, not `mysqld_db_t`.
