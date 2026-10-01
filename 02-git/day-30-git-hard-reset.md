# Day 30: Git hard reset

## Question

The Nautilus application development team was working on a git repository
`/usr/src/kodekloudrepos/cluster` present on `Storage server` in `Stratos DC`. This was just a
test repository and one of the developers pushed a couple of changes for testing, but now they
want to clean this repository along with the commit history/work tree, so they want to point back
the `HEAD` and the branch itself to a commit with message `add data.txt file`. Find below more
details:

1. In `/usr/src/kodekloudrepos/cluster` git repository, reset the git commit history so that there
   are only two commits in the commit history, i.e. `initial commit` and `add data.txt file`.
2. Also make sure to push your changes.

## Solution

```bash
ssh natasha@ststor01

sudo -i
cd /usr/src/kodekloudrepos/cluster

git log --oneline
# 4c39589 (HEAD -> master, origin/master) Test Commit10
# 4d14560 Test Commit9
# ...
# e3233fc Test Commit1
# ce2664f add data.txt file      <- land here
# 0a1284b initial commit

git reset --hard ce2664f

git log --oneline
# ce2664f (HEAD -> master) add data.txt file
# 0a1284b initial commit
```

The push is refused, and that is expected:

```bash
git push origin master
# ! [rejected]        master -> master (non-fast-forward)
# error: failed to push some refs to '/opt/cluster.git'
# hint: Updates were rejected because the tip of your current branch is behind
# hint: its remote counterpart. If you want to integrate the remote changes,
# hint: use 'git pull' before pushing again.

# pulling would drag the deleted commits straight back, so force instead
git push -f origin master
# + 4c39589...ce2664f master -> master (forced update)
```

## Why

| Command | Why |
|:--|:--|
| `git log --oneline` | Gets the hash of `add data.txt file`, since the task names the commit by message, not by id. |
| `git reset --hard <hash>` | Moves the branch pointer, the staging area and the working tree to that commit in one go. The later commits become unreachable and the files they added are deleted. |
| `git push origin master` (the failed one) | Worth running, because the rejection explains itself. The remote still holds the extra commits, so your branch is not ahead of it and git refuses to lose them. |
| `git push -f origin master` | Replaces the remote branch with yours. Following the hint and pulling would restore exactly the commits you just deleted. |

## The three reset modes

What each one throws away, confirmed by running them:

| Mode | Commits | Staging area | Files on disk |
|:--|:--|:--|:--|
| `--soft` | removed | changes kept, **staged** | untouched |
| `--mixed` (default) | removed | changes kept, **unstaged** | untouched |
| `--hard` | removed | wiped | wiped |

`--soft` cancels the commits but keeps the file changes from them, left staged, so a single
`git commit` re-records the lot as one commit. That is how you squash several commits into one.
`--hard` is the only one that also clears the work tree, which is what this task asks for.

In the test run the extra file showed as `A junk.txt` (staged) after `--soft`, as `?? junk.txt`
(untracked) after `--mixed`, and was gone entirely after `--hard`.

## Notes

- The rejected push is git protecting a shared branch from losing commits, not a mistake on your part. "Behind its remote counterpart" is literally true: the remote has commits you no longer do.
- `git push --force-with-lease` is the safer force. It refuses if someone else pushed since your last fetch, where `-f` overwrites no matter what.
- Force pushing rewrites shared history. Anyone who already pulled still has the old commits and will push them back unless they reset too. Fine on a test repo like this, risky on a real branch.
- Reset is not revert (Day 27). Revert adds a new commit that undoes an old one and keeps the history; reset erases it. This task explicitly wanted the history gone.
- Escape hatch: `git reflog` still lists the discarded commits for a couple of weeks, so `git reset --hard <old-sha>` brings them back right up until git garbage collects them.
