# Day 85: Create Files on App Servers using Ansible

## Question

The Nautilus DevOps team is testing various Ansible modules on servers in `Stratos DC`. They're
currently focusing on file creation on remote hosts using Ansible. Here are the details:

a. Create an inventory file `~/playbook/inventory` on `jump host` and include `all app servers`.

b. Create a playbook `~/playbook/playbook.yml` to create a blank file `/opt/opt.txt` on
`all app servers`.

c. Set the permissions of the `/opt/opt.txt` file to `0744`.

d. Ensure the user/group owner of the `/opt/opt.txt` file is `tony` on `app server 1`, `steve` on
`app server 2` and `banner` on `app server 3`.

> `Note:` Validation will execute the playbook using the command
> `ansible-playbook -i inventory playbook.yml`, so ensure the playbook functions correctly without
> any additional arguments.

## Solution

```bash
cd ~/playbook
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
```

```bash
vi playbook.yml
```

```yaml
---
- name: create a blank file
  hosts: all
  become: true
  tasks:
    - name: create a blank file /opt/opt.txt
      file:
        path: /opt/opt.txt
        state: touch
        mode: '0744'
        owner: "{{ ansible_user }}"
        group: "{{ ansible_user }}"
```

```bash
ansible-playbook -i inventory playbook.yml
```

Verify on a host:

```bash
ssh tony@stapp01
ls -lah /opt/opt.txt
# -rwxr--r-- 1 tony tony 0 Nov 26 06:27 /opt/opt.txt
```

## Why

| Piece | Why |
|:--|:--|
| `state: touch` | Creates the file if it is missing and leaves it empty, which is what "blank file" means. `state: file` only adjusts a file that already exists. |
| `mode: '0744'` | Requirement c, giving `rwxr--r--`. Owner can read, write and execute; everyone else can only read. |
| `owner`/`group` as `{{ ansible_user }}` | Requirement d wants a different owner per host, and that owner happens to be each host's own login user, which the inventory already holds. One task covers all three. |
| `become: true` | `/opt` is root-owned, so creating anything there needs escalation even though the finished file belongs to the login user. |

## Notes

- `{{ ansible_user }}` works **only because** the required owner matches the login user on every host. If the task had asked for some other owner, the clean way is a separate per-host variable in the inventory, such as `file_owner=tony`, then `owner: "{{ file_owner }}"`.
- Quote the mode. `'0744'` leaves nothing to interpret, while unquoted `0744` relies on YAML reading a leading zero as octal. Both work here, but the quoted form is the habit worth having.
- `state: touch` reports `changed` on every run because it updates timestamps, same as Day 83.
- `become: true` escalates to root, so the file is created as root and then handed to the named owner. That is why the ownership lines are needed at all.
