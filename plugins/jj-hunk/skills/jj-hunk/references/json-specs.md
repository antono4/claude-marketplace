# JSON/YAML Specs and Raw `jj --tool` Usage

Hunkset expressions are the recommended selector. Reserve JSON/YAML specs for
programmatic generation, for reusable allowlists, and for the raw `jj --tool` path.

## Format

```json
{
  "files": {
    "src/svc.rs": {"ids": ["hunk-e338f6bc", "hunk-7844072c"]},
    "src/bar.rs": {"action": "keep"},
    "src/qux.rs": {"action": "reset"}
  },
  "default": "reset"
}
```

| Entry | Effect |
|-------|--------|
| `{"hunks": [0, 2]}` | Include only hunks 0 and 2 (indices or ID strings) |
| `{"ids": ["hunk-e338f6bc"]}` | Include hunks by ID (full, short, or unambiguous prefix) |
| `{"action": "keep"}` | Include all changes in the file |
| `{"action": "reset"}` | Discard all changes in the file |
| `"default": "reset"` | Unlisted files are discarded |
| `"default": "keep"` | Unlisted files are kept |

`ids` and `hunks` are merged when both are given. `"default": "reset"` is the safer
choice: name what you want rather than excluding what you do not.

Specs may be inline, read from stdin with `-`, or loaded with `-f`/`--spec-file`:

```bash
echo '{"files":{"f.txt":{"ids":["hunk-14a47eb4"]}},"default":"reset"}' | jj-hunk split - "msg"
jj-hunk split --spec-file /tmp/spec.json "msg"
```

Keys are paths **relative to the current directory**, and an ID belongs to the path it was
listed under, because the path is part of the hash.

## `--spec-template`

The quickest way to a correctly-shaped starting spec. It writes short IDs and covers the
**whole** diff, including changes that have no hunks:

```
$ jj-hunk list --spec-template --format json
{
  "files": {
    "blob.bin": {
      "action": "keep"
    },
    "brand_new_empty.txt": {
      "action": "keep"
    },
    "config": {
      "action": "keep"
    },
    "moved.txt": {
      "action": "keep",
      "from": "tomove.txt"
    },
    "script.sh": {
      "action": "keep"
    }
  },
  "default": "reset"
}
```

JSON and YAML only — `--format text` and `--format diff` are rejected.

## Pinning a hunkless change

To pin down a binary, symlink, mode-only change, pure rename or empty add — the changes no
content-level predicate can reach — use `{"action": "keep"}`. `{"ids": []}` resets it,
restoring the parent byte-for-byte, and works on symlinks and non-UTF-8 paths too.

## Renamed files carry a `from` field

`select` is handed two directories and nothing else, so it cannot tell that `right/new_name`
used to be `left/old_name`. `from` supplies that link:

```json
{"files": {"new_name.txt": {"ids": ["hunk-111b8dea"], "from": "old_name.txt"}}, "default": "reset"}
```

The mutating verbs fill this in from jj's rename detection, so a hand-written spec that
omits it still works through them. It is load-bearing only on the raw `jj --tool` path
below, where omitting it drops the file from the commit entirely.

Key the entry under the **new** name. Using the old one is an error naming the replacement:

```
$ echo '{"files":{"f.txt":{"action":"keep"}},"default":"reset"}' | jj-hunk split - "x"
Error: spec does not resolve against the diff:
  f.txt: renamed to renamed.txt in this diff -- file the entry under renamed.txt instead
Those entries do not name exactly what they meant to. Check them against `jj-hunk list --spec-template`.
```

Two names for matching, one for keying: a hunkset expression may *select* the file by its
old path, but the spec it produces is always keyed by the new one.

## Raw `jj --tool` usage

The subcommands are wrappers around this. For direct control:

```bash
echo '{"files": {"src/foo.rs": {"hunks": [0]}}, "default": "reset"}' > /tmp/spec.json
JJ_HUNK_SELECTION=/tmp/spec.json jj split -i --tool=jj-hunk -m "message"
```

On this path nothing pre-processes the spec, so **you must write `from` yourself for any
renamed file**. Without it the selection matches no hunk in the recomputed diff and the
file is lost.

This is also the only path where jj resolves the tool itself, so it needs config:

```toml
[merge-tools.jj-hunk]
program = "jj-hunk"                          # on PATH, or an absolute path
edit-args = ["select", "$left", "$right"]
```

The subcommands do not read this block — each pins `merge-tools.jj-hunk.program` to the
running executable and passes it to `jj` on the command line, so a stale or wrong block
cannot break them. Registering a *different* tool that wraps jj-hunk therefore has to use
a name other than `jj-hunk`, and be driven through `jj` directly.
