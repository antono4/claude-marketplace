---
name: jj-hunk
description: Programmatic hunk selection for the jj (Jujutsu) VCS. Select individual diff hunks with a query language instead of an interactive TUI, then split, commit, squash, diffedit, restore, or absorb them. Use when splitting a jj commit into logical pieces, committing or squashing only part of the working copy, undoing individual hunks, selecting changes by file/content/line-range or by function/class/decorator via tree-sitter, or when `jj split -i` / `jj squash -i` would open an editor that cannot be driven non-interactively. Also use when several agents share one working copy and each needs to extract its own changes. Triggers on mentions of jj-hunk, hunkset, hunk IDs, or non-interactive hunk selection in jj.
---

# jj-hunk: Programmatic Hunk Selection for jj

## Overview

`jj-hunk` selects individual diff hunks from a query expression, so hunk-level jj
operations that normally require the `:builtin` diff editor become one-shot commands.
It is an optional external tool, not part of jj.

**Check availability first:**

```bash
command -v jj-hunk >/dev/null && jj-hunk --version   # jj-hunk 0.4.1-my-jj-hunk
```

If absent, fall back to jj's own interactive forms (`jj split -i`, `jj squash -i`,
`jj diffedit`) — which need a human — or to path-level selection
(`jj split -m "msg" src/foo.rs`, `jj squash 'glob:**/*.rs'`), which is non-interactive
but cannot go finer than a whole file.

**Install from the fork's git repo, not from crates.io:**

```bash
cargo install --git https://github.com/dashed/jj-hunk
```

`cargo install jj-hunk` resolves to a **different project** — crates.io carries
`laulauland/jj-hunk`, whose latest published version is 0.3.0, has no `[features]`
section and no hunkset query language at all. Everything below is written against
`dashed/jj-hunk` (`0.4.1-my-jj-hunk`), where the `semantic` feature is **on by default**
(`--no-default-features` turns it off). See
[Semantic predicates](#semantic-predicates).

No jj configuration is required. Every `jj-hunk` verb pins the merge tool to the running
executable and passes it to `jj` on the command line, so a stale `[merge-tools.jj-hunk]`
block in `~/.jjconfig.toml` cannot affect it.

## When to Use

- Splitting a mixed working copy into several logical commits
- Committing or squashing only some hunks, non-interactively
- Undoing one hunk out of many (`restore`)
- Reducing a revision to just the hunks you name (`diffedit`)
- Selecting by semantic unit — everything in one function, class, or decorator
- Multiple agents sharing a working copy, each extracting its own changes

## Commands

Eight subcommands. `list` is read-only; `select` is plumbing that `jj` invokes; the other
six rewrite history.

| Command | What it does |
|---------|--------------|
| `list` | Show the hunks in a diff. The only one safe to run speculatively |
| `split <spec> <msg>` | Move the selected hunks into a new commit |
| `commit <spec> <msg>` | Commit the selected hunks |
| `squash <spec>` | Move the selected hunks into the parent |
| `diffedit <spec>` | Rewrite a revision to contain **only** the selected hunks |
| `restore <spec>` | **Undo** the selected hunks, taking their content from another revision |
| `absorb [<spec>]` | Route each hunk into the mutable ancestor that last touched its lines |
| `select <left> <right>` | Called by `jj --tool=jj-hunk`; not for direct use |

**The message is a positional argument, not `-m`.** `jj-hunk commit -m "msg" 'all()'`
fails with `error: unexpected argument '-m' found`. The signature is
`jj-hunk commit [OPTIONS] [SPEC] [MESSAGE]`.

```bash
jj-hunk split 'import()' "chore: add io import"    # correct
```

## Critical: the verbs disagree about what a named hunk MEANS

Read this before writing any selector. The same expression does different things per verb:

| Verb | The hunks you name are... | The hunks you do NOT name... |
|------|---------------------------|------------------------------|
| `split` | the ones that **leave**, into the new commit | stay in the original revision |
| `commit` | the ones that **are committed** | stay in the working copy |
| `squash` | the ones that **move** to the destination | stay where they are |
| `restore` | the ones that are **UNDONE** | are left alone |
| `diffedit` | the ones that are **KEPT** | are **discarded** |

`diffedit` and `restore` are near-inverses. Against the same diff, `diffedit 'id(X)'`
throws away everything *except* X; `restore 'id(X)'` throws away *only* X. Confusing them
destroys the wrong half of the diff.

## Core Workflow

### 1. List

```bash
$ jj-hunk list --format text
M README.md
  hunk 0 insert hunk-ce5a66e9 (before 3+0 after 3+1)
    + line three
M src/svc.rs
  hunk 0 insert hunk-399b086c (before 2+0 after 2+1)
    + use std::io;
  hunk 1 insert hunk-e338f6bc (before 5+0 after 6+1) in UserService
    +     hits: u64,
  hunk 2 replace hunk-7844072c (before 9+1 after 11+1) in UserService::handle_request
    -         let a = 1;
    +         let a = 111;
```

The trailing `in UserService::handle_request` comes from the semantic analyzer and is the
fastest way to see which hunks belong to which logical change.

### 2. Preview the selection

`--spec` filters `list` with the same expression a mutating verb would take. Always do
this before a verb that rewrites history:

```bash
$ jj-hunk list --spec 'scope("UserService")' --format text
M src/svc.rs
  hunk 1 insert hunk-e338f6bc (before 5+0 after 6+1) in UserService
    +     hits: u64,
  hunk 2 replace hunk-7844072c (before 9+1 after 11+1) in UserService::handle_request
    -         let a = 1;
    +         let a = 111;
```

### 3. Act

```bash
jj-hunk split 'scope("UserService")' "refactor: track hit count"
```

A selection matching nothing is a **hard error** from every mutating verb — the main
guard against a typo'd selector silently becoming a no-op:

```
Error: selection matched no hunks: content("zzzznomatch")
An empty selection is nearly always a mistyped selector rather than an intent, so it is refused.
Check it against `jj-hunk list --spec ...`, or pass --allow-empty if that is what you meant.
```

### 4. Prefer IDs when changes may arrive concurrently

Resolve a query to hunk IDs, then act on the IDs. An ID is a hash over the hunk's path,
type, text, and parent-side context — not its position — so it survives other hunks
appearing, disappearing, or shifting elsewhere in the same file:

```bash
jj-hunk split 'id("hunk-e338f6bc", "hunk-7844072c")' "feat: track hit count"
```

IDs do **not** survive a rename, a rebase, or an edit on a line immediately adjacent to
the hunk. Re-run `list` after any of those. See
[references/hunk-ids.md](references/hunk-ids.md).

## Hunkset Query Language

The selector language. Anything that is not JSON/YAML is parsed as a hunkset.

### Operators

| Operator | Meaning | Example |
|----------|---------|---------|
| `x \| y` | Union | `type(insert) \| type(delete)` |
| `x & y` | Intersection | `type(insert) & glob("src/**")` |
| `x ~ y` | Difference | `all() ~ type(delete)` |
| `~x` | Negation | `~type(delete)` |
| `(x)` | Grouping | `(type(insert) \| type(replace)) & file("x")` |

Union and intersection chain freely; **difference does not**. `a ~ b ~ c` is a parse
error telling you to write `(a ~ b) ~ c` — unlike jj's revsets, where `~` is
left-associative. `!` is not an operator here. `all` and `none` may be written bare when
they are the whole expression.

### Predicates

**File-level** — these reach every change in the diff, including ones with no hunks:

| Function | Description |
|----------|-------------|
| `all()` / `none()` | Everything / nothing |
| `file("src/svc.rs")` | Exact path match (whole path, not a suffix) |
| `glob("src/**/*.rs")` | Glob on the file path |
| `extension("rs")` | File extension, written without the dot |
| `status(modified)` | `modified`, `added`, `removed`, `renamed`, `copied` — a bare identifier, not a string |

**Content-level** — these can never match a change that has no hunks:

| Function | Description |
|----------|-------------|
| `type(insert)` | `insert`, `delete`, or `replace` |
| `content("text")` | Added or removed text contains "text" |
| `added("text")` / `removed("text")` | Only the added / only the removed side |
| `lines(10..20)` | Hunks touching lines 10-20, either side of the diff |
| `id("hunk-e338f6bc")` | By hunk ID — full, short, or any unambiguous prefix |

Ranges are **inclusive of both endpoints** despite the Rust-looking `..`: `lines(7..7)`
selects the hunk on line 7.

`id()` accepts several ids in one call and the quotes are optional. It is the one
predicate that **errors** rather than returning nothing when it matches nothing, because
it resolves a name rather than narrowing a set. An ambiguous prefix is likewise an error
naming the candidates, never a guess.

### Bare strings do not mean the same thing everywhere

| Predicate | Bare-string default | Consequence |
|-----------|--------------------|-------------|
| `function()`, `scope()` | **exact** | `function("alpha")` does NOT match `alpha_beta` |
| `file()`, `extension()` | **exact** | `file("svc.rs")` does NOT match `src/svc.rs` |
| `content()`, `added()`, `removed()` | substring | `added("TODO")` matches any line containing TODO |
| `annotation()`, `decorator()` | substring | `annotation("test")` matches `#[test]` |

Opt into a family match explicitly with a prefix — **outside** the quotes:

```bash
jj-hunk list --spec 'function(substring:"test")'
jj-hunk list --spec 'function(regex:"^handle_")'
```

Writing it inside the quotes is a parse error that says so. Prefixes are `exact:`,
`substring:`, `glob:`, `regex:`. A malformed glob is an error everywhere, never a pattern
that matches nothing — which matters most under `~`, where "matches nothing" would invert
into "matches everything".

Full reference, including line-range and semantic predicates:
[references/hunkset-language.md](references/hunkset-language.md).

### Semantic predicates

`function()`, `scope()`, `annotation()`, `decorator()`, `doc()`, `import()`, `toplevel()`
and `depth()` are backed by tree-sitter and exist only in a binary built with the
`semantic` feature. A binary without it does not silently return nothing — each fails
with an explicit error:

```
Error: hunkset evaluation error: function() requires the 'semantic' feature (build with --features semantic)
```

That means the binary was built with `--no-default-features`; reinstall with
`cargo install --git https://github.com/dashed/jj-hunk`, where the feature is on by
default. Every other predicate works in any build.

```bash
jj-hunk split 'scope("UserService")' "refactor: update UserService"
jj-hunk split 'annotation("test")' "test: add unit tests"
jj-hunk split 'all() ~ doc()' "feat: implementation"
```

## Sharp Edges

### `restore` reads its ids from a REVERSED listing

`jj restore` hands its editor the destination on the left and the source on the right, so
`restore` builds its spec against `destination -> source` — the reverse of `jj diff`. An
id copied from a plain `list` **does not resolve there**: the removed and added lines are
swapped, and the id is a hash over them. The hunk *type* flips too, so `type(insert)`
inverts as well.

```bash
$ jj-hunk list --format text                  # forward
M README.md
  hunk 0 insert hunk-ce5a66e9 (before 3+0 after 3+1)
    + line three

$ jj-hunk restore 'id(hunk-ce5a66e9)'
Error: hunkset evaluation error: hunk id 'hunk-ce5a66e9' matches no hunk in this diff -- ids change when the hunk's content or its file changes, so one copied from an earlier listing may be stale. Run 'list' again for the current ids.

$ jj-hunk list --from @ --to @- --format text  # the listing restore actually sees
M README.md
  hunk 0 delete hunk-8460af87 (before 3+1 after 3+0)
    - line three

$ jj-hunk restore 'id(hunk-8460af87)'          # undoes that hunk, leaves the rest
```

In general `restore -c REV` reads from `jj-hunk list --from REV --to REV-`, and
`restore --from A --into B` reads from `jj-hunk list --from B --to A`.

### `absorb` exits 0 when it refuses

An absorb that can route nothing is **not an error**. It prints its reason and exits `0`,
so an agent checking only the exit code reads a refusal as success. **Read the summary
line, not the status.**

```bash
$ jj-hunk absorb                              # renamed.txt was renamed from f.txt
  1 hunks: 0 moving into 0 ancestors, 1 staying
...
Nothing to absorb: every hunk stays in lrsxttwn.
$ echo $?
0
```

Run `--dry-run` first — the plan names every destination commit and is the only preview
there is. Renames and copies are always refused (commit the rename on its own first), and
pure insertions stay put by default (`--insertions=surrounding` opts into routing them by
their neighbours). The source revision is left **empty, not abandoned**, and the undo is
`jj op restore <id>` — printed on the last line — **not `jj undo`**, which reverses only
the last of absorb's several operations. See
[references/commands.md](references/commands.md).

### Changes with no hunks: half the predicates cannot see them

Five kinds of change have nothing to select *within* — a **binary file**, a **symlink
retarget**, a **mode-only flip**, a **pure rename or copy**, and an **add or remove of an
empty file**. Each is selected whole or not at all:

| Reach them | Never reach them |
|------------|------------------|
| `all()`, `file()`, `glob()`, `extension()`, `status()`, any negation `~x` | `content()`, `added()`, `removed()`, `lines()`, `id()` |

They carry a stand-in hunk with a path and a status and no text, *marked* as such so a
content-level predicate declines it outright rather than answering a question about bytes
that were never diffed.

**The trap: a `content()`-only selector silently leaves every one of them behind, at exit
0, with nothing on stderr.**

```bash
$ jj-hunk list --format text
M blob.bin [binary]
A brand_new_empty.txt
M config [symlink, whole-file only]
R moved.txt (tomove.txt -> moved.txt)
M script.sh [mode 100644 -> 100755, not selectable]
M text.txt
  hunk 0 replace hunk-1a94e681 (before 2+1 after 2+1)
    - world
    + WORLD

$ jj-hunk split 'content("WORLD")' "text only"   # exit 0
$ jj-hunk list --format text                     # ...but all of this stayed behind
M blob.bin [binary]
A brand_new_empty.txt
M config [symlink, whole-file only]
R moved.txt (tomove.txt -> moved.txt)
M script.sh [mode 100644 -> 100755, not selectable]
```

When a selection is meant to cover everything, use a predicate from the left column —
`all()`, `all() ~ content("...")`, `glob("**")` — or `--spec-template`, which names every
change explicitly. Run `list` again afterwards to confirm nothing is left.

### A renamed file matches under BOTH its paths

`file()`, `glob()` and `extension()` reach a renamed or copied file by where it is now
*and* by where it came from. The old path is the point — it is the name you have when
looking for what used to be somewhere:

```bash
$ jj-hunk list --format text
R exposed.txt (secret/keys.txt -> exposed.txt)
  ...
$ jj-hunk list --spec 'glob("secret/*")'    # matches: the old path
$ jj-hunk list --spec 'file("exposed.txt")' # matches: the new path
```

The consequence is that **`~glob("secret/*")` drops a file renamed out of `secret/`** —
keeping it would hand you what you excluded. `extension()` likewise matches both sides of
a rename that changed the extension.

Matching only: the spec is always **keyed by the new path**. Keying an entry under the old
one is an error that names the replacement (`f.txt: renamed to renamed.txt in this diff --
file the entry under renamed.txt instead`). Applies only when jj reports a rename; a move
too large for its rename detection is an ordinary add plus delete.

## Output Formats

```bash
jj-hunk list --format json    # structured (default)
jj-hunk list --format yaml
jj-hunk list --format text    # human-readable, with semantic context inline
jj-hunk list --format diff    # unified diff with hunk IDs in the @@ headers
```

`--format diff` is a bare unified patch — apply it with **`git apply`, not `git am`**
(which rejects it: `Patch format detection failed`). Combined with `--spec` it exports
just the hunks a query selects, re-anchored to their own context. Renames are the
exception: the patch records one as a plain modification of the new path.

Paths in output, in spec keys, and in `file()`/`glob()` are **relative to the current
directory**. Hunk IDs are not — the path folded into the hash is workspace-root-relative,
so the same hunk keeps its id whether listed from the repo root or a subdirectory.

## Troubleshooting

| Message | Cause |
|---------|-------|
| `selection matched no hunks: <expr>` | Typo'd selector, or a `content()` query against a hunkless change. Preview with `list --spec` |
| `hunk id '...' matches no hunk in this diff` | Stale id (edit/rebase/rename since listing), or a forward id handed to `restore` |
| `hunk id 'hunk-1' is ambiguous -- it matches 2 hunks` | Prefix too short. Use more characters (a JSON spec words this as `id hunk-1 is ambiguous, it names 2 hunks`) |
| `spec does not resolve against the diff` | A JSON/YAML spec names a missing path or id. `--allow-empty` does **not** silence this |
| `... requires the 'semantic' feature` | Binary built `--no-default-features`. Reinstall with `cargo install --git https://github.com/dashed/jj-hunk` |
| `pattern prefix 'regex:' must go outside the quotes` | Write `regex:"..."`, not `"regex:..."` |
| `difference does not chain` | Write `(a ~ b) ~ c` |
| `error: unexpected argument '-m' found` | The message is positional: `jj-hunk commit '<spec>' '<msg>'` |
| Query returns nothing unexpectedly | `function()` and `file()` are **exact** by default — try `substring:"..."` |
| `warning: ... found no semantic metadata` | No tree-sitter parser for those files. Only printed when **nothing** in the diff could be parsed — a silent empty result with some parseable file present is a real answer |

## References

- [references/hunkset-language.md](references/hunkset-language.md) — every predicate, pattern syntax, precedence, semantic languages and decorator attribution
- [references/commands.md](references/commands.md) — full flags for all eight verbs, `absorb` and `restore` semantics in depth
- [references/hunk-ids.md](references/hunk-ids.md) — ID anatomy, exactly what invalidates one, resolution errors
- [references/json-specs.md](references/json-specs.md) — JSON/YAML spec format, `--spec-template`, rename `from`, raw `jj --tool` usage
- [references/multi-agent.md](references/multi-agent.md) — extracting per-agent changes from a shared working copy
