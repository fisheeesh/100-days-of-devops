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
| `sudo -i` | A root shell. The repo belongs to root, so this avoids both prefixing every command with sudo and the dubious-ownership refusal from Day 22. |
| `git log --oneline` | Gives you the two hashes that matter: `HEAD`, the commit to undo, and the `initial commit` the task names as the target state. |
| `git revert HEAD -n` | Applies the **inverse** of HEAD to the working tree and stages it without committing. `-n` is `--no-commit`. Without it git opens an editor and writes its own `Revert "..."` message. |
| `git add .` | Already staged by the revert, so this is redundant, but harmless. It earns its place when a revert stops on a conflict and leaves something unstaged. |
| `git commit -m "revert ecommerce"` | The reason `-n` was used. The task demands an exact lowercase message, and this is the only way to set it. |
| `git diff <initial> HEAD` | The real proof. Empty output means the files now match the initial commit, which is what "revert to the previous commit" actually means. |

## Notes

- **`-m` is not the message flag on `git revert`.** It is `--mainline`, the parent number to pick when reverting a merge commit. `git revert -m "revert ecommerce" HEAD` fails with `option 'mainline' expects a number greater than zero`. That is exactly why the `-n` then `commit -m` route exists.
- **revert is not reset.** `revert` adds a new commit that undoes an older one, so nothing leaves the history and anyone who already pulled is unaffected. `reset --hard` moves the branch pointer backwards and discards commits, which rewrites history and breaks every other clone. The task says "revert repo HEAD to last commit", but it wants the safe one.
- After this the log still holds all three commits. The bad commit stays as a record; the undo sits on top of it.
- If git answers `Please tell me who you are`, set an identity for root first: `git config --global user.email "you@example.com"` and the matching `user.name`.
- If later commits touched the same lines, the revert stops on a conflict. Fix the files, `git add` them, then commit with the required message. `-n` makes that flow natural, since you were committing by hand anyway.
- The task does not ask you to push.
