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
| `sudo` on every git command | The repo belongs to root and you are natasha. Same situation as Day 22, where the fix was `safe.directory`. Either works: `sudo` runs git as the owner, `safe.directory` tells git to trust a path it does not own. |
| `git branch -a` | Lists local and remote-tracking branches, so you see what is already there before adding to it. `-a` means all. |
| `git checkout master` | A new branch starts wherever `HEAD` currently is. Landing on `master` first is what makes it branch **from master**, which is the actual requirement. |
| `git checkout -b <name>` | Creates the branch and switches onto it in one step. |
| `git branch -v` | Shows the commit each branch points at. Matching SHAs are the proof, rather than trusting that the command did the right thing. |

## Notes

- Creating a branch changes nothing in the working tree. `git status` stays clean, which is how requirement 2 is satisfied without thinking about it. A branch is only a movable pointer to a commit, which is why this is instant even on a huge repo.
- Same result, fewer steps, from whatever branch you happen to be on:
  - `sudo git checkout -b xfusioncorp_official master`
  - `sudo git switch -c xfusioncorp_official master` (git 2.23+, `switch` only changes branches, while `checkout` also does file restores)
  - `sudo git branch xfusioncorp_official master` creates it without switching to it
- You are left standing on the new branch. Harmless here. `sudo git checkout master` puts the repo back the way you found it if you would rather leave it on master.
- The branch is local to this repo. Publishing it would need `git push origin xfusioncorp_official`, which this task does not ask for.
