# Day 1: Linux User Setup with Non-Interactive Shell

## Question

To accommodate the backup agent tool's specifications, the system admin team at
`xFusionCorp Industries` requires the creation of a user with a non-interactive shell.
Here's your task:

Create a user named `kareem` with a non-interactive shell on `App Server 3`.

> Note: You can find the infrastructure details by clicking on the **Details of all Users
> and Servers** button on the top-right section of the page.

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
| `ls -l /sbin/nologin` | `useradd` accepts any string as a shell. Check the path exists or you get a broken account. |
| `useradd -s` | Without `-s` you get `/bin/bash` from `/etc/default/useradd`. |
| `grep kareem /etc/passwd` | Field 7 is the login shell. That is what the validator reads. |

## Notes

- `/sbin/nologin` refuses and prints a message. `/bin/false` just exits 1. Either works.
- The point: an account that can own files and run a service, but never get a shell.
