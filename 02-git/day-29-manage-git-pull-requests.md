# Day 29: Manage Git Pull Requests

## Question

`Max` wants to push some new changes to one of the repositories, but we don't want people pushing
directly to `master`, since that is the final version of the code. It should only ever hold
content that has been reviewed and approved. So let's do it the right way:

1. SSH into `storage server` as user `max` / `Max_pass123`. An already cloned repo sits in max's
   home. Max has written his story about The Fox and Grapes and pushed it to the Gitea branch
   `story/fox-and-grapes`.
2. Check the contents of the cloned repo. Confirm you can see the story and the commit history
   with `git log`, and validate the author info, commit message and so on.
3. His story is not in `master` yet. Open a Pull Request to merge `story/fox-and-grapes` into
   `master`, using the Gitea UI (login `max` / `Max_pass123`).
   - PR title: `Added fox-and-grapes story`
   - Pull from (source): `story/fox-and-grapes`
   - Merge into (destination): `master`
4. Add `tom` as a reviewer on the PR.
5. Log out, log in as `tom` / `Tom_pass123`, then review, approve and merge the PR.

> **Note:** For scenarios requiring web UI changes, take screenshots so you can share them for
> review in case the task is marked incomplete.

## Solution

On the box, just look around. Nothing here changes anything:

```bash
ssh max@ststor01            # password: Max_pass123
cd ~/story-blog

git branch -a
# * master
#   remotes/origin/master
#   remotes/origin/story/fox-and-grapes

git log --oneline

# the author details the task asks you to confirm
git log -1 --format='%an <%ae>%n%s' origin/story/fox-and-grapes
```

Everything else is in the Gitea UI:

1. Open Gitea from the top bar, sign in as `max`.
2. Open the repo, **Pull Requests**, **New Pull Request**.
3. Set the direction: merge **into** `master`, pull **from** `story/fox-and-grapes`.
4. Title it `Added fox-and-grapes story`, create the PR.
5. On the PR page, **Reviewers** on the right, add `tom`.
6. Sign out. Sign in as `tom` / `Tom_pass123`.
7. Open the same PR, review the changes, **Approve**, then **Merge Pull Request**.
8. Screenshot each step, since there is no command output to show for any of it.

## Why

| Command | Why |
|:--|:--|
| `git branch -a` | Shows the story branch exists on the remote (`remotes/origin/story/fox-and-grapes`). If it is not there, max never pushed and there is nothing to open a PR from. |
| `git log --oneline` | Confirms the history is what the task describes before you go near the UI. |
| `git log -1 --format='%an <%ae>%n%s'` | Prints author name, email and subject directly, which is exactly the "validate author info, commit message" step, rather than eyeballing full log output. |

## Notes

- A pull request is a **forge** feature (Gitea, GitHub, GitLab), not a git one. There is no `git pull-request` command. What Gitea runs at the end is an ordinary git merge, performed on the server.
- The direction is the part people get backwards. Base is `master` (destination), compare is `story/fox-and-grapes` (source). Swapping them proposes merging master into the story branch, which is not what is wanted.
- The reviewer step is the actual point of the task. The rule is nobody pushes to `master` directly, so the review is the gate the change has to pass. On a real protected branch max could not approve his own PR.
- Gitea offers merge commit, rebase and squash. A plain merge commit is the right default here.
- After the merge, `master` on the server holds the story. The clone in max's home is still behind until `git pull`.
- Day 23 was the other route into a repo: fork it when you have no write access, then open a PR from the fork. Here max can push a branch to the same repo, so no fork is needed.
