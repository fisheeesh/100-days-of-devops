# Day 27: Git Revert Some Changes

## Question

The Nautilus application development team was working on a git repository
`/usr/src/kodekloudrepos/ecommerce` present on `Storage server` in `Stratos DC`. However, they
reported an issue with the recent commits being pushed to this repo. They have asked the DevOps
team to revert repo HEAD to last commit. Below are more details about the task:

1. In `/usr/src/kodekloudrepos/ecommerce` git repository, revert the latest commit `( HEAD )` to
   the previous commit (JFYI the previous commit hash should be with `initial commit` message).
2. Use `revert ecommerce` message (please use all small letters for commit message) for the new
   revert commit.

## Solution

```bash
ssh natasha@ststor01

# root shell, so the root-owned repo needs no sudo on each command
sudo -i
cd /usr/src/kodekloudrepos/ecommerce

git status
git log --oneline
# 042124e add feature      <- HEAD, the commit to undo
# 35e7c3a initial commit   <- the state we want the files back in
```

```bash
# apply the inverse of HEAD and stage it, but do not commit yet
git revert HEAD -n

git status
# M  app.txt      (staged, ready to commit)

git add .
git commit -m "revert ecommerce"

git log --oneline
# 50df346 revert ecommerce
# 042124e add feature
# 35e7c3a initial commit
git status
```

Check the files really are back to the initial commit:

```bash
git diff 35e7c3a HEAD
# no output means the two trees are identical
```

## Why

| Command | Why |
|:--|:--|
| `git log --oneline` | The two hashes that matter: `HEAD` and the `initial commit` you are returning to. |
| `git revert HEAD -n` | Applies the **inverse** of HEAD and stages it without committing. `-n` is `--no-commit`. |
| `git commit -m "revert ecommerce"` | Why `-n` was used. The task demands an exact lowercase message. |
| `git diff <initial> HEAD` | Empty output means the files match the initial commit. That is the real proof. |

## Notes

- **`-m` is not the message flag on revert.** It is `--mainline`, for merge commits, and fails with `option 'mainline' expects a number greater than zero`.
- revert is not reset. Revert adds an undo commit and keeps the history; reset erases it. All three commits stay in the log.
- On a conflict: fix the files, `git add` them, then commit with the required message.
