---
name: openlog
description: |-
  Start or resume per-module development work — browse the module catalogue under the project root's `OpenLog/`, quickly pick up an existing module's latest progress, or scaffold a fresh module's documentation (architecture / dev log / current stage). When a task wraps up, automatically maintain these three documents so the next session or agent can re-enter with minimal tokens.
  Trigger conditions: user types `/openlog`, or says things like "start working on module X", "pick up module X", "which module are we doing", "kick off module work", "write a dev-log entry for this module" — any module-level start / wrap-up signal in a large project.
---

# OpenLog — Modular Development Workflow

OpenLog is a lightweight "per-module development journal" mechanism. It pins each module's **architecture, dev log, and current breakpoint** into three fixed files under `OpenLog/<Module>/`, so any handoff target (human or agent) can rebuild the context in under 1k tokens.

**Core goal**: stable, replayable, low-token.

---

## Data layout (project root)

```
OpenLog/
├── INDEX.md                          # Module catalogue (minimal table)
├── <ModuleA>/
│   ├── ARCHITECTURE.md               # Module architecture (structured, kept current)
│   ├── DEV_LOG.md                    # Dev log (reverse chronological, newest on top)
│   ├── DEV_LOG.archive/              # Auto-archive (only when DEV_LOG > 500 lines)
│   │   └── YYYY-MM.md
│   └── CURRENT.md                    # Current breakpoint (< 30 lines)
└── <ModuleB>/...
```

`OpenLog/` follows the project's git repository, so teams / multiple worktrees / different agents share the same journal.

---

## Startup phase (the most token-efficient entry)

Execute in order:

### Step 1: Make sure `OpenLog/INDEX.md` exists

- If the project root has no `OpenLog/`, create it.
- If there is no `INDEX.md`, write the empty template below:

  ```markdown
  # OpenLog Module Index

  | Module | One-line definition | Status |
  |--------|---------------------|--------|
  ```

### Step 2: Read `OpenLog/INDEX.md` (once, in full)

Load the entire module catalogue into context. This is the only "full read" allowed in this phase.

### Step 3: Try **fuzzy module-name matching** (preferred over asking)

If the user **already mentioned a module name or description** when invoking the skill (e.g. "work on the combat module", "pick up X's config"), try to match first:

1. Do a fuzzy keyword match against both the "Module" and "One-line definition" columns of INDEX (substring, case-insensitive, supports CN/EN cross-reference).
2. **Exactly 1 hit** → go straight to branch A, skip the question, but add a confirmation line before the handoff summary: "(auto-matched module `<Module>`)".
3. **2+ hits** → use AskUserQuestion to let the user pick among candidates (use AskUserQuestion when ≤ 4 candidates; otherwise fall back to step 4's text-list flow).
4. **0 hits** → proceed to step 4.

If the user invoked the skill with no module hint (e.g. plain `/openlog`), skip straight to step 4.

### Step 4: Let the user pick a module

**Choose interaction by module count**:

- **≤ 3 modules** (so total options including "+ new module" ≤ 4) → use `AskUserQuestion`
- **≥ 4 modules** → AskUserQuestion can't fit (max 4 options). Print the INDEX table directly and end with:

  > Reply with a module name (any substring works), or reply `new` to create a new module.

  Wait for the user's text reply, then do a fuzzy match again. Same matching rules as step 3.

### Step 5: Branch based on the user's choice

#### Branch A: Pick up an existing module

Read the three files in this **strict order** — **do not parallelise, do not read whole files**:

1. `Read OpenLog/<Module>/ARCHITECTURE.md` (full, usually 1-2 KB)
2. `Read OpenLog/<Module>/DEV_LOG.md`, **first 40 lines only** (use Read's `limit: 40`) — covers the most recent ~5 entries, enough to recover the recent thread
3. `Read OpenLog/<Module>/CURRENT.md` (full, fixed < 30 lines)

**Architecture-drift check** (lightweight, optional):

- Use Bash `stat -f "%m" OpenLog/<Module>/ARCHITECTURE.md` (macOS) or `stat -c %Y ...` (Linux) to get ARCHITECTURE's mtime.
- If > 30 days old AND the DEV_LOG head you just read shows ≥ 5 log entries newer than that mtime (compare entry timestamps to mtime), append to the handoff summary:

  > ⚠️ ARCHITECTURE hasn't been updated for N days, with M new log entries since — consider reviewing the architecture doc.

- Don't force the update; just hint.

Then output a **2-3 sentence handoff summary**:

> **Module purpose**: <one line>
> **Recent progress**: <synthesised from the 1-2 newest log entries>
> **Current breakpoint / next step**: <synthesised from CURRENT's "In Progress" + "Next">
> [⚠️ drift hint, optional]

Then start executing the user's actual instruction.

#### Branch B: Create a new module

1. Use `AskUserQuestion` to collect (in one go):
   - Module name (kebab-case or PascalCase — match the project's code style)
   - One-line purpose
2. Create the directory `OpenLog/<Module>/`
3. Inside, create the three files using the "File templates" section below
4. Append a row to `OpenLog/INDEX.md`: `| <Module> | <one-line purpose> | 🟢 active |`
5. Output one line of confirmation: "Module `<Module>` is set up. Ready to start development." — then wait for the user's next instruction.

#### Branch C: Change module status (pause / done / resume)

If the user's invocation is a status-change action ("pause module X", "X is done", "archive X", "resume X"), skip reading the module docs and:

1. Find the row in INDEX.md, update the "Status" column:
   - Pause → 🟡 paused
   - Done → ✅ done
   - Resume → 🟢 active
2. If becoming ✅ done, optionally rewrite CURRENT.md as a final delivery summary (5-10 lines)
3. Append a status-change record to the top of DEV_LOG.md
4. Output confirmation; do not enter development flow.

---

## Wrap-up phase (maintain documents after a task)

### Trigger gate (important: avoid over-triggering)

**Only** execute wrap-up if the user's instruction is one of:

- Explicit completion signal: "done", "finished", "all good", "ok that's it", "wrap it up"
- Explicit log signal: "wrap up", "log it", "archive this", "record this"
- Explicit transition signal: "next task", "switch modules", "next one"
- Explicit commit signal: "commit it", "update the log"
- User types `/openlog commit` or `/openlog wrapup`

**Forbidden**: triggering wrap-up just because "files were changed" or "a chunk of work seems done". **Wrap-up must be confirmed by the user's linguistic signal**, otherwise you pollute the log.

### Wrap-up steps

Execute in order:

#### 1. Prepend a DEV_LOG.md entry (mandatory)

Insert at the **top** of the file (right after the `# <Module> Dev Log` title):

```markdown
## YYYY-MM-DD HH:MM — <one-line change summary>
- <key change 1>
- Changed: `<path/to/file>:<line>` (one line per file)
- Linked: CURRENT item "<X>" resolved / new blocker "<Y>" added
```

Use local time (e.g. `date "+%Y-%m-%d %H:%M"`). Keep summary ≤ 60 chars.

**Concurrency note**: if the project uses git with remote collaborators, remind the user: "When multiple worktrees / agents are active, run `git pull --rebase` before writing to avoid conflicts in OpenLog/." Don't auto-pull — let the user decide.

#### 2. Update CURRENT.md (mandatory)

- Move completed "In Progress" items to "Done" or delete (keep Done ≤ 5)
- New blockers → write into "Known Issues / TODO"
- Refresh the top "Last updated" timestamp
- **Keep total < 30 lines**. Over the limit → push stale "Done" rows into the DEV_LOG "Linked" field, or delete.

#### 3. Update ARCHITECTURE.md (only when structure changes)

**Only** if this task triggered one of:

- Added / removed / renamed a public interface (method, component, event)
- Added / removed / renamed a module dependency (upstream or downstream)
- Added / removed / renamed a key file
- Landed a design decision worth recording ("why we did it this way", 1-3 lines)

**Pure implementation details don't pollute ARCHITECTURE.** This is the key token-saving rule.

#### 4. Check DEV_LOG archive (only when file is large)

After writing, check DEV_LOG.md line count:

```bash
wc -l OpenLog/<Module>/DEV_LOG.md
```

If **> 500 lines**, archive:

1. Create `OpenLog/<Module>/DEV_LOG.archive/` if missing
2. Find all entries **> 90 days old** (by entry timestamp)
3. Group them by month, append to `DEV_LOG.archive/YYYY-MM.md` (merge with existing files in reverse chronological order)
4. Delete those entries from the main DEV_LOG.md
5. Keep a footer line in DEV_LOG.md: `> Older entries: [DEV_LOG.archive/](DEV_LOG.archive/)`

Archiving doesn't need to happen on every wrap-up — checking once every ~10 wrap-ups is fine.

#### 5. Do NOT update INDEX.md unless status changes

- Module → 🟡 paused (via branch C)
- Module → ✅ done (via branch C)
- Otherwise leave it.

---

## File templates (use when creating new modules)

### ARCHITECTURE.md

```markdown
# <Module> Architecture

## Purpose
(2-3 lines: what problem does this module solve, where does it sit in the system)

## Key Interfaces
- `<Interface>.<Method>(...)` — `<path/to/file>:<line>` — one-line responsibility
- ...

## Dependencies
- Upstream: <list of upstream modules>
- Downstream: <list of downstream modules>

## File List
- `<path/to/file>` — one-line responsibility
- ...

## Design Decisions
(Append as needed, 1-3 lines each. Record "why", not "what".)
```

### DEV_LOG.md

```markdown
# <Module> Dev Log

(Newest entries on top. Each entry 3-5 lines.)
```

### CURRENT.md

```markdown
# <Module> Current Stage

**Last updated**: YYYY-MM-DD HH:MM
**Stage goal**: <one-line goal for this stage>

## Done
- <key delivered items>

## In Progress
- <ongoing work; note blockers in parentheses if any>

## Next
1. <next action>
2. <next action>

## Known Issues / TODO
- [ ] <to-do>
```

---

## Forbidden behaviours (written down so we don't forget)

- ❌ **Don't re-read the entire DEV_LOG** to "understand history" — the reverse-chronological first 40 lines is the designed boundary
- ❌ **Don't dump scratch discussions into ARCHITECTURE** — only commit settled interfaces / dependencies / decisions
- ❌ **Don't let CURRENT exceed 30 lines** — archive or delete
- ❌ **Don't paste full code diffs into any doc** — use `path/to/file:line` references
- ❌ **Don't trigger wrap-up unilaterally** — only when the user gives an explicit linguistic signal
- ❌ **Don't grep the codebase to rebuild module structure** — ARCHITECTURE is the source of truth; if it's stale, update it before acting
- ❌ **Don't write tech-stack filler into ARCHITECTURE** (Unity / C# / React / etc.) — unless that fact has architectural meaning

---

## Expected token cost per invocation

- Startup + pick up existing module: ~700-1000 tokens (INDEX + full ARCHITECTURE + first 40 lines of DEV_LOG + full CURRENT + summary output)
- Startup + new module: ~300 tokens (INDEX + template fill + one-row index update)
- Status change: ~100 tokens (just modify one INDEX row)
- Wrap-up: ~150-400 tokens (DEV_LOG prepend + CURRENT local edit; ARCHITECTURE usually skipped)
- Archive (rare): ~500 tokens

If you exceed these budgets in practice, stop and check whether you violated a "forbidden behaviour".
