# Day 24: Git Create Branches

## Question

Nautilus developers are actively working on one of the project repositories,
`/usr/src/kodekloudrepos/official`. Recently, they decided to implement some new features in the
application, and they want to maintain those new changes in a separate branch. Below are the
requirements that have been shared with the DevOps team:

1. On `Storage server` in Stratos DC create a new branch `xfusioncorp_official` from `master`
   branch in `/usr/src/kodekloudrepos/official` git repo.
2. Please do not try to make any changes in the code.

## Solution

```bash
ssh natasha@ststor01

cd /usr/src/kodekloudrepos/official

# the repo is root-owned, so every git command needs sudo
sudo git branch -a
# * master

sudo git checkout master
sudo git checkout -b xfusioncorp_official
# Switched to a new branch 'xfusioncorp_official'
```

Verify:

```bash
sudo git branch -v
#   master               e143ac5 second
# * xfusioncorp_official e143ac5 second
#   same commit on both, so it really did branch from master

sudo git status --short
# no output: creating a branch touched nothing
```

## Why

| Command | Why |
|:--|:--|
| `git branch -a` | What already exists, local and remote-tracking. |
| `git checkout master` | A new branch starts at `HEAD`. Landing on master is what makes it branch **from master**. |
| `git branch -v` | Shows the commit each branch points at. Matching SHAs are the proof. |

## Notes

- Creating a branch touches no files, so `git status` stays clean. That is requirement 2 satisfied.
- One step instead of two: `git checkout -b xfusioncorp_official master`, or `git switch -c ...` on git 2.23+.
- The repo is root-owned, so every git command needs `sudo` (or `safe.directory`, Day 22).
- The branch is local. Publishing it would need `git push origin xfusioncorp_official`.
