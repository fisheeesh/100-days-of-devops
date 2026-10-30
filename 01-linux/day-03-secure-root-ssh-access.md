# Day 3: Secure Root SSH Access

## Question

Following security audits, the `xFusionCorp Industries` security team has rolled out new
protocols, including the restriction of direct root SSH login.

Your task is to disable direct SSH root login on all app servers within the
`Stratos Datacenter`.

## Solution

Repeat on `stapp01`, `stapp02`, `stapp03`. Each has its own sudo user: `tony`, `steve`, `banner`.

```bash
ssh tony@stapp01
sudo -i

vi /etc/ssh/sshd_config
# /PermitRootLogin   search for it
# i  insert · Esc  stop editing · :wq  write and quit
# set:  PermitRootLogin no

# check the config parses before restarting; a typo here locks you out
sshd -t

systemctl restart sshd
systemctl status sshd
```

## Why

| Command | Why |
|:--|:--|
| `PermitRootLogin no` | Admins log in as themselves then `sudo`, so the audit log names a person. |
| `sshd -t` | Syntax check. Restarting on a broken config locks you out of the box. |
| `systemctl restart sshd` | Config is only read at startup. Existing sessions survive the restart. |

## Notes

- `ssh_config` is the client side, `sshd_config` the server side. You want the `d`.
- First match wins. Edit the existing `PermitRootLogin` line, do not append a second.
- `sshd_config.d/*.conf` drop-ins override the main file if it has an `Include`.
- Values: `yes` · `no` · `prohibit-password` · `forced-commands-only`.
