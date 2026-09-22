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
| `ssh natasha@ststor01` | Connects to the storage server, which is the host where the repository must be created. |
| `yum install -y git` | Installs Git through the package manager required by the task. `-y` automatically confirms the installation prompt. |
| `git --version` | Confirms that the Git executable is installed and available in `PATH`. |
| `git init --bare /opt/games.git` | Creates a repository containing only Git's internal data, with no checked-out working tree. Supplying the absolute path also avoids accidentally creating it in the wrong directory. |
| `rev-parse --is-bare-repository` | Asks Git itself whether the target is a bare repository. The expected result is `true`. |
| `ls -lah /opt/games.git` | Confirms that the exact directory exists and contains Git's repository structure. |

## Notes

- A **bare repository** stores Git's internal database directly in the repository directory. That is why `/opt/games.git` contains entries such as `objects`, `refs`, `HEAD`, and `config` instead of normal application files.
- It has no working tree, so files cannot be checked out and edited directly inside `/opt/games.git`. Make changes in a regular clone, commit them there, and push the commits back to the bare repository.
- Bare repositories are commonly used as central repositories. Developers or other servers clone from them and use them as remotes for `fetch`, `pull`, and `push` operations.
- The `.git` suffix is a convention for bare repositories. In this task it is also part of the required name, so `/opt/games.git` must be used exactly.
- The repository is empty immediately after `git init --bare`. Its `objects` and `refs` directories receive project history only after someone pushes commits to it.
- Because the repository was created with `sudo`, it is owned by `root`. A real shared server would also need an appropriate Unix group and write permissions for the developers who are allowed to push.
