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
