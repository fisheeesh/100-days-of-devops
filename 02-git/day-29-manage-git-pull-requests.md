# Day 29: Manage Git Pull Requests

## Question

`Max` want to push some new changes to one of the repositories but we don't want people to push
directly to `master` branch, since that would be the final version of the code. It should always
only have content that has been reviewed and approved. We cannot just allow everyone to directly
push to the master branch. So, let's do it the right way as discussed below:

SSH into `storage server` using user `max`, password `Max_pass123`. There you can find an already
cloned repo under `Max` user's home.

Max has written his story about The Fox and Grapes

Max has already pushed his story to remote git repository hosted on `Gitea` branch
`story/fox-and-grapes`

Check the contents of the cloned repository. Confirm that you can see Sarah's story and history of
commits by running `git log` and validate author info, commit message etc.

Max has pushed his story, but his story is still not in the `master` branch. Let's create a Pull
Request(PR) to merge Max's `story/fox-and-grapes` branch into the `master` branch

Click on the `Gitea UI` button on the top bar. You should be able to access the `Gitea` page.

UI login info:

- Username: `max`
- Password: `Max_pass123`

PR title : `Added fox-and-grapes story`

PR pull from branch: `story/fox-and-grapes` (source)

PR merge into branch: `master` (destination)

Before we can add our story to the `master` branch, it has to be reviewed. So, let's ask `tom` to
review our PR by assigning him as a reviewer

Add tom as reviewer through the Git Portal UI

- Go to the newly created PR
- Click on Reviewers on the right
- Add tom as a reviewer to the PR

Now let's review and approve the PR as user `Tom`

Login to the portal with the user `tom`

Logout of `Git Portal UI` if logged in as `max`

UI login info:

- Username: `tom`
- Password: `Tom_pass123`

PR title : `Added fox-and-grapes story`

Review and merge it.

> `Note:` For these kind of scenarios requiring changes to be done in a web UI, please take
> screenshots so that you can share it with us for review in case your task is marked incomplete.
> You may also consider using a screen recording software such as loom.com to record and share
> your work.

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
| `git branch -a` | Confirms the story branch exists on the remote. If it does not, there is nothing to open a PR from. |
| `git log -1 --format='%an <%ae>%n%s'` | Author, email and subject directly, which is the "validate author info" step. |

## Notes

- A pull request is a **forge** feature, not a git one. What Gitea runs at the end is an ordinary merge on the server.
- Direction is what people get backwards: base is `master` (destination), compare is `story/fox-and-grapes` (source).
- The reviewer step is the point of the task. Nobody pushes to master, so review is the gate.
- Day 23 was the other route: fork when you have no write access. Here max can push a branch to the same repo.
