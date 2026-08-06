# Git to jj Command Comparison

This reference maps common Git commands and workflows to their jj equivalents.

## Quick Reference Table

| Git Command | jj Equivalent | Notes |
|-------------|---------------|-------|
| `git init` | `jj git init` | Creates colocated repo by default |
| `git clone` | `jj git clone` | Creates colocated repo by default |
| `git status` | `jj status` | Alias: `jj st` |
| `git log` | `jj log` | Shows graph by default |
| `git log --oneline` | `jj log --no-graph` | Or customize template |
| `git show` | `jj show` | |
| `git diff` | `jj diff` | |
| `git diff --staged` | N/A | No staging area in jj |
| `git add` | N/A | Auto-tracked |
| `git add -p` | `jj split -i` | Interactive commit splitting |
| `git commit` | `jj commit` or `jj new` | Different workflow |
| `git commit --amend` | `jj describe` + changes | Working copy is always amendable |
| `git commit --amend -m` | `jj describe -m "msg"` | `jj describe --editor` opens the editor; `--author`/`--reset-author` were removed in 0.42 |
| `git commit --author` | `jj metaedit --author "Name <email>"` | Metadata only, content unchanged; `--update-author` reads `JJ_USER`/`JJ_EMAIL` |
| `git reset HEAD~` | `jj squash` | Move changes to parent |
| `git reset --hard` | `jj restore` | |
| `git checkout <file>` | `jj restore <file>` | |
| `git checkout <branch>` | `jj new <bookmark>` | Creates new working copy |
| `git switch` | `jj new` | |
| `git branch` | `jj bookmark` | Alias: `jj b` |
| `git branch -d` | `jj bookmark delete` | |
| `git branch --set-upstream-to` | `jj bookmark track <name>@<remote>` | Or `jj bookmark track <name> --remote <remote>`. The `<kind>:<name>@<remote>` pattern form was removed in 0.43 |
| N/A | `jj bookmark untrack <name>@<remote>` | Stop tracking a remote bookmark |
| `git merge` | `jj new <A> <B>` | Creates merge commit |
| `git rebase` | `jj rebase` | More powerful |
| `git rebase -i` | `jj squash -i`, `jj split` | Different approach |
| `git cherry-pick` | `jj duplicate <rev> -o <dest>` | One step — no follow-up rebase needed |
| `git revert` | `jj revert` | |
| `git stash` | N/A | Not needed - use `jj new` |
| `git stash pop` | N/A | Use `jj squash` |
| `git fetch` | `jj git fetch` | Also fetches tags (0.44+); rebases descendants of rewritten revisions (0.43+) |
| `git fetch --tags` | `jj git fetch` | Tags fetched like bookmarks (0.44+); Git's `tagOpt` is ignored |
| `git pull` | `jj git fetch` + `jj rebase` | No single command |
| `git push` | `jj git push` | New bookmarks are tracked automatically (`--allow-new` removed in 0.42) |
| `git push --force-with-lease` | `jj git push` | Default behavior — the remote moves only if it matches what jj last fetched |
| `git push --tags` | `jj git push --tag <name>` | `--all` pushes all bookmarks *and* tags (0.44+) |
| `git blame` | `jj file annotate` | |
| `git grep` | `jj file search -p <pattern>` | `-p`/`--pattern` is a required flag (0.44+); positional args are filesets |
| `git grep -l` | `jj file search --name-only -p <pattern>` | Files only — the pre-0.44 default output |
| `git grep -n` | `jj file search -n -p <pattern>` | `-n`/`--line-number` (0.44+) |
| `git reflog` | `jj op log` | More powerful |
| `git tag` | `jj tag list` / `jj tag set` | Tags are tracked, fetched, and pushed like bookmarks (0.44+) — see below |
| `git rebase --exec <cmd>` | `jj run -- <cmd>` | Runs across a whole revset, each revision in its own working copy (0.43+) |
| `git filter-branch --tree-filter` | `jj run --restore-descendants -- <cmd>` | Rewrite content across many commits |
| `git push -o <opt>` | `jj git push --option <opt>` | Or `-o <opt>` shorthand |
| N/A | `jj git push --allow-conflicts` | Push commits that contain conflicts (0.44+) — see "Conflicts as First-Class Citizens" |
| N/A | `jj arrange` | Reorder commits; closest git equivalent is interactive rebase reordering |
| N/A | `jj bookmark advance` | Automatically fast-forward a bookmark to a descendant |
| N/A | `jj util snapshot` | Snapshot working copy into the commit (jj-specific concept) |

## Workflow Comparisons

### Creating a New Commit

**Git:**
```bash
git add .
git commit -m "message"
```

**jj:**
```bash
# Changes are auto-tracked
jj describe -m "message"
jj new  # Start new work
# Or:
jj commit -m "message"  # Same effect
```

### Amending the Last Commit

**Git:**
```bash
git add .
git commit --amend
```

**jj:**
```bash
# Changes automatically amend current working copy
# Just edit files, done!
# To change message:
jj describe -m "new message"
```

### Interactive Staging

**Git:**
```bash
git add -p
git commit
```

**jj:**
```bash
# Split current changes into separate commits
jj split -i
# Or squash parts into parent
jj squash -i
```

### Undoing Last Commit (Keep Changes)

**Git:**
```bash
git reset HEAD~
```

**jj:**
```bash
jj squash  # Moves changes to parent, abandons if empty
```

### Discarding Changes

**Git:**
```bash
git checkout -- .
git reset --hard
```

**jj:**
```bash
jj restore  # Restore from parent
```

### Switching Branches

**Git:**
```bash
git checkout feature
# or
git switch feature
```

**jj:**
```bash
jj new feature  # Create working copy on feature
# Or to edit feature directly:
jj edit feature
```

### Creating a Branch

**Git:**
```bash
git checkout -b feature
# or
git switch -c feature
```

**jj:**
```bash
jj bookmark create feature
# Then work - changes go to working copy
```

### Stashing Changes

**Git:**
```bash
git stash
# ... do other work ...
git stash pop
```

**jj:**
```bash
# Not needed! Working copy is already a commit.
# To work on something else:
jj new main  # Start new work from main
# ... do other work ...
jj edit <original-change-id>  # Go back
```

### Rebasing a Branch

**Git:**
```bash
git checkout feature
git rebase main
```

**jj:**
```bash
jj rebase -b feature -o main
# Or if on feature:
jj rebase -o main
```

### Interactive Rebase

**Git:**
```bash
git rebase -i HEAD~5
```

**jj:**
```bash
# Different approach - use individual commands:
jj squash        # Combine commits
jj split         # Split commits
jj rebase        # Reorder
jj describe      # Edit messages
jj abandon       # Drop commits
```

### Cherry-picking

**Git:**
```bash
git cherry-pick <commit>
```

**jj:**
```bash
jj duplicate <commit> -o main   # Copy the commit onto main, one step
jj duplicate <commit> -A @      # Or place it after the working copy
```

`-A`/`--insert-after` and `-B`/`--insert-before` place the copy relative to an
existing revision instead of onto a destination.

### Resolving Conflicts

**Git:**
```bash
git merge feature
# Conflicts occur
# Edit files
git add .
git commit
```

**jj:**
```bash
jj new main feature  # Create merge (may have conflicts)
# Conflicts are recorded in commit
jj log  # Shows conflict marker
# Edit files
# Changes auto-commit, conflict resolved
```

### Undoing Operations

**Git:**
```bash
git reflog
git reset --hard HEAD@{2}
```

**jj:**
```bash
jj op log
jj op revert <op-id>  # Revert a specific operation
# Or simply:
jj undo               # Undo last operation
jj redo               # Redo last undone operation
# Note: `jj op undo` was removed in 0.39 — use `jj op revert` or `jj undo`
```

### Viewing History at a Point

**Git:**
```bash
git log HEAD@{yesterday}
```

**jj:**
```bash
jj --at-op=<op-id> log
```

## Conceptual Differences

### No Staging Area

Git has a staging area (index) between working directory and commits. jj doesn't:

- **Git**: working directory → staging area → commit
- **jj**: working copy (IS a commit) → new commit

### Working Copy is a Commit

In jj, the working copy is always a commit. Changes are automatically recorded:

- No "dirty" working directory
- No lost changes from checkout
- Can always undo

### Change IDs vs Commit IDs

- **Git**: Only commit hashes (SHA), change when commit is amended
- **jj**: Change IDs (stable) + Commit IDs (change on rewrite)

Use change IDs (`kntqzsqt`) when referring to commits.

### Conflicts as First-Class Citizens

- **Git**: Conflicts block operations, must resolve immediately
- **jj**: Conflicts are recorded in commits, resolve when convenient

A conflicted commit has no native Git representation. When one is exported (or
pushed with `--allow-conflicts`), the Git commit gets root directories named
`.jjconflict-base-*/` and `.jjconflict-side-*/`, and the real path holds the
**first side** of the conflict — not conflict markers. Those directories exist
only to keep the relevant trees from being garbage-collected; the authoritative
state lives in a non-standard `jj:trees` commit header.

You won't see any of this while using jj. But if Git tooling checks such a
commit out (e.g. `git switch`), those directories appear in the working copy,
and a subsequent `jj status` will snapshot them as if you had added them —
`jj abandon` gets you back to the unresolved-conflict state.

### Operations are Atomic

Every jj operation is recorded and reversible:

```bash
jj op log      # See all operations
jj undo        # Undo last operation
jj redo        # Redo last undone operation
jj op revert   # Revert a specific operation
```

### Bookmarks vs Branches

- **Git branches**: Automatically move with commits
- **jj bookmarks**: Named pointers, move explicitly

```bash
# Git: branch moves with HEAD
git commit  # branch advances

# jj: bookmark stays unless moved
jj new      # bookmark doesn't move
jj bookmark set <name>  # explicit move
```

## Common Patterns

### "Pull and Rebase"

**Git:**
```bash
git pull --rebase
```

**jj:**
```bash
jj git fetch
jj rebase -o main@origin  # <bookmark>@<remote>
```

Since 0.43, `jj git fetch` also rebases the descendants of revisions the remote
rewrote (matched by change ID), so recovering from someone else's force-push is
usually just a fetch. Immutable descendants are not rebased.

### "Push New Branch"

**Git:**
```bash
git push -u origin feature
```

**jj:**
```bash
jj git push --bookmark feature
# Or create bookmark from change:
jj git push --change <change-id>
```

A bookmark that isn't tracking anything yet is tracked automatically, so there
is no upstream to set separately. The `--allow-new` flag that used to gate this
was removed in 0.42 and is now an unknown-argument error.

### "Push a Tag"

**Git:**
```bash
git tag v1.0
git push origin v1.0
git push --tags
```

**jj:**
```bash
jj tag set v1.0 -r @
jj git push --tag v1.0
jj git push --all          # all bookmarks AND tags (0.44+)
```

As of 0.44 tags behave like bookmarks: `jj git fetch` fetches them as
`<name>@<remote>`, fetched tags are tracked by local tags of the same name, and
tracked tags are pushed by default. `jj tag track` / `jj tag untrack` control
tracking, and `<name>@<remote>` resolves as a revision:

```bash
jj tag list --all-remotes     # see local + remote tags and tracking state
jj tag untrack 'v1.0@origin'
jj log -r 'v1.0@origin'
```

Git's `tagOpt` is no longer respected; disable tag fetching in jj config with
`remotes.<remote>.fetch-tags = '~*'`. Note that `tags()` is part of
`builtin_immutable_heads()`, so newly tracked tags can make more history
immutable.

### "Run a Command Over Every Commit"

**Git:**
```bash
git rebase --exec 'cargo check' main   # stops at the first failure
```

**jj:**
```bash
jj run -- cargo check                       # rewrites commits with the result
jj run --ignore-changes -- cargo check      # read-only; nothing is rewritten
jj run --ignore-changes --ignore-errors -- cargo test   # check the whole stack
```

`jj run` (0.43+) checks out each revision in its own isolated working copy,
runs the command, and amends the revision with the result — see
[faq-patterns.md](faq-patterns.md#running-a-command-over-a-stack-jj-run).
Unlike `git rebase --exec` it doesn't leave you in a detached, half-finished
rebase when a command fails, and unlike `git filter-branch` it operates on a
revset and propagates conflicts instead of aborting.

### "Squash Last N Commits"

**Git:**
```bash
git rebase -i HEAD~3
# Mark commits as squash
```

**jj:**
```bash
# Squash into parent repeatedly:
jj squash -r <commit>
jj squash -r <commit>
# Or use revsets:
jj squash --from 'trunk()..@'
```

### "Search File Contents"

**Git:**
```bash
git grep "pattern"
git grep -n "pattern" -- "*.py"
git grep -l "pattern"
```

**jj:**
```bash
jj file search -p "pattern"                   # path:line for every match
jj file search -n -p "pattern" 'glob:**/*.py' # + line numbers, limited to a fileset
jj file search --name-only -p "pattern"       # file paths only
```

Two 0.44 changes make this a much closer analog of `git grep`:

- Output is now every matched line prefixed by its file path (previously: only
  file paths). `--name-only` restores the old behavior.
- `-p`/`--pattern` is a **required flag**. Positional arguments are
  [filesets](filesets.md) that narrow *which files* are searched — they are not
  the pattern. `jj file search "foo"` is an error on 0.44.

The pattern takes `kind:pattern` syntax and defaults to `regex:`. A `glob:`
pattern must match the **whole line**, so wrap it in wildcards:

```bash
jj file search -p 'glob:*foo*'   # not 'glob:foo'
jj file search -r <rev> -p foo   # search a revision other than @
```

### "Push with Options"

**Git:**
```bash
git push -o ci.skip
```

**jj:**
```bash
jj git push --option ci.skip
jj git push -o ci.skip  # Shorthand
```

### "Reorder Commits"

**Git:**
```bash
git rebase -i HEAD~5
# Manually reorder lines in editor
```

**jj:**
```bash
jj arrange   # Reorder commits interactively
```

### "Edit Old Commit"

**Git:**
```bash
git rebase -i <commit>^
# Mark commit as edit
# Make changes
git commit --amend
git rebase --continue
```

**jj:**
```bash
jj edit <commit>
# Make changes (auto-committed)
jj new  # Continue with new work
# Descendants auto-rebased!
```
