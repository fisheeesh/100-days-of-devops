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
| `systemctl status mariadb` | Establishes the state before touching anything. `inactive (dead)` says it is not running; `disabled` separately says it would not come back after a reboot either. |
| `systemctl start mariadb` | The failure message is itself the next clue. "the control process exited with error code" means it died in a pre-start step, before mysqld ever ran. |
| `journalctl -xeu mariadb.service` | The real diagnosis. `-u` filters to that unit, `-e` jumps to the end, `-x` adds systemd's explanation text. `systemctl status` only prints the last handful of lines and the actual cause is usually above the cut. |
| `ls -lah /var/lib/mysql` | Verifies what the log claimed. Here the directory was missing altogether, which is why the pre-start step could not proceed. |
| `mkdir /var/lib/mysql` | That path is MariaDB's data directory. Without it there is nowhere for the database to live. |
| `chown -R mysql:mysql` | mysqld drops privileges to the `mysql` user right after start. A root-owned datadir means the daemon cannot write to its own data and dies again with a different error. This is the step that gets skipped. |
| `show databases;` | Proves the server is actually answering queries. `active (running)` only means systemd is happy with the process. |

## Notes

- The chain: `mariadb.service` runs `mariadb-prepare-db-dir` as an `ExecStartPre` before it ever launches mysqld. That is why the failure was reported as a "control process" rather than as mysqld crashing.
- With an empty but correctly owned datadir, that pre-start step initialises a **fresh** database. You get the system schemas and nothing else. On a real incident that is very different from repairing the data, so look for backups before recreating the directory.
- `systemctl status` truncates its log output. `journalctl -xeu <unit>` is the one worth memorising: `-u` unit, `-e` end, `-x` explain.
- The unit showed `disabled`. Starting it fixes right now; `sudo systemctl enable mariadb` is what makes it survive a reboot, and `enable --now` does both. The task only asks for it to be running, but on a real box you want both.
- Do not assume the path. Confirm where the datadir actually points with `grep -r datadir /etc/my.cnf /etc/my.cnf.d/` before creating anything.
- Same "will not start" symptom, other common causes: datadir owned by the wrong user after a restore, a full disk, a stale `/var/lib/mysql/mysql.sock`, or SELinux denying access to a datadir that was moved.
- `sudo mysql` with no password works because it connects over the unix socket as root. That is a local-only path, which is why the application failing to connect and your shell failing to connect are the same underlying outage but different code paths.
