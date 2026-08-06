# GitHub/GitLab Workflow

## Pushing Changes

### Using Auto-Generated Bookmarks

Let jj create a bookmark name from the change ID — no manual naming needed:

```shell
# Work on your changes, then commit
jj new main
jj commit -m 'feat(bar): add support for bar'

# Push parent of working copy (@ is empty), auto-creates push-XXXX bookmark
jj git push --change @-
# Short form: jj git push -c @-

# Push a specific change by its short ID
jj git push -c mw  # creates bookmark like "push-mwmpwkwknuz"
```

### Using Named Bookmarks

```shell
jj new main
jj commit -m 'feat(bar): add support for bar'

# Create bookmark on parent of working copy
jj bookmark create bar -r @-

# Push it by name — a brand-new bookmark is tracked automatically (0.42+)
jj git push --bookmark bar

# Afterwards a bare `jj git push` picks it up (only tracked bookmarks are pushed)
jj git push
```

A bare `jj git push` refuses to create a *new* remote bookmark (`Refusing to create new remote bookmark bar@origin`). Either name it once with `--bookmark`/`--change`/`--named` — all three auto-track — or run `jj bookmark track bar@origin` first.

> `jj git push --allow-new` was **removed in 0.42**. `--bookmark` now does what that flag used to gate: "If a bookmark isn't tracking anything yet, the remote bookmark will be tracked automatically."

## Updating Your Repository

There is no `jj pull`. Use fetch + rebase:

```shell
# Fetch latest from remote
jj git fetch

# Rebase current branch onto updated main
jj rebase -o main

# Rebase multiple branches
jj rebase -b feature-a -b feature-b -o main
```

### When Upstream Force-Pushed (rewritten history)

Since 0.43, `jj git fetch` **rebases descendants of rewritten revisions by itself**, matching them by change ID. No manual rebase needed:

```shell
jj git fetch
# bookmark: feat@origin [updated] untracked
# Updated 1 rewritten commits.
# Rebased 1 descendant commits.
```

Your stacked work lands on the amended upstream commit automatically. Caveats:

- Requires the change ID to survive the rewrite (jj writes a `change-id` Git header; preserved by `git commit --amend`, **not** by a plain `git rebase`).
- **Immutable descendants are not rebased.**
- If change IDs were lost, fall back to `jj rebase -s <your-work> -o <new-upstream>`.

## Addressing PR Review Comments

### Additive Style (GitHub default)

GitHub compares branches, so adding commits on top is the standard workflow:

```shell
# Start editing on top of your feature bookmark
jj new your-feature

# Make fixes, then commit
jj diff
jj commit -m 'address pr comments'

# Move bookmark forward to include new commit
jj bookmark move your-feature --to @-

# Push (normal push, not force)
jj git push
```

### Rewriting Style (clean history)

For projects that prefer clean commits (force-push workflow):

```shell
# Edit the specific commit that needs fixing (trailing - = parent)
jj new your-feature-

# Make fixes, then squash into the parent
jj diff
jj squash

# Push — jj automatically force-pushes when history is rewritten
jj git push --bookmark your-feature
```

The `-` suffix is revset syntax: `your-feature-` means "parent of your-feature".

### Reordering an Already-Pushed / Stacked Branch

```shell
jj rebase --ignore-immutable -s <change> -o <dest>   # reorder; descendants auto-rebase
jj git push --bookmark <name>                        # force-with-lease-safe by default (no flag)

# when interleaving with plain git tooling instead:
git push --force-with-lease origin <branch>
```

`jj git push` only updates the remote if it still matches what jj last fetched (built-in lease safety) — don't reach for `--force`. `--ignore-immutable` works in global *or* subcommand position (`jj rebase --ignore-immutable …` or `jj --ignore-immutable rebase …`).

Push also refuses commits that still contain conflicts (`Won't push commit … since it has conflicts`). `--allow-conflicts` (0.44+) overrides that — see [conflicts.md](conflicts.md#pushing-conflicted-commits) before using it.

## Working with Others' Bookmarks

By default, `jj git fetch` doesn't import new remote bookmarks locally:

```shell
# Check out someone's branch without importing it
jj new feature-x@origin

# Or import all remote bookmarks automatically (in config):
# remotes.<name>.auto-track-bookmarks = "*"
```

With auto-tracking enabled, use `jj new feature-x` directly.

## Tags (0.44+)

Tags are fetched and pushed like bookmarks: as `<name>@<remote>`, tracked by a local tag of the same name.

```shell
jj tag list --all-remotes      # see local tags and their remote counterparts
jj git fetch                   # fetches tags too; new remote tags are tracked by default
jj git push --tag 'v*'         # push tags matching a glob (can be repeated)
jj tag untrack 'v*'            # stop tracking — leaves v1.0@origin standalone
jj tag track 'v1.0@origin'     # start tracking again
```

Three things to watch for:

- **`jj git push --all` now pushes all tags in addition to bookmarks.** If `--all` is your habit, it will publish local tags you never meant to share. Prefer `--tracked`, `--bookmark`, or `--change`.
- **The first `jj git fetch` in a pre-0.44 repo re-fetches every tag** to initialize tracking state. A wall of tag output is expected once, not a sign of breakage.
- **Git's `tagOpt` is no longer respected.** Disable tag fetching per remote in jj config instead — `remotes.<name>.fetch-tags = '~*'` (a *pattern*, not the old `all|included|none` enum). See [configuration.md](configuration.md).

`tags()` is part of `builtin_immutable_heads()`, so tracking a new tag can make more commits immutable.

`jj git clone --fetch-tags=all|none|included` was removed in 0.44; use `jj git clone --tag=PATTERN`.

## Fork Workflow (Multiple Remotes)

### Initial Setup

```shell
jj git clone --remote upstream https://github.com/upstream-org/repo
cd repo
jj git remote add origin git@github.com:your-name/repo-fork
```

### Configure Fetch/Push Defaults

```shell
# Fetch from upstream (and optionally origin), push to fork
jj config set --repo git.fetch '["upstream", "origin"]'
jj config set --repo git.push origin

# Track main from both remotes
jj bookmark track main

# Set trunk to upstream so it's immutable
jj config set --repo 'revset-aliases."trunk()"' main@upstream
```

### Keeping Fork in Sync

```shell
jj git fetch  # fetches from configured remotes
jj git push --bookmark main  # pushes main to origin (your fork)
```

## Using GitHub CLI

For non-colocated repos, `gh` can't find the Git directory. Fix with `$GIT_DIR`:

```shell
# One-off
GIT_DIR=$(jj git root) gh pr create --title "My PR"

# Permanent fix: add to .envrc (requires direnv)
echo 'export GIT_DIR=$(jj git root)' >> .envrc
direnv allow
```

In colocated repos (`jj git init --colocate`), `gh` works without configuration.

## GitLab Push Options

Create merge requests and control CI directly from push:

```shell
# Skip CI
jj git push -o ci.skip

# Create merge request on push (--bookmark auto-tracks a new bookmark)
jj git push --bookmark your-feature \
  -o merge_request.create \
  -o merge_request.target=main \
  -o 'merge_request.title=Add feature X' \
  -o merge_request.draft
```

## Useful Revsets for PR Workflows

```shell
# All local work not yet on any remote
jj log -r 'remote_bookmarks()..@'

# All your bookmarks not pushed to remote
jj log -r 'mine() & bookmarks() & ~remote_bookmarks()'

# Local bookmarks diverged from main
jj log -r 'bookmarks() & ~(main | remote_bookmarks())'

# All remote bookmarks you authored
jj log -r 'remote_bookmarks() & mine()'

# Commits on a specific bookmark not yet in main
jj log -r 'main..your-feature'
```
