# Command Reference

Full flags and semantics for all eight `jj-hunk` subcommands.

## Contents

- [What the verbs disagree about](#what-the-verbs-disagree-about)
- [list](#list)
- [split](#split)
- [commit](#commit)
- [squash](#squash)
- [diffedit](#diffedit)
- [restore](#restore)
- [absorb](#absorb)
- [select](#select)
- [--allow-empty](#--allow-empty)

## What the verbs disagree about

| Verb | The hunks you name are... | The hunks you do NOT name... |
|------|---------------------------|------------------------------|
| `split` | the ones that **leave**, into the new commit | stay in the original revision |
| `commit` | the ones that **are committed** | stay in the working copy |
| `squash` | the ones that **move** to the destination | stay where they are |
| `restore` | the ones that are **UNDONE** | are left alone |
| `diffedit` | the ones that are **KEPT** | are **discarded** |

Every verb takes the spec as a **positional** argument, and `split`/`commit` take the
message as a second positional. There is no `-m`.

## list

Read-only. The only subcommand safe to run speculatively.

```
Usage: jj-hunk list [OPTIONS]
```

| Flag | Meaning |
|------|---------|
| `-r, --rev <REV>` | Revset to diff against its parent (must resolve to one revision) |
| `--from <FROM>` / `--to <TO>` | Diff two revisions explicitly; mutually exclusive with `--rev` |
| `-i, --include <GLOB>` / `-x, --exclude <GLOB>` | Filter paths; one pattern per flag, repeatable |
| `--group <none\|directory\|extension\|status>` | Group output as `groups: [{name, files}]` |
| `--format <json\|yaml\|text\|diff>` | Output format (default `json`) |
| `--binary <skip\|mark\|include>` | Binary handling (default `mark`: file listed with 0 hunks) |
| `--max-bytes <n>` / `--max-lines <n>` | Truncate file contents before diffing |
| `--spec <SPEC>` / `-f, --spec-file <FILE>` | Filter output with a hunkset or JSON/YAML spec |
| `--files` | List files with hunk counts only |
| `--spec-template` | Emit an ID-based starting spec (JSON/YAML only) |

`--spec-template` with `--format text` or `diff` is rejected:
`Error: --spec-template does not support text output (use json or yaml)`.

`list --spec` does **not** run the JSON-spec resolution check, so a spec naming a stale id
or a missing path simply selects nothing — which is what makes it safe to iterate with. A
hunkset is different: `id()` resolves as it evaluates, so `list --spec 'id(hunk-deadbeef)'`
errors even on `list`. So do a malformed glob and a semantic predicate in a
non-`semantic` build.

## split

```
Usage: jj-hunk split [OPTIONS] [SPEC] [MESSAGE]
```

Moves the named hunks into a **new commit**; the rest stay in the original revision.

| Flag | Meaning |
|------|---------|
| `-r, --rev <REV>` | Revision to split (default `@`) |
| `-f, --spec-file <FILE>` | Read spec from a file |
| `--allow-empty` | Allow a selection that keeps nothing |

```
$ jj-hunk split 'import()' "chore: add io import"
Selected changes : oyntkmtn 2e36033d chore: add io import
Remaining changes: mpmqyukx d843e54a (no description set)
Working copy  (@) now at: mpmqyukx d843e54a (no description set)
Parent commit (@-)      : oyntkmtn 2e36033d chore: add io import
```

The new change is created as a child of the current revision. Use `jj rebase -r <new> -d
<dest>` to position it elsewhere in the graph.

## commit

```
Usage: jj-hunk commit [OPTIONS] [SPEC] [MESSAGE]
```

Commits the named hunks; the rest stay in the working copy. Same flags as `split` minus
`--rev`.

```
$ jj-hunk commit 'file("other.txt")' "chore: tweak other"
Working copy  (@) now at: troynkur 385d513e (no description set)
Parent commit (@-)      : kwumlwlv 87751cc9 chore: tweak other
```

## squash

```
Usage: jj-hunk squash [OPTIONS] [SPEC]
```

Moves the named hunks into the parent; the rest stay where they are.

| Flag | Meaning |
|------|---------|
| `-r, --rev <REV>` | Revision to squash (default `@`) |
| `-f, --spec-file <FILE>`, `--allow-empty` | As above |

## diffedit

```
Usage: jj-hunk diffedit [OPTIONS] [SPEC]
```

Rewrites a revision to contain **only** the named hunks. Everything else is **discarded**.

| Flag | Meaning |
|------|---------|
| `-r, --rev <REV>` | Revision to edit (default `@`) |
| `--from <FROM>` | Show changes from this revision (default `@`) |
| `-t, --to <TO>` | Edit changes in this revision (default `@`) |
| `--allow-empty` | Allow a selection that keeps nothing — **discards the whole diff** |

```
$ jj-hunk list --format text
M f.txt
  hunk 0 replace hunk-455638d1 ...   - A1  + A1-x
  hunk 1 replace hunk-f500a113 ...   - A9  + A9-x

$ jj-hunk diffedit 'id(hunk-455638d1)'
$ cat f.txt        # A1-x kept, A9-x discarded
A1-x
...
A9
```

## restore

```
Usage: jj-hunk restore [OPTIONS] [SPEC]
```

**Undoes** the named hunks, taking their content from another revision. The hunks you do
not name are left alone — the inverse of `diffedit`.

| Flag | Meaning |
|------|---------|
| `-c, --changes-in <REV>` | Undo the changes in this revision (default `@`) |
| `--from <FROM>` | Revision to restore from, the source (default `@`) |
| `-t, --into <INTO>` | Revision to restore into, the destination (default `@`) |
| `-f, --spec-file <FILE>` | Read spec from a file |
| `--allow-empty` | Allow a selection that undoes nothing |

### It reads its ids from a REVERSED listing

`jj restore` hands its editor the destination on the left and the source on the right, so
`restore` builds its spec against `destination -> source`. An id copied from a forward
`list` **never resolves**, because removed and added lines are swapped and the id hashes
them. The hunk **type flips** too — an insertion in the forward diff is a `delete` in the
reversed one — so `type()`, `added()` and `removed()` invert along with it.

| What you are running | The listing to read ids from |
|----------------------|------------------------------|
| `restore` (default, `--changes-in @`) | `jj-hunk list --from @ --to @-` |
| `restore -c REV` | `jj-hunk list --from REV --to REV-` |
| `restore --from A --into B` | `jj-hunk list --from B --to A` |

```
$ jj-hunk list --format text                     # forward
M f.txt
  hunk 0 replace hunk-455638d1 (before 1+1 after 1+1)
    - A1
    + A1-x
  hunk 1 replace hunk-f500a113 (before 9+1 after 9+1)
    - A9
    + A9-x

$ jj-hunk list --from @ --to @- --format text    # what restore sees
M f.txt
  hunk 0 replace hunk-f4a41f4a (before 1+1 after 1+1)
    - A1-x
    + A1
  hunk 1 replace hunk-9670f6bf (before 9+1 after 9+1)
    - A9-x
    + A9

$ jj-hunk restore 'id(hunk-f4a41f4a)'            # undoes A1 only; A9-x survives
```

When a spec fails to resolve, the error names the reversed listing with the revisions
already substituted, rather than sending you back to the forward one:

```
Error: spec does not resolve against the diff:
  src/svc.rs: no hunk with id hunk-deadbeef
Those entries do not name exactly what they meant to. Check them against `jj-hunk list --from f0e487697bba --to 2e36033d7a36 --spec-template`.
```

## absorb

```
Usage: jj-hunk absorb [OPTIONS] [SPEC]
```

Routes each hunk into the mutable ancestor that last touched its lines, using
`jj file annotate`. With no spec it considers every hunk in the revision.

| Flag | Meaning |
|------|---------|
| `-r, --rev <REV>` | Revision to absorb from (default `@`) |
| `--dry-run` | Print the routing plan without changing anything |
| `--insertions <skip\|surrounding>` | What to do with hunks that only add lines (default `skip`) |
| `-f, --spec-file <FILE>` | Read spec from a file |

**Run `--dry-run` first.** The plan names every destination commit and is the only preview
there is.

```
$ jj-hunk absorb --dry-run
absorb from lrsxttwn (no description)
  2 hunks: 2 moving into 2 ancestors, 0 staying

move into tswkprov c2: touch line 2
  f.txt:2  -1 +1  hunk-81f851ca

move into xvwsktkv c3: touch line 7
  f.txt:7  -1 +1  hunk-d5f65728

line numbers are in the parent of lrsxttwn; an insertion is listed at the line it goes before
--dry-run: nothing was changed
```

### A refusal exits 0

An absorb that can route nothing is **not an error**. Check the summary line
(`N moving into N ancestors, M staying`), not the exit status:

```
$ jj-hunk absorb
absorb from lrsxttwn (no description)
  1 hunks: 0 moving into 0 ancestors, 1 staying

stay in lrsxttwn (no description)
  renamed.txt:2  -1 +1  hunk-0b1afc48
    renamed.txt was renamed from f.txt, and a rename is a whole-file change that would ride into the ancestor with the first hunk that moved -- commit the rename on its own first, then absorb

line numbers are in the parent of lrsxttwn; an insertion is listed at the line it goes before
Nothing to absorb: every hunk stays in lrsxttwn.
$ echo $?
0
```

### What stays behind, and why

- **Renamed and copied files are refused**, with the reason printed beside the hunk.
  Commit the rename on its own first, then absorb.
- **Pure insertions stay put by default.** No line of an insertion blames to an ancestor:
  `it only adds lines, so no line of it blames to an ancestor (--insertions=surrounding
  routes these by their neighbours)`. With `--insertions=surrounding`, an insertion is
  routed only when the lines above and below agree on a destination — otherwise it still
  stays, reported as `it is inserted exactly on the boundary between lines owned by
  tswkprov and xzkwvypz`.

### Undo

The source revision is left **empty, not abandoned**. The undo is **`jj op restore`, not
`jj undo`** — absorb performs several operations and `jj undo` reverses only the last.
The id is printed on the final line:

```
Absorbed 2 hunks into 2 revisions:
  tswkprov c2: touch line 2
  xvwsktkv c3: touch line 7
Undo all of it with: jj op restore da254dd4512b
```

## select

Plumbing invoked by `jj --tool=jj-hunk`. Not for direct use. See
[json-specs.md](json-specs.md) for the raw `jj --tool` path.

## --allow-empty

Every mutating verb refuses a selection that matches nothing:

```
Error: selection matched no hunks: content("zzzznomatch")
An empty selection is nearly always a mistyped selector rather than an intent, so it is refused.
Check it against `jj-hunk list --spec ...`, or pass --allow-empty if that is what you meant.
```

What it would otherwise have carried out differs per verb — an empty commit for `split`,
a **discarded diff** for `diffedit`, nothing at all for `restore`. Pass `--allow-empty`
only when that outcome is genuinely what you want.

`--allow-empty` says an empty *result* is acceptable. It never meant "do not check whether
what I wrote refers to anything", so a typo'd path or a stale id in a JSON spec is still
reported with the flag set.
