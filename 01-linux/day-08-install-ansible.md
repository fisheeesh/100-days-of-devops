# Day 8: Install Ansible

## Question

During the weekly meeting, the Nautilus DevOps team discussed about the automation and
configuration management solutions that they want to implement. While considering several
options, the team has decided to go with `Ansible` for now due to its simple setup and
minimal pre-requisites. The team wanted to start testing using Ansible, so they have decided
to use `jump host` as an Ansible controller to test different kind of tasks on rest of the
servers.

Install `ansible` version `4.7.0` on `Jump host` using `pip3` only. Make sure Ansible binary
is available globally on this system, i.e all users on this system are able to run Ansible
commands.

## Solution

Already on the jump host as `thor`, no ssh needed.

```bash
which pip3
# /usr/bin/pip3

sudo pip3 install ansible==4.7.0

which ansible
# /usr/local/bin/ansible

ansible --version
# ansible [core 2.11.x]   <- core version, not 4.7.0, see notes

pip3 show ansible
# Version: 4.7.0          <- this is the one the task asks for
```

At this point `ansible` works for `thor` but not through `sudo`:

```bash
echo $PATH
# ... :/usr/local/bin: ...   present for the normal shell

sudo ansible --version
# sudo: ansible: command not found

sudo visudo
# find the secure_path line and append /usr/local/bin
# Defaults    secure_path = /sbin:/bin:/usr/sbin:/usr/bin:/usr/local/bin

sudo ansible --version
# same output as ansible --version
```

## Why

| Command | Why |
|:--|:--|
| `which pip3` | The task says pip3 only. Confirm it exists before anything else. |
| `sudo pip3 install` | `sudo` makes it a system-wide install under `/usr/local/`, which is what "all users can run it" means. `pip3 install --user` would land in `~/.local/bin` for `thor` alone and fail the requirement. |
| `==4.7.0` | Pins the exact version. Without it pip installs the newest release, which is a different major series entirely. |
| `which ansible` | Tells you where the binary actually landed. That path is what you need for the next step. |
| `pip3 show ansible` | `ansible --version` does not print the package version. This is the only command that shows `4.7.0`. |
| `sudo visudo` | sudo discards your `PATH` and uses `secure_path` from sudoers instead. `/usr/local/bin` is not in the default `secure_path`, which is why `ansible` works but `sudo ansible` does not. |
| `visudo`, never `vi /etc/sudoers` | visudo syntax-checks the file before saving. A broken sudoers locks everyone out of sudo, and you cannot use sudo to fix it. |

## Notes

- The version confusion is the whole trap here. `ansible` 4.7.0 is the **community package**, a bundle of collections. `ansible --version` reports the **ansible-core** version underneath it, which is 2.11.x for the 4.x line. Both numbers are right, they name different things. Check the one the task asked for with `pip3 show ansible`.
- pip will warn about running as root. Expected: a system-wide install is exactly the requirement.
- Do not reach for `dnf install ansible`. The task says pip3 only, and the repo version will not be 4.7.0.
- `secure_path` exists so that a tampered user `PATH` cannot trick root into running a planted binary. Adding `/usr/local/bin` is normal practice, just know you are widening that surface.
- If `which ansible` shows a different directory, put that directory in `secure_path` instead. Do not copy `/usr/local/bin` blindly.
- Verify as another user if you want to be sure it is really global: `sudo su - <someone> -c 'ansible --version'`.
