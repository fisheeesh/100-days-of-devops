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
# test it the way the scheduled scripts will connect: non-interactive,
# with password fallback off so a failure is loud instead of a prompt
for h in tony@stapp01 steve@stapp02 banner@stapp03; do
  ssh -o BatchMode=yes -o PasswordAuthentication=no $h hostname
done
# prints the three hostnames

ssh tony@stapp01
cat ~/.ssh/authorized_keys    # on the server
# ssh-ed25519 AAAAC3Nza...Bzglq1 thor@jump-host
exit

cat ~/.ssh/id_ed25519.pub     # back on the jump host, identical line
```

## Why

| Command | Why |
|:--|:--|
| `ssh-keygen -t ed25519` | Short, fast, no key size to get wrong. OpenSSH 9.5+ defaults to it, older to RSA. |
| Empty passphrase | A passphrase prompts on every connection, and scheduled scripts have nobody to type it. |
| `ssh-copy-id user@host` | Appends the `.pub` to the remote `authorized_keys` and sets the modes correctly. |
| One run per server | Each app server has its own sudo user. Same key, different account. |

## Notes

- The private key never leaves the jump host. Only the `.pub` travels.
- A public key grants access to the one account whose `authorized_keys` holds it.
- `StrictModes` rejects the key if home, `~/.ssh` or `authorized_keys` is group or world **writable**, or owned by someone else. `755` and `644` pass, `775` and `664` do not.
- A StrictModes rejection shows nothing useful on the client. The reason is in the server's `/var/log/secure`.
