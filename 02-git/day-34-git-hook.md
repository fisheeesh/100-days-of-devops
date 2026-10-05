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
| `/opt/beta.git/hooks/post-update` | Receive-side hooks belong in the bare repository that accepts pushes. A hook placed in the working clone would run only for local operations and would not react to pushes received by `origin`. |
| `for ref in "$@"` | Git passes every updated ref to `post-update` as a separate argument. The loop handles a push that updates more than one branch or tag. |
| `if [ "$ref" = "refs/heads/master" ]` | Restricts release creation to pushes that actually update `master`. A feature-branch-only push must not create a release tag. |
| `date +%Y-%m-%d` | Produces the required ISO-style date used in a tag such as `release-2026-10-05`. The value is calculated when the push reaches the server. |
| `git show-ref --verify --quiet` | Checks whether today's tag already exists. This makes the hook safe when `master` is pushed more than once on the same date. |
| `git tag "$tag_name" "$ref"` | Creates a lightweight tag on the newly updated `master` commit explicitly. Using only `git tag "$tag_name"` would rely on the bare repository's `HEAD` pointing to the correct branch. |
| `chmod +x` | Git ignores a hook that is not executable. This changes only the new hook file's executable bit, not the repository or directory permissions prohibited by the task. |
| `sh -n` | Parses the shell script without executing it. It catches syntax errors without creating the release tag before the real push test. |
| `git push origin master` | Publishes the merge and triggers the server's `post-update` hook after the remote ref has been updated. |
| `git fetch origin --tags` | Downloads the server-created tag into the working clone. A normal push does not automatically copy a tag created afterward by a remote hook back to the client. |

## Notes

- `post-update` runs once after all refs in a successful receive operation are updated. Its arguments are ref names such as `refs/heads/master`; it does not receive old and new commit IDs like `post-receive` does.
- The hook must be installed before pushing the merge. If the push happens first, there is no master update left to trigger the required test.
- The tag is created directly inside `/opt/beta.git`, so a separate `git push --tags` is not needed. The hook has already written the tag to the remote repository.
- `git fetch -a` means `git fetch --append`; it is not the command for retrieving every remote or specifically retrieving tags. Use `git fetch origin --tags` here.
- A second master push on the same day cannot create another tag with the same name. The existence check leaves the original daily release tag unchanged and lets the hook exit successfully.
- This script creates a lightweight tag. The task asks only for the name, not an annotated tag with a message and tagger metadata.
- No `chown`, recursive `chmod`, or directory permission changes are necessary. The task is performed entirely as `natasha`; only the new hook itself receives its required executable bit.
