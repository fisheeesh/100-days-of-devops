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
| `systemctl start mariadb` | The failure message sends you to the journal, but it is vaguer than it looks: systemd prints "the control process exited with error code" for **any** start job that exits non-zero, whether that was an `ExecStartPre` or mysqld itself. It tells you to go read the log, not where it broke. |
| `journalctl -xeu mariadb.service` | The real diagnosis. `-u` filters to that unit, `-e` jumps to the end, `-x` adds systemd's explanation text. `systemctl status` only prints the last handful of lines and the actual cause is usually above the cut. |
| `ls -lah /var/lib/mysql` | Verifies what the log claimed. Here the directory was missing altogether, which is why the pre-start step could not proceed. |
| `mkdir /var/lib/mysql` | That path is MariaDB's data directory. Without it there is nowhere for the database to live. |
| `chown -R mysql:mysql` | On RHEL 9 the unit sets `User=mysql`, and `ExecStartPre` inherits it, so the pre-start script runs **unprivileged**. It cannot create a directory under `/var/lib` or take ownership of a root-owned one, which is exactly why root has to do both first. This is the step that gets skipped. |
| `show databases;` | Proves the server is actually answering queries. `active (running)` only means systemd is happy with the process. |

## Notes

- The chain: `mariadb.service` runs `mariadb-prepare-db-dir` as an `ExecStartPre` before it ever launches mysqld, and that is the step that failed here. The wording of `systemctl`'s error is not what tells you so, it says "control process" for a failed `ExecStart` too. The journal is what identified it.
- Red Hat deliberately runs that pre-start script as the `mysql` user, not root. mysqld itself never runs as root either on RHEL 9: the unit sets `User=mysql`, so systemd starts it unprivileged from the beginning. The old "starts as root, then drops privileges" model is `mysqld_safe` behaviour and does not apply here.
- That log message normally fires when the datadir **exists but is not empty** and has no `mysql/` subdirectory inside it. If you hit it with the directory actually present, the fix is different: clear it out or point `datadir` elsewhere. Do not blindly `mkdir`.
- With an empty but correctly owned datadir, that pre-start step initialises a **fresh** database. You get the system schemas and nothing else. On a real incident that is very different from repairing the data, so look for backups before recreating the directory.
- `systemctl status` truncates its log output. `journalctl -xeu <unit>` is the one worth memorising: `-u` unit, `-e` end, `-x` explain.
- The unit showed `disabled`. Starting it fixes right now; `sudo systemctl enable mariadb` is what makes it survive a reboot, and `enable --now` does both. The task only asks for it to be running, but on a real box you want both.
- Do not assume the path. Confirm where the datadir actually points with `grep -r datadir /etc/my.cnf /etc/my.cnf.d/` before creating anything.
- If SELinux is enforcing, add `sudo restorecon -Rv /var/lib/mysql`. A directory created by hand under `/var/lib` inherits `var_lib_t`, while the policy expects `mysqld_db_t`. A recreated datadir has exactly the same mislabelling problem as a moved one.
- Same "will not start" symptom, other common causes: datadir owned by the wrong user after a restore, a full disk, or a stale `/var/lib/mysql/mysql.sock`.
- `sudo mysql` with no password works because it connects over the unix socket as root. That is a local-only path, which is why the application failing to connect and your shell failing to connect are the same underlying outage but different code paths.
