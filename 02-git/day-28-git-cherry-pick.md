# Day 28: Git Cherry Pick

## Question

The Nautilus application development team has been working on a project repository
`/opt/media.git`. This repo is cloned at `/usr/src/kodekloudrepos` on `storage server` in
`Stratos DC`. They recently shared the following requirements with the DevOps team:

There are two branches in this repository, `master` and `feature`. One of the developers is
working on the `feature` branch and their work is still in progress, however they want to merge
one of the commits from the `feature` branch to the `master` branch, the message for the commit
that needs to be merged into `master` is `Update info.txt`. Accomplish this task for them, also
remember to push your changes eventually.

## Solution

Cherry-pick means: out of all the commits on a branch, take only the one you want.

```bash
ssh natasha@ststor01

sudo -i
cd /usr/src/kodekloudrepos/media

git status

# find the commit by its message, without switching branches to hunt for it
git log --all --oneline --grep="Update info.txt"
# 8855791 Update info.txt
```

```bash
git checkout master
git log --oneline
# 1aae358 (HEAD -> master, origin/master) Add welcome.txt
# ae51bff initial commit

git cherry-pick 8855791

git log --oneline
# afc2a9c (HEAD -> master) Update info.txt      <- new commit, new hash
# 1aae358 (origin/master) Add welcome.txt
# ae51bff initial commit

git push origin master
```

## Why

| Command | Why |
|:--|:--|
| `git log --all --oneline --grep="..."` | Finds the commit by its message without checking out `feature` to hunt for it. |
| `git checkout master` | Cherry-pick applies onto whatever branch you are standing on. |
| `git cherry-pick <hash>` | Replays that one commit. The rest of `feature`, including the unfinished work, stays behind. |
| `git push origin master` | `origin/master` only catches up once you push. |

## Notes

- It is `git cherry-pick`, hyphenated. `git cherry pick <hash>` runs the real `git cherry` command and fails with `fatal: unknown commit pick`, which reads like a bad hash.
- Cherry-pick makes a **new commit**: `abdabb7` on feature became `9decd8e` on master. A commit's identity includes its parent.
- The WIP commit on `feature` did not follow. That is the difference from merging the whole branch.
- On a conflict: fix, `git add`, `git cherry-pick --continue`. `--abort` puts it back.
