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
| `ls -lah /opt/apps.git` | Confirms it is a **bare** repo: `HEAD`, `config`, `objects/` and `refs/` sit at the top level with no working tree. The `.git` suffix in the name is the convention for exactly this. |
| `sudo git clone` | `/usr/src/kodekloudrepos` is root-owned and natasha cannot write there. Without sudo the clone stops at `Permission denied`. The task forbids changing permissions, so sudo is the answer, not `chown`. |
| Naming the destination `.../apps` | Cloning `apps.git` creates a directory called `apps`. Spelling the path out does the same thing and leaves no doubt where it landed. |
| `git config --global --add safe.directory` | The repo is owned by root and you are natasha, so git refuses to run. This marks that one path as trusted. `--global` writes to natasha's `~/.gitconfig`, so it changes your own config, not the repository. |
| `git remote -v` | Shows `origin` as `/opt/apps.git`. A clone from a local path records that path, so `git pull` later works with no network at all. |

## Notes

- A bare repo has no working tree, so there are no files to edit inside it. It exists to be pushed to and cloned from, which is what a server-side repo is for.
- `detected dubious ownership` arrived in Git 2.35.2 as a security fix: a repository owned by someone else could otherwise run commands at you through its config. `safe.directory` is the opt-in trust list.
- The other way round it is to run every git command with `sudo`. Adding `safe.directory` is nicer, because plain `git status` keeps working as natasha.
- Cloning from a local path **hardlinks** the object files instead of copying them. The object files in the clone share an inode with the ones in `/opt/apps.git`, so the clone is near-instant and costs almost no disk. `--no-hardlinks` forces real copies. Git never rewrites an object in place, so sharing them is safe.
- "No modifications to the repository or existing directories" means no `chown`, no `chmod`, no commits. Cloning only reads the source.
- Running `git clone /opt/apps.git` from inside `/usr/src/kodekloudrepos` gets the same result, but only if you are already in that directory and can write to it.
