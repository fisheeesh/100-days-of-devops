# Day 2: Temporary User Setup with Expiry

## Question

Create a user named `jim` on App Server 3 with expiry date `2026-12-07`.

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
| `useradd -e YYYY-MM-DD` | Sets the **account** expiry date. On that date the account is disabled, no login at all. |
| `chage -l jim` | Reads back the ageing info. `/etc/shadow` stores expiry as days-since-epoch, so reading the file raw is useless. Look for `Account expires`. |

## Notes

- Account expiry ≠ password expiry. `-e` disables the whole account; `chage -E` changes it after the fact.
- Username must be lowercase. Linux is case-sensitive and `Jim` would fail the check.

## Handy

```bash
sudo !!              # re-run the last command with sudo, no retyping
<command> | grep kw  # narrow long output to the line you want
```
