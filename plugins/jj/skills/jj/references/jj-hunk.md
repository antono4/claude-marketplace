# Hunk-Level Selection with `jj-hunk`

## Overview

Several jj commands need a diff editor to go finer than a whole file: `jj split -i`,
`jj squash -i`, `jj diffedit`, `jj restore -i`, `jj absorb -i`. The `:builtin` editor is a
TUI, so none of them can be driven non-interactively. Path arguments
(`jj split -m "msg" src/foo.rs`) are the built-in workaround and stop at file granularity.

[`jj-hunk`](https://github.com/dashed/jj-hunk) is an **optional external tool** that
selects individual hunks from a query expression instead, making those operations one-shot
commands. It is not part of jj and is not installed by default.

## Detect before using

```bash
command -v jj-hunk >/dev/null && jj-hunk --version   # jj-hunk 0.4.1-my-jj-hunk
```

| Available | Not available |
|-----------|---------------|
| `jj-hunk split '<query>' "msg"` | `jj split -i` (needs a human), or `jj split -m "msg" <paths>` (file granularity) |
| `jj-hunk squash '<query>'` | `jj squash -i`, or `jj squash <paths>` |
| `jj-hunk diffedit '<query>'` | `jj diffedit` (no non-interactive form) |
| `jj-hunk restore '<query>'` | `jj restore <paths>` (whole files only) |
| `jj-hunk absorb --dry-run` | `jj absorb` (whole-hunk routing, no preview), `jj absorb -i` (0.44+, needs a human) |

Install with `cargo install --git https://github.com/dashed/jj-hunk` — **not**
`cargo install jj-hunk`, which resolves to a different project on crates.io
(`laulauland/jj-hunk`, latest published 0.3.0) with no hunkset query language. The fork
builds the tree-sitter `semantic` feature by default, giving `function()`, `scope()`,
`annotation()` and `import()`; a `--no-default-features` build makes those predicates
error explicitly rather than return nothing.

No jj configuration is required — each subcommand pins the merge tool to its own
executable and passes it to `jj` on the command line.

## Where it fits in the jj workflows

| jj workflow | Interactive form | jj-hunk equivalent |
|-------------|------------------|--------------------|
| Split a mixed working copy | `jj split -i` | `jj-hunk split 'glob("src/api/**")' "feat: api"` |
| Move part of a change to the parent | `jj squash -i` | `jj-hunk squash 'function("handle_request")'` |
| Reduce a revision to some of its diff | `jj diffedit` | `jj-hunk diffedit 'id(hunk-e338f6bc)'` |
| Undo one hunk | `jj restore -i` | `jj-hunk restore 'id(...)'` — see the reversed-listing trap below |
| Fold fixes into the right ancestors | `jj absorb` | `jj-hunk absorb --dry-run`, then `jj-hunk absorb` |

Explore first — `list` is the only read-only subcommand:

```bash
jj-hunk list --format text                        # every hunk, with enclosing function/scope
jj-hunk list --spec 'scope("UserService")' --format text   # preview a selection
```

Selectors combine with `|` (union), `&` (intersection), `~` (difference/negation).
Predicates include `file()`, `glob()`, `extension()`, `status()`, `type()`, `content()`,
`added()`, `removed()`, `lines()`, `id()`, plus the semantic ones. A selection matching
nothing is a hard error from every mutating verb, which is the guard against a typo'd
selector silently becoming a no-op.

## Traps that bite an automated caller

These are the ones worth knowing before the first call. Full detail lives in the
`jj-hunk` skill.

### The verbs disagree about what a named hunk means

`diffedit` **keeps** the hunks you name and discards the rest. `restore` **undoes** the
hunks you name and leaves the rest alone. Against the same diff they are near-inverses,
so confusing them destroys the wrong half.

### `restore` ids come from a reversed listing

`jj restore` hands its editor the destination on the left and the source on the right, so
`jj-hunk restore` builds its spec against `destination -> source`. A hunk id copied from a
forward `jj-hunk list` **never resolves** there — the added and removed lines are swapped
and the id hashes them. List the diff the way `restore` sees it:

```bash
jj-hunk list --from @ --to @- --format text    # the ids restore can use
jj-hunk restore 'id(hunk-8460af87)'
```

For `restore -c REV` the feeding listing is `--from REV --to REV-`; for
`restore --from A --into B` it is `--from B --to A`.

### `absorb` exits 0 when it refuses

A refusal is not an error. Renames are always refused and pure insertions stay put by
default, and in both cases the command prints its reason and exits `0`:

```
  1 hunks: 0 moving into 0 ancestors, 1 staying
Nothing to absorb: every hunk stays in lrsxttwn.
```

Check the summary line, not the exit status. Run `--dry-run` first — it names every
destination commit and is the only preview there is. The undo is `jj op restore <id>`
(printed on the last line), **not `jj undo`**, which reverses only the last of absorb's
several operations.

### Content predicates cannot see changes with no hunks

A binary file, a symlink retarget, a mode-only flip, a pure rename or copy, and an empty
file add or remove have nothing to select *within*. `content()`, `added()`, `removed()`,
`lines()` and `id()` never match one; `all()`, `file()`, `glob()`, `extension()`,
`status()` and any negation select it whole.

So `jj-hunk split 'content("...")'` leaves every one of them behind — at exit 0, with
nothing on stderr. Use `all()`, `all() ~ content("...")` or `glob("**")` when the
selection is meant to cover everything, and re-run `list` afterwards to confirm.

### A renamed file matches under both its paths

`file()`, `glob()` and `extension()` reach a renamed file by its new path *and* its old
one. The consequence is that `~glob("secret/*")` **drops** a file renamed out of
`secret/`. Specs are always keyed by the new path; using the old one is an error naming
the replacement.

### The message is positional

`jj-hunk commit -m "msg" 'all()'` fails with `error: unexpected argument '-m' found`. The
signature is `jj-hunk split|commit [OPTIONS] [SPEC] [MESSAGE]`.

## Hunk IDs

An id is a SHA-256 over the hunk's path, type, text, and parent-side context — not its
position. It survives other hunks appearing, disappearing or shifting elsewhere in the
same file, which is what makes it the right handle when several agents share a working
copy. It does **not** survive a rename, a rebase, a squash into the parent, or an edit on
an immediately adjacent line. Re-run `list` after any of those.

The recommended pattern is: query with a hunkset to explore, then act on the resolved ids.

```bash
jj-hunk list --spec 'scope("UserService")' --format text
jj-hunk split 'id("hunk-e338f6bc", "hunk-7844072c")' "feat: track hit count"
```
