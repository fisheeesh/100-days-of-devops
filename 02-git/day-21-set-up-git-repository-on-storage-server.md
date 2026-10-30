# Day 21: Set Up Git Repository on Storage Server

## Question

The Nautilus development team has provided requirements to the DevOps team for a new application
development project, specially requesting the establishment of a Git repository. Follow the
instructions below to create the Git repository on the `Storage Server` in the Stratos DC:

1. Utilize `yum` to install the `git` package on the `Storage Server`.
2. Create a bare repository named `/opt/games.git` (ensure exact name usage).

## Solution

Connect to the storage server:

```bash
ssh natasha@ststor01
```

Install Git and confirm that it is available:

```bash
sudo yum install -y git

git --version
# git version 2.x.x
```

Create the bare repository at the exact path required by the task:

```bash
sudo git init --bare /opt/games.git
# Initialized empty Git repository in /opt/games.git/
```

Verify both the repository type and its contents:

```bash
sudo git --git-dir=/opt/games.git rev-parse --is-bare-repository
# true

sudo ls -lah /opt/games.git
# HEAD  branches  config  description  hooks  info  objects  refs
```

## Why

| Command | Why |
|:--|:--|
| `git init --bare /opt/games.git` | A bare repo is git's database with no working tree. The absolute path stops it landing in the wrong directory. |
| `rev-parse --is-bare-repository` | Asks git itself instead of guessing. Expected `true`. |

## Notes

- A bare repo holds `objects`, `refs`, `HEAD` and `config` at the top level instead of project files.
- No working tree, so you cannot edit inside it. Clone it, commit there, push back.
- The `.git` suffix is convention, and part of the required name here.
