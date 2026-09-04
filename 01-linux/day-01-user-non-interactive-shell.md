# Day 1: Linux User Setup with Non-Interactive Shell

## Question

Create a user named `kareem` with a non-interactive shell on App Server 3.

## Solution

```bash
ssh banner@stapp03

# confirm the nologin shell exists on this box before using it
ls -l /sbin/nologin /usr/sbin/nologin

sudo useradd -s /sbin/nologin kareem

# verify: last field of the passwd entry must be the nologin shell
grep kareem /etc/passwd
```

## Why

| Command | Why |
|:--|:--|
| `ls -l /sbin/nologin` | `useradd` won't reject a bad shell path, so you'd get a broken account. On RHEL `/sbin` is a symlink to `/usr/sbin`, so both paths work. |
| `useradd -s <shell>` | `-s` sets the login shell. Without it you get the default from `/etc/default/useradd` (usually `/bin/bash`). |
| `grep kareem /etc/passwd` | The check is on field 7 of the passwd entry, which is exactly what the validator reads. |

## Notes

- `/sbin/nologin` refuses the login and prints a message. `/bin/false` just exits 1, no message. Either is non-interactive; `nologin` is the conventional choice.
- Non-interactive = the account can own files and run services, but nobody can get a shell with it. Standard for service/agent accounts.
