# Day 4: Script Execution Permissions

## Question

In a bid to automate backup processes, the `xFusionCorp Industries` sysadmin team has
developed a new bash script named `xfusioncorp.sh`. While the script has been distributed to
all necessary servers, it lacks executable permissions on `App Server 1` within the Stratos
Datacenter.

> Your task is to grant executable permissions to the `/tmp/xfusioncorp.sh` script on
> `App Server 3`. Additionally, ensure that all users have the capability to execute it.

## Solution

```bash
ssh tony@stapp01

# see what the script has right now, before changing anything
ls -lah /tmp
# ----------  1 root root   40 Sep  5 16:45 xfusioncorp.sh
# mode 000, nobody can do anything with it

sudo chmod a+rx /tmp/xfusioncorp.sh

# verify
ls -lah /tmp
# -r-xr-xr-x  1 root root   40 Sep  5 16:45 xfusioncorp.sh
```

## Why

| Command | Why |
|:--|:--|
| `ls -lah /tmp` | Mode was `----------`, so the fix is adding bits, not editing them. |
| `chmod a+rx` | `a` is user, group and other at once. `+` adds without clearing what is set. |
| `r` as well as `x` | The shell has to read a script to interpret it. `x` alone gives Permission denied. |

## Notes

- A compiled binary runs with `x` alone. Scripts need `r` too.
- The task says App Server 1 in one sentence and App Server 3 in the next. Go where the file is.
