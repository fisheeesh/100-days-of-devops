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
| `ls -lah /tmp` | Read the current mode before touching it. Here it was `----------`, so the fix is adding bits, not editing them. |
| `chmod a+rx` | `a` is all three classes at once (user, group, other), which is what "all users" in the task means. `+` adds bits without clearing the ones already set. |
| `r` as well as `x` | A shell script is not run directly by the kernel. The shell has to open and read the file to interpret it, so execute alone is not enough. |
| `sudo` | The file is owned by `root`, and only the owner or root can change a file's mode. |
| `ls -lah /tmp` again | Confirms `-r-xr-xr-x`. This is what the validator reads. |

## Notes

- `a+rx` vs `chmod 555`: both land on the same mode here, but `+` is additive and `555` sets absolutely, wiping anything already there. Additive is the safer habit.
- A compiled binary only needs `x`. Scripts need `r+x`. Easy one to get bitten by.
- Nobody gets write. The task only asked for execute, and `/tmp` is world-writable enough as it is.
- The task text says App Server 1 in the first sentence and App Server 3 in the second. Go with wherever the file actually is; `ls /tmp` tells you.
