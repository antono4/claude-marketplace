# Changelog - jj

All notable changes to the jj skill in this marketplace will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

## [Unreleased]

## [1.9.0] - 2026-08-12

### Added
- `references/jj-hunk.md`: conditional integration with the external [`jj-hunk`](https://github.com/dashed/jj-hunk) tool, which brings non-interactive hunk selection to `jj split`, `jj squash`, `jj diffedit`, `jj restore` and `jj absorb`. Covers detection (`command -v jj-hunk`), the fallback when it is absent, a verb-mapping table from each jj command to its jj-hunk counterpart, the per-verb selection semantics, and the traps that bite an automated caller — `restore`'s reversed hunk ids, `absorb` exiting 0 on a refusal, and content-level predicates silently skipping binaries, symlinks, mode flips, pure renames and empty adds
- SKILL.md pointers to that reference from the three places where hunk-level selection is the natural next step: "No Staging Area", "Editing a Previous Commit", and "Inherently Interactive Commands"

### Changed
- SKILL.md line budget: "Working Copy as a Commit" and "First-Class Conflicts" each lost a code block whose commands are already covered elsewhere in the file (the Essential Commands table and the "Resolving Conflicts" section respectively), becoming prose. That paid for the jj-hunk pointers with room to spare - 499 lines to 491, against the 500-line guideline

## [1.8.0] - 2026-08-06

Updates the skill from a jj 0.41.0 baseline to **jj 0.44.0** (covering the 0.42, 0.43, and 0.44 releases). Every claim was verified against the live 0.44.0 binary — `--help`, `jj config list --include-defaults`, the v0.44.0 config schema, and scratch repos with real bare-git remotes. Documentation pinned at the `v0.44.0` tag was used in preference to the upstream working tree, which is already past the release.

This is primarily a **correctness repair**. The skill had drifted into recommending flags and config keys that no longer exist, and a mechanical audit (extracting every documented `jj <subcommand> --flag` pair and diffing it against the binary — 510 pairs) also surfaced a number of errors that predate the 0.42–0.44 window.

### Fixed

Removed in 0.42–0.44 but still documented as current:

- `jj git push --allow-new` (removed 0.42) in SKILL.md, `commands.md`, and `github-workflow.md`, including a three-row "incompatible combinations" table. `--bookmark`/`--tag`/`--change`/`--named` now auto-track new remote refs, so the flag is simply unnecessary.
- `git.auto-local-bookmark` (removed 0.42) → per-remote `remotes.<name>.auto-track-bookmarks` (default `~*`; `"*"` is the exact equivalent of the old `= true`).
- `git.push-new-bookmarks` (removed 0.42), including a comment that wrongly described another key as its replacement.
- `git_head()` / `git_refs()` (removed 0.43 from both revsets and templates), Git-like symbols such as `refs/heads/main` (no longer resolve), `ui.revsets-use-glob-by-default` (removed 0.43), and the `<kind>:<bookmark>@<remote>` form of `jj bookmark track`/`untrack` (removed 0.43). Retained as a migration table with replacements rather than deleted outright.
- `jj file search`: `-p`/`--pattern` is now a **required flag** and positional arguments are filesets — the documented `jj file search <pattern>` form was invalid. Default output is now `path:line` per match; `--name-only` restores the pre-0.44 paths-only behavior; `-n`/`--line-number` is new.
- `jj git import` / `jj git export` were recommended as the colocated-repo fix-up; as of 0.44 both are no-ops there (`No import needed in colocated workspaces.`, exit 0). Documented the `--ignore-working-copy` force path and that ordinary jj commands re-sync automatically.
- `fetch-tags`: the `all|included|none` enum no longer exists. `git.fetch-tags` is gone entirely; only `remotes.<name>.fetch-tags` remains, as a string pattern (`'~*'` disables, `'v*'` selects). `jj git clone --fetch-tags=…` → `-t/--tag=PATTERN`.
- `jj commit`/`jj describe` `--author`/`--reset-author`, `jj describe --edit`/`--no-edit` (→ `--editor`), and `jj metaedit --update-committer-timestamp` (→ `--force-rewrite`), all removed in 0.42.

Errors predating the 0.42–0.44 window, found by mechanical audit:

- `jj show -p` does not exist (patch is on by default; `--no-patch` disables).
- `jj bisect run -s <good> -e <bad>` — neither flag exists; the real interface is `--range`/`-r` (repeatable) plus `--find-good`. The documented invocation could never have worked. Added the exit-code contract and `$JJ_BISECT_TARGET`.
- `jj revert -s <rev>` — no such flag; `-r` **and** a destination are both required.
- `jj bookmark advance -r` → `-t`/`--to`.
- `references/conflicts.md`: the auto-rebase opt-out recipe used `jj rebase --branch C --destination K`, but `-b C` expands to `-s roots(K..C)` = **B**, dragging B onto the duplicate and defeating the recipe. Corrected to `--source`.
- `references/revsets.md`: `:x` and `x:` were documented as "exclusive ancestors/descendants" operators but have never existed on 0.44 (`` `:` is not a prefix/postfix operator ``); `x..` was described as "descendants of x minus x" when it is `~::x` and includes sibling branches; and an entire section taught the `all:` modifier, removed in **0.38**, which is now a hard parse error.
- `references/templates.md`: `trailers().filter(…)` was a hard parse error (`trailers` is a keyword, not a function); `List.join()` was documented unconditionally but does not exist for non-printable element types such as `List<Commit>`.
- `references/configuration.md`: the `[[--when]]` + `environments` conditional-config syntax was a hard parse error (correct form is `[[--scope]]` + `--when.<key>`); `signing.sign-all` does not exist (→ `signing.behavior` = `drop`/`keep`/`own`/`force`); `signing.key` is top-level, not under `signing.backends.ssh`; `fix.tools.<name>.line-range-arg` used non-existent `$start`/`$end` placeholders instead of `$first`/`$last`; `git.shallow-clone-depth` and `snapshot.use-watchman` do not exist; and wrong defaults were recorded for `working-copy.exec-bit-change`, `revsets.bookmark-advance-from`/`-to`, and `revsets.op-diff-changes-in`.
- `references/filesets.md`: `glob:` is non-recursive (`glob:"*.rs"` at the root matches nothing; use `glob:"**/*.rs"`).
- `references/git-comparison.md`: the `jj bookmark track` note claimed `--remote` had to be used "not `<name>@<remote>` syntax" — the `BOOKMARK@REMOTE` symbol form is still supported; 0.43 removed only the `<kind>:<bookmark>@<remote>` *pattern* form. The "Pull and Rebase" section contained the nonsense placeholder `jj rebase -d <remote>@origin`, and "Cherry-picking" taught a two-step `jj duplicate` + `jj rebase -r` workaround that `jj duplicate <rev> -o main` does in one step.
- `references/faq-patterns.md`: "Push Says Nothing Changed" said `--all` pushes all bookmarks — on 0.44 it pushes bookmarks **and tags**; and `jj arrange` was described as experimental, which its help text no longer claims.

### Added

- **`jj run` (0.43)** — the headline feature of that release, previously absent from the skill entirely. Full reference section in `commands.md`, a "Running a Command Over a Stack" recipe in `faq-patterns.md`, and a SKILL.md entry. Covers `-r`, `-j`, `--root`, `--clean`, `--restore-descendants`, the 0.44 flags `--passthrough`/`--ignore-changes`/`--ignore-errors`, the `JJ_CHANGE_ID`/`JJ_COMMIT_ID`/`JJ_WORKSPACE_ROOT` environment variables, and oldest→newest ordering.
- **Tags as first-class refs (0.44)** across SKILL.md, `commands.md`, `configuration.md`, and `github-workflow.md`: `jj git fetch` fetches tags as `<name>@<remote>` and auto-tracks them, `jj tag track`/`untrack`, `jj git push --tag`, and the footgun that `jj git push --all` now pushes all tags. Includes the immutability consequence — `tags()` is part of `builtin_immutable_heads()`, so fetching a tag can make commits you were editing immutable.
- `jj absorb -i`/`--interactive`/`--tool` (0.44), which obsoletes the previously-documented "split first, then absorb" workaround; `jj git push --allow-conflicts` (0.44) with an accurate description of how conflicts appear to Git tooling — `.jjconflict-base-*/` and `.jjconflict-side-*/` root directories with the first side's content at the real path, not conflict markers; `jj util backend name` (0.42), `jj config gc` (0.43), `jj show --reversed` (0.43) and multi-revision `jj show` (0.42).
- Revsets: `merge_point()` (0.44, arity 1), `forks()` (0.43), and `builtin_log()` (0.44) with its real expansion and a "Extending the Default Log" section.
- Templates: `try()` (0.44), the `FsPath` / `Option<FsPath>` return-type changes to `WorkspaceRef.root()` and `RepoPath.absolute()`, `max_bar_width` on `TreeDiff.stat()`, and eight previously-undocumented types.
- Config: `/etc/jj` and `conf.d/` in the precedence list (0.43), `diff.stat.max-bar-width` (0.44), `merge-tools.<name>.edit-invocation-mode` (0.42), the alias `.doc` table form (0.42), `crossed-out` colors (0.43), `revsets.log = "builtin_log()"`, and a warning that jj **silently ignores unknown config keys** — the mechanism by which all of the above rotted invisibly.
- `github-workflow.md`: a "When Upstream Force-Pushed" recipe built on 0.43's change-ID-based descendant rebasing, verified end-to-end.
- The 0.44 rule that repeated CLI arguments are no longer an error (last occurrence wins).

### Changed

- `jj rebase` examples now use `-o`/`--onto` and `jj restore` uses `-t`/`--into` throughout, matching upstream's own documentation (jj's v0.44.0 docs use `-o` exclusively and `-d` nowhere). `-d`/`--destination` and `--to` remain accepted aliases and are noted once, at each command's definition in `commands.md`.

## [1.7.0] - 2026-06-05

Closes the "already-pushed / colocated-repo" gap that greenfield examples never exercised. Every claim was re-verified against live `jj` 0.41.0 before shipping (including correcting two inaccuracies in the source improvement proposal itself).

### Added
- SKILL.md: "Colocated repos: Git sees `@` as uncommitted" pitfall — Git's `HEAD` tracks `@`'s parent, so `@`'s changes look uncommitted to Git; `git checkout`/`git switch` (and git-based tooling: gh, git-chain, hooks) can abort, but *only* when the target branch touches the same files as the `@`-vs-parent diff. Fix: park `@` with `jj new <bookmark>` before the git operation.
- SKILL.md: one-line "reorder a pushed/stacked branch" pointer in Pushing Changes, linking to the full recipe.
- references/github-workflow.md: "Reordering an Already-Pushed / Stacked Branch" recipe — `jj rebase --ignore-immutable -s <change> -d <dest>` then `jj git push --bookmark <name>` (force-with-lease-safe by default; no `--force` flag), with the plain-git `git push --force-with-lease origin <branch>` alternative noted.
- references/commands.md: `jj squash --from <child> --into <parent>` relocates the child's bookmark onto the parent (with `-m`/`--use-destination-message` non-interactive caveat); `--ignore-immutable` also works in subcommand position.

### Changed
- SKILL.md: broadened the "Immutable commit error" troubleshooting note to include **untracked remote bookmarks** (e.g. branches pushed via git/gh/git-chain, not `jj git push`) and to show `--ignore-immutable` in both global and subcommand position.
- SKILL.md: refined the "Working Copy Changes on Merge Commits" pitfall — an empty, description-less, bookmark-less `@` is auto-abandoned when you move off it, so a manual `jj abandon` is usually unnecessary.
- SKILL.md: removed a duplicate "Searching Files (Non-Interactive)" block (the retained "Searching File Contents" section is a superset) and folded the `jj split` file-path workaround inline, to stay under the 500-line cap.

### Fixed
- references/configuration.md: corrected the immutable-commits default at two sites. The real default is `builtin_immutable_heads()` = `trunk() | tags() | untracked_remote_bookmarks()` (trunk, tags, and **untracked** remote bookmarks) — not `trunk() | tags()` as previously labeled "the default". Branches pushed with `jj git push` are tracked (mutable); branches managed by external git tooling arrive untracked (immutable). Now consistent with `references/revsets.md`, which was already correct.

## [1.6.0] - 2026-05-25

### Added
- Megamerge pattern for parallel integration testing with jj absorb routing (faq-patterns.md)
- `all:` prefix pattern and WIP batch rebase recipe (revsets.md)
- Auto-rebase opt-out pattern with duplicate + rebase workaround (conflicts.md)
- `squash --from` with revset ranges documentation (commands.md)
- `describe -r` with revsets for batch descriptions (commands.md)
- `active()`/`wip` custom revset alias examples (revsets.md)
- Per-repo config isolation tip (revsets.md)

## [1.5.0] - 2026-05-24

### Added
- New references/faq-patterns.md: 14 common patterns from jj FAQ (private commits, resume work, revert merges, lost commits, etc.)
- New references/conflicts.md: Conflict resolution deep dive (marker styles, workflows, multi-sided, binary)
- Enhanced commands.md: Added missing flags for squash (--from/--into/-k), split (-r), log (-p/--stat/--no-graph/-n), show (--stat/--git/-T), resolve (--list/--tool), duplicate (behavior notes), redo

## [1.4.0] - 2026-05-24

### Added
- New references/github-workflow.md: GitHub/GitLab PR workflows (push patterns, review comments, fork setup, useful revsets)
- Added `jj absorb`, `jj evolog`, `jj commit`, `jj parallelize`, `jj revert`, `jj simplify-parents` to commands.md
- Added CLI revision options guide (-r/-s/-b/-f and -o/-A/-B/-t flag patterns) to commands.md
- Added conflict marker styles (diff/snapshot/git) to configuration.md
- Added multiple remotes / fork workflow configuration
- Added mutable()/immutable() system explanation
- Added more snapshot.auto-track examples

### Changed
- SKILL.md: Added absorb, evolog, commit to Essential Commands table
- SKILL.md: Added `jj absorb` to "Editing a Previous Commit" section
- SKILL.md: Added github-workflow.md link to Advanced Topics
- Condensed bookmark gotchas section for brevity

## [1.3.0] - 2026-05-23

### Added
- New references/templates.md: Comprehensive template language reference (operators, global functions, types with methods, aliases, practical examples)
- New references/filesets.md: Fileset language reference (file patterns, operators, functions, aliases)
- Enhanced references/revsets.md: Added 20+ missing functions (reachable, fork_point, bisect, first_parent, at_operation, etc.), string/date patterns, alias documentation
- Added "Minimum Version Requirements" table to SKILL.md
- Added "Template Language" section to SKILL.md with quick reference
- Added "Filesets" section to SKILL.md with practical examples
- Added fileset aliases (0.39+) to references/configuration.md
- Added revset alias with doc property syntax to references/configuration.md

### Changed
- Condensed snapshot trigger and colocated mode sections in SKILL.md for brevity
- Updated Advanced Topics section with links to all 6 reference files

## [1.2.0] - 2026-05-23

### Added
- New commands: `jj arrange`, `jj bookmark advance`, `jj file search`, `jj util snapshot`
- New revset functions: `divergent()`, `remote_tags()`, `diff_lines()`, `diff_lines_added()`, `diff_lines_removed()`
- `xyz/n` versioned access syntax for hidden/divergent changes
- `--no-integrate-operation` global flag documentation
- `jj rebase --simplify-parents` flag
- `jj git push --option` for push options
- `jj bookmark rename --overwrite-existing` flag
- Pattern alias support (`name:x` syntax) in revsets
- Configuration: `remotes.<name>.fetch-bookmarks/fetch-tags`, `working-copy.exec-bit-change`, `--when.environments`, `JJ_PAGER`, `revsets.bookmark-advance-from/to`, `revsets.op-diff-changes-in`

### Changed
- Update skill to cover jj 0.37.0 through 0.41.0 (from 0.36.0)
- Update `jj bookmark track/untrack` to use `--remote` flag (old `@remote` syntax deprecated)
- Update push flag table: `--all`/`--tracked`/`-r` now skip ineligible bookmarks instead of failing
- Note `jj op undo` removed (use `jj op revert` or `jj undo`/`jj redo`)
- Note `diff_contains()` renamed to `diff_lines()`
- Note `git_head()` and `git_refs()` deprecated
- Note per-repo config moved outside repo (0.38)
- Note removed configs: `ui.always-allow-large-revsets`, `all:` modifier, `git.push-bookmark-prefix`, `ui.default-description`, `ui.diff.format`, `ui.diff.tool`, `core.fsmonitor`, `core.watchman.register-snapshot-trigger`
- Note minimum git version now 2.41.0
- Condensed Colocated Mode Deep Dive for brevity (progressive disclosure)
- Condensed binary/merge conflict resolution sections

## [1.1.0] - 2026-01-15

### Added
- Editor Settings section in references/configuration.md (for interactive/human use):
  - Priority order: `$JJ_EDITOR` > `ui.editor` > `$VISUAL` > `$EDITOR`
  - Terminal editor examples (vim, nvim, nano, emacs, micro, helix)
  - GUI editor configurations with wait flags (VS Code, Sublime, BBEdit, TextMate, IntelliJ)
  - Quick config commands (`jj config set --user ui.editor`)
  - Comparison table of `ui.editor` vs `ui.diff-editor` vs `ui.merge-editor`

- Working copy snapshot trigger explanation (when snapshots occur, how to force them)
- Binary file conflict resolution using `jj restore --from`

- Multi-parent (merge) conflict resolution workflow
- Colocated Mode Deep Dive section covering:
  - Git status interpretation ("HEAD detached from X" meaning)
  - Git index synchronization issues after jj conflict resolution
  - When git and jj disagree (import/export commands)
- Bookmark gotchas subsection:
  - `--allow-backwards` flag for moving bookmarks to ancestors
  - `*` suffix meaning (diverged from remote)
  - Create vs Set bookmark behavior
- Non-Interactive Workflows section covering:
  - Commit messages without editor (`-m` flag for describe, commit, new, squash, split)
  - Squash without editor (`-u` or `-m` flags)
  - Conflict resolution without merge tool (`--tool :ours/:theirs` or `jj restore`)
  - Inherently interactive commands and workarounds
- Common Pitfalls section covering:
  - Push flag combinations that don't work together
  - Working copy changes on merge commits
  - Git status in colocated mode
  - Bookmark movement refused scenarios
- Advanced Revset Recipes in revsets.md:
  - `roots(X ~ Y)` pattern for rebasing entire branch trees
  - Branch divergence analysis
  - Working with multiple feature branches
  - Finding merge commits
  - Complex rebase scenarios
- Push flag compatibility table in commands.md
- `--allow-new` flag documentation for `jj git push`
- Updated bookmark set command with `--allow-backwards` flag

### Changed
- SKILL.md Configuration section now emphasizes non-interactive workflows for LLM/automation use
- Removed editor setup from main SKILL.md (moved to references for human users)

## [1.0.0] - 2025-11-24

### Added
- Initial addition to marketplace
- Comprehensive SKILL.md covering:
  - Key concepts (working copy as commit, change IDs, no staging, first-class conflicts)
  - Essential commands quick reference table
  - Common workflows (new changes, editing, rebasing, bookmarks, pushing, conflicts)
  - Basic revsets reference
  - Git interoperability
  - Configuration basics
  - Troubleshooting guide
- Reference documentation in references/:
  - `revsets.md` - Complete revset language reference with all operators, functions, and patterns
  - `commands.md` - Full command reference organized by category with all options
  - `git-comparison.md` - Git to jj command mapping and workflow comparisons
  - `configuration.md` - Configuration reference including templates, filesets, aliases
- Skill triggers for "jj", "jujutsu", change IDs, operation log, revsets
