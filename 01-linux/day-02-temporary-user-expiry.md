# Day 2: Temporary User Setup with Expiry

## Question

As part of the temporary assignment to the `Nautilus` project, a developer named `jim`
requires access for a limited duration. To ensure smooth access management, a temporary
user account with an expiry date is needed. Here's what you need to do:

Create a user named `jim` on `App Server 3` in Stratos Datacenter. Set the expiry date to
`2026-12-07`, ensuring the user is created in lowercase as per standard protocol.

## Solution

```bash
ssh banner@stapp03

sudo useradd -e 2026-12-07 jim

# verify
sudo chage -l jim
```

## Why

| Command | Why |
|:--|:--|
| `useradd -e YYYY-MM-DD` | Sets **account** expiry. On that date the whole account is disabled. |
| `chage -l jim` | `/etc/shadow` stores expiry as days since epoch. `chage -l` renders it as a date. |

## Notes

- Account expiry, not password expiry. `chage -E` changes it afterwards.
- Lowercase matters: `Jim` and `jim` are two different users.

## Handy

```bash
sudo !!              # re-run the last command with sudo
<command> | grep kw  # narrow long output
```
