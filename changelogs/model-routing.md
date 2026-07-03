# Changelog - model-routing

All notable changes to the model-routing skill in this marketplace will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

## [1.1.0] - 2026-07-03

### Added
- `references/codex-cli.md`: "Timeouts and hangs" section — `codex exec` can
  hang indefinitely at zero CPU on a dead network wait (observed in
  production: two exec chains idle 5+ hours with ~0.1 s CPU each), so every
  invocation must be wrapped in a hard `timeout` (900 s implementation /
  300 s read-only); includes the CPU-time-vs-process-age forensic for
  diagnosing a suspected hang and kill-the-whole-chain guidance
- `references/codex-cli.md`: documented codex's built-in deadlines
  (`stream_idle_timeout_ms` default 5 min, `stream_max_retries` 5,
  `request_max_retries` 4; MCP `startup_timeout_sec` 30 s /
  `tool_timeout_sec` 300 s — verified in the codex source) and why they are
  a first line, not a guarantee: the observed hangs occurred with all of
  them at defaults, so only an external `timeout` bounds the whole invocation

### Changed
- Template and SKILL.md: every codex example invocation now carries the
  `timeout` wrapper (mechanics bullet, task-shape table, smoke test), and the
  wrapper-agent pattern gained two rules — the wrapper prompt must state the
  timeout policy explicitly (the wrapper is the process blocked when codex
  hangs, so it can't apply judgment after the fact), and each wrapper gets
  ONE codex task or must checkpoint-report between tasks (a hang in a
  sequential relay stalls everything queued behind it)
- SKILL.md troubleshooting: added the "delegated task goes silent" entry
  (near-zero CPU on an hours-old process = hung network wait; kill and re-run
  under `timeout`)

## [1.0.0] - 2026-07-01

### Added
- Initial addition to marketplace
- Skill that installs and maintains a "model routing" section in a project's
  CLAUDE.md — a per-model cost/intelligence/taste rankings table plus
  application rules and mechanics — so the orchestrating Claude routes
  subagent and workflow tasks to the cheapest model that clears the quality
  bar (pattern adapted from Theo's multi-model CLAUDE.md workflow:
  https://x.com/theo/status/2072481845363822914 and
  https://x.com/theo/status/2072482460122964067)
- `references/claude-md-template.md`: the customizable CLAUDE.md section
  template (rankings table, defaults-not-limits escalation rules, Codex CLI
  mechanics, thin-wrapper pattern for using GPT models inside workflows and
  subagents where the `model` parameter only accepts Claude models)
- `references/codex-cli.md`: verified `codex exec` non-interactive reference
  (sandbox modes, output capture, model override, session resume) sourced
  from the Codex CLI implementation
