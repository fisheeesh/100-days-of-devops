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
| `sudo pip3 install` | System-wide under `/usr/local/`, which is what "all users" means. `--user` would be thor only. |
| `==4.7.0` | Pins the version. Without it pip takes the newest, a different major series. |
| `pip3 show ansible` | `ansible --version` does not print the package version. Only this shows `4.7.0`. |
| `sudo visudo` | sudo ignores your `PATH` and uses `secure_path`, which is why `ansible` works but `sudo ansible` does not. |

## Notes

- `ansible` 4.7.0 is the community bundle; `ansible --version` reports ansible-core 2.11.x. Both are correct, they name different things.
- `visudo`, never `vi /etc/sudoers`. A broken sudoers locks everyone out and you cannot sudo to fix it.
- If `which ansible` shows a different directory, put that one in `secure_path`.
