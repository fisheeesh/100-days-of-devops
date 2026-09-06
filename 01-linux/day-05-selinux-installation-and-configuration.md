# Day 5: SELinux Installation and Configuration

## Question

Following a security audit, the xFusionCorp Industries security team has opted to enhance
application and server security with SELinux. To initiate testing, the following requirements
have been established for `App Server 3` in the `Stratos Datacenter`:

1. Install the required `SELinux` packages.
2. Permanently disable SELinux for the time being; it will be re-enabled after necessary
   configuration changes.
3. No need to reboot the server, as a scheduled maintenance reboot is already planned for
   tonight.
4. Disregard the current status of SELinux via the command line; the final status after the
   reboot should be `disabled`.

## Solution

```bash
ssh banner@stapp03

# find the distro first, package manager and package names depend on it
hostnamectl                 # look at "Operating System"
cat /etc/os-release         # or look at PRETTY_NAME
# mine: CentOS Stream 9

sudo yum install selinux-policy selinux-policy-targeted policycoreutils

# config lives here
ls /etc/selinux/

sudo vi /etc/selinux/config
# set:  SELINUX=disabled

# verify
grep ^SELINUX= /etc/selinux/config
# expected: SELINUX=disabled
```

## Why

| Command | Why |
|:--|:--|
| `hostnamectl` / `cat /etc/os-release` | Package names and the package manager differ per distro. Confirm the family before reaching for `yum`. |
| `selinux-policy` | The policy framework itself. Nothing else works without it. |
| `selinux-policy-targeted` | The actual policy the system loads at boot. `selinux-policy` alone gives you a framework with no rules. |
| `policycoreutils` | The management tooling: `setenforce`, `restorecon`, `load_policy`. |
| `ls /etc/selinux/` | Confirms the package landed and there is a `config` file to edit. |
| `vi /etc/selinux/config` | This file is read at boot, which is what makes the change permanent. `setenforce 0` would only last until the next reboot. |
| `grep ^SELINUX=` | The file also contains `SELINUXTYPE=` and a commented block listing the valid values. Anchoring on `^SELINUX=` pins the one line that matters. |

## Notes

- `setenforce 0` is runtime only and resets on reboot. `/etc/selinux/config` is the permanent one. The task asks for the state *after* the reboot, so the file is the answer.
- That is also why point 4 says to ignore the current CLI status: `getenforce` and `sestatus` will keep reporting Enforcing until the box actually reboots.
- Valid values in that file are `enforcing`, `permissive`, `disabled`. `permissive` logs violations without blocking, which is how you'd normally stage this before turning it on for real.
- On RHEL 9 and CentOS Stream 9, `SELINUX=disabled` in the config is deprecated. The kernel still boots with SELinux enabled and simply loads no policy. The supported way to fully disable it is `grubby --update-kernel ALL --args selinux=0`. KodeKloud validates the config file, so the edit above is the right answer for this task.
- `yum` is a`dn symlink to f` on CentOS Stream 9. Either works.
- `policycoreutils-python-utils` is the package that gives you `semanage`, needed later if you ever add port or file-context rules.
