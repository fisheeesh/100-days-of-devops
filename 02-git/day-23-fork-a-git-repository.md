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
| **Fork** | Your own copy of someone else's repo, under your account. You can push to it; the original stays untouched. | On the Git server |
| **Clone** (Day 22) | A copy of a repo on a machine, with its history on disk. | On your machine |
| **Branch** | A separate line of work inside one repo. No second repo, no second owner. | Inside a repo |

## Notes

- Forking happens entirely on the server. Nothing is downloaded. Working on it means cloning your fork afterwards.
- Gitea records where the fork came from, which is what lets you open a pull request back to `sarah/story-blog` later (Day 29).
- `jon` can push to `jon/story-blog` but not to `sarah/story-blog`. That is the whole point: forking is how you contribute to a repo you do not have write access to.
- A fork is a snapshot at that moment and does not follow the original. To pick up later changes you add the original as a second remote, conventionally named `upstream`, and fetch from it.
