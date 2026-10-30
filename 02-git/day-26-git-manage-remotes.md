# Day 26: Git Manage Remotes

## Question

The xFusionCorp development team added updates to the project that is maintained under
`/opt/media.git` repo and cloned under `/usr/src/kodekloudrepos/media`. Recently some changes were
made on the Git server that is hosted on `Storage Server` in `Stratos DC`. The DevOps team added
some new Git remotes, so we need to update the remote on `/usr/src/kodekloudrepos/media`
repository as per the details mentioned below:

a. In `/usr/src/kodekloudrepos/media` repo, add a new remote `dev_media` and point it to
`/opt/xfusioncorp_media.git` repository.

b. There is a file `/tmp/index.html` on the same server; copy this file to the repo and add and
commit it to the `master` branch.

c. Finally, push the `master` branch to this new remote.

## Solution

Connect to the storage server and move into the existing clone:

```bash
ssh natasha@ststor01

cd /usr/src/kodekloudrepos/media
```

Inspect the current branch, working tree, and remotes before changing anything:

```bash
sudo git status
sudo git branch -a
sudo git remote -v
```

Add the new remote and verify its URL:

```bash
sudo git remote add dev_media /opt/xfusioncorp_media.git

sudo git remote -v
# dev_media  /opt/xfusioncorp_media.git (fetch)
# dev_media  /opt/xfusioncorp_media.git (push)
# origin     /opt/media.git (fetch)
# origin     /opt/media.git (push)

sudo git remote get-url dev_media
# /opt/xfusioncorp_media.git
```

Switch to `master`, copy the supplied file, and commit it:

```bash
sudo git checkout master

sudo cp /tmp/index.html .

sudo git status --short
# ?? index.html

sudo git add index.html
sudo git commit -m "Add index.html"

sudo git status --short
# no output: the change is committed
```

Push `master` to the new remote:

```bash
sudo git push dev_media master
```

Verify that the new remote received the branch and commit:

```bash
sudo git ls-remote --heads dev_media master
# <commit-sha>  refs/heads/master

sudo git log -1 --oneline --decorate
sudo git status
# nothing to commit, working tree clean
```

## Why

| Command | Why |
|:--|:--|
| `git remote add dev_media <path>` | Stores a name for the new repo's path. Nothing is copied yet. |
| `git remote -v` before and after | Proves `dev_media` was added without replacing `origin`. |
| `git push dev_media master` | Pushing to `origin` would update `/opt/media.git`, the wrong destination. |
| `git ls-remote --heads dev_media master` | Reads master from the new remote, proving the push arrived. |

## Notes

- A remote is just a local name for another repo. Both of these are paths on the same box.
- `origin` is a convention, not a keyword. The task names `dev_media`, so that is where it goes.
- No `-u` on purpose: master may already track `origin/master`, and this push should not change that.
