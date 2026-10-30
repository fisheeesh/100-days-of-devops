# Day 32: Git Rebase

## Question

The Nautilus application development team has been working on a project repository `/opt/beta.git`.
This repo is cloned at `/usr/src/kodekloudrepos` on `storage server` in `Stratos DC`. They recently
shared the following requirements with the DevOps team:

One of the developers is working on the `feature` branch and their work is still in progress,
however there are some changes which have been pushed into the `master` branch. The developer now
wants to `rebase` the `feature` branch with the `master` branch without losing any data from the
`feature` branch, and they do not want to add any `merge commit` by simply merging `master` into
`feature`. Accomplish this task as per the requirements mentioned.

Also remember to push your changes once done.

## Solution

```bash
ssh natasha@ststor01

sudo -i
cd /usr/src/kodekloudrepos/beta

git branch
git status

git log --oneline
# 5a1279d (HEAD -> feature, origin/feature) Add new feature
# 4b4e768 initial commit

git checkout master
git log --oneline
# 9b44099 (origin/master, master) Update info.txt
# 4b4e768 initial commit
# master has a commit feature has never seen

git checkout feature
```

```bash
git remote -v
git fetch origin master

git rebase origin/master
# Successfully rebased and updated refs/heads/feature.

git log --oneline
# 0d75832 (HEAD -> feature) Add new feature     <- same work, new hash
# 9b44099 (origin/master, master) Update info.txt
# 4b4e768 initial commit
```

The push is refused, which is expected after a rebase:

```bash
git push -u origin feature
# ! [rejected]        feature -> feature (non-fast-forward)

git push -u origin feature -f
# + 5a1279d...0d75832 feature -> feature (forced update)
```

## Why

| Command | Why |
|:--|:--|
| `git fetch origin master` | Rebasing onto a stale `origin/master` replays the commits onto an old base. |
| `git rebase origin/master` | Replays `feature`'s commits on top of master's tip. The result is a straight line, which is what "no merge commit" means. |
| `git push -f` | The rebase rewrote the commits, so the remote's branch and yours no longer share a tip. |

## Rebase vs merge

Merging master into feature works too, but it creates a merge commit, which the task forbids.
Rebase lifts your commits off and replays them on the new base instead.

Confirmed in a test run: `git log --merges` counted **0**, the feature file was still there, and
the commit's hash changed from `0de211f` to `0d75832`. Same change, new commit, because a commit's
identity includes its parent (same reason as the cherry-pick on Day 28).

## Notes

- The rejected push is not a mistake. Your branch was rewritten, so git sees the two as diverged.
- `--force-with-lease` is the safer force: it refuses if someone else pushed since your last fetch.
- Conflicts stop the rebase partway: fix, `git add`, `git rebase --continue`. `--abort` restores the branch exactly.
- `git rebase master` uses your local master, which may be behind. `origin/master` after a fetch is the safer habit.
