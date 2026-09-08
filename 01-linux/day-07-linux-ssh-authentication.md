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
| `ssh-keygen -t ed25519` | `-t` is the key **type**. ed25519 keys are short and fast, with security comparable to RSA-3072 at a fraction of the size and no key length or padding options to get wrong. OpenSSH 9.5 and newer default to ed25519 with no `-t` at all; older versions default to RSA. Being explicit gets you the same key on any box. |
| Empty passphrase | A passphrase means ssh prompts on every connection. The scripts running on a schedule have nobody to type it, so the whole task fails. |
| `ssh-copy-id user@host` | Appends `id_ed25519.pub` to that user's `~/.ssh/authorized_keys` on the server, creating `.ssh` and the file if needed, and setting the modes correctly. Doing it by hand means getting all of that right yourself. |
| One run per server | Each app server has a different sudo user (`tony`, `steve`, `banner`). The key is the same every time; the account it is being installed into is not. |
| `cat ~/.ssh/authorized_keys` | The line on the server should be byte for byte what is in `id_ed25519.pub`. If it does not match, the key went to the wrong account. |

## Notes

- The private key never leaves the jump host. Only the `.pub` travels. If you ever see `id_ed25519` without the `.pub` sitting on a server, something went wrong.
- A public key grants access to the **one account** whose `authorized_keys` it sits in. The same key in `tony`'s file on stapp01 and `steve`'s file on stapp02 is two separate grants that happen to use the same key.
- Permissions are the number one reason this silently fails, but the real rule is narrower than people assume. `StrictModes` refuses the key if the home directory, `~/.ssh`, or `authorized_keys` is **group or world writable** (any of the `022` bits), or is owned by anyone other than that user or root. Read bits are irrelevant: `755` on `~/.ssh` and `644` on `authorized_keys` pass fine, `775` and `664` do not. `700` and `600` are the safe habit, not the enforced requirement.
- Ownership bites after copying keys around as root. `chown -R tony:tony ~tony/.ssh` matters as much as the mode.
- When it does fail the client says nothing useful: you fall back to a password prompt, or `Permission denied (publickey)` if password auth is off.
- `ssh-copy-id -i ~/.ssh/other_key.pub user@host` installs exactly one named key. Without `-i` it installs **every** key your agent reports from `ssh-add -L`, and only falls back to the most recent `~/.ssh/id*.pub` when the agent has none. On a box with several keys that is not necessarily the one you meant.
- `known_hosts` fills up on first connect. That is the **server** proving its identity to you, the opposite direction from what you just set up. `known_hosts.old` is a backup that `ssh-keygen` writes when it rewrites the file (`-R` to drop a host, `-H` to hash it). Ordinary connections just append and never create it.
- Password-less here means no password for `ssh`. `sudo` on the server is separate and may still prompt.
- `ssh -v tony@stapp01` shows which key the client offers, but a StrictModes rejection never appears there. The reason only exists in the server's auth log: `sudo tail -f /var/log/secure` on RHEL, `journalctl -u sshd -f` on systemd, where it reads `Authentication refused: bad ownership or modes for directory /home/tony/.ssh`.
