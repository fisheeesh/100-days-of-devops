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
| `git stash list` | Stashes are addressed by position, newest first. The task names `stash@{1}`, so you need the list in front of you before touching anything. |
| `git stash apply stash@{1}` | Restores that stash's changes into the working tree and **leaves the stash in the list**. `pop` would apply it and delete it. |
| `git add .` | The step that is easy to skip. `apply` restores changes **unstaged**, so without this the commit has nothing to record. |
| `git commit -m "..."` | Records the restored work. |
| `git push origin master` | The task asks for it explicitly. |

## Notes

- Running `git commit -m "..."` straight after `apply` does nothing. Git answers `no changes added to commit (use "git add" and/or "git commit -a")` and creates no commit, so the task silently stays unfinished. `git commit -a -m "..."` is the one-step version for files git already tracks; brand new files still need `git add`.
- `apply` keeps the stash, `pop` applies and drops it. When a task names a particular stash, `apply` is the safer choice: if the result is not what you wanted, the stash is still sitting there.
- Stash numbers are **positions, not ids**. Drop or pop one and everything underneath renumbers. After dropping `stash@{1}`, the old `stash@{2}` becomes the new `stash@{1}`. Re-run `git stash list` between operations instead of trusting numbers you read earlier.
- `git stash apply --index` also restores which files were staged at the time. Without it everything comes back unstaged, which is exactly what happened here.
- `git stash show -p stash@{1}` prints the diff without applying it, so you can confirm you picked the right one first.
- `git stash save "message"` is deprecated. The current form is `git stash push -m "message"`.
- If your shell complains about the braces, quote it: `git stash apply 'stash@{1}'`.
