# Multi-Agent Concurrent Workflow

When several agents work in parallel against one working copy, their changes intermingle.
`jj-hunk` lets each agent retrospectively extract its own changes into a clean,
independent commit.

## Why hunk IDs are the right handle here

A hunk ID is a SHA-256 over the hunk's path, type, text, and parent-side context — not
its position. Another agent adding, removing, or shifting hunks elsewhere in the same file
does **not** disturb yours. That is exactly the property this workflow needs.

It does not survive a rename or a rebase, so collect IDs and act on them from the same
working-copy state. See [hunk-ids.md](hunk-ids.md).

## Setup: create the merge point

Before agents start working, establish the topology:

```bash
jj new main -m "merge: combine agent work"
MERGE=$(jj log -r @ --template 'change_id' --no-graph)
```

## Each agent's workflow

```bash
# 1. Make changes normally. They land in the working copy alongside other agents' work.

# 2. Query to identify YOUR changes, using semantic context where possible
jj-hunk list --spec 'scope("UserService") & glob("src/api/**")' --format text

# 3. Narrow until the query captures exactly your work
jj-hunk list --spec 'function("handle_request") & file("src/api/handler.rs")' --format text

# 4. Confirm the IDs you are about to use cover the whole scope.
#    Empty output means those ids account for everything the query matched.
jj-hunk list --spec 'scope("UserService") ~ id("hunk-e338f6bc", "hunk-7844072c")' --format text

# 5. Split by ID, from the same state you listed
jj-hunk split 'id("hunk-e338f6bc", "hunk-7844072c")' "feat: add request handler"

# 6. Position the new commit in the graph
jj rebase -r <new_change> -d main

# 7. Update the merge to include your branch
jj rebase -r $MERGE -d <new_change> -d <other_branches...>
```

## Verification

```bash
jj diff -r $MERGE     # should be empty, or contain only conflict resolutions
```

If the merge has unexpected content, some agent's split was incomplete. Find what is left
with `jj-hunk list -r $MERGE --format text` and dispatch it.

**Also re-run a plain `list` after each split.** A `content()`-, `added()`- or
`lines()`-based selector cannot reach a binary, symlink, mode-only change, pure rename or
empty add, so those stay behind at exit 0 with nothing on stderr. See
[hunkset-language.md](hunkset-language.md#the-file-level--content-level-split).

## Example: three agents

```
main
├── agent-1: feat: add database schema
│   (jj-hunk split 'glob("src/db/**")' ...)
├── agent-2: feat: add API endpoints
│   (jj-hunk split 'scope("Router") & glob("src/api/**")' ...)
├── agent-3: refactor: update shared utils
│   (jj-hunk split 'function("parse_config") | function("validate")' ...)
└── merge: combine agent work (should be empty)
```

## Key principles

- **Query first, split by ID.** Use hunkset queries to explore; resolve to IDs before
  executing. IDs are content-addressed and immune to concurrent hunks elsewhere in the
  file. Re-list if you rebase or rename in between.
- **Combine predicates for precision.** `scope("MyClass") & file("src/models.rs")` is
  safer than either half alone: it will not capture unrelated changes that happen to sit
  in the same file, nor an identically-named scope in a different file.
- **Verify the merge is empty.** If it is not, something was missed.
- **Rebase, do not move.** `jj-hunk split` creates the new change as a child of the
  current revision; `jj rebase` puts it where it belongs.
- **Do not trust exit 0 to mean "everything moved."** Re-list and look.
