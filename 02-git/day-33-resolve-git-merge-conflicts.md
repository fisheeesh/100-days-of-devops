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
| `git push origin master` | Reproduces the problem instead of guessing. The `fetch first` rejection proves that the remote contains work missing from Max's local branch. |
| `git pull --rebase origin master` | Fetches the latest remote `master`, places Max's local commit aside, updates to Sarah's commit, and then replays Max's commit on top. This integrates both histories without creating a merge commit. |
| `git status` | During the stopped rebase, it identifies `story-index.txt` as unmerged and tells you the exact recovery command to run next. |
| Editing `story-index.txt` | Git cannot decide which titles to keep. The correct resolution combines both contributors' work, fixes `Mooose`, and removes the marker lines. |
| `git add story-index.txt` | Tells Git that the file has been resolved. Merely saving the file does not clear its unmerged state. |
| `git rebase --continue` | Resumes the paused rebase and rebuilds Max's commit using the resolved file. A separate `git commit` is unnecessary in this rebase workflow. |
| `git push origin master` | Publishes the successfully rebased `master`. This push is now a fast-forward from the remote's point of view. |
| `git show origin/master:story-index.txt` | Reads the file from the remote-tracking branch after the push, confirming what Git recorded rather than only checking the working-tree copy. |

## Notes

- The rejected push is Git protecting Sarah's commit from being overwritten. Do not force-push here; the requirement is to preserve both developers' work.
- `add/add` means both sides independently added a file at the same path with different content. Git has no shared earlier version it can use to choose between them.
- Conflict markers come as a set: `<<<<<<<`, `=======`, and `>>>>>>>`. All three must be removed. Leaving even the separator line behind produces an invalid final file and can fail validation.
- Do not blindly choose only "ours" or "theirs." Sarah's version has the corrected spelling, while Max's version adds the fourth title. The required result is a deliberate combination of both.
- During a rebase, the meaning of `ours` and `theirs` can feel reversed because Max's commit is being replayed on top of the updated remote branch. Editing the file directly is clearer for this conflict.
- Seeing the same conflict in `git status` immediately after editing is normal. Git considers it unresolved until `git add story-index.txt` records the resolution.
- If the resolution goes wrong, `git rebase --abort` restores the repository to exactly where it was before `git pull --rebase`, allowing you to start again safely.
- The typo and all four titles belong in `story-index.txt`; editing `story-blog.txt` is not part of the required resolution.
