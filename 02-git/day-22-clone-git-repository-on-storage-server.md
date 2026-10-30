# Day 22: Clone Git Repository on Storage Server

## Question

The DevOps team established a new Git repository last week, which remains unused at present.
However, the Nautilus application development team now requires a copy of this repository on the
`Storage Server` in the Stratos DC. Follow the provided details to clone the repository:

1. The repository to be cloned is located at `/opt/apps.git`
2. Clone this Git repository to the `/usr/src/kodekloudrepos` directory. Perform this task using
   the natasha user, and ensure that no modifications are made to the repository or existing
   directories, such as changing permissions or making unauthorized alterations.

## Solution

```bash
ssh natasha@ststor01

# the source: a bare repo, git internals at the top level, no working tree
ls -lah /opt/apps.git
# HEAD  config  description  hooks  info  objects  refs

# /usr/src/kodekloudrepos is root-owned, so the clone needs sudo
sudo git clone /opt/apps.git /usr/src/kodekloudrepos/apps

ls -lah /usr/src/kodekloudrepos/apps
```

The clone belongs to root while you are natasha, so git refuses to touch it:

```bash
cd /usr/src/kodekloudrepos/apps

git status
# fatal: detected dubious ownership in repository at '/usr/src/kodekloudrepos/apps'

git config --global --add safe.directory /usr/src/kodekloudrepos/apps

git status
git log --oneline
git remote -v
# origin  /opt/apps.git (fetch)
# origin  /opt/apps.git (push)
```

## Why

| Command | Why |
|:--|:--|
| `ls -lah /opt/apps.git` | Confirms it is bare: `HEAD`, `config`, `objects/`, `refs/` at the top level, no working tree. |
| `sudo git clone` | `/usr/src/kodekloudrepos` is root-owned. The task forbids changing permissions, so sudo is the answer, not `chown`. |
| `git config --global --add safe.directory` | The clone belongs to root and you are natasha, so git refuses. `--global` writes your own config, not the repo. |
| `git remote -v` | `origin` is `/opt/apps.git`, so a later `git pull` works with no network at all. |

## Notes

- `detected dubious ownership` is a Git 2.35.2 security fix: another user's repo could otherwise run commands at you through its config.
- Local clones **hardlink** the objects instead of copying them, so the clone is near-instant and costs almost no disk. `--no-hardlinks` forces real copies.
- "No modifications" means no `chown`, no `chmod`, no commits. Cloning only reads the source.
