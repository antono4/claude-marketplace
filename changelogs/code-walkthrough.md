# Changelog - code-walkthrough

All notable changes to the code-walkthrough skill in this marketplace will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

## [1.0.0] - 2026-07-29

### Added
- Initial skill: create self-contained interactive HTML documents that teach a code change (branch, PR, diff, or subsystem) to a reviewer with no prior knowledge
- 6-phase pipeline: extract (read-only agent fan-out with exact git refs, verbatim-first) → verify (author independently re-checks every critical claim) → structure (spiral pedagogy: BLUF → foundations → mechanism → per-unit walkthroughs → flows → scope boundaries → reviewer checklist → glossary) → author (fragments + concatenate) → browser-verify (structure checks, render checks, light/dark screenshots) → deliver (single file, fragments kept for iteration)
- Component library: sidebar TOC with scrollspy, before/after code toggles, diff-colored code lines, collapsible deep-dives, step-through code walker with line highlighting, mermaid diagrams, callouts, glossary-linked terms, light/dark theme, playground widgets that mirror the logic under study
- `references/template-shell.html`: generic, domain-agnostic HTML shell with the full CSS/JS component library and inline usage documentation for every component
- Pitfalls section from production use: mermaid dynamic-import timing (startOnLoad never fires → explicit `mermaid.run()`), dark-mode diagram canvas stays light, empty-grep false negatives (wrong-cwd), agent-claims-are-leads-not-facts, cwd discipline during assembly
