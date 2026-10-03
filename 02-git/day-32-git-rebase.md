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
| `git checkout master` then back | Just to see what `master` has that `feature` does not. It is the reason a rebase is being asked for at all. |
| `git fetch origin master` | Updates `origin/master` from the server. Rebasing onto a stale `origin/master` would replay the commits onto an old base. |
| `git rebase origin/master` | Replays `feature`'s commits on top of `master`'s tip, one at a time. The history ends up as a straight line, which is what "no merge commit" means. |
| `git push -f` | A rebase rewrites the feature commits, so the remote's version and yours no longer share a tip. Force replaces the remote branch with the rebuilt one. |

## Rebase vs merge

Merging `master` into `feature` would have worked too, but it creates a merge commit joining the
two lines of history, which the task forbids. Rebase instead lifts your commits off and replays
them on the new base, leaving one straight line and no merge commit.

Confirmed in a test run: after the rebase, `git log --merges` counted **0** merge commits, the
feature file was still present, and the feature commit's hash changed from `0de211f` to `0d75832`.
Same change, new commit, because a commit's identity includes its parent (the same reason a
cherry-pick gets a new hash on Day 28).

## Notes

- "Without losing any data" is satisfied by rebase itself. Nothing is dropped, the commits are rebuilt on a newer base.
- The rejected push is not a mistake. Your `feature` was rewritten, so git sees the remote branch and yours as having diverged and refuses to clobber it.
- `git push --force-with-lease` is the safer force, as on Day 30. It refuses if someone else pushed to `feature` since your last fetch. Plain `-f` overwrites regardless.
- Force pushing a rebased branch is normal and expected for a personal feature branch. It is rude on a branch other people are working on, because their clones still hold the old commits.
- Conflicts stop the rebase partway: fix the files, `git add` them, then `git rebase --continue`. `git rebase --abort` puts the branch back exactly as it was.
- `git rebase origin/master` and `git rebase master` differ. The first uses what the server has, the second uses your local `master`, which may be behind. Fetching first and rebasing onto `origin/master` is the safer habit.
- `git log --oneline --graph --all` draws the branches, which makes "straight line" versus "merge" obvious at a glance.
