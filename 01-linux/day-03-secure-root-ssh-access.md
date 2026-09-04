# Day 3: Secure Root SSH Access

## Question

Disable direct SSH root login on **all** app servers in Stratos Datacenter.

## Solution

Repeat on `stapp01`, `stapp02`, `stapp03`:

```bash
ssh banner@stapp01
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
| `sudo -i` | Root shell, so you stop prefixing every command. `sshd_config` is `0600 root:root`, so a normal user can't even read it. |
| `vi /etc/ssh/sshd_config` | `PermitRootLogin no` blocks root from authenticating over SSH. Admins log in as themselves, then `sudo`, so actions are attributable. |
| `sshd -t` | Syntax check. Restarting on a broken config kills sshd and you lose the box. |
| `systemctl restart sshd` | Config is read at startup, so the edit does nothing until the service reloads. |
| `systemctl status sshd` | Confirms it actually came back up. |

## Notes

- `ssh_config` = **client** side, this box connecting out. `sshd_config` = **server** side, others connecting in. You want the `d` one.
- First matching directive wins in `sshd_config`. If `PermitRootLogin yes` already exists, edit that line, don't append a second one.
- `sshd_config.d/*.conf` drop-ins can override the main file if it has an `Include` line at the top. Check there if the setting won't take.
- Values: `yes` · `no` · `prohibit-password` (keys only) · `forced-commands-only`.
