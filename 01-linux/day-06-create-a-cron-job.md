# Day 6: Create a Cron Job

## Question

The `Nautilus` system admins team has prepared scripts to automate several day-to-day tasks.
They want them to be deployed on all app servers in `Stratos DC` on a set schedule. Before
that they need to test similar functionality with a sample cron job. Therefore, perform the
steps below:

a. Install `cronie` package on all `Nautilus` app servers and start `crond` service.

b. Add a cron `*/5 * * * * echo hello > /tmp/cron_text` for `root` user.

## Solution

Repeat on `stapp01`, `stapp02`, `stapp03`:

```bash
ssh tony@stapp01

sudo yum install -y cronie

sudo systemctl status crond
# ○ crond.service - Command Scheduler
#      Loaded: loaded (/usr/lib/systemd/system/crond.service; enabled; preset: enabled)
#      Active: inactive (dead)
# enabled but not running, so it needs starting

sudo systemctl start crond
sudo systemctl status crond
# Active: active (running)

# or both in one step
sudo systemctl enable --now crond
```

Then the job itself:

```bash
# what is already there. no sudo is the current user, sudo is root
crontab -l
sudo crontab -l
# expect nothing on a fresh box

sudo crontab -e
# add:  */5 * * * * echo hello > /tmp/cron_text

sudo crontab -l          # confirm it registered to root
sudo ls -lah /var/spool/cron

# wait for the next multiple of 5 minutes, then
cat /tmp/cron_text
# hello
```

## Why

| Command | Why |
|:--|:--|
| `yum install -y cronie` | The package is `cronie`, the service is `crond`. `systemctl start cron` fails. |
| `systemctl status crond` | `enabled` but `inactive`: enabled means at boot, not now. The task needs both. |
| `sudo crontab -e` | `sudo` edits **root's** crontab. Without it you edit your own. |
| `cat /tmp/cron_text` | Proves it ran. `crontab -l` only proves it was scheduled. |

## Notes

- `*/5 * * * *` fires at `:00`, `:05`, `:10`, not 5 minutes from when you saved.
- Fields: minute, hour, day of month, month, day of week.
- Cron runs with a minimal `PATH` (`/usr/bin:/bin`). Use absolute paths in jobs.
- All app servers, so three boxes.

**Q: Why `sudo yum` here when I normally use `sudo apt install`?**

The package manager depends on the distro family, not on what you are installing.

| Family | High level | Low level |
|:--|:--|:--|
| Debian, Ubuntu | `apt` | `dpkg` (`.deb`) |
| RHEL, CentOS, Fedora | `dnf`, `yum` | `rpm` (`.rpm`) |

These are CentOS Stream 9, so `apt` is not installed. The package name changes too: `cronie` on RHEL, `cron` on Debian.
