# Changelog - jj-hunk

All notable changes to the jj-hunk skill in this marketplace will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

## [Unreleased]

## [1.0.0] - 2026-08-12

### Added

Initial addition to the marketplace. `jj-hunk` is an optional external tool for the jj
(Jujutsu) VCS that selects individual diff hunks from a query expression, so hunk-level
operations that normally need the `:builtin` diff editor become one-shot commands.

Every command, output listing, and error string in the skill was verified against a
`jj-hunk 0.4.1-my-jj-hunk` binary (the `dashed/jj-hunk` fork, semantic build) on jj
0.44.0, in throwaway repos built for the purpose.

- `SKILL.md` (397 lines) — availability check and fallback to `jj split -i` /
  path-level selection, the eight subcommands, the per-verb semantics table, the
  list → preview → act-by-id workflow, the hunkset essentials, five sharp edges, output
  formats, and a message-to-cause troubleshooting table
- `references/hunkset-language.md` — every predicate, operator precedence, pattern
  prefixes and their per-predicate bare-string defaults, semantic predicates with the
  supported-language list, and the file-level/content-level reachability split
- `references/commands.md` — full flags for all eight verbs, `restore`'s reversed
  listing, `absorb`'s routing plan and refusal modes
- `references/hunk-ids.md` — ID anatomy, frame-independence, exactly what invalidates
  one, and every resolution-failure message
- `references/json-specs.md` — spec format, `--spec-template`, the rename `from` field,
  and raw `jj --tool=jj-hunk` usage
- `references/multi-agent.md` — extracting per-agent changes from a shared working copy

Sharp edges the skill documents because an agent hits them silently:

- The verbs disagree about what a named hunk means. `diffedit` **keeps** what you name;
  `restore` **undoes** what you name. They are near-inverses against the same diff
- `restore` builds its spec from a reversed `destination -> source` diff, so an id copied
  from a forward `list` never resolves. The feeding listing is `jj-hunk list --from @ --to @-`
- `absorb` exits **0** when it refuses (a rename, a pure insertion). An agent checking
  only the exit code reads `0 moving into 0 ancestors, 1 staying` as success
- Content-level predicates (`content`, `added`, `removed`, `lines`, `id`) can never reach
  a change with no hunks — binary, symlink retarget, mode-only flip, pure rename, empty
  add — so a `content()`-only split leaves all of them behind at exit 0 with nothing on
  stderr
- `file()`/`glob()` match a renamed file under **both** paths, so `~glob("secret/*")`
  drops a file renamed out of `secret/`
- The commit message is **positional**: `jj-hunk commit '<spec>' '<message>'`; `-m` is
  `error: unexpected argument`
- Semantic predicates need the `semantic` build feature and error explicitly without it

The skill installs from `cargo install --git https://github.com/dashed/jj-hunk`, **not**
`cargo install jj-hunk`. The crates.io name resolves to a different project
(`laulauland/jj-hunk`, latest published 0.3.0) with no `[features]` section and no
hunkset query language, so the plain crates.io install would silently produce a binary
that cannot run most of what this skill documents. On the fork the `semantic` feature is
on by default.
