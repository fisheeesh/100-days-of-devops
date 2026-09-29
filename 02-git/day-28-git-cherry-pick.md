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
| `git log --all --oneline --grep="..."` | Searches commit messages across every branch, so you get the hash straight from the task's description without checking out `feature` to look for it. |
| `git checkout master` | Cherry-pick applies a commit **onto whatever branch you are standing on**. Being on master is what puts the change there. |
| `git cherry-pick <hash>` | Replays that one commit's changes onto the current branch. Everything else on `feature`, including the unfinished work, stays behind. |
| `git log --oneline` afterwards | Confirms the commit landed and, just as importantly, that nothing else tagged along. |
| `git push origin master` | The task asks for it, and `origin/master` in the log only catches up once you push. |

## Notes

- The command is `git cherry-pick`, hyphenated. Typing `git cherry pick <hash>` runs a **different real command**, `git cherry`, which fails with `fatal: unknown commit pick`. The error points at the word `pick`, so it reads like a bad hash when the problem is the missing hyphen.
- Cherry-pick creates a **new commit**. In a test run, `abdabb7` on `feature` became `9decd8e` on `master` with the same change. A commit's identity includes its parent, so replaying it somewhere else produces a different hash. The same change now exists as two separate commits.
- The in-progress `WIP` commit on `feature` did not follow. That is exactly the difference between cherry-picking one commit and merging the whole branch.
- `feature` is untouched and still holds the original commit. Git copes when that branch is merged later.
- `-x` appends `(cherry picked from commit <full sha>)` to the message. Worth using on a shared repo so the copy can be traced back.
- On a conflict: fix the files, `git add` them, then `git cherry-pick --continue`. `git cherry-pick --abort` puts everything back.
- `-n` stages the change without committing, the same idea as the revert on Day 27.
- The repos under `/usr/src/kodekloudrepos` are root-owned, so `sudo -i` saves prefixing every command (Day 27), or use `safe.directory` (Day 22).
