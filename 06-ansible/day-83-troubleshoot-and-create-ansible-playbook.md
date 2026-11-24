# Day 83: Troubleshoot and Create Ansible Playbook

## Question

An Ansible playbook needs completion on the `jump host`, where a team member left off. Below are
the details:

1. The inventory file `/home/thor/ansible/inventory` requires adjustments. The playbook must run
   on `App Server 1` in `Stratos DC`. Update the inventory accordingly.
2. Create a playbook `/home/thor/ansible/playbook.yml`. Include a task to create an empty file
   `/tmp/file.txt` on `App Server 1`.

> `Note:` Validation will run the playbook using the command
> `ansible-playbook -i inventory playbook.yml`. Ensure the playbook works without any additional
> arguments.

## Solution

```bash
cd /home/thor/ansible
vi inventory
```

First attempt, no credential:

```ini
stapp01 ansible_user=tony ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

```bash
ansible all -i inventory -m ping
# stapp01 | UNREACHABLE! => {
#     "msg": "Invalid/incorrect password: ... Permission denied, please try again.",
#     "unreachable": true
# }
```

`Permission denied` means the host resolved and SSH answered, so only the credential is missing:

```ini
stapp01 ansible_user=tony ansible_ssh_pass='Ir0nM@n' ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

```bash
ansible all -i inventory -m ping
# stapp01 | SUCCESS => { "changed": false, "ping": "pong" }
```

Then the playbook:

```bash
vi playbook.yml
```

```yaml
- name: Create Empty File
  hosts: stapp01
  tasks:
    - name: Creating file /tmp/file.txt using file module
      file:
        path: /tmp/file.txt
        state: touch
```

```bash
ansible-playbook -i inventory playbook.yml
```

## Why

| Piece | Why |
|:--|:--|
| Pinging before fixing anything | The failure **is** the diagnosis. `Permission denied` rules out DNS and reachability and points straight at authentication. |
| `ansible_ssh_pass` in the inventory | Validation passes no extra arguments, so the credential cannot come from `--ask-pass` and has to live in the file. |
| `ansible_ssh_common_args='-o StrictHostKeyChecking=no'` | Skips the first-connection host key prompt, which a non-interactive run has no way to answer. |
| `file` with `state: touch` | Creates the file if it is missing and leaves it empty, which is exactly requirement 2. |

## Notes

- `ansible_ssh_pass` and `ansible_password` are the same setting; the first is the older alias. Day 82 used the other spelling.
- Check for stray whitespace inside the quotes. `'Ir0nM@n '` with a trailing space or tab is a different password, and it fails with the same `Permission denied` that sent you looking elsewhere.
- `state: touch` reports `changed` on **every** run, because it updates the timestamps. Add `modification_time: preserve` and `access_time: preserve` to make it idempotent, in contrast to the `changed=0` second run on Day 82.
- A bare host line with no `[group]` header is valid ini. The host simply lands in the ungrouped set, and `hosts: stapp01` targets it by name.
