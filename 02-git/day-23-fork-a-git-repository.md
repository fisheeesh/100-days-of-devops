# Day 23: Fork a Git Repository

## Question

There is a Git server utilized by the Nautilus project teams. Recently a new developer named Jon
joined the team and needs to begin working on a project. To begin, he must fork an existing Git
repository. Follow the steps below:

1. Click on the `Gitea` UI button located on the top bar to access the Gitea page.
2. Login to `Gitea` server using username `jon` and password `Jon_pass123`.
3. Once logged in, locate the Git repository named `sarah/story-blog` and `fork` it under the
   `jon` user.

> **Note:** For tasks requiring web UI changes, screenshots are necessary for review purposes.

## Solution

No terminal work on this one. In the Gitea web UI:

1. Open Gitea from the button in the top bar.
2. Sign in as `jon` / `Jon_pass123`.
3. Find `sarah/story-blog` (search, or go straight to `/sarah/story-blog`).
4. Click **Fork**, top right of the repo page.
5. Set the owner to `jon`, leave the name as `story-blog`, confirm.
6. You land on `jon/story-blog`, showing **forked from sarah/story-blog** under the title.
7. Screenshot it, since there is no command output to prove the work.

## Why

| | What it is | Where it lives |
|:--|:--|:--|
| **Fork** | Your own copy of someone else's repo. You can push to it, the original stays untouched. | On the Git server |
| **Clone** (Day 22) | A copy of a repo with its history on disk. | On your machine |
| **Branch** | A separate line of work inside one repo. | Inside a repo |

## Notes

- Forking happens entirely on the server. Nothing is downloaded; you clone your fork afterwards.
- `jon` can push to `jon/story-blog` but not `sarah/story-blog`. That is the whole point of forking.
- A fork is a snapshot and does not follow the original. Add the original as `upstream` to pull later changes.
