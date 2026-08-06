# Revsets Reference

Revsets are a functional language for selecting commits in jj. This reference covers all operators, functions, and patterns.

## Table of Contents

- [Basic Symbols](#basic-symbols)
- [Symbol Resolution Priority](#symbol-resolution-priority)
- [Operators](#operators)
- [Functions](#functions)
- [String Patterns](#string-patterns)
- [Date Patterns](#date-patterns)
- [Revset Aliases](#revset-aliases)
- [Common Patterns](#common-patterns)
- [Deprecations and Removals](#deprecations-and-removals)

## Basic Symbols

| Symbol | Description |
|--------|-------------|
| `@` | Working copy commit |
| `<workspace>@` | Working copy in another workspace |
| `root()` | Repository root (empty commit) |
| `<change_id>` | Commit by change ID (e.g., `kntqzsqt`) |
| `<commit_id>` | Commit by commit hash (prefix ok) |
| `<bookmark>` | Commit at bookmark (e.g., `main`) |
| `<tag>` | Commit at tag |
| `<name>@<remote>` | Remote-tracking bookmark **or tag** (e.g., `main@origin`, `v1.0@origin`) |

## Symbol Resolution Priority

jj resolves a symbol in this order:

1. Tag name
2. Bookmark name
3. Commit ID or change ID

To override, use `commit_id(abc)` or `change_id(abc)` explicitly — useful in scripts where a bookmark might shadow a commit ID.

**Removed in 0.43:** the Git-ref lookup step. Git-like symbols (`refs/heads/main`, `refs/tags/v1.0`) no longer resolve to revisions and now error with "Revision ... doesn't exist". Use the plain `<name>` or `<name>@<remote>` form instead.

## Operators

### Parent/Child Navigation

| Operator | Description | Example |
|----------|-------------|---------|
| `x-` | Parents of x | `@-` (parent of working copy) |
| `x+` | Children of x | `main+` (children of main) |
| `x--` | Grandparents | `@--` |
| `x++` | Grandchildren | `main++` |

### Ancestry/Descendant

| Operator | Description | Example |
|----------|-------------|---------|
| `::x` | Ancestors of x (inclusive) | `::@` |
| `x::` | Descendants of x (inclusive) | `main::` |
| `x::y` | DAG path from x to y | `main::@` |
| `::` | All visible commits — same as `all()` | `jj log -r ::` |

### Range

| Operator | Description | Example |
|----------|-------------|---------|
| `x..y` | Ancestors of y minus ancestors of x | `main..@` |
| `x..` | Commits that are **not ancestors** of x (`~::x`) | `main..` |
| `..y` | Ancestors of y, excluding the root commit | `..@` |
| `..` | All visible commits except root — same as `~root()` | `jj log -r ..` |

`x..` is *not* "descendants of x" — it includes sibling branches. Given `A` with
children `B` and `C`, both merged into `D`, `B..` is `{C, D}` (C is not a
descendant of B). Note also that `..` does not distribute over union on its left:
`(A | B)..` equals `A.. & B..`, not `A.. | B..`.

### Set Operations

| Operator | Description | Example |
|----------|-------------|---------|
| `x \| y` | Union (x or y) | `main \| develop` |
| `x & y` | Intersection (x and y) | `mine() & ::@` |
| `x ~ y` | Difference (x minus y) | `::@ ~ ::main` |
| `~x` | Complement (not x) | `~empty()` |

### Grouping

Use parentheses for grouping: `(x | y) & z`

## Functions

### Commit Selection

| Function | Description |
|----------|-------------|
| `all()` | All visible commits |
| `none()` | Empty set |
| `visible_heads()` | Visible branch heads |
| `heads(x)` | Commits in x with no descendants in x |
| `roots(x)` | Commits in x with no ancestors in x |
| `latest(x, [count])` | Latest `count` commits from x by committer timestamp (default: 1) |
| `fork_point(x)` | Common ancestor(s) of all commits in x. Equivalent to `heads(::x_1 & ::x_2 & ... & ::x_N)`. Single commit resolves to itself |
| `merge_point(x)` | Takes **one** revset. Common descendant(s) of all commits in x that have no ancestors which are also common descendants of all commits in x. Equivalent to `roots(x_1:: & x_2:: & ... & x_N::)`. Single commit resolves to itself (0.44+) |
| `bisect(x)` | Finds commits where roughly half of x are descendants — useful for bisection workflows |
| `exactly(x, count)` | Returns x, errors if set size is not exactly `count`. Use `exactly(x, 1)` to assert single commit |
| `merges()` | Merge commits (more than 1 parent) |
| `forks()` | Fork commits — more than 1 child (0.43+) |

### Bookmarks and Tags

| Function | Description |
|----------|-------------|
| `bookmarks([pattern])` | All local bookmark targets, optionally filtered by pattern |
| `remote_bookmarks([name], [remote=pattern])` | All remote bookmark targets. Use `remote="git"` or `remote="*"` to include `@git` bookmarks |
| `tracked_remote_bookmarks([name], [remote=pattern])` | Tracked remote bookmarks, same optional args as `remote_bookmarks()` |
| `untracked_remote_bookmarks([name], [remote=pattern])` | Untracked remote bookmarks, same optional args as `remote_bookmarks()` |
| `tags([pattern])` | All tag targets |
| `remote_tags([name], [remote=pattern])` | Remote tags (0.38+) |
| `trunk()` | Main branch (main, master, trunk) |

### Author/Committer

| Function | Description |
|----------|-------------|
| `author(pattern)` | Match author name or email. Equivalent to `author_name(p) \| author_email(p)` |
| `author_name(pattern)` | Match author name only |
| `author_email(pattern)` | Match author email only |
| `author_date(pattern)` | Match author date (see [Date Patterns](#date-patterns)) |
| `committer(pattern)` | Match committer name or email. Equivalent to `committer_name(p) \| committer_email(p)` |
| `committer_name(pattern)` | Match committer name only |
| `committer_email(pattern)` | Match committer email only |
| `committer_date(pattern)` | Match committer date (see [Date Patterns](#date-patterns)) |
| `mine()` | Commits by configured user email. Equivalent to `author_email(exact-i:<user-email>)` |

### Content

| Function | Description |
|----------|-------------|
| `description(pattern)` | Match commit description. Note: non-empty descriptions usually end with newline |
| `subject(pattern)` | Match first line of description only (without trailing newline) |
| `empty()` | Empty commits (no file changes). Includes `merges()` without user modifications and `root()` |
| `files(expression)` | Commits modifying paths matching fileset expression. Paths relative to cwd |
| `diff_lines(text, [files])` | Commits with matching diff content (renamed from `diff_contains()` in 0.38) |
| `diff_lines_added(text, [files])` | Match content on added side of diff only (0.40+) |
| `diff_lines_removed(text, [files])` | Match content on removed side of diff only (0.40+) |

### Conflicts and Status

| Function | Description |
|----------|-------------|
| `conflicts()` | Commits containing conflicts |
| `divergent()` | Divergent changes — multiple visible commits with the same change ID (0.38+) |
| `signed()` | Cryptographically signed commits |
| `working_copies()` | All working copy commits across all workspaces |
| `at_operation(op, x)` | Evaluate revset x at a specific operation. E.g., `at_operation(@-, visible_heads())` returns heads at the previous operation |

### Mutability

| Function | Description |
|----------|-------------|
| `mutable()` | Commits that can be rewritten (`~immutable()`) |
| `immutable()` | Protected commits (`::(immutable_heads() \| root())`) |
| `immutable_heads()` | Heads of immutable set (default: `trunk() \| tags() \| untracked_remote_bookmarks()`) |

**Tags now grow the immutable set (0.44+).** `jj git fetch` fetches tags as `<name>@<remote>` and tracks them with a local tag of the same name automatically, so a fetch that brings in a new tag adds it to `tags()` and therefore to `immutable_heads()`. If a newly-fetched tag makes commits you were editing immutable, delete the local tag with `jj tag delete <name>` (`jj tag untrack <name>@<remote>` only stops tracking — it leaves the local tag, and `tags()`, unchanged), or override `immutable_heads()` to drop `tags()`.

### Ancestry and Navigation

| Function | Description |
|----------|-------------|
| `ancestors(x, [depth])` | Same as `::x`. With depth, limits ancestor traversal |
| `descendants(x, [depth])` | Same as `x::`. With depth, limits descendant traversal |
| `parents(x, [depth])` | Parents of x. With depth, `parents(x, 3)` is equivalent to `x---` |
| `children(x, [depth])` | Children of x. With depth, `children(x, 3)` is equivalent to `x+++` |
| `first_parent(x, [depth])` | Like `parents(x)`, but only returns first parent of merges. `first_parent(x, 2)` = `first_parent(first_parent(x))` |
| `first_ancestors(x, [depth])` | Like `ancestors(x)`, but only traverses first parent of each commit. Useful for following the merge-target branch, excluding changes from other branches |
| `connected(x)` | Same as `x::x` — all commits both descended from and ancestral to commits in x |
| `reachable(srcs, domain)` | All commits reachable from srcs within domain, traversing parent and child edges. **Key pattern:** `reachable(@, mutable())` returns your working stack |

### Identity and Utility

| Function | Description |
|----------|-------------|
| `change_id(prefix)` | Commits with given change ID prefix (handles divergent changes) |
| `commit_id(prefix)` | Commits with given commit ID prefix |
| `present(x)` | x if all commits exist, else `none()` — prevents errors on unknown bookmarks |
| `coalesce(revsets...)` | First non-empty revset from the list. If all empty, returns `none()` |

## String Patterns

Used in functions like `bookmarks()`, `description()`, `author()`.

Since 0.37, the default pattern type is `glob` (previously `substring`). Use explicit prefixes to be clear:

| Pattern | Description | Example |
|---------|-------------|---------|
| `glob:"pattern"` | Glob pattern, must match the **whole** string (default since 0.37) | `bookmarks("feature-*")` |
| `substring:"text"` | Contains text | `description(substring:"fix")` |
| `exact:"text"` | Exact match | `description(exact:"")` |
| `regex:"pattern"` | Regular expression, **unanchored** (matches substrings) | `author(regex:"^J.*")` |

Note the asymmetry: `description(glob:"world")` does *not* match "hello world"
(use `glob:"*world*"`), but `description(regex:"world")` does.

### Case-Insensitive Matching

Append `-i` after the pattern kind to match case-insensitively:

```
glob-i:"fix*jpeg*"
exact-i:"TODO"
regex-i:"error|warning"
substring-i:"refactor"
```

### Pattern Logical Operators

String patterns support logical operators for combining:

| Operator | Description | Example |
|----------|-------------|---------|
| `~x` | Not matching x | `bookmarks(~glob:"ci/*")` |
| `x & y` | Matching both x and y | `bookmarks(glob:"feature-*" & ~glob:"*wip*")` |
| `x ~ y` | Matching x but not y | `bookmarks(glob:"release-*" ~ glob:"*rc*")` |
| `x \| y` | Matching x or y | `bookmarks(glob:"fix-*" \| glob:"bug-*")` |

### Pattern Aliases (0.39+)

Define custom pattern prefixes in the config:

```toml
[revset-aliases]
'grep:x' = 'description(regex:x)'
```

Then use as `grep:"pattern"` in revset expressions.

## Date Patterns

For `author_date()` and `committer_date()`:

| Pattern | Description | Example |
|---------|-------------|---------|
| `after:"date"` | At or after the given date | `author_date(after:"2024-01-01")` |
| `before:"date"` | Before (not including) the given date | `committer_date(before:"yesterday")` |

### Supported Date Formats

| Format | Example |
|--------|---------|
| Date only | `2024-02-01` |
| Date and time | `2024-02-01T12:00:00` |
| With timezone | `2024-02-01T12:00:00-08:00` |
| Space separator | `2024-02-01 12:00:00` |
| Relative | `2 days ago`, `5 minutes ago` |
| Named relative | `yesterday`, `yesterday 5pm`, `yesterday 15:30` |

## Revset Aliases

Define custom symbols, functions, and patterns in the config:

```toml
[revset-aliases]
HEAD = '@-'
'user()' = 'user("me@example.org")'
'user(x)' = 'author(x) | committer(x)'
```

Alias functions can be overloaded by number of parameters.

### Alias with Documentation

Aliases can include descriptions surfaced in shell completions:

```toml
[revset-aliases]
HEAD = { definition = '@-', doc = 'Parent of working copy' }
```

### Pattern Aliases (0.39+)

Custom `<name>:<value>` patterns can be defined as aliases:

```toml
[revset-aliases]
'grep:x' = 'description(regex:x)'
```

### Built-in Aliases

| Alias | Default Definition |
|-------|-------------------|
| `trunk()` | Head of default bookmark on default remote (falls back to `main`/`master`/`trunk` on `upstream`/`origin`) |
| `immutable_heads()` | `trunk() \| tags() \| untracked_remote_bookmarks()` |
| `immutable()` | `::(immutable_heads() \| root())` |
| `mutable()` | `~immutable()` |
| `builtin_immutable_heads()` | Same as default `immutable_heads()` — override `immutable_heads()` instead of this |
| `builtin_log()` | `present(@) \| ancestors(immutable_heads().., 2) \| trunk()` — the default value of `revsets.log` (0.44+) |
| `visible()` | `::visible_heads()` — equals `all()` unless the revset mentions hidden commits |
| `hidden()` | `~visible()` |

Override `trunk()` for custom setups:

```toml
[revset-aliases]
'trunk()' = 'your-bookmark@your-remote'
```

### Extending the Default Log (0.44+)

`revsets.log` now defaults to `builtin_log()`, so a custom log revset can extend
the built-in default instead of copying its full expression:

```toml
[revsets]
log = "builtin_log() | mybranch"
```

### Useful Custom Aliases

```toml
[revset-aliases]
# All mutable ancestors + descendants of a revision (your "active" work context)
'active(rev)' = '(ancestors(rev) | descendants(rev)) ~ immutable()'

# Everything related to current work
work = 'active(@)'

# WIP commits by description pattern
'wip' = 'description(regex:"^\\[(wip|WIP|todo|TODO)\\]|(wip|WIP|todo|TODO):?")'
```

Use per-repo config (`jj config edit --repo`) to keep project-specific aliases isolated.

## Common Patterns

### Working with Current Work

```bash
# My work in progress
jj log -r 'trunk()..@'

# My working stack (all connected mutable commits)
jj log -r 'reachable(@, mutable())'

# My recent changes
jj log -r 'mine() & ancestors(@, 20)'

# Empty commits I made (WIP markers)
jj log -r 'mine() & empty()'

# Commits with empty descriptions
jj log -r 'description(exact:"")'
```

### Branch Operations

```bash
# Commits on feature branch not on main
jj log -r 'main..feature'

# All commits on any feature branch
jj log -r 'bookmarks(glob:"feature-*")::'

# Diverged commits
jj log -r 'heads(trunk()..)'

# Bookmarks not matching a pattern
jj log -r 'bookmarks(~glob:"ci/*")'
```

### Finding Commits

```bash
# Commits touching specific file
jj log -r 'files("src/main.rs")'

# Commits containing "TODO" in diff
jj log -r 'diff_lines("TODO")'

# Commits by specific author
jj log -r 'author("alice@")'

# Commits by author name only
jj log -r 'author_name("Alice")'

# Commits from last week
jj log -r 'committer_date(after:"1 week ago")'

# Search first line of commit message
jj log -r 'subject(glob:"fix*")'

# Signed commits in my branch
jj log -r 'signed() & trunk()..@'
```

### Conflicts

```bash
# All conflicted commits
jj log -r 'conflicts()'

# Conflicted commits in my branch
jj log -r 'conflicts() & trunk()..@'
```

### Rebasing Patterns

```bash
# Rebase entire branch onto trunk
jj rebase -s 'roots(trunk()..@)' -o trunk()

# Rebase all mutable descendants
jj rebase -s 'roots(mutable())' -o <dest>

# Find commits to squash (empty changes)
jj log -r 'empty() & trunk()..@'
```

### Working Copies (Multiple Workspaces)

```bash
# All working copy commits
jj log -r 'working_copies()'

# Current workspace's working copy
jj log -r '@'
```

### Operation History

```bash
# Heads visible at the previous operation
jj log -r 'at_operation(@-, visible_heads())'

# What changed since last operation
jj log -r 'at_operation(@-, @) ~ @'
```

### Safety and Assertions

```bash
# Ensure exactly one commit matches (errors otherwise)
jj log -r 'exactly(bookmarks("release-*"), 1)'

# Use present() to avoid errors on missing bookmarks
jj log -r 'present(feature-branch) | trunk()'

# First non-empty of several fallback revsets
jj log -r 'coalesce(bookmarks("main"), bookmarks("master"), trunk())'
```

## Combining Expressions

Complex queries combine operators and functions:

```bash
# My non-empty commits on feature branch, excluding conflicts
jj log -r '(mine() & feature::@) ~ (empty() | conflicts())'

# Latest 5 commits touching src/ by any author
jj log -r 'latest(files("src/**"), 5)'

# All commits between two tags
jj log -r 'v1.0::v2.0'
```

## Deprecations and Removals

Kept as a migration table — you will still meet these in older configs and blog posts.

| Old form | Status | Replacement |
|----------|--------|-------------|
| `git_head()` | **Removed in 0.43** (deprecated 0.37) | `first_parent(@)` in colocated repos |
| `git_refs()` | **Removed in 0.43** (deprecated 0.37) | `remote_bookmarks(remote=glob:*) \| tags()` |
| Git-like symbols (`refs/heads/main`, `refs/tags/v1.0`) | **Removed in 0.43** | `<name>` or `<name>@<remote>` |
| `ui.revsets-use-glob-by-default` config | **Removed in 0.43** (now a silently-ignored key) | Glob is the default pattern kind since 0.37; use an explicit `substring:` prefix for the old behavior |
| `<kind>:<bookmark>@<remote>` in `jj bookmark track`/`untrack` | **Removed in 0.43** | Plain `<bookmark>@<remote>` symbol (still supported) |
| `diff_contains(pattern)` | Renamed in 0.38 | `diff_lines(pattern)` |
| The `all:` revset modifier and `ui.always-allow-large-revsets` setting | Removed in 0.38 — `all:x` is now a **parse error** | Multi-commit revsets are accepted directly by most commands |
| `file(pattern)` | Renamed | `files(expression)` — now takes fileset expressions |

## Advanced Recipes

### Rebasing Entire Branch Tree onto New Base

When you have multiple feature branches and want to rebase all onto new upstream:

```bash
# Find roots of all branches leading to integration commit, excluding main:
jj rebase -s 'roots(::my-integration ~ ::main)' -o main
```

This pattern works by:
1. `::my-integration` - All ancestors of integration branch
2. `~ ::main` - Subtract ancestors of main (the difference)
3. `roots(...)` - Find root commits of that set (the branch points)

jj automatically rebases all descendants when you rebase the roots.

**Example workflow:**
```bash
# Verify current state
jj log -r 'main | feature-a | feature-b | integration'

# Rebase all feature branch roots onto updated main
jj rebase -s 'roots(::integration ~ ::main)' -o main

# Handle any conflicts that arise
jj log -r 'conflicts()'
```

### Finding Branch Divergence

```bash
# Commits in feature not in main:
jj log -r '::feature ~ ::main'

# Commits that diverged (in both since common ancestor):
jj log -r '(::feature | ::main) ~ ::trunk()'

# Common ancestor of two branches:
jj log -r 'heads(::feature & ::main)'

# Fork point of multiple branches:
jj log -r 'fork_point(feature-a | feature-b)'

# Merge point of multiple branches — one revset arg, like fork_point() (0.44+):
jj log -r 'merge_point(feature-a | feature-b)'

# Every point in your stack where history branched (0.43+):
jj log -r 'forks() & mutable()'
```

### Working with Multiple Feature Branches

```bash
# All feature branch heads:
jj log -r 'heads(bookmarks(glob:"feature-*")::)'

# Commits unique to each feature branch:
jj log -r '(::feature-a ~ ::main) | (::feature-b ~ ::main)'

# Find all roots of your work (useful before complex rebase):
jj log -r 'roots(mine() & trunk()..@)'
```

### Navigating Merge History

```bash
# Follow only first-parent lineage (like git log --first-parent):
jj log -r 'first_ancestors(@, 20)'

# First parent of a merge commit:
jj log -r 'first_parent(@)'

# Commits on the merge-target branch only:
jj log -r 'first_ancestors(@) & trunk()..@'
```

### Finding Merge Commits

```bash
# All merge commits in your branch:
jj log -r 'merges() & trunk()..@'

# Commits with merge descriptions in your branch:
jj log -r 'trunk()..@ & description(glob:"*merge*")'
```

### Multi-Commit Arguments (no more `all:`)

The `all:` revset modifier was **removed in 0.38** — `all:wip` is now a parse
error (`` `:` is not an infix operator ``). Most commands that take revsets
(`jj rebase`, `jj new`, `jj abandon`, `jj duplicate`, `jj describe`, ...) accept
multi-commit revsets directly — their usage line reads `[REVSETS]...`:

```bash
# Rebase every WIP-tagged branch at once — no prefix needed
jj rebase -b wip -o main
```

Useful WIP revset alias:

```toml
[revset-aliases]
'wip' = 'description(regex:"^\\[(wip|WIP|todo|TODO)\\]|(wip|WIP|todo|TODO):?")'
```

Commands that genuinely operate on one commit (`jj edit`, `jj split -r`) still
reject multi-commit revsets — their usage line reads `<REVSET>`:

```
Error: Revset `B|C` resolved to more than one revision
```

Narrow the revset instead — e.g. `heads(wip)`, `latest(wip)`, or
`exactly(wip, 1)` to assert there is only one.

### Complex Rebase Scenarios

```bash
# Rebase preserving branch structure (roots only):
jj rebase -s 'roots(::feature-integration ~ ::main)' -o main

# Rebase single commit without descendants:
jj rebase -r <rev> -o main

# Insert commit between two others:
jj rebase -r <commit> -A <after-this>
jj rebase -r <commit> -B <before-this>
```
