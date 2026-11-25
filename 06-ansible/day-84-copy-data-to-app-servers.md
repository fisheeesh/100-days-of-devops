# Day 84: Copy Data to App Servers using Ansible

## Question

The Nautilus DevOps team needs to copy data from the `jump host` to all `application servers` in
`Stratos DC` using Ansible. Execute the task with the following details:

a. Create an inventory file `/home/thor/ansible/inventory` on `jump_host` and add all application
servers as managed nodes.

b. Create a playbook `/home/thor/ansible/playbook.yml` on the `jump host` to copy the
`/usr/src/finance/index.html` file to all application servers, placing it at `/opt/finance`.

> `Note:` Validation will run the playbook using the command
> `ansible-playbook -i inventory playbook.yml`. Ensure the playbook functions properly without any
> extra arguments.

## Solution

```bash
cd /home/thor/ansible
vi inventory
```

```ini
[app_servers]
stapp01 ansible_user=tony ansible_password='Ir0nM@n'
stapp02 ansible_user=steve ansible_password='Am3ric@'
stapp03 ansible_user=banner ansible_password='BigGr33n'
```

```bash
ansible all -i inventory -m ping
# all three should answer pong
```

```bash
vi playbook.yml
```

```yaml
---
- name: Copy Data
  hosts: all
  become: true
  tasks:
    - name: Copy to app server
      copy:
        src: /usr/src/finance/index.html
        dest: /opt/finance/
        remote_src: false
```

```bash
ansible-playbook -i inventory playbook.yml
# changed on all three the first time, ok=... changed=0 on a second run
```

## Why

| Piece | Why |
|:--|:--|
| All three hosts, each with its own credential | The task says all application servers, and every app server has a different sudo user. Validation passes no extra arguments, so each credential has to live in the inventory. |
| `become: true` | `/opt` is root-owned. Without escalation the copy fails with permission denied on every host. |
| `remote_src: false` | Says the source is on the **control node**. Set it `true` and Ansible looks for the file on the managed node instead. |
| `dest: /opt/finance/` with the trailing slash | Marks the destination as a directory, so the file lands inside it rather than becoming it. |

## Notes

- Two directions of copying, and `remote_src` is what picks between them:
  - `false` (the default): jump host to managed node, which is this task.
  - `true`: the path is read **on the managed node**, so the copy happens within that host.
- Ping all three before running the playbook. With more than one host, a bad credential shows up straight away and the output names which host it was.
- `copy` compares checksums, so a second run reports `changed=0`. That is real idempotence, unlike Day 83's `state: touch`, which always reports changed.
- `hosts: all` picks up every host in the inventory regardless of group name, so the `[app_servers]` header is for readability rather than targeting.
