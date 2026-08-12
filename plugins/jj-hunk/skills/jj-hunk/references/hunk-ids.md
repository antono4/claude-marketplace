# Hunk IDs

## Anatomy

A hunk ID is `hunk-` plus a SHA-256 over the hunk's **path**, its type, its removed and
added lines, and up to three lines of surrounding context from the **parent** side of the
diff. Positions are not hashed.

| Form | Length | Where it appears |
|------|--------|------------------|
| Full | `hunk-` + 64 hex | The `id` field in `--format json` / `--format yaml` |
| Short | `hunk-` + 8 hex | The `short_id` field, `--format text`, `--format diff` headers, `--spec-template` |

The short form is the shortest prefix unique across the diff, never under eight hex
digits. Everywhere an ID is *accepted* — `ids` and `hunks` in a spec, and `id()` — the
full form, the short form, and any unambiguous prefix all work, and forms may be mixed in
one call. A trailing `...` is tolerated for ids copied out of older diff headers.

`exact:` does **not** demand the full 64 hex. `id(exact:"hunk-f5696093")` resolves the
short form fine; all `exact:` does is switch off abbreviation, accepting the full id or
the exact short id and rejecting every other prefix.

## IDs are frame-independent; paths are not

The path folded into the hash is relative to the **workspace root**, so the same hunk
keeps its id whether listed from the repo root or from a subdirectory — even though the
`path` field printed alongside it changes:

```
$ jj-hunk list --format json | grep -E '"path"|"id"'
      "path": "src/svc.rs",
          "id": "hunk-e338f6bc9cc6977089d2e8271e1360b8686dccb649af8e6107a9d527e5658432",

$ cd src && jj-hunk list --format json | grep -E '"path"|"id"'
      "path": "svc.rs",
          "id": "hunk-e338f6bc9cc6977089d2e8271e1360b8686dccb649af8e6107a9d527e5658432",
```

So an ID collected from one working directory is usable from another, but a **spec key**
or a `file()` argument is not — those follow the current directory.

## How long an ID stays valid

Long enough to carry from one command to the next against the same working-copy state.
That is the workflow — `list`, choose, `split` — and within it IDs are solid.

**Survives:**

- other hunks appearing or disappearing elsewhere in the same file (concurrent agent work)
- edits to other files
- line numbers shifting — positions are not hashed
- an edit *inside* its own three-line context window, because the context is read from the
  parent side and the parent side did not change
- being listed from a different working directory

**Does not survive:**

- **renaming or moving the file** — the path is hashed
- **an edit touching a line immediately adjacent to the hunk**, which merges the two into
  one larger hunk with different text. A line of untouched code in between keeps them
  separate
- **a rebase, or a squash into the parent**, when it rewrites the lines around the hunk.
  Context is read from the parent side; treat any rebase as invalidating
- **reversing the diff.** `--from A --to B` and `--from B --to A` swap removed and added
  lines, so every id differs. This is what makes `restore` ids distinct — see
  [commands.md](commands.md#it-reads-its-ids-from-a-reversed-listing)

**So: re-run `list` after editing, and use the IDs from that run.** Do not cache IDs
across an editing session, a rebase, or a rename.

## When an ID does not resolve

### In a hunkset

`id()` resolves as it evaluates, so this fires even on `list --spec`:

```
Error: hunkset evaluation error: hunk id 'hunk-deadbeef' matches no hunk in this diff -- ids change when the hunk's content or its file changes, so one copied from an earlier listing may be stale. Run 'list' again for the current ids.
```

Ambiguity is an error naming the candidates, never a guess:

```
Error: hunkset evaluation error: hunk id 'hunk-1' is ambiguous -- it matches 2 hunks: hunk-14a47eb4 (f.txt), hunk-1c12dc45 (f.txt). Use more characters, or exact:"<full-id>".
```

### In a JSON/YAML spec

Every mutating verb validates each entry of a spec before touching anything and refuses
the whole operation if one does not name exactly what it meant to:

```
Error: spec does not resolve against the diff:
  f.txt: no hunk with id hunk-deadbeef
Those entries do not name exactly what they meant to. Check them against `jj-hunk list --spec-template`.
```

The middle line names the specific problem. All of these are real messages:

| Message | Cause |
|---------|-------|
| `no hunk with id hunk-deadbeef` | Usually a stale ID |
| `id hunk-1 is ambiguous, it names 2 hunks -- use a longer prefix` | Prefix too short |
| `no hunk with index 999 (file has 67)` | Out-of-range index in `hunks` |
| `no such path in the diff` | Typo'd path, or a path with `ids`/`hunks` that is absent |
| `f.txt: renamed to renamed.txt in this diff -- file the entry under renamed.txt instead` | Spec keyed by a rename's old name |

`list --spec` does **not** run this check, which is what makes it safe to iterate with.

### What is tolerated

A spec **may** name paths absent from the diff — a reusable allowlist stays usable as
files come and go — as long as it keeps at least one path that *is* present. Only a spec
that keeps nothing real is rejected. That applies to bare `{"action": "keep"}` entries.

An entry naming `ids` or `hunks` under an absent path is **always** rejected, however much
else resolved: those ids were read off a real diff, so the path is a typo. Tolerating it
would have committed a subset of what the spec asked for, at exit 0, with nothing on
stderr.

`--allow-empty` does not silence any of this — it concerns an empty *result*, not whether
what you wrote refers to anything.
