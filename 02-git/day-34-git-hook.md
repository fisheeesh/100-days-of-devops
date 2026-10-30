# Day 34: Git Hook

## Question

The Nautilus application development team was working on a Git repository `/opt/beta.git`, which
is cloned under the `/usr/src/kodekloudrepos` directory on `Storage Server` in `Stratos DC`. The
team wants to set up a hook on this repository. Find the details below:

- Merge the `feature` branch into the `master` branch, but complete the next requirement before
  pushing the changes.
- Create a `post-update` hook in this Git repository so that whenever changes are pushed to the
  `master` branch, it creates a release tag named `release-YYYY-MM-DD`, using the current date. For
  example, if today is June 20, 2023, the tag must be `release-2023-06-20`. Test the hook at least
  once and create a tag for today's release.
- Finally, push the changes.

> **Note:** Perform this task using the `natasha` user, and do not alter the repository or existing
> directory permissions.

## Solution

Connect to the storage server as the required user and inspect the bare repository:

```bash
ssh natasha@ststor01

cd /opt/beta.git

git rev-parse --is-bare-repository
# true

git show-ref
# ... refs/heads/master
# ... refs/heads/feature

ls -lah hooks
# includes post-update.sample
```

Create the server-side hook with the exact name `post-update`:

```bash
cd /opt/beta.git/hooks
vi post-update
```

Put this script in the file. The shebang must be its first line:

```sh
#!/bin/sh

for ref in "$@"; do
    if [ "$ref" = "refs/heads/master" ]; then
        tag_name="release-$(date +%Y-%m-%d)"

        if ! git show-ref --verify --quiet "refs/tags/$tag_name"; then
            git tag "$tag_name" "$ref"
        fi
    fi
done

exit 0
```

Make the hook executable and check its syntax without running it:

```bash
chmod +x /opt/beta.git/hooks/post-update

sh -n /opt/beta.git/hooks/post-update
# no output: shell syntax is valid

ls -l /opt/beta.git/hooks/post-update
# -rwxr-xr-x ... natasha natasha ... post-update
```

Move to the working clone, verify that it is clean, and merge `feature` into `master`:

```bash
cd /usr/src/kodekloudrepos/beta

git status
git branch -a

git checkout master
git merge feature

git log --oneline --decorate --graph --all -5
```

Push `master`. This updates `refs/heads/master` in the bare repository and executes the
`post-update` hook, which is the required test:

```bash
git push origin master
```

Verify the tag directly on the remote repository. On October 5, 2026, the expected tag is
`release-2026-10-05`:

```bash
TODAY_TAG="release-$(date +%Y-%m-%d)"

git --git-dir=/opt/beta.git show-ref --verify "refs/tags/$TODAY_TAG"
# <commit-sha> refs/tags/release-2026-10-05

git ls-remote --tags origin "$TODAY_TAG"
# <same-commit-sha> refs/tags/release-2026-10-05
```

The hook creates the tag in the remote bare repository, not automatically in this clone. Fetch it
before checking the local tag list:

```bash
git fetch origin --tags

git tag -l "$TODAY_TAG"
# release-2026-10-05

git rev-parse origin/master
git rev-parse "$TODAY_TAG"
# both commands print the same commit id

git status
# On branch master
# Your branch is up to date with 'origin/master'.
# nothing to commit, working tree clean
```

## Why

| Command or code | Why |
|:--|:--|
| `/opt/beta.git/hooks/post-update` | Receive-side hooks belong in the bare repo that accepts pushes. One in a clone would never see them. |
| `if [ "$ref" = "refs/heads/master" ]` | `post-update` gets every updated ref as an argument. Without the check, a feature-branch push would also tag. |
| `git show-ref --verify --quiet` | Checks whether today's tag exists, so a second push on the same day does not fail. |
| `git tag "$tag_name" "$ref"` | Tags the commit that was just pushed. Without the ref it relies on the bare repo's `HEAD`. |
| `chmod +x` | Git silently ignores a hook that is not executable. |
| `git fetch origin --tags` | A tag created by a server hook does not come back on its own. |

## Notes

- `post-update` runs once after all refs are updated, and receives ref names only. `post-receive` is the one that gets old and new commit IDs.
- Install the hook **before** pushing the merge. Push first and there is no master update left to trigger it.
- The tag is written directly inside `/opt/beta.git`, so no `git push --tags` is needed.
- `git fetch -a` is `--append`, not "all tags". Use `git fetch origin --tags`.
- This is a lightweight tag. The task asks for the name only, not an annotated tag.
