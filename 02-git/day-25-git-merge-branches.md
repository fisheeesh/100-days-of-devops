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
| `git checkout -b datacenter master` | Creates `datacenter` at the commit currently referenced by `master` and switches to it. Naming `master` explicitly guarantees the correct starting point regardless of the branch that was previously checked out. |
| `cp /tmp/index.html .` | Copies the supplied file into the repository's working tree. The final `.` means the current directory, `/usr/src/kodekloudrepos/blog`. |
| `git status --short` | Shows `?? index.html` before staging, proving that Git sees the new untracked file. No output after the commit means the working tree is clean. |
| `git add index.html` | Places the new file in Git's staging area so it will be included in the next commit. |
| `git commit -m "Add index.html"` | Records the staged file in the history of the currently checked-out `datacenter` branch. |
| `git push -u origin datacenter` | Creates `datacenter` on the remote and sets `origin/datacenter` as its upstream, making later pushes and pulls possible without repeating the remote and branch names. |
| `git merge datacenter` | Moves the committed change into `master`. Because `master` has not diverged, Git can normally perform a fast-forward merge without creating an unnecessary merge commit. |
| `git push origin master` | Updates the remote `master` branch after the local merge. Pushing `datacenter` earlier does not update `master`; the two remote branches must be pushed separately. |
| `git ls-remote --heads` | Reads branch references directly from `origin`, proving that both required branches exist there and point to the expected commit. |

## Notes

- The correct clone path is `/usr/src/kodekloudrepos/blog`. The shorter path `/usr/src/kodekoudrepos` is a typo and does not exist.
- The repository is root-owned in this lab, so `sudo` is needed for the copy and Git commands. Otherwise the copy can fail with `Permission denied`, and Git can reject the repository because of dubious ownership.
- A fast-forward is still a merge. Since `master` has no new commits after `datacenter` branches from it, Git only advances the `master` pointer to the commit already on `datacenter`.
- `git add` stages content, while `git commit` saves that staged snapshot to the current branch. Copying the file into the directory alone does not put it in Git history.
- A commit is local until it is pushed. This task explicitly requires both remote branches, so completing only `git push origin datacenter` or only `git push origin master` is not enough.
- `git status` verifies the local working tree. `git ls-remote` is the stronger final check for this task because it verifies the state stored in the bare `origin` repository.
