# Day 82: Create Ansible Inventory for App Server Testing

## Question

The Nautilus DevOps team is testing Ansible playbooks on various servers within their stack.
They've placed some playbooks under `/home/thor/playbook/` directory on the `jump host` and now
intend to test them on `app server 3` in `Stratos DC`. However, an inventory file needs creation
for Ansible to connect to the respective app. Here are the requirements:

a. Create an ini type Ansible inventory file `/home/thor/playbook/inventory` on `jump host`.

b. Include `App Server 3` in this inventory along with necessary variables for proper
functionality.

c. Ensure the inventory hostname corresponds to the `server name` as per the wiki, for example
`stapp01` for `app server 1` in `Stratos DC`.

> `Note:` Validation will execute the playbook using the command
> `ansible-playbook -i inventory playbook.yml`. Ensure the playbook functions properly without any
> extra arguments.

## Solution

```bash
cd /home/thor/playbook
vi inventory
```

```ini
[app_servers]
stapp03 ansible_user=banner ansible_password='BigGr33n' ansible_host=stapp03 ansible_connection=ssh ansible_port=22
```

Check the connection before touching the playbook:

```bash
ansible all -m ping -i inventory
# stapp03 | SUCCESS => {
#     "changed": false,
#     "ping": "pong"
# }
```

Then run it:

```bash
ansible-playbook -i inventory playbook.yml
# TASK [Install httpd package] ... changed: [stapp03]
# TASK [Start service httpd]   ... changed: [stapp03]
# stapp03 : ok=3  changed=2  unreachable=0  failed=0

# run it again
# stapp03 : ok=3  changed=0  unreachable=0  failed=0
```

## Why

| Piece | Why |
|:--|:--|
| `ansible_user` and `ansible_password` in the file | Validation runs `ansible-playbook -i inventory playbook.yml` with no extra arguments, so every connection detail has to live in the inventory itself. |
| Hostname `stapp03` | Requirement c. The checker reads the inventory hostname, not just whether the connection works. |
| `ansible all -m ping` | Tests reachability and authentication on their own. A `pong` means the inventory is right before any playbook logic is involved. |
| ini format with inline variables | Requirement a asks for ini, where host variables sit on the same line as the host. |

## Notes

- The second run reporting `changed=0` is **idempotence**, the idea the whole tool is built on. Ansible checks the current state before acting, so re-running a playbook that already matches does nothing and takes far less time.
- `ansible_password` means password authentication, which needs `sshpass` on the control node. Without it Ansible stops and tells you to install it.
- `become: yes` in the playbook escalates to root on the target. If sudo there asked for a password you would also need `ansible_become_password` in the inventory.
- PLAY RECAP: `ok` counts tasks that ran fine, `changed` counts the ones that altered something, `unreachable` means the connection failed rather than the task.
