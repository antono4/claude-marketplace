# Configuration Reference

Comprehensive reference for jj configuration options, templates, filesets, and aliases.

## Table of Contents

- [Config Files](#config-files)
- [User Settings](#user-settings)
- [UI Settings](#ui-settings)
  - [Basic UI](#basic-ui)
  - [Editor Settings](#editor-settings)
  - [Colors and Styles](#colors-and-styles)
  - [Diff Options](#diff-options)
  - [External Diff Tools](#external-diff-tools)
  - [Conflict Marker Styles](#conflict-marker-styles)
- [Aliases](#aliases)
- [Templates](#templates)
- [Filesets](#filesets)
- [Git Settings](#git-settings)
  - [Multiple Remotes](#multiple-remotes)
  - [Per-Remote Settings](#per-remote-settings)
  - [Tags (0.44+)](#tags-044)
- [Working Copy](#working-copy)
- [Conditional Configuration](#conditional-configuration)
- [Signing](#signing)
- [Immutable Commits](#immutable-commits)
- [Fix Tools (Formatters)](#fix-tools-formatters)
- [Merge Tools](#merge-tools)
- [Snapshot Settings](#snapshot-settings)
- [Merge Settings](#merge-settings)
- [Run Settings (0.43+)](#run-settings-043)
- [Debug Settings](#debug-settings)

## Config Files

jj loads configuration from multiple sources (in order of precedence — later wins):

1. **Built-in defaults** - Cannot be edited; inspect with `jj config list --include-defaults`
2. **System (0.43+, Unix only)** - `/etc/jj/config.toml`, then `/etc/jj/conf.d/*.toml`
3. **User** - `$HOME/.jjconfig.toml`, then `<config-dir>/jj/config.toml`, then `<config-dir>/jj/conf.d/*.toml`
4. **Repo** - Stored outside the repo (managed by jj)
5. **Workspace** - Stored outside the repo (managed by jj)
6. **Command-line** - `--config key=value` / `--config-file PATH`

`<config-dir>` is `$XDG_CONFIG_HOME` (or `~/.config`) on Linux/macOS and `%APPDATA%` on Windows. Files in `conf.d/` load in lexicographic order — handy for splitting config across files (see [Conditional Configuration](#conditional-configuration)).

Setting `JJ_CONFIG` (even to an empty string) replaces **all** config-file locations, including `/etc/jj`. `JJ_CONFIG=` disables user/system config entirely.

> **Tip:** Enable JSON Schema validation in your editor by adding
> `#:schema https://docs.jj-vcs.dev/latest/config-schema.json` at the top of your TOML config files.

> **Note (0.38+):** Per-repo and per-workspace config are now stored outside the repository directory. The legacy locations `.jj/repo/config.toml` and `.jj/workspace-config.toml` are auto-migrated on first access. Use `jj config edit --repo` / `--workspace` to edit — jj manages the paths.

```bash
jj config path --user            # Show user config path
jj config edit --user            # Edit user config
jj config edit --repo            # Edit repo config
jj config list                   # Show all config values
jj config list --include-defaults # Include built-in defaults (use this to audit)
jj config get <key>              # Get specific value
jj config gc                     # (0.43+) Prune repo-level config for repos that no longer exist
```

> **Warning:** jj does **not** warn about unknown config keys. A key removed in a newer release (e.g. `git.auto-local-bookmark`) is silently ignored — `jj config get <key>` reporting "Value not found" for something you set means the key is no longer recognized.

## User Settings

```toml
[user]
name = "Your Name"
email = "your@email.com"
```

## UI Settings

### Basic UI

```toml
[ui]
# Color output: always, never, auto, debug
color = "auto"

# Default command when running 'jj' with no args
default-command = "log"
# Or with arguments:
default-command = ["log", "--reversed"]

# Pager command (also overridable via JJ_PAGER env var)
pager = "less -FRX"

# Diff format: :color-words, :git, :summary, :stat, :types, :name-only
diff-formatter = ":color-words"

# Movement commands (next/prev) edit instead of creating new commit
movement.edit = false
```

### Editor Settings

The editor is used for commands that need text input: `jj describe`, `jj squash` (when combining messages), `jj commit`, `jj split` (for commit messages).

**Editor priority order** (highest to lowest):

```
$JJ_EDITOR > ui.editor > $VISUAL > $EDITOR
```

**Pager priority order** (highest to lowest):

```
$JJ_PAGER > ui.pager > $PAGER
```

If none are set, defaults to `nano` (Unix) or `notepad` (Windows).

**Terminal editors:**

```toml
[ui]
editor = "vim"
editor = "nvim"
editor = "nano"
editor = "emacs"
editor = "micro"
editor = "helix"
```

**GUI editors** (require wait flag to block until closed):

```toml
[ui]
editor = "code -w"           # VS Code
editor = "code.cmd -w"       # VS Code on Windows
editor = "subl -n -w"        # Sublime Text
editor = "bbedit -w"         # BBEdit
editor = "mate -w"           # TextMate
editor = "idea --temp-project --wait"  # IntelliJ IDEA

# Array syntax for complex commands:
editor = ["C:/Program Files/Notepad++/notepad++.exe",
    "-multiInst", "-notabbar", "-nosession", "-noPlugin"]
```

**Quick config commands:**

```bash
# Set editor in user config
jj config set --user ui.editor "nvim"

# Set editor in repo config
jj config set --repo ui.editor "code -w"

# Check current editor setting
jj config get ui.editor
```

**Editor vs Diff-Editor:**

| Setting | Used For | Default |
|---------|----------|---------|
| `ui.editor` | Commit messages (`describe`, `squash`, `commit`) | `nano` |
| `ui.diff-editor` | Interactive diff editing (`split`, `squash -i`, `diffedit`) | `:builtin` |
| `ui.merge-editor` | Conflict resolution (`resolve`) | (none) |

### Colors and Styles

```toml
[colors]
# Simple foreground color
commit_id = "green"
change_id = "magenta"

# Hex colors
bookmark = "#ff1525"

# ANSI 256-color palette
commit_id = "ansi-color-81"

# Full style specification.
# Table keys: fg, bg, bold, dim, italic, underline, reverse, crossed-out (0.43+)
commit_id = { fg = "green", bg = "black", bold = true }
change_id = { underline = true, italic = true }
elided = { crossed-out = true }        # 0.43+

# Combined labels (like CSS selectors)
"working_copy commit_id" = { underline = true }
"conflict description" = "red"

# Diff colors
"diff removed" = "red"
"diff added" = "green"
"diff removed token" = { bg = "#221111", underline = false }
"diff added token" = { bg = "#002200", underline = false }
```

### Diff Options

```toml
[diff.color-words]
# Max removed/added alternation to inline (-1 = all)
max-inline-alternation = 3
# Lines of context
context = 3

[diff.git]
context = 3
# Suppress a/ b/ path prefixes in diff --git output
show-path-prefix = true

# 0.44+: cap the width of the `++--` bar in diff stats (jj show --stat, jj diff --stat).
# Unset = unbounded (uses remaining width, minimum 30% of total).
[diff.stat]
max-bar-width = 10
```

### External Diff Tools

```toml
[ui]
diff-formatter = ["difft", "--color=always", "$left", "$right"]
# Or reference a named tool:
diff-formatter = "difftastic"

[merge-tools.difftastic]
program = "difft"
diff-args = ["--color=always", "$left", "$right"]
# How to invoke for diff *viewing*: "dir" (default) or "file-by-file"
diff-invocation-mode = "dir"
# 0.42+: same, but for diff *editing* (jj diffedit, jj split, jj squash -i).
# Set independently of diff-invocation-mode. "file-by-file" launches the editor
# once per changed file, which is what makes per-file tools like vimdiff usable.
edit-invocation-mode = "file-by-file"
```

### Conflict Marker Styles

```toml
[ui]
# Conflict marker style: "diff" (default), "snapshot", "git"
conflict-marker-style = "diff"
```

**Available styles:**

| Style | Markers | Best For |
|-------|---------|----------|
| `diff` (default) | `<<<<<<<`, `%%%%%%%` (diff), `+++++++` (snapshot), `>>>>>>>` | Most powerful — shows a snapshot section plus a diff to apply. Apply the diff hunks to the snapshot to resolve. |
| `snapshot` | `+++++++`, `-------` | Shows full content of each side. Simpler but requires manual comparison. |
| `git` | `<<<<<<<`, `\|\|\|\|\|\|\|`, `=======`, `>>>>>>>` | Git-compatible diff3 style. Best for tools that expect Git conflict markers. |

## Aliases

### Command Aliases

```toml
[aliases]
# Simple alias
l = ["log", "-r", "@::"]

# Complex alias
show-tree = ["log", "-r", "@::", "--no-graph", "-T", "commit_id.short() ++ ' ' ++ description.first_line()"]

# External command (via util exec)
my-script = ["util", "exec", "--", "my-jj-script"]

# Inline script
format = ["util", "exec", "--", "bash", "-c", """
set -euo pipefail
jj fix
""", ""]

# Alias with a description (0.42+): a table with .definition and .doc.
# Shell completions surface .doc as the alias description.
wip = { definition = ["log", "-r", "mine() & description(exact:'')"], doc = "List my undescribed changes" }
```

> **Note:** Command aliases cannot be loaded from `--config` / `--config-file`, nor from the repo config reached via `-R`/`--repository` — jj warns and ignores them. The other alias kinds work from any source.

### Revset Aliases

```toml
[revset-aliases]
# Custom revsets
'wip' = 'description(exact:"") & mine()'
'stacked' = 'trunk()..@'
'recent' = 'ancestors(@, 20) & mine()'
'feature(x)' = 'bookmarks(glob:"feature-" ++ x ++ "*")::'

# Override built-in trunk() detection
'trunk()' = 'latest(remote_bookmarks(exact:"main", exact:"origin") | remote_bookmarks(exact:"master", exact:"origin"))'

# Customize immutable commits (narrower than the default — DROPS protection of
# untracked remote bookmarks; default is trunk() | tags() | untracked_remote_bookmarks())
'immutable_heads()' = 'trunk() | tags()'

# Pattern aliases (0.39+): use as name:value in revsets
'grep:x' = 'description(regex:x)'

# Alias with description for shell completions (0.39+)
HEAD = { definition = '@-', doc = 'The parent of the working-copy commit' }
```

### Fileset Aliases (0.39+)

```toml
[fileset-aliases]
'src' = 'glob:"src/**"'
'tests' = 'glob:"**/test_*" | glob:"**/tests/**"'
'rust' = 'glob:"**/*.rs"'
'no-lock' = '~glob:"**/Cargo.lock" ~ glob:"**/package-lock.json"'

# 0.42+: the { definition, doc } table form works for fileset-aliases and
# template-aliases too, not just revset-aliases and command aliases.
'src' = { definition = 'glob:"src/**"', doc = 'All source files' }
```

### Built-in Revsets

jj provides built-in revset settings that can be overridden. Defaults below are the real 0.44 values (`jj config list --include-defaults revsets`):

```toml
[revsets]
# Default 'jj log' revset. 0.44+: defaults to the builtin_log() alias, so a
# custom log revset can extend the built-in default instead of copying it.
log = "builtin_log()"
# e.g. also always show your own commits:
log = "builtin_log() | (mine() & mutable())"

# Which commit the graph's left column follows instead of @
log-graph-prioritize = "present(@)"

# Revsets for 'jj bookmark advance'.
# 'from' has access to 'to', so it can find bookmarks relative to the destination.
bookmark-advance-to = "@"
bookmark-advance-from = "heads(::to & bookmarks())"

# Defines "interesting" revisions for op show/diff/log; others are elided
# into a summary count.
op-diff-changes-in = "mutable() | immutable_heads()"

# Default revision sets for commands that operate over a range
fix = "reachable(@, mutable())"
run = "reachable(@, mutable())"          # 0.43+, for 'jj run'
arrange = "reachable(@, mutable())"
simplify-parents = "reachable(@, mutable())"
sign = "reachable(@, mutable())"
```

### Template Aliases

```toml
[template-aliases]
# Custom formatting
'format_short_id(id)' = 'id.shortest(8)'
'format_timestamp(ts)' = 'ts.ago()'

# Custom commit format
'my_log' = '''
change_id.short() ++ " " ++
if(description, description.first_line(), "(no description)") ++
if(conflict, " CONFLICT", "")
'''
```

## Templates

Templates are a functional language for customizing output.

### New Description Template

Populated when `jj new` is run without `-m`:

```toml
[templates]
new_description = '""'
# Example: auto-populate with a prefix
new_description = '"wip: "'
```

### Log Template

```toml
[templates]
log = 'builtin_log_oneline'
# Or custom:
log = '''
separate(" ",
  format_short_change_id_with_hidden_and_divergent_info(self),
  format_short_commit_id(commit_id),
  bookmarks,
  tags,
  if(conflict, label("conflict", "conflict")),
  if(empty, label("empty", "(empty)")),
  if(description, description.first_line(), description_placeholder),
) ++ "\n"
'''
```

### Draft Description Template

```toml
[templates]
draft_commit_description = '''
concat(
  builtin_draft_commit_description,
  "\nJJ: ignore-rest\n",
  diff.git(),
)
'''

[template-aliases]
default_commit_description = '''
"

Closes #NNNN
"
'''
```

### Workspace List Template (0.44+)

`jj workspace list` now shows each workspace's root path by default. Customize with `-T` or:

```toml
[templates]
# Default is the builtin_workspace_list alias
workspace_list = 'name ++ ": " ++ target.commit_id().short() ++ "\n"'
```

### Commit Trailers

```toml
[templates]
commit_trailers = '''
format_signed_off_by_trailer(self)
++ if(!trailers.contains_key("Change-Id"), format_gerrit_change_id_trailer(self))
'''
```

### Template Syntax

```
# Literals
"string"
true / false
42

# Operators
x ++ y           # Concatenate
x && y           # Logical and
x || y           # Logical or
!x               # Logical not
x == y           # Equality

# Conditionals
if(condition, then, else)

# Functions
separate(sep, items...)   # Join non-empty with separator
concat(items...)          # Join all items
coalesce(items...)        # First non-empty
surround(prefix, suffix, content)  # Wrap if non-empty
label(name, content)      # Apply color label
indent(prefix, content)   # Indent lines
fill(width, content)      # Word wrap
```

### Commit Methods

Available in log/show templates:

```
self.commit_id()
self.change_id()
self.description()
self.author()
self.committer()
self.parents()
self.bookmarks()
self.tags()
self.working_copies()
self.conflict()
self.empty()
self.immutable()
self.divergent()
self.hidden()
self.mine()
self.contained_in(revset)
self.diff([fileset])
config(name)              # Returns Option<ConfigValue>; accepts Stringify expression
```

## Filesets

Filesets select files for commands like `jj diff`, `jj split`, `jj squash`.

### Patterns

```bash
# Path prefix (default)
jj diff src                    # Files under src/

# Exact file
jj diff 'file:README.md'

# Glob patterns
jj diff 'glob:*.rs'            # .rs in current dir
jj diff 'glob:**/*.rs'         # All .rs files
jj diff 'glob-i:*.TXT'         # Case-insensitive

# Root-relative (from repo root)
jj diff 'root:src'
jj diff 'root-glob:**/*.rs'
```

### Operators

```bash
# Negation
jj diff '~Cargo.lock'          # Everything except Cargo.lock

# Intersection
jj diff 'src & glob:**/*.rs'   # Rust files in src/

# Difference
jj diff 'src ~ glob:**/*.rs'   # Non-Rust files in src/

# Union
jj diff 'glob:*.rs | glob:*.toml'
```

### Functions

```bash
all()                          # All files
none()                         # No files
```

### Examples

```bash
# Diff excluding lock files
jj diff '~Cargo.lock'

# Split: put all except foo in first commit
jj split '~foo'

# List non-Rust files in src
jj file list 'src ~ glob:**/*.rs'

# Squash only specific files
jj squash 'glob:*.md'
```

## Git Settings

> **Minimum git version:** 2.41.0 (required since jj 0.38)

```toml
[git]
# Default fetch remote(s): a single remote, a list, or a string pattern
# (glob by default; "regex:'^(origin|upstream)'" also works).
fetch = "origin"
# Default push remote. Unlike git.fetch, this can only be a single remote.
push = "origin"

# Private commits (won't be pushed). Default: "none()"
private-commits = "description(glob:'wip:*')"

# Colocate by default (default: true)
colocate = true

# Create a local bookmark tracking the default remote bookmark on clone
# (default: true). Set false if you never update trunk locally.
track-default-bookmark-on-clone = true

# Abandon unreachable commits from remote (default: true). See note below.
abandon-unreachable-commits = true

# Reconstruct evolution history for imported Git commits via change IDs
# (default: true). Disable temporarily when importing very many commits.
record-synthetic-predecessors = true

# Sign commits when pushing rather than when creating them (see Signing)
sign-on-push = false

# Hash for new Git repos: "sha1" (default) or "sha256"
object-hash = "sha1"

# Remote interactions spawn a `git` subprocess; point at a specific binary
# if `git` isn't on PATH
executable-path = "git"
```

> **Removed in 0.42:** `git.auto-local-bookmark` no longer exists. Its replacement is per-remote — see [Per-Remote Settings](#per-remote-settings). `auto-local-bookmark = true` becomes:
>
> ```toml
> [remotes.origin]
> auto-track-bookmarks = "*"
> ```
>
> **Removed in 0.44:** `git.fetch-tags` (the `all`/`included`/`none` enum) is gone; tag fetching is now a per-remote string *pattern*. `git.shallow-clone-depth` does not exist either — use `jj git clone --depth`.
>
> These keys are silently ignored if left in your config.

#### `git.abandon-unreachable-commits`

When jj imports refs from Git, commits that used to be reachable but no longer are get **abandoned** in jj to match, and their descendants are rebased off them. If an abandoned commit is a working copy, it is replaced with a new empty commit. Set to `false` to leave Git-unreachable commits in the repo instead:

```toml
[git]
abandon-unreachable-commits = false
```

This matters when external tooling prunes branches out from under you (`gh pr merge --delete-branch`, `git branch -D`, `git-chain`).

> **0.44+ scope change:** `jj git import` / `jj git export` are now no-ops in colocated workspaces — both print `No import needed in colocated workspaces.` and exit 0, because colocated repos import continuously and the explicit commands had a race. `--ignore-working-copy` forces the real import. So this setting now mostly bites on non-colocated repos, on forced imports, and on the implicit imports colocated repos still do.

### Multiple Remotes

Configure fetch/push for fork workflows:

```toml
[git]
# Fetch from multiple remotes by default
fetch = ["upstream", "origin"]
# Push only to your fork
push = "origin"
```

Fork workflow trunk override:

```toml
[revset-aliases]
# Point trunk() at upstream instead of origin
'trunk()' = 'main@upstream'
```

### Per-Remote Settings

All four values are name patterns. `*` is the only wildcard (no `?`), combinable with `|` (union) and `~` (negation). Other [string-pattern](revsets.md#string-patterns) kinds such as `regex:` are **not** accepted here.

```toml
[remotes.origin]
# Which fetched/created bookmarks to track automatically. Default "~*" (none).
# "*" is the replacement for the removed git.auto-local-bookmark = true.
auto-track-bookmarks = "*"

# Narrower alternative: only track bookmarks *you* create with
# `jj bookmark create` / `jj bookmark set`. Default "~*" (none).
# Use when bookmark names don't encode an owner prefix. Downside: you must
# track your own bookmarks manually after fetching them on another machine.
auto-track-created-bookmarks = "*"

# Which bookmarks/tags to fetch by default.
# If fetch-bookmarks is unset, the remote's Git refspecs are used.
fetch-bookmarks = "~gh-pages"
fetch-tags = "v*"
```

**Fork workflow** — track everything from your fork, only trunk from upstream:

```toml
[remotes.origin]
auto-track-bookmarks = "*"
[remotes.upstream]
auto-track-bookmarks = "main"
```

**Personal-prefix workflow** — don't track collaborators' bookmarks:

```toml
[remotes.origin]
auto-track-bookmarks = "alice/*"
```

#### Tags (0.44+)

`jj git fetch` now fetches tags the same way it fetches bookmarks — as `<name>@<remote>`, automatically tracked by a local tag of the same name. Git's `tagOpt` is **no longer respected**; control tag fetching from jj config only:

```toml
[remotes.origin]
fetch-tags = "~*"   # fetch no tags at all
fetch-tags = "v*"   # fetch only release tags
# unset          -> fetch all tags (the default)
```

The first `jj git fetch` after upgrading re-fetches all tags to initialize tracking state.

> **This changes what is immutable.** `tags()` is part of `builtin_immutable_heads()`, so every fetched tag makes its target (and all ancestors) immutable. If a fetch suddenly protects commits you were editing, `fetch-tags = "~*"` — or a narrower `immutable_heads()` — is the lever. See [Immutable Commits](#immutable-commits).

Related: `jj tag track` / `jj tag untrack` manage tracking manually; `jj git push --all` now pushes all tags in addition to bookmarks; `jj git clone` uses `-t/--tag PATTERN` (the old `--fetch-tags=all|none|included` was removed in 0.44).

## Working Copy

```toml
[working-copy]
# Executable-bit handling on Unix (unused on Windows):
#   "auto" (default) - probe the filesystem, fall back to "respect"
#                      (or "ignore" if permission is denied)
#   "respect"        - track and apply the executable bit
#   "ignore"         - never read or write it; new files are non-executable
exec-bit-change = "auto"

# CRLF <-> LF conversion: "none" (default), "input", "input-output"
eol-conversion = "none"
```

Set `exec-bit-change` to `"ignore"` on filesystems that don't support executable bits or when spurious permission changes cause noise. `jj file chmod` always updates the recorded bit regardless.

## Conditional Configuration

Config can be scoped to a condition using `[[--scope]]` tables plus a `--when.*` key.

```toml
[user]
name = "Your Name"
email = "default@example.com"

# Different email for repos under ~/oss
[[--scope]]
--when.repositories = ["~/oss"]
[--scope.user]
email = "you@oss.example.org"

# Only when the REMOTE_DEV env var is set
[[--scope]]
--when.environments = ["REMOTE_DEV"]
[--scope.ui]
editor = "code -w"
pager = "cat"
```

If a scope has no conditions it always applies; if it has several, they must **all** match (intersection). Values *within* one condition key are OR'd.

**Condition keys:**

| Key | Matches |
|-----|---------|
| `--when.repositories` | Repository path prefixes (absolute; `~` expands) |
| `--when.workspaces` | Workspace path prefixes |
| `--when.hostnames` | Exact, case-sensitive match against `operation.hostname` |
| `--when.commands` | Subcommand prefixes — `["file"]` matches `jj file show` and `jj file list`; `["file show"]` matches only the former |
| `--when.platforms` | `windows`, `macos`, `linux`, `unix`, … |
| `--when.environments` | `"NAME=VALUE"` (variable equals value) or `"NAME"` (variable is set) |

`--when.*` can also be used at the **top level** of a file, which pairs well with `conf.d/`:

```toml
# ~/.config/jj/conf.d/work.toml — applies to the whole file
--when.repositories = ["~/work"]

[user]
email = "you@work.example.com"
```

> **Gotcha:** `[[--when]]` with a nested `[--when.ui]` table is **not** valid and fails with `Invalid type or value for --when`. The scope table is `[[--scope]]`; `--when.*` is the condition inside it.

## Signing

```toml
[signing]
# Signing backend: "none" (default, disables signing), "gpg", "gpgsm", "ssh"
backend = "ssh"

# When to sign. NOT a boolean — there is no `sign-all`.
#   "drop"  - never auto-sign; drop an existing signature on rewrite
#   "keep"  - (default) re-sign a change that was already signed by you
#   "own"   - sign every commit you author when you create or modify it
#   "force" - sign everything, even commits you didn't author
behavior = "own"

# Key: top-level, shared by all backends. Defaults to the key for user.email.
# gpg/gpgsm: anything `gpg -u` accepts. ssh: a literal key or a path.
key = "~/.ssh/id_ed25519.pub"

# GPG settings
[signing.backends.gpg]
program = "gpg"
allow-expired-keys = false      # consider expired keys valid when verifying

# PKCS#12 certificates
[signing.backends.gpgsm]
program = "gpgsm"
allow-expired-keys = false

# SSH settings
[signing.backends.ssh]
program = "ssh-keygen"
allowed-signers = "~/.ssh/allowed_signers"   # needed to *verify* signatures
revocation-list = "~/.ssh/revoked_keys"      # signatures from these keys are invalid
```

**Sign on push instead of on every rewrite** — useful when signing is slow or needs a hardware key, since it batches into one operation:

```toml
[signing]
behavior = "drop"
backend = "ssh"
key = "~/.ssh/id_ed25519.pub"

[git]
sign-on-push = true
```

Signature verification/display is **off** by default (it costs time on large logs). Enable with `ui.show-cryptographic-signatures = true`. Sign manually with `jj sign` / `jj unsign`.

## Immutable Commits

The immutable system protects commits from accidental modification:

- `immutable_heads()` — defines which commit heads are protected (configurable below)
- `immutable()` — defined as `::immutable_heads()` (all ancestors of immutable heads)
- `mutable()` — defined as `~immutable()` (everything NOT immutable)

`mutable()` is useful in revsets — e.g., `reachable(@, mutable())` returns your working stack (all commits reachable from `@` that aren't immutable).

The default (`builtin_immutable_heads()`) protects `trunk()`, tags, and **untracked** remote bookmarks. Branches you push with `jj git push` are *tracked* → remain mutable; branches managed by external git tooling (`git`/`gh`/`git-chain`) arrive *untracked* → immutable.

> **0.44+:** `jj git fetch` now fetches tags automatically, so tagged commits become immutable as soon as you fetch. To opt out, set `remotes.<name>.fetch-tags = "~*"` (see [Tags](#tags-044)) or drop `tags()` from `immutable_heads()`.

```toml
[revset-aliases]
# Default: trunk, tags, and untracked remote bookmarks are immutable
'immutable_heads()' = 'trunk() | tags() | untracked_remote_bookmarks()'
# Equivalently, reuse the built-in and extend it:
'immutable_heads()' = 'builtin_immutable_heads() | bookmarks(glob:"release-*")'

# Narrower: trunk and tags only (DROPS untracked-remote-bookmark protection)
'immutable_heads()' = 'trunk() | tags()'

# Most restrictive: only trunk
'immutable_heads()' = 'trunk()'

# Include release branches
'immutable_heads()' = 'trunk() | tags() | bookmarks(glob:"release-*")'
```

## Fix Tools (Formatters)

```toml
[fix.tools.rustfmt]
command = ["rustfmt", "--emit=stdout"]
patterns = ["glob:'**/*.rs'"]

[fix.tools.black]
command = ["black", "-", "--stdin-filename=$path"]
patterns = ["glob:'**/*.py'"]

[fix.tools.prettier]
command = ["prettier", "--stdin-filepath=$path"]
patterns = ["glob:'**/*.{js,ts,jsx,tsx,json,md}'"]

[fix.tools.clang-format]
command = ["clang-format"]
patterns = ["glob:'**/*.{c,cc,cpp,h}'"]
# Format only modified lines. The placeholders are $first and $last (1-based,
# inclusive) — NOT $start/$end. The argument is repeated once per changed range.
# Omit entirely to always format the whole file.
line-range-arg = "--lines=$first:$last"
# Run the tool even when a file has zero line ranges to format (default false).
# Useful for whole-file work like import sorting.
run-tool-if-zero-line-ranges = true
# Define a tool disabled by default; enable it per-repo with one setting.
enabled = true
```

`command` may also use `$path` (the file's repo-relative path) and `$root` (the workspace root) — `jj fix` feeds content on stdin anonymously, so tools that need a filename read it from these.

## Merge Tools

```toml
[merge-tools.meld]
program = "meld"
merge-args = ["$left", "$base", "$right", "-o", "$output"]
edit-args = ["$left", "$right"]

[merge-tools.vimdiff]
program = "vim"
merge-args = ["-d", "$left", "$base", "$right", "$output"]
merge-tool-edits-conflict-markers = true

[ui]
merge-editor = "meld"
diff-editor = ":builtin"
```

## Snapshot Settings

```toml
[snapshot]
# Files larger than this are not snapshotted. String with units, or a plain
# byte count. Set to 0 to disable the limit entirely.
max-new-file-size = "1MiB"

# Auto-track patterns (default: all)
auto-track = "all()"
# Or selective:
auto-track = "glob:**/*.rs"

# Auto-run `jj workspace update-stale` when the working copy is stale
auto-update-stale = false

# Filesystem monitor lives in its own table, NOT under [snapshot].
# (`snapshot.use-watchman` no longer exists.)
[fsmonitor]
backend = "none"        # or "watchman"

[fsmonitor.watchman]
register-snapshot-trigger = false
```

### Auto-Track Examples

```toml
[snapshot]
# Only track specific file types
auto-track = 'glob:"**/*.rs" | glob:"**/*.toml" | glob:"Cargo.lock"'

# Exclude large generated directories
auto-track = 'all() ~ glob:"node_modules/**" ~ glob:"target/**"'

# Track everything except build artifacts and vendored deps
auto-track = 'all() ~ glob:"dist/**" ~ glob:"vendor/**" ~ glob:".next/**"'
```

## Merge Settings

```toml
[merge]
# Granularity of hunks when merging files: "line" (default) or "word"
hunk-level = "line"

# When all sides make the same change: "accept" (default) or "keep" the conflict
same-change = "accept"
```

## Run Settings (0.43+)

Settings for `jj run`, which executes a command across a set of revisions, each in its own private working copy.

```toml
[run]
# Max parallel working copies / jobs. Default 1. Overridden by --jobs.
jobs = 4

[revsets]
# Default revisions jj run operates on
run = "reachable(@, mutable())"
```

## Debug Settings

```toml
[debug]
# Fixed RNG seed for deterministic commit/change IDs (testing only).
# Integer, not a string.
randomness-seed = 42
```
