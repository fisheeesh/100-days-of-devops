# Day 31: Git Stash

## Question

The Nautilus application development team was working on a git repository
`/usr/src/kodekloudrepos/media` present on `Storage server` in `Stratos DC`. One of the developers
stashed some in-progress changes in this repository, but now they want to restore some of the
stashed changes. Find below more details to accomplish this task:

Look for the stashed changes under `/usr/src/kodekloudrepos/media` git repository, and restore the
stash with `stash@{1}` identifier. Further, commit and push your changes to the origin.

## Solution

```bash
ssh natasha@ststor01

sudo -i
cd /usr/src/kodekloudrepos/media

git status
git log --oneline

git stash list
# stash@{0}: On master: wip two
# stash@{1}: On master: wip one     <- the one the task names
# stash@{2}: On master: wip zero
```

```bash
git stash apply stash@{1}
# Changes not staged for commit:
#   modified:   a.txt

git status --short
#  M a.txt        <- restored into the working tree, but NOT staged

git add .
git commit -m "apply stash@{1}"

git push origin master
```

## Why

| Command | Why |
|:--|:--|
| `git stash list` | Stashes are addressed by position, newest first. The task names `stash@{1}`. |
| `git stash apply stash@{1}` | Restores the changes and **leaves the stash in the list**. `pop` would apply and delete it. |
| `git add .` | The step that is easy to skip. `apply` restores changes **unstaged**. |
| `git push origin master` | The task asks for it explicitly. |

## Notes

- Committing straight after `apply` does nothing: git says `no changes added to commit` and creates no commit, so the task silently stays unfinished. `git commit -a -m` is the one-step version for tracked files.
- Stash numbers are **positions, not ids**. Drop one and everything below renumbers, so re-run `git stash list` between operations.
- `git stash show -p stash@{1}` prints the diff without applying it.
- `git stash save "msg"` is deprecated. Use `git stash push -m "msg"`.
