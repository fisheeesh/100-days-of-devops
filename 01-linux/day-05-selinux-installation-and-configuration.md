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
| `hostnamectl` | Package names differ per distro. Confirm the family before reaching for `yum`. |
| `selinux-policy-targeted` | The policy actually loaded at boot. The framework alone has no rules. |
| `vi /etc/selinux/config` | Read at boot, which is what makes it permanent. `setenforce 0` lasts until reboot. |
| `grep ^SELINUX=` | The file also holds `SELINUXTYPE=`. The anchor pins the one line that matters. |

## Notes

- Valid values: `enforcing`, `permissive`, `disabled`.
- `getenforce` keeps saying Enforcing until the reboot, which is why point 4 says to ignore it.
- On RHEL 9 `SELINUX=disabled` is deprecated: the kernel boots with SELinux on and no policy loaded. Fully disabling it needs `grubby --update-kernel ALL --args selinux=0`. KodeKloud checks the file.
