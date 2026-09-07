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
| `yum install -y cronie` | RHEL ships the cron daemon in a package called `cronie`, but the service it installs is called `crond`. The names differ, which is why `systemctl start cron` fails. |
| `systemctl status crond` | Showed `enabled` but `inactive (dead)`. Enabled means "starts at boot", not "running right now". Two separate things, and the task needs both. |
| `systemctl enable --now crond` | Does enable and start in one command. Same result as running the two separately. |
| `sudo crontab -e` | `sudo` edits **root's** crontab. Without it you edit the crontab of whoever you are logged in as. The task specifically asks for root. |
| `crontab -e` rather than editing the file | `crontab` validates the syntax before installing it, and writes the file with the ownership and mode cron expects. A bad line typed straight into the spool file is silently ignored. |
| `sudo crontab -l` | Reads it back. Confirms the entry landed on root and not on `tony`. |
| `cat /tmp/cron_text` | Proves the job actually ran. `crontab -l` only proves it was scheduled. |

## Notes

- `*/5 * * * *` means every 5 minutes on the clock (`:00`, `:05`, `:10` and so on), not 5 minutes from when you saved it. Wait for the next multiple of 5 before checking.
- Field order is minute, hour, day of month, month, day of week.
- `>` truncates, so `/tmp/cron_text` always holds exactly one line. `>>` would append forever.
- Crontabs live in `/var/spool/cron/<user>` on RHEL. Fine to look at with `sudo ls -lah /var/spool/cron`, but edit through `crontab -e`.
- To remove one: `sudo crontab -r -u tony`. Deleting `/var/spool/cron/tony` by hand works too, but `-r` is the supported path.
- Cron runs with a stripped environment and a minimal `PATH` (`/usr/bin:/bin`). Scripts that run fine in your shell often fail under cron for this reason. Use absolute paths inside cron jobs.
- The task says all app servers, so this is three boxes, not one.

**Q: Why `sudo yum` here? Normally when installing things like Java or Tomcat I use `sudo apt install`. And `apt` is not used here at all.**

Because the package manager depends on the distro family, not on what you are installing.

| Family | High level | Low level (single package file) |
|:--|:--|:--|
| Debian, Ubuntu | `apt` | `dpkg` (`.deb`) |
| Red Hat, RHEL, CentOS, Fedora | `dnf`, `yum` | `rpm` (`.rpm`) |

These servers are CentOS Stream 9, so `apt` is not installed at all and `yum` is what is there. Same package, different name and different command depending on where you are: `cronie` on RHEL, `cron` on Debian.

This is exactly why day 5 started with `hostnamectl`. Check the distro first, then you know which tool to reach for.
