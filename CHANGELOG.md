# Changelog

All notable changes to OpenLog are documented in this file.

Format is loosely based on [Keep a Changelog](https://keepachangelog.com/),
versioning follows [Semantic Versioning](https://semver.org/).

---

## [0.3.0] — 2026-06-06

Session-end handoff mechanism + step-0 wrap-up guard + slim-down pass.

### Added
- **`HANDOFF.md`** — optional, one-shot per-session "scene snapshot" file. Captures session goal, files changed, key implementation, validation status, incomplete items, and review requests for the next agent / session to pick up. File-existence-as-signal: present = there's a half-finished work waiting; absent = standard handoff.
- **Branch D — "leave handoff"** — explicit user-triggered creation of HANDOFF.md (trigger phrases: "留个交接" / "write a handoff" / `/openlog handoff` etc.). Runs alongside standard wrap-up, not as a replacement.
- **Branch E — "close handoff"** — absorbs HANDOFF content into DEV_LOG / CURRENT / ARCHITECTURE based on per-item disposition, then deletes the file. Deletion without absorption is forbidden.
- **Wrap-up step 0 — pre-check HANDOFF.md** — runs before any other wrap-up step; if an open HANDOFF exists, proactively asks "close it together with this wrap-up?" — bridges any natural wrap-up phrase ("done" / "finished" / etc.) to a handoff close prompt.

### Fixed
- **Handoff close was effectively dependent on the user remembering the specific phrase "handoff done"**, causing Codex (and any agent) to silently leave HANDOFF.md open after a normal wrap-up. Step 0 now guarantees the handoff close question is always asked when relevant.

### Changed
- **Slim-down pass** — pure wording compression without removing features:
  - "Forbidden behaviours" reorganised from 12 individual bullets into 3 grouped categories (read limits / write limits / HANDOFF discipline)
  - Token budget reformatted from a 10-row list into a 6-row table (baseline + conditional increments)
  - Multiple sections (data layout split, conditional CONVENTIONS trigger conditions, HANDOFF discipline, branch A close-handoff hint) condensed
  - SKILL.md: 482 → 454 lines · SKILL.en.md: 481 → 453 lines · ~500-600 tokens saved (~8% of skill body)

---

## [0.2.0] — 2026-05-31

Cross-module collaboration mechanism + Codex CLI support.

### Added
- **`CONVENTIONS.md`** — optional cross-cutting engineering conventions file. Project-specific tech practices (asmdef templates, unit-test layouts, naming rules, etc.) live here instead of in the skill itself, keeping the skill distributable.
- **Cross-module collaboration rules** in INDEX.md — 4 universal meta-rules covering contract ownership, additive evolution, contract location in ARCHITECTURE, and inbox-based request routing.
- **External Requests (inbox)** field in CURRENT.md — the official channel for cross-module requests. Only the contract owner may triage and either accept (move into ARCHITECTURE Key Interfaces) / reject (with reason) / defer (move into Known Issues).
- **Contract inbox triage** as part of standard wrap-up step 3 — open inbox items must not accumulate.
- **Conditional `CONVENTIONS.md` read** at startup — only loaded when the task involves a foundational sub-module / cross-module contract / user explicitly references project conventions. Keeps the per-call token cost unchanged for routine work.
- **New convention append flow** (wrap-up step 3.5) — triggered only when the user explicitly confirms a reusable engineering practice should be recorded.
- **Codex CLI support** — same SKILL.md format works in both `~/.claude/skills/openlog/` and `~/.codex/skills/openlog/`. README documents both install paths plus a recommended symlink approach so a single `~/openlog-skill/` source updates both clients.

### Changed
- ARCHITECTURE "Key Interfaces" section explicitly designated as the **contract-owner declaration** — downstream consumers may not edit it; they must route requests via the inbox.

---

## [0.1.0] — 2026-05-23

Initial release.

### Added
- **Three-file per-module structure**: `ARCHITECTURE.md` / `DEV_LOG.md` / `CURRENT.md` — long-term-stable / accumulating-history / current-breakpoint.
- **Five interaction branches** at startup: pick up existing module (A) / create new module (B) / change module status (C).
- **Project-level `INDEX.md`** — minimal module catalogue (name / one-line definition / status).
- **Wrap-up triggered by explicit user signal only** — list of permitted trigger phrases; "files were changed" is not enough.
- **Conditional DEV_LOG archive** — once the log file exceeds 500 lines, entries older than 90 days are split into `DEV_LOG.archive/YYYY-MM.md`.
- **Architecture drift hint** — when ARCHITECTURE.md is 30+ days behind the latest 5 log entries, the startup summary appends a non-blocking warning.
- **Fuzzy module-name matching** at startup — match against both module name and one-line definition before falling back to `AskUserQuestion`.
- **AskUserQuestion graceful fallback** — when module count > 3 (limit of AskUserQuestion options), the skill falls back to a text-list prompt.
- **Token budget per operation** — target of ~1k tokens for typical handoff.

---

## Versioning policy

- **Major** (1.0+) — once the skill stabilises after real-world use; reserved for incompatible workflow changes (e.g., renaming the three core files).
- **Minor** (0.x.0) — new mechanisms or new branches; backward-compatible with existing OpenLog directories.
- **Patch** (0.x.y) — bug fixes, wording compression, documentation updates; never changes the file format or workflow.
