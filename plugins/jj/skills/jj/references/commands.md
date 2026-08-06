# Commands Reference

Complete reference for jj commands and their options.

## Table of Contents

- [CLI Revision Options](#cli-revision-options)
- [Repository Setup](#repository-setup)
- [Status and History](#status-and-history)
- [Creating and Editing Commits](#creating-and-editing-commits)
- [History Rewriting](#history-rewriting)
- [Running Commands Across Revisions](#running-commands-across-revisions)
- [Bookmarks (Branches)](#bookmarks-branches)
- [Git Operations](#git-operations)
- [Operation Log](#operation-log)
- [File Operations](#file-operations)
- [Tags](#tags)
- [Workspaces](#workspaces)
- [Configuration](#configuration)

## CLI Revision Options

Many jj commands share a consistent pattern of flags for selecting revisions and destinations. Understanding this pattern makes the CLI predictable across commands.

### Source flags (what to operate on)

| Flag | Short | Meaning |
|------|-------|---------|
| `--revision` (or `--revisions`) | `-r` | Select specific revision(s) — default for most commands |
| `--source` | `-s` | Revision AND all its descendants (same as `-r REV::`) |
| `--from` | `-f` | The _contents_ of a revision, or the bookmarks on a revision |
| `--branch` | `-b` | Whole branch relative to destination |

### Destination flags (where to put it)

| Flag | Short | Meaning |
|------|-------|---------|
| `--onto` | `-o` | Place as children of target (aliases: `-d`, `--destination`) |
| `--insert-after` | `-A` | Insert between target and its children |
| `--insert-before` | `-B` | Insert between target and its parents |
| `--into` | `-t` | Target for contents/bookmarks (alias: `--to`; use with `--from`) |

### Key examples

```bash
# Rebase operations
jj rebase -r xyz -o main              # Rebase single commit onto main
jj rebase -s xyz -o main              # Rebase xyz and all descendants onto main
jj rebase -b @ -o main                # Rebase current branch onto main

# Content operations
jj squash --from xyz --into abc        # Move xyz's changes into abc
jj restore --from main --into @        # Copy file state from main to working copy

# Insertion
jj rebase -r xyz -A main              # Insert xyz after main
jj rebase -r xyz -B main              # Insert xyz before main
```

### Commands that allow omitting `-r`

Some commands accept revisions as positional arguments (e.g., `jj new xyz` instead of `jj new -r xyz`):
`abandon`, `describe`, `duplicate`, `metaedit`, `new`, `parallelize`, `show`.

## Repository Setup

### `jj git clone`

Clone a Git repository:

```bash
jj git clone <url> [destination]
jj git clone --colocate <url>     # Allow git commands (default)
jj git clone --no-colocate <url>  # jj-only repo
jj git clone -b <pattern>         # Clone specific branch(es)
jj git clone -t <pattern>         # Clone only matching tags (0.44+)
jj git clone --depth <n>          # Shallow clone
jj git clone --remote <name>      # Name for the created remote (default: origin)
```

`--tag=PATTERN` (0.44+) replaces the removed `--fetch-tags=all|none|included`.

### `jj git init`

Initialize a new repository:

```bash
jj git init                       # New colocated repo (default)
jj git init --no-colocate         # New jj-only repo
jj git init --git-repo <path>     # Use existing git repo as backend
```

## Status and History

### `jj status` (alias: `st`)

Show working copy status:

```bash
jj status
jj st [paths...]                  # Status for specific paths
```

### `jj log`

Show commit history:

```bash
jj log                            # Default: mutable commits
jj log -r <revset>                # Specific revisions
jj log -r '::'                    # All commits
jj log -n 10                      # Limit to 10 commits (--limit)
jj log -p                         # Show patches
jj log -s                         # Summary (files changed)
jj log --stat                     # Show diffstat
jj log --name-only                # Just the changed file paths
jj log -G                         # Flat list, no graph (--no-graph)
jj log --reversed                 # Oldest first
jj log --count                    # Print the number of commits instead of showing them
jj log -T <template>              # Custom template
jj log [paths...]                 # Commits touching paths
```

### `jj show`

Show commit details. Patch output is on by default; there is no `-p` flag:

```bash
jj show                           # Current working copy
jj show <rev>                     # Specific revision
jj show <A> <B> <C>               # Multiple revisions, one after the other (0.42+)
jj show --reversed                # Oldest first (0.43+)
jj show -s                        # Summary only
jj show --no-patch                # Metadata only, no diff
jj show --stat                    # Diffstat
jj show --git                     # Git-format diff
jj show -T <template>             # Custom output template
```

### `jj diff`

Show changes:

```bash
jj diff                           # Changes in working copy
jj diff -r <rev>                  # Changes in revision vs parent
jj diff --from <rev>              # From specific revision
jj diff --to <rev>                # To specific revision
jj diff --from <A> --to <B>       # Between two revisions
jj diff -s                        # Summary
jj diff --stat                    # Diffstat
jj diff --git                     # Git format
jj diff --color-words             # Word-level diff
jj diff [paths...]                # Specific paths only
```

### `jj evolog` (evolution log)

Show the history of a single change over time. Each time a change is rewritten (rebased, squashed, amended), the update appears here:

```bash
jj evolog                         # Evolution of working copy change
jj evolog -r <rev>                # Evolution of specific change
jj evolog -p                      # Show patches between versions
jj evolog -s                      # Summary of changes
jj evolog -n 5                    # Limit entries
jj evolog -T <template>           # Custom template
```

Use with `jj restore --from` to recover a previous version of a change.

## Creating and Editing Commits

### `jj new`

Create a new commit:

```bash
jj new                            # New commit on working copy parent
jj new <rev>                      # New commit on specific revision
jj new <A> <B>                    # Merge commit with multiple parents
jj new -m "message"               # With description
jj new --no-edit                  # Don't make it working copy
jj new -A <rev>                   # Insert after revision
jj new -B <rev>                   # Insert before revision
```

### `jj describe` (alias: `desc`)

Edit commit description:

```bash
jj describe                       # Edit working copy description
jj describe <rev>                 # Edit specific revision
jj describe -m "message"          # Set message directly
jj describe --stdin               # Read from stdin
jj describe -m "msg" --editor     # Prefill the editor with -m/--stdin, then edit
jj describe -r 'trunk()..@'      # Describe all commits in branch (editor per commit)
jj describe -r 'description("")' # Describe all commits with empty messages
```

`--author`, `--reset-author`, `--edit`, and `--no-edit` were removed in 0.42. Use
`--editor` to force an editor, and `jj metaedit` for author changes.

### `jj edit`

Switch working copy to edit existing commit:

```bash
jj edit <rev>                     # Edit specific revision
```

### `jj commit` (alias: `ci`)

Snapshot working copy into the current commit, then create a new empty commit on top. The description editor opens for the CURRENT commit (contrast with `jj new`, which creates a new commit without editing the current one's description):

```bash
jj commit                         # Describe current, create new on top
jj commit -m "message"            # With message
jj commit -m "msg" --editor       # Prefill editor with the message, then edit
jj commit -i                      # Interactive selection
jj commit --tool <name>           # Pick a diff editor (implies -i)
jj commit [paths...]              # Only specified paths
```

`--author` / `--reset-author` were removed in 0.42 — use `jj metaedit --author`.

## History Rewriting

### `jj squash`

Move changes into parent:

```bash
jj squash                         # All changes to parent
jj squash -r <rev>                # From specific revision
jj squash -i                      # Interactive selection
jj squash [paths...]              # Only specific paths
jj squash --from <A> --into <B>   # Between arbitrary commits (-f / -t)
jj squash -m "message"            # Set combined description
jj squash -u                      # Use the destination's description as-is
jj squash -k                      # --keep-emptied: don't abandon the source
jj squash --tool <name>           # Pick a diff editor (implies -i)

# Gather changes to specific paths from a range of commits:
jj squash vendor/ --from r1::rN   # All vendor/ changes from r1..rN into @
jj squash -i --from r1::rN        # Interactive pick from range
```

`--from` accepts full revset expressions (not just single commits). `--into` defaults to `@` when omitted.

**Bookmark relocation:** `jj squash --from <child> --into <parent>` folds the child into the parent *and* moves the child's bookmark onto the parent. Squashing two *described* commits opens an editor — pass `-m "<msg>"` or `--use-destination-message` for agent/non-interactive use.

### `jj split`

Split commit into two:

```bash
jj split                          # Interactive split of working copy
jj split -r <rev>                 # Split specific revision
jj split [paths...]               # Put paths in first commit
jj split -i                       # Interactive mode
jj split --tool <name>            # Pick a diff editor (implies -i)
jj split -p                       # Parallel (sibling) commits
jj split -m "message"             # First commit message
jj split -A <rev>                 # Move the selected changes after <rev>
jj split -B <rev>                 # Move the selected changes before <rev>
```

### `jj arrange`

TUI for reordering and abandoning revisions interactively:

```bash
jj arrange                        # Reorder revisions in TUI
jj arrange -r <revset>            # Arrange specific revisions
```

Supports swapping commits up/down along graph edges. Includes parents/children in the view and uses the default log template.

### `jj rebase`

Move commits to different parents:

```bash
# What to rebase:
jj rebase -b <rev>                # Branch containing rev (default: -b @)
jj rebase -s <rev>                # Source and descendants
jj rebase -r <rev>                # Only revision (not descendants)

# Where to rebase (-d / --destination are aliases of -o / --onto):
jj rebase -o <dest>               # Onto destination
jj rebase -A <rev>                # Insert after
jj rebase -B <rev>                # Insert before

# Examples:
jj rebase -o main                 # Rebase current branch onto main
jj rebase -s X -o Y               # Rebase X and descendants onto Y
jj rebase -r X -o Y               # Rebase only X onto Y
jj rebase -r X -A Y               # Insert X after Y
jj rebase -r X -B Y               # Insert X before Y

# Options:
jj rebase --skip-emptied          # Abandon commits that become empty
jj rebase --keep-divergent        # Keep divergent commits instead of abandoning them
jj rebase --simplify-parents      # Remove redundant parent edges
```

Without `--keep-divergent`, a commit is abandoned during rebase if the destination
already contains a commit with the same change ID and identical changes.

### `jj diffedit`

Interactively edit commit contents:

```bash
jj diffedit                       # Edit working copy
jj diffedit -r <rev>              # Edit specific revision
jj diffedit --from <A> --to <B>   # Edit diff between revisions
jj diffedit --tool <tool>         # Use specific diff editor
jj diffedit --restore-descendants # Preserve descendant content
```

### `jj duplicate`

Copy commits. Duplicated commits have the same file content and description but receive new change IDs, so they are independent from the originals:

```bash
jj duplicate                      # Duplicate working copy
jj duplicate <revs>               # Duplicate specific revisions
jj duplicate -A <rev>             # Insert duplicates after
jj duplicate -B <rev>             # Insert duplicates before
```

### `jj abandon`

Remove commits (keep content in descendants):

```bash
jj abandon                        # Abandon working copy
jj abandon <revs>                 # Abandon specific revisions
jj abandon --retain-bookmarks     # Move bookmarks to parents
jj abandon --restore-descendants  # Leave children's content untouched
```

### `jj restore`

Restore files from another revision:

```bash
jj restore                        # Restore all from parent
jj restore [paths...]             # Restore specific paths
jj restore --from <rev>           # Source revision (-f)
jj restore --into <rev>           # Destination revision (-t; --to is an accepted alias)
jj restore -c <rev>               # --changes-in: undo changes made in revision
jj restore -i                     # Interactive mode
jj restore --tool <name>          # Pick a diff editor (implies -i)
jj restore --restore-descendants  # Leave descendants' content untouched
```

### `jj absorb`

Auto-squash working copy changes into the commits where those lines were last modified. Like `git commit --fixup` + `rebase --autosquash` in one step:

```bash
jj absorb                         # Absorb all working copy changes into mutable()
jj absorb [paths...]              # Absorb specific paths only
jj absorb --from <rev>            # Absorb from specific revision (-f, default: @)
jj absorb --into <revset>         # Target specific destinations (-t, default: mutable())
jj absorb -i                      # Interactively pick which hunks to absorb (0.44+)
jj absorb --tool <name>           # Choose the diff editor (implies -i) (0.44+)
```

**Behavior:**
- Only considers ancestors of the source revision as destinations
- If the destination for a hunk can't be determined unambiguously, the change stays in the source
- Source is abandoned if all changes are absorbed AND it has no description
- With `-i` (0.44+), only the selected hunks are considered — unselected and unabsorbable
  hunks stay in the source. This absorbs *part* of a commit without splitting it first.
- Review what happened with `jj op show -p`

### `jj interdiff`

Compare the *diffs* of two revisions — shows how one implementation differs from another:

```bash
jj interdiff --from <rev> --to <rev>   # Compare what two changes do differently
jj interdiff --from push-xyz@origin --to push-xyz  # What changed since last push
jj interdiff --from @- --to other -s   # Summary of implementation differences
```

Unlike `jj diff --from A --to B` (which compares file contents), interdiff compares *patches* — what each change adds/removes relative to its own parents. Use `jj evolog -p` to see the full evolution history instead.

### `jj fix`

Run configured formatting tools on files in mutable commits:

```bash
jj fix                            # Fix files in reachable(@, mutable())
jj fix -s <rev>                   # Fix from specific revision + descendants
jj fix [paths...]                 # Fix only specific paths
jj fix -a                         # --all-lines: format entire files, not just modified lines (0.41+)
jj fix --include-unchanged-files  # Fix all files, not just changed ones
```

Default scope is `reachable(@, mutable())`. Descendants are also fixed to preserve formatting. Review with `jj op show -p`. Requires `[fix.tools.*]` configuration.

### `jj bisect`

Find a bad revision by running a command across history. The range is given as a
single `--range`/`-r` revset — its heads are assumed bad, its excluded ancestors good:

```bash
jj bisect run --range v1.0..main -- <command>   # Find first bad revision
jj bisect run -r v1.0..main --find-good -- <cmd> # Find first *good* revision instead
```

Each candidate is checked out (becomes `@`) before the command runs. Exit status
drives the search: `0` = good, `125` = skip this revision, `127` = abort the
bisection, anything else = bad (`--find-good` inverts good/bad but not the special
codes). The candidate's commit ID is exported as `$JJ_BISECT_TARGET`.

`--range` can be repeated; the union of the ranges is bisected.

### `jj next` / `jj prev`

Navigate the commit graph by moving the working copy forward/backward:

```bash
jj next                           # Move WC to child commit
jj next 2                         # Move 2 commits forward
jj next --edit                    # Edit the child instead of creating new WC on top (-e)
jj next --conflict                # Jump to the next commit with a conflict
jj prev                           # Move WC to parent commit
jj prev 2                         # Move 2 commits backward
jj prev --edit                    # Edit the parent directly
jj prev --no-edit                 # Force new-WC behavior even if edit is the default (-n)
```

Without `--edit`: creates a new empty WC commit as sibling. With `--edit`: changes WC to the target commit directly (like `jj edit`).

### `jj metaedit`

Modify commit metadata without changing file content:

```bash
jj metaedit -m "new message"      # Update description (default: @)
jj metaedit -r <revs> -m "msg"    # Update specific revisions
jj metaedit --update-change-id    # Generate a new change ID
jj metaedit --update-author       # Set author to the configured user
jj metaedit --author "N <e@x>"    # Set author explicitly
jj metaedit --update-author-timestamp     # Set author date to now
jj metaedit --author-timestamp <date>     # Set author date (RFC2822 or RFC3339)
jj metaedit --force-rewrite       # Rewrite even if nothing else changed (refreshes committer)
```

Updating any metadata also refreshes the committer name/email/timestamp on every
rewritten commit. `JJ_USER` / `JJ_EMAIL` override the user for `--update-author` and
`--force-rewrite`. `--update-committer-timestamp` was removed in 0.42 — use
`--force-rewrite`.

### `jj sign` / `jj unsign`

Cryptographically sign or remove signatures from commits:

```bash
jj sign                           # Sign revision (uses revsets.sign default)
jj sign -r <revs>                 # Sign specific revisions
jj sign --key <key>               # Sign with a specific key
jj unsign -r <revs>               # Remove signature
```

### `jj sparse`

Manage sparse checkouts — control which paths are materialized in the working copy:

```bash
jj sparse list                    # Show current sparse patterns
jj sparse set --add src --remove tests  # Modify patterns
jj sparse set --clear --add src   # Start fresh with only src/
jj sparse reset                   # Include all files again
jj sparse edit                    # Edit patterns in editor
```

### `jj revert`

Create new commit(s) that are the reverse of specified changes. Unlike `jj restore`, this creates new commits rather than modifying existing ones.

Both `-r/--revision` **and** a destination (`-o/--onto`, `-A`, or `-B`) are required —
`jj revert -r <rev>` on its own is an error:

```bash
jj revert -r xyz -o main          # Revert xyz, place the reverse onto main
jj revert -r 'a::b' -o @          # Revert a range on top of the working copy
jj revert -r xyz -A @             # Insert the reverse right after @
jj revert -r xyz -B main          # Insert the reverse before main
```

Reverses are applied sequentially in reverse topological order. Customize the generated
description with `templates.revert_description`.

### `jj parallelize`

Make sequential revisions into parallel siblings (children of the same parent):

```bash
jj parallelize <revs>             # Make revisions parallel
jj parallelize abc::xyz           # Parallelize a range of commits
```

### `jj resolve`

Resolve merge conflicts interactively:

```bash
jj resolve                        # Resolve conflicts in working copy
jj resolve -r <rev>               # Resolve conflicts in specific revision
jj resolve -l                     # List conflicted files
jj resolve --tool <name>          # Use specific merge tool (e.g., meld, kdiff3)
jj resolve --tool :ours           # Resolve all conflicts using "our" side
jj resolve --tool :theirs         # Resolve all conflicts using "their" side
jj resolve [paths...]             # Resolve specific files only
```

Built-in tools `:ours` and `:theirs` are useful for bulk-resolving conflicts without opening an editor.

### `jj simplify-parents`

Remove redundant parent edges from merge commits. Useful after rebasing when a merge commit has a parent that's also an ancestor through another parent:

```bash
jj simplify-parents               # Simplify working copy parents
jj simplify-parents -r <revs>     # Simplify specific revisions
```

## Running Commands Across Revisions

### `jj run` (0.43+)

Run a command once per revision. Each revision is checked out in its own private
working copy, the command runs there, and the revision is amended with the result.
Descendants are then rebased on top, so the change propagates through the stack —
including conflicts, if the amendment creates any.

```bash
jj run -r 'trunk()..@' -- cargo fix          # Apply a fixer across the stack
jj run -r 'mutable()' -- cargo fmt           # Reformat every mutable commit
jj run -j 4 -- pre-commit run --all-files    # 4 parallel jobs
jj run --ignore-changes -r 'trunk()..@' -- cargo test   # Read-only check per commit
```

**Options:**

| Flag | Meaning |
|------|---------|
| `-r`, `--revision <revsets>` | Revisions to run against (default: `revsets.run`, i.e. `reachable(@, mutable())`) |
| `-j`, `--jobs <n>` | Parallel processes (overrides `run.jobs`; default 1) |
| `--root` | Run from each commit's workspace root instead of the current subdirectory |
| `--clean` | Start each commit from a fresh checkout (working copies are reused otherwise, preserving build artifacts) |
| `--restore-descendants` | Preserve descendants' content instead of their diff |
| `--passthrough` (0.44+) | Connect the subprocess's stdout/stderr to the terminal instead of capturing them. Forces a single job; stdin is not inherited |
| `--ignore-changes` (0.44+) | Run without rewriting any commits — working-copy changes are discarded. Good for tests/linters, and permits immutable commits without `--ignore-immutable` |
| `--ignore-errors` (0.44+) | Keep going when a command exits nonzero. Failed revisions are left unchanged; successful ones still apply atomically at the end, and `jj run`'s own exit code is unaffected |

Each invocation gets `JJ_CHANGE_ID`, `JJ_COMMIT_ID`, and `JJ_WORKSPACE_ROOT` in its
environment. As of 0.44 revisions are processed oldest to newest, and the *start*
order is guaranteed even with `--jobs` above 1.

Use `--` before the command whenever its own arguments start with `-`.

## Bookmarks (Branches)

### `jj bookmark` (alias: `b`)

Manage bookmarks:

```bash
# List
jj bookmark list
jj bookmark list -a               # Include all remotes
jj bookmark list -r <revs>        # Bookmarks at revisions
jj bookmark list -t               # Tracked remote bookmarks only
jj bookmark list -c               # Conflicted bookmarks only
jj bookmark list --sort <key>     # name / author-date / committer-date (suffix `-` = descending)

# Create/Set
jj bookmark create <name>         # At working copy
jj bookmark create <name> -r <rev>
jj bookmark set <name>            # Move to working copy
jj bookmark set <name> -r <rev>   # Move to revision
jj bookmark set <name> -r <rev> --allow-backwards  # Move to ancestor

# Modify
jj bookmark move <name>           # Move to working copy (--to defaults to @)
jj bookmark move <name> --to <rev>
jj bookmark move --from <rev>     # Move whichever bookmarks point at <rev>
jj bookmark rename <old> <new>
jj bookmark rename <old> <new> --overwrite-existing  # Replace existing
jj bookmark advance               # Move bookmarks forward to @
jj bookmark advance --to <rev>    # Move bookmarks forward to revision (-t)

# Delete
jj bookmark delete <name>         # Delete (will push deletion)
jj bookmark forget <name>         # Forget (won't push deletion)
jj bookmark forget <name> --include-remotes  # Also forget the remote-tracking state

# Remote tracking
jj bookmark track <name> --remote <remote>
jj bookmark untrack <name> --remote <remote>
```

`bookmark advance` uses `revsets.bookmark-advance-from` and `revsets.bookmark-advance-to` for customization.

## Git Operations

### `jj git fetch`

Fetch from remote:

```bash
jj git fetch                      # From default remote
jj git fetch --remote <name>      # From specific remote (repeatable, glob patterns OK)
jj git fetch --all-remotes        # From all remotes
jj git fetch -b <pattern>         # Specific branches (--branch)
jj git fetch -t <pattern>         # Specific tags (--tag)
jj git fetch --tracked            # Only already-tracked bookmarks and tags
```

Shows details of abandoned commits after fetch.

**Tags are fetched like bookmarks (0.44+).** They arrive as `<name>@<remote>` and are
**automatically tracked** by local tags of the same name (bookmarks are *not*
auto-tracked). The first fetch in an existing repo re-fetches all tags to initialize
tracking state. Git's `tagOpt` is no longer respected — disable tag fetching with
`remotes.<name>.fetch-tags = '~*'` in jj config.

**Fetch rebases descendants of rewritten revisions (0.43+),** matching them by change
ID. This means a force-push upstream that rewrote your stack's base leaves your local
descendants rebased onto the new parents. Immutable descendants are not rebased.

### `jj git push`

Push to remote:

```bash
jj git push -b <name>             # Push specific bookmark (--bookmark, repeatable)
jj git push -t <name>             # Push specific tag (--tag, repeatable) (0.44+)
jj git push --all                 # Push all bookmarks AND tags, including new ones
jj git push --tracked             # Push all tracked bookmarks and tags
jj git push --deleted             # Push deletions
jj git push -c <rev>              # --change: create + push an auto-named bookmark
jj git push --named <name>=<rev>  # Push <rev> under a new bookmark name
jj git push -r <revs>             # Push bookmarks/tags pointing at these commits
jj git push --remote <name>       # To specific remote
jj git push --dry-run             # Show what would be pushed
jj git push -o <key>=<value>      # Pass push options to remote
```

**No `--allow-new` (removed in 0.42).** Its job is now automatic: `--bookmark`,
`--tag`, `--change`, and `--named` track the remote ref automatically when it doesn't
exist yet, and `--all` explicitly includes new bookmarks and tags. Just drop the flag.

With no flags, push defaults to tracking bookmarks and tags in
`remote_bookmarks(remote=<remote>)..@`.

**Skip guards** — push refuses ineligible commits unless told otherwise:

| Flag | Allows pushing |
|------|----------------|
| `--allow-empty-description` | Commits with no description |
| `--allow-private` | Commits matched by `git.private-commits` |
| `--allow-conflicts` (0.44+) | Commits containing conflicts |

`--all`, `--tracked`, and `-r` skip ineligible bookmarks instead of failing.

Push is force-with-lease by default: the remote ref moves only if its current state
matches what jj last fetched.

### `jj git remote`

Manage remotes:

```bash
jj git remote list
jj git remote add <name> <url>
jj git remote remove <name>
jj git remote rename <old> <new>
jj git remote set-url <name> <url>
jj git remote set-url <name> --push <url>  # Separate push URL
```

### `jj git import` / `jj git export`

Sync with the underlying Git repo. Only relevant in **non-colocated** repos:

```bash
jj git import                     # Import Git changes to jj
jj git export                     # Export jj changes to Git
```

**Disabled by default in colocated workspaces (0.44+).** Import/export happen
automatically there, so both commands now print `No import needed in colocated
workspaces.` (or the export equivalent) and do nothing. They previously had a race
condition. Pass `--ignore-working-copy` to force the operation anyway.

If a colocated repo looks out of sync, run any ordinary jj command (which snapshots
and re-imports) rather than reaching for `jj git import`.

`jj git import` also **abandons** jj commits that are no longer reachable from any
branch in the Git repo (an abandoned working-copy commit is replaced with a new empty
one). Set `git.abandon-unreachable-commits = false` to disable that. The same setting
governs `jj git fetch`.

## Operation Log

### `jj op log`

View operation history:

```bash
jj op log                         # Full operation log
jj op log -n 10                   # Limit entries
jj op log -p                      # Show patches
jj op log -d                      # Show operation diffs (--op-diff)
jj op log --show-changes-in <revs> # Limit the shown changed revisions to a revset
```

### `jj op revert`

Revert an operation (replaces removed `jj op undo`):

```bash
jj op revert                      # Revert the latest operation
jj op revert <op-id>              # Revert specific operation
jj op revert --what repo          # Restore only repo state, not remote-tracking refs
```

### `jj op restore`

Restore to previous state:

```bash
jj op restore <op-id>             # Restore to operation
jj op restore <op-id> --what repo # Leave remote-tracking bookmarks alone (so you can still push)
```

`--what` is experimental. Values: `repo` (repo state and local bookmarks) and
`remote-tracking` (remote-tracking bookmarks); both are restored by default. Omit
`remote-tracking` if you want to push after the undo.

### `jj op show`

Show operation details:

```bash
jj op show                        # Current operation
jj op show <op-id>                # Specific operation
jj op show -p                     # With patches
jj op show --no-op-diff           # Metadata only
```

### `jj op diff` / `jj op abandon`

```bash
jj op diff --from <op> --to <op>  # Compare repo state between two operations
jj op diff --operation <op>       # Changes made by one operation, vs its parent (--op)
jj op abandon ..<op-id>           # Discard <op-id> and all its ancestors
jj op abandon <op-id>..@-         # Discard recent operations (after jj op restore <op-id>)
```

`jj op abandon` takes an operation **or an operation range**. Descendants of abandoned
operations are reparented onto the root operation; unreachable commits can then be
collected with `jj util gc`.

## File Operations

### `jj file`

File-related commands:

```bash
jj file list                      # List files in working copy
jj file list -r <rev>             # Files in specific revision
jj file show <path>               # Show file content
jj file show -r <rev> <path>      # Content at revision
jj file annotate <path>           # Blame (line origins)
jj file chmod x <path>            # Make executable
jj file chmod n <path>            # Remove executable
jj file track <paths>             # Start tracking
jj file track --include-ignored <paths>  # Track even if gitignored
jj file untrack <paths>           # Stop tracking
```

### `jj file search`

Search file contents (like `git grep`). **`--pattern`/`-p` is a required flag** —
positional arguments are filesets, not the pattern:

```bash
jj file search -p 'TODO'          # Every matching line, prefixed by file path
jj file search -p 'TODO' src/     # Restrict to a fileset
jj file search -p 'TODO' -r <rev> # Search at a specific revision (default: @)
jj file search -p 'TODO' -n       # Prefix matches with 1-based line numbers (0.44+)
jj file search -p 'TODO' --name-only  # Just the file paths (pre-0.44 behavior)
```

**Output changed in 0.44:** the default now prints each matched line prefixed by its
file path (`path:line`, or `path:N:line` with `-n`). Use `--name-only` for the old
paths-only output.

The pattern uses `kind:pattern` string-pattern syntax and defaults to `regex` when the
kind is omitted. A glob must match the **whole line**, so wrap it:
`--pattern 'glob:*foo*'`.

This is an early version of the command — it does not search files concurrently.

## Tags

As of 0.44 tags are fetched, pushed, and tracked much like bookmarks. Remote tags live
at `<name>@<remote>`, and fetched tags are **tracked by default** (unlike bookmarks).

### `jj tag list`

List and filter tags:

```bash
jj tag list                       # Local tags (tracked remotes shown only if they differ)
jj tag list <patterns>            # Filter by tag name (glob by default)
jj tag list -a                    # All tracked and untracked remote tags (--all-remotes)
jj tag list --remote <name>       # Tags from one remote
jj tag list -t                    # Tracked remote tags only
jj tag list -c                    # Conflicted tags only
jj tag list -r <revset>           # Tags whose local targets are in <revset>
jj tag list --sort <key>          # name / author-date / committer-date (suffix `-` = descending)
jj tag list -T <template>         # Custom template (templates.tag_list)
```

`jj tag list -r 'REV::'` ≈ `git tag --contains REV`; `-r '::REV'` ≈ `git tag --merged REV`.

### `jj tag set` / `jj tag delete`

```bash
jj tag set <name>                 # Create tag at @
jj tag set <name> -r <rev>        # Create tag at a revision
jj tag set <name> --allow-move    # Move an existing tag
jj tag delete <names>             # Delete local tags
```

### `jj tag track` / `jj tag untrack` (0.44+)

```bash
jj tag track 'v1.0@origin'        # Track a specific remote tag
jj tag track 'v*' --remote origin # Track by pattern
jj tag untrack 'v1.0@origin'      # Stop importing this remote tag locally
```

A tracked remote tag is imported as a local tag of the same name and keeps updating on
fetch. An untracked one is just a pointer to the last-fetched remote state. With no
`--remote`, all remotes matching the tag names are affected.

Because `tags()` is part of `builtin_immutable_heads()`, changing which tags you track
changes which commits are immutable.

## Workspaces

### `jj workspace`

Manage multiple working copies:

```bash
jj workspace list                 # List workspaces (shows roots by default, 0.44+)
jj workspace list -T <template>   # Custom template (templates.workspace_list)
jj workspace add <path>           # Add workspace (uses relative paths)
jj workspace add -r <rev> <path>  # At specific revision
jj workspace add --name <name> <path>
jj workspace forget [names...]    # Remove workspace(s)
jj workspace rename <new-name>    # Rename the current workspace
jj workspace root                 # Show workspace root
jj workspace root --name <name>   # Root of specified workspace
jj workspace update-stale         # Update stale workspace
```

`jj workspace list` output is `<name>: <root> <change-id> <commit-id> <description>`.
Before 0.44 the root was omitted.

## Configuration

### `jj config`

Manage configuration:

```bash
jj config list                    # Show all config
jj config get <key>               # Get specific value
jj config set --user <key> <val>  # Set user config
jj config set --repo <key> <val>  # Set repo config
jj config unset --user <key>      # Remove a setting
jj config edit --user             # Edit user config
jj config edit --repo             # Edit repo config
jj config path --user             # Show config file path
jj config gc                      # Prune repo-level config dirs whose repo is gone (0.43+)
```

`jj config gc` (0.43+) finds and optionally deletes config directories under
`~/.config/jj/repos` that point at repos which no longer exist.

## Utility Commands

### Other useful commands

```bash
jj root                           # Show repo root (shortcut for workspace root)
jj version                        # Show jj version
jj util backend name              # Print the commit backend in use, e.g. "git" (0.42+)
jj util completion <shell>        # Generate shell completions
jj util config-schema             # JSON schema for the jj TOML config format
jj util exec <cmd> [args...]      # Run an external command via jj
jj util gc                        # Garbage collect (prunes objects/ops older than 2 weeks)
jj util gc --expire now           # Prune immediately (only "now" is accepted today)
jj util install-man-pages <path>  # Install manpages
jj util snapshot                  # Manually trigger working copy snapshot
```

## Global Options

Available on all commands:

```bash
jj -R <path>                      # Use different repo
jj --at-op <op-id>                # Load at operation (alias of --at-operation)
jj --ignore-working-copy          # Skip working copy snapshot
jj --ignore-immutable             # Allow modifying immutable (also valid after the subcommand: jj <cmd> --ignore-immutable)
jj --no-integrate-operation       # Run without impacting repo state
jj --color <when>                 # always/never/auto
jj --no-pager                     # Disable pager
jj --config <key=value>           # Override config
jj --config-file <path>           # Additional config file
jj --quiet                        # Less output
jj --debug                        # Debug output
```

**Repeated arguments are no longer an error (0.44+).** The last occurrence wins, so
`jj log -n 5 -n 10` limits to 10 and an argument baked into an alias or wrapper script
can be overridden on the command line. Arguments that legitimately repeat — `--config`,
`jj log -r`, `jj git push -b` — still collect every value.
