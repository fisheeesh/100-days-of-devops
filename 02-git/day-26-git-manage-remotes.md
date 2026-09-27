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
| `git remote -v` | Lists every remote with its fetch and push URLs. Running it before and after the change proves that `dev_media` was added without replacing the existing `origin`. |
| `git remote add dev_media /opt/xfusioncorp_media.git` | Stores the name `dev_media` as an alias for the new bare repository path. Future commands can use the short name instead of repeating the full path. |
| `git remote get-url dev_media` | Reads back the configured URL and catches spelling or path mistakes before anything is pushed. |
| `git checkout master` | Ensures that the file is committed to the branch named in the requirement, rather than whichever branch happened to be checked out. |
| `cp /tmp/index.html .` | Copies the supplied file into the repository's working tree. The final `.` means the current directory, `/usr/src/kodekloudrepos/media`. |
| `git add index.html` | Stages the copied file so its content will be included in the next commit. |
| `git commit -m "Add index.html"` | Records the staged file in the local `master` branch history. |
| `git push dev_media master` | Sends the local `master` branch to the new remote named `dev_media`. Pushing to `origin` would update `/opt/media.git`, which is not the requested destination. |
| `git ls-remote --heads dev_media master` | Reads `master` directly from the new remote, proving that the push reached `/opt/xfusioncorp_media.git`. |

## Notes

- A **remote** is a local name for another Git repository. It is not necessarily on another machine; both remotes in this task are paths on the same storage server.
- `origin` is only the conventional name Git assigns to the source of a clone. It is not a special Git keyword. The newly required remote is named `dev_media`, so the correct destination is `git push dev_media master`.
- Adding `dev_media` does not copy commits by itself. It only records the URL. Data moves when commands such as `fetch`, `pull`, or `push` use that remote.
- The command intentionally omits `-u`. The local `master` branch may already track `origin/master`, and this one-time push should not silently change its upstream to `dev_media/master`.
- If `index.html` already exists, `cp` replaces its contents and `git status --short` shows `M index.html` instead of `?? index.html`. Either is valid as long as the supplied file is staged and committed.
- The repository is root-owned in this lab, so `sudo` is used for the copy and Git commands. Without it, Git may report dubious ownership or the copy may fail with `Permission denied`.
