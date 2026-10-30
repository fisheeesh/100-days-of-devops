# Day 25: Git Merge Branches

## Question

The Nautilus application development team has been working on a project repository
`/opt/blog.git`. This repo is cloned at `/usr/src/kodekloudrepos` on `Storage Server` in
`Stratos DC`. They recently shared the following requirements with the DevOps team:

Create a new branch `datacenter` in the `/usr/src/kodekloudrepos/blog` repo from `master` and
copy the `/tmp/index.html` file (present on `Storage Server` itself) into the repo. Further,
add and commit this file in the new branch and merge that branch back into the `master` branch.
Finally, push the changes to the origin for both branches.

## Solution

Connect to the storage server and inspect the remote repository:

```bash
ssh natasha@ststor01

sudo ls -lah /opt/blog.git
# HEAD  config  description  hooks  info  objects  refs
```

Move into the existing clone and confirm its current state:

```bash
cd /usr/src/kodekloudrepos/blog

sudo git status
sudo git remote -v
sudo git branch -a
```

Create `datacenter` directly from `master`, then copy and commit the required file:

```bash
sudo git checkout -b datacenter master
# Switched to a new branch 'datacenter'

sudo cp /tmp/index.html .

sudo git status --short
# ?? index.html

sudo git add index.html
sudo git commit -m "Add index.html"

sudo git status --short
# no output: the change is committed
```

Push the new branch to `origin`:

```bash
sudo git push -u origin datacenter
```

Merge the change into `master` and push `master` as well:

```bash
sudo git checkout master
sudo git merge datacenter
# Fast-forward

sudo git push origin master
```

Verify the local repository and both remote branches:

```bash
sudo git status
# On branch master
# Your branch is up to date with 'origin/master'.
# nothing to commit, working tree clean

sudo git branch -vv
sudo git log --oneline --decorate --graph --all -5

sudo git ls-remote --heads origin master datacenter
# both refs/heads/master and refs/heads/datacenter point to the new commit
```

## Why

| Command | Why |
|:--|:--|
| `git checkout -b datacenter master` | Creates the branch at master's commit, whatever branch you were standing on. |
| `git add` then `git commit` | Copying the file into the directory is not enough: `add` stages, `commit` records. |
| `git push -u origin datacenter` | Creates the branch on the remote and sets its upstream. |
| `git ls-remote --heads` | Reads the refs from `origin` itself, the only real proof both branches landed. |

## Notes

- A fast-forward is still a merge. Master had no new commits, so git just moves the pointer.
- Both remote branches are required. Pushing only one of them is not enough.
- The repo is root-owned, so the copy and the git commands need sudo.
- The path is `/usr/src/kodekloudrepos/blog`. `kodekoudrepos` is a typo and does not exist.
