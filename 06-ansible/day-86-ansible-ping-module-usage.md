# Day 86: Ansible Ping Module Usage

## Question

The Nautilus DevOps team is planning to test several Ansible playbooks on different app servers in
`Stratos DC`. Before that, some pre-requisites must be met. Essentially, the team needs to set up
a password-less SSH connection between Ansible controller and Ansible managed nodes. One of the
tickets is assigned to you; please complete the task as per details mentioned below:

a. `Jump host` is our Ansible controller, and we are going to run Ansible playbooks through `thor`
user from `jump host`.

b. There is an inventory file `/home/thor/ansible/inventory` on `jump host`. Using that inventory
file test `Ansible ping` from `jump host` to `App Server 1`, make sure ping works.

## Solution

See what the inventory actually holds before connecting:

```bash
ansible all -i inventory --list-hosts
# three app servers

cat inventory
# stapp01 ansible_ssh_pass=Ir0nM@n
# stapp02 ansible_ssh_pass=Am3ric@
# stapp03 ansible_ssh_pass=BigGr33n
# passwords, but no users
```

```bash
ansible stapp01 -i inventory -m ping
# stapp01 | FAILED! => {
#     "msg": "Using a SSH password instead of a key is not possible because Host Key
#             checking is enabled and sshpass does not support this. Please add this
#             host's fingerprint to your known_hosts file to manage this host."
# }
```

Add the users **and** turn off the first-connection fingerprint prompt:

```bash
vi inventory
```

```ini
stapp01 ansible_host=stapp01 ansible_user=tony   ansible_password=Ir0nM@n  ansible_ssh_common_args='-o StrictHostKeyChecking=no'
stapp02 ansible_host=stapp02 ansible_user=steve  ansible_password=Am3ric@  ansible_ssh_common_args='-o StrictHostKeyChecking=no'
stapp03 ansible_host=stapp03 ansible_user=banner ansible_password=BigGr33n ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

```bash
ansible stapp01 -i inventory -m ping
# stapp01 | SUCCESS => { "changed": false, "ping": "pong" }
```

## Why

| Piece | Why |
|:--|:--|
| `--list-hosts` | Shows what the inventory contains before you try to use it. Here it exposed passwords with no users attached. |
| `ansible_user` | Without it Ansible connects as the local user, `thor`, who has no account on the app servers. |
| `ansible_ssh_common_args='-o StrictHostKeyChecking=no'` | Skips the first-connection fingerprint prompt, which is the thing actually blocking the connection. |
| `ansible stapp01 -i inventory -m ping` | Targets one host by name instead of `all`, which is what requirement b asks for. |

## Notes

- **Why the error blames sshpass.** The jump host has never connected to these servers, so their fingerprints are not in `~/.ssh/known_hosts`. On a first connection SSH stops and asks "Are you sure you want to continue connecting (yes/no)?". Password login goes through `sshpass`, which cannot answer that question, so Ansible gives up before it ever tries the password. Key-based login hits the same check, but there a human can type "yes". With a password you cannot.
- The task's preamble says password-less SSH, and the more literal route is `ssh-keygen` then `ssh-copy-id` to each host. Accepting the fingerprint during `ssh-copy-id` also clears the known_hosts problem, after which the inventory only needs `ansible_user`.
- Other ways past the same host-key check: `ssh-keyscan stapp01 >> ~/.ssh/known_hosts`, or `host_key_checking = False` under `[defaults]` in `ansible.cfg`.
- `ansible_ssh_pass` and `ansible_password` are the same setting. The original file used the older spelling.
