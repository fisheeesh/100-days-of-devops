# Day 33: Resolve Git Merge Conflicts

## Question

Sarah and Max were working on writing some stories which they have pushed to the repository. Max
has recently added some new changes and is trying to push them to the repository, but he is facing
some issues. Below you can find more details:

SSH into `Storage Server` using user `max` and password `Max_pass123`. Under `/home/max` you will
find the `story-blog` repository. Try to push the changes to the origin repo and fix the issues.
The `story-index.txt` file must have titles for all four stories. Additionally, there is a typo in
the `The Lion and the Mooose` line where `Mooose` should be `Mouse`.

Click on the `Gitea UI` button on the top bar. You should be able to access the Gitea page. You can
log in using username `sarah` and password `Sarah_pass123`, or username `max` and password
`Max_pass123`.

> **Note:** For scenarios requiring changes in a web UI, take screenshots for review in case the
> task is marked incomplete. A screen recording can also be used to document the work.

## Solution

Connect as Max and inspect the repository before changing anything:

```bash
ssh max@ststor01
# password: Max_pass123

cd /home/max/story-blog

git status
git remote -v
git log --oneline --decorate --graph --all -5
```

Try the requested push first. It is rejected because Sarah has already pushed a commit that Max's
local `master` does not contain:

```bash
git push origin master
# ! [rejected]        master -> master (fetch first)
# error: failed to push some refs to 'http://gitea:3000/sarah/story-blog.git'
```

Fetch Sarah's work and replay Max's local commit on top of it:

```bash
git pull --rebase origin master
# Auto-merging story-index.txt
# CONFLICT (add/add): Merge conflict in story-index.txt
# error: could not apply <commit>... <message>
```

Git pauses the rebase. Inspect the conflict before resolving it:

```bash
git status
# interactive rebase in progress
# both added: story-index.txt

cat story-index.txt
```

The file contains two competing versions separated by conflict markers:

```text
<<<<<<< HEAD
1. The Lion and the Mouse
2. The Frogs and the Ox
3. The Fox and the Grapes
=======
1. The Lion and the Mooose
2. The Frogs and the Ox
3. The Fox and the Grapes
4. The Donkey and the Dog
>>>>>>> <max-commit>
```

Edit the file and combine the valid content from both versions:

```bash
vi story-index.txt
```

The finished `story-index.txt` must contain exactly the four story titles, with no conflict
markers and with the typo corrected:

```text
1. The Lion and the Mouse
2. The Frogs and the Ox
3. The Fox and the Grapes
4. The Donkey and the Dog
```

Check the resolved content before staging it:

```bash
cat story-index.txt

grep -nE '^(<<<<<<<|=======|>>>>>>>)' story-index.txt
# no output: all conflict markers are gone

grep -cE '^[0-9]+\. ' story-index.txt
# 4

grep -n 'Mooose' story-index.txt
# no output

grep -n 'The Lion and the Mouse' story-index.txt
# 1:1. The Lion and the Mouse
```

Mark the conflict as resolved and continue the paused rebase:

```bash
git add story-index.txt

git status
# all conflicts fixed: run "git rebase --continue"

git rebase --continue
# keep the existing commit message if an editor opens, then save and exit
# Successfully rebased and updated refs/heads/master.
```

Push the rebased branch and verify it:

```bash
git push origin master

git status
# On branch master
# Your branch is up to date with 'origin/master'.
# nothing to commit, working tree clean

git log --oneline --decorate --graph --all -5
git show origin/master:story-index.txt
```

Finally, open Gitea and verify the result:

1. Sign in as `max` / `Max_pass123` or `sarah` / `Sarah_pass123`.
2. Open `sarah/story-blog`.
3. Open `story-index.txt` and confirm that all four titles are present and `Mouse` is spelled
   correctly.
4. Confirm that the latest commit is visible, then take a screenshot for review.

## Why

| Command | Why |
|:--|:--|
| `git push origin master` (the failed one) | Reproduces the problem. `fetch first` proves the remote holds work Max does not have. |
| `git pull --rebase origin master` | Replays Max's commit on top of Sarah's, integrating both without a merge commit. |
| Editing `story-index.txt` | Git cannot pick between two independently added files. The resolution keeps both names and fixes `Mooose`. |
| `git add story-index.txt` | Marks it resolved. Saving the file alone does not clear the unmerged state. |
| `git rebase --continue` | Rebuilds the commit with the resolved file. No separate `git commit` needed. |

## Notes

- The rejected push is git protecting Sarah's commit. Do not force here: the task needs both developers' work.
- `add/add` means both sides added the same path with different content, so git has no common version to fall back on.
- Remove all three markers: `<<<<<<<`, `=======`, `>>>>>>>`. A leftover separator makes the file invalid.
- Do not take only "ours" or "theirs". Sarah has the spelling fix, Max has the fourth title. The answer is both.
- `git rebase --abort` puts everything back exactly as it was if the resolution goes wrong.
