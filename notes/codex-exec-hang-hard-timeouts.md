# Codex exec hangs: why delegation needs mandatory hard timeouts

**Date:** 2026-07-03
**Context:** first production day of the model-routing workflow (`plugins/model-routing`) in the media_glue repo — Claude (fable, orchestrating) delegating mechanical build tasks to gpt-5.5 via `codex exec`, wrapped in thin relay subagents.
**Outcome:** two silent multi-hour hangs, ~5 hours of lost wall-clock on one workstream, recovered by process forensics + a takeover protocol. Hard timeouts are now mandatory on every `codex exec` invocation in this setup.

## What happened

A relay subagent was driving four sequential `codex exec -s workspace-write` build
tasks in a git worktree. Tasks 1 and 2 completed and verified green. Then the
pipeline went quiet.

Two separate codex process chains were later found hung:

| Chain | Started | CPU time when killed | State |
|---|---|---|---|
| `codex exec -s …` (pid 27049/27051/27052) | 05:09 | **0.12 s** | never did work |
| `codex exec -s …` (pid 31710/31712/31714/31715) | ~06:45 | **0.10 s** | never did work |

Both sat for **5+ hours at effectively zero CPU** — the signature of a dead
network wait (an API/stream call that never returned and had no client-side
deadline), not a long computation. No file mtimes changed after 06:44; no build
fingerprints after 06:43.

The relay wrapper (a subagent whose job is to run codex via Bash and relay its
output) was itself blocked on the never-returning Bash call — so it could not
even report a status when pinged. From the orchestrator's mailbox view, "hung
child" and "working quietly" look identical. That ambiguity cost hours: an
earlier activity check at ~07:00 found fresh file edits and a *just-spawned*
codex process and reasonably concluded "not hung, mid-task" — the just-spawned
process then never accumulated CPU again.

## How it was detected (the forensic that works)

File mtimes and `git diff --stat` tell you the last time codex *did* anything.
The decisive signal is **CPU time vs process age**:

```bash
ps aux | grep "codex exec" | grep -v grep | awk '{print $2, $9, "cpu:"$10}'
# a healthy codex run accumulates CPU continuously;
# 0:00.10 CPU on a process started hours ago = hung, kill it
```

Cross-check with build activity (`ls -lt target/debug/.fingerprint | head`) and
worktree mtimes. Three stale signals together = dead, no further benefit in
waiting.

## Recovery protocol that worked

1. **Kill the hung chains** (the full pid chain: the zsh wrapper, the node
   shim, the vendor binary). Leave unrelated interactive codex sessions alone —
   identify yours by start time / command shape, not by name.
2. **Stand down the relay wrapper** by message *before* it wakes from its
   killed Bash call, so it doesn't resume confused and double-run anything.
3. **Salvage completed work**: tasks already edited into the worktree were
   verified directly (their tests passed — a killed-mid-write file would not
   compile, so green named tests ≈ complete edits; still diff-check substance).
4. **Re-run the remaining tasks under the orchestrator's own control** with a
   hard timeout (below). Both remaining tasks then completed inside a single
   15-minute capped run.

## The fix: mandatory hard timeouts

Every `codex exec` in this workflow is now invoked as:

```bash
timeout 900 codex exec -s workspace-write --skip-git-repo-check -C <worktree> "<prompt>"
```

- **900 s** fits multi-file build tasks comfortably (the recovered double-task
  run finished well under it). Use smaller caps (120–300 s) for read-only /
  investigation calls.
- On timeout: the orchestrator (or the relay) does that chunk itself, or
  retries once. Never wait-and-see — a zero-CPU codex process does not recover.
- Relay-wrapper prompts must carry the rule explicitly ("ALWAYS wrap codex in
  `timeout 900`; a hung exec must die, then do that chunk yourself"), because
  the wrapper is the one blocked when the child hangs — it cannot apply
  judgment after the fact.

## Lessons for the model-routing skill/template

1. The template's mechanics section should ship the `timeout` wrapper in every
   example invocation, not as a footnote. A routing section that references a
   command that can silently hang for hours degrades much worse than one that
   occasionally re-runs a 15-minute cap.
2. **Sequential multi-task relays amplify the failure**: one hang stalls every
   queued task behind it. Prefer one codex task per relay for parallelizable
   work, or require the relay to checkpoint-report between tasks (ours did,
   which is how tasks 1–2 were known-good at takeover time).
3. **Idle-notification ambiguity**: a relay blocked on Bash looks identical to
   one that finished and forgot to report. The orchestrator's escalation ladder
   that worked: (1) mailbox ping, (2) worktree/diff inspection, (3) process
   forensics (CPU-vs-age), (4) kill + takeover. Go to (3) directly when (2)
   shows no file activity for longer than the task should take.
4. The hangs are an argument for timeouts, **not** against the routing itself:
   the same day, gpt-5.5 landed six changes (test repairs, parser fixtures, a
   four-fix download-pipeline batch) at effectively zero marginal cost, all
   passing an unchanged review bar. The economics hold; the plumbing needed a
   deadline.

## Quick reference

```bash
# Detect
ps aux | grep "codex exec" | grep -v grep          # CPU column vs start time
ls -lt <worktree>/src | head                        # last real edit
ls -lt <worktree>/target/debug/.fingerprint | head  # last build activity

# Kill (whole chain, not just the leaf)
kill <zsh-pid> <node-pid> <codex-pid>

# Invoke (always)
timeout 900 codex exec -s workspace-write --skip-git-repo-check -C <dir> "<self-contained prompt>"
```

## Appendix: what codex ships built-in (verified in source, 2026-07-03)

Codex is *not* designed to hang forever — the source has client-side deadlines
(checked at `/Users/me/aaa/github/codex`, codex-cli 0.142.x era):

- **Model streaming** (`codex-rs/model-provider-info/src/lib.rs:26-28`):
  `stream_idle_timeout_ms` default **300 000 (5 min)**, `stream_max_retries`
  default **5**, `request_max_retries` default **4** — configurable per
  `[model_providers.<id>]` in config.toml.
- **MCP servers** (`codex-rs/codex-mcp/src/rmcp_client.rs:85-86`): startup
  timeout default **30 s** (`startup_timeout_sec`), per-tool-call timeout
  default **300 s** (`tool_timeout_sec`) — per `[mcp_servers.<name>]`.

A dead *established* stream should therefore die within ~5 idle minutes and
retry, worst case tens of minutes. Our chains hung 5+ hours with ~0.1 s CPU
from birth — i.e. the wait was in a phase these deadlines don't cover (or a
path where the idle timer never armed), very early in the run before any real
work. Conclusion unchanged and reinforced: the built-ins are a first line, but
only an external `timeout` bounds the whole invocation regardless of which
internal phase wedged.
