# Day 7: Linux SSH Authentication

## Question

The system admins team of `xFusionCorp Industries` has set up some scripts on `jump host`
that run on regular intervals and perform operations on all app servers in
`Stratos Datacenter`. To make these scripts work properly we need to make sure the `thor`
user on jump host has password-less SSH access to all app servers through their respective
sudo users (i.e `tony` for app server 1). Based on the requirements, perform the following:

Set up a password-less authentication from user `thor` on jump host to all app servers
through their respective sudo users.

## Solution

Password-less SSH is a key pair. The private key stays with you, the public key goes on the
server. Whoever holds the private key can log in as the account that has the matching public
key in its `authorized_keys`.

On the jump host, as `thor`:

```bash
ssh-keygen -t ed25519
# accept the default path
# leave the passphrase EMPTY, a passphrase would defeat the point

ls -lah ~/.ssh/
# -rw------- 1 thor thor 411 id_ed25519       private, never leaves this box
# -rw-r--r-- 1 thor thor  96 id_ed25519.pub   public, this is what gets copied out
```

Push the public key to each app server, each through its own sudo user:

```bash
ssh-copy-id tony@stapp01
ssh-copy-id steve@stapp02
ssh-copy-id banner@stapp03
# enter each user's password once, that is the last time you need it
```

Verify:

```bash
ssh tony@stapp01              # lands with no password prompt

cat ~/.ssh/authorized_keys    # on the server
# ssh-ed25519 AAAAC3Nza...Bzglq1 thor@jump-host
exit

cat ~/.ssh/id_ed25519.pub     # back on the jump host, identical line
```

## Why

| Command | Why |
|:--|:--|
| `ssh-keygen -t ed25519` | `-t` is the key **type**. ed25519 keys are short, fast, and stronger than RSA at a fraction of the size. OpenSSH 9.5 and newer default to ed25519 with no `-t` at all, which is why the bare `ssh-keygen` produced one here. Older versions default to RSA, so being explicit gets you the same key on any box. |
| Empty passphrase | A passphrase means ssh prompts on every connection. The scripts running on a schedule have nobody to type it, so the whole task fails. |
| `ssh-copy-id user@host` | Appends `id_ed25519.pub` to that user's `~/.ssh/authorized_keys` on the server, creating `.ssh` and the file if needed, and setting the modes correctly. Doing it by hand means getting all of that right yourself. |
| One run per server | Each app server has a different sudo user (`tony`, `steve`, `banner`). The key is the same every time; the account it is being installed into is not. |
| `cat ~/.ssh/authorized_keys` | The line on the server should be byte for byte what is in `id_ed25519.pub`. If it does not match, the key went to the wrong account. |

## Notes

- The private key never leaves the jump host. Only the `.pub` travels. If you ever see `id_ed25519` without the `.pub` sitting on a server, something went wrong.
- A public key grants access to the **one account** whose `authorized_keys` it sits in. The same key in `tony`'s file on stapp01 and `steve`'s file on stapp02 is two separate grants that happen to use the same key.
- Permissions are the number one reason this silently fails. sshd's `StrictModes` ignores the key if `~/.ssh` is not `700`, `authorized_keys` is not `600`, or the home directory itself is group or world writable. The client shows no error, you just get a password prompt as if nothing happened.
- `ssh-copy-id -i ~/.ssh/other_key.pub user@host` when the key is not the default name. Without `-i` it uses the default identity.
- `known_hosts` fills up on first connect. That is the **server** proving its identity to you, the opposite direction from what you just set up. `known_hosts.old` is the previous version, written when the file gets rewritten.
- Password-less here means no password for `ssh`. `sudo` on the server is separate and may still prompt.
- Debug a failure from the client with `ssh -v tony@stapp01` and watch which key it offers.
