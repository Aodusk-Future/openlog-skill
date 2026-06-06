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
├── INDEX.md                          # Module catalogue + cross-module collaboration rules
├── CONVENTIONS.md                    # Optional: cross-cutting engineering conventions (asmdef / test templates / naming...)
├── <ModuleA>/
│   ├── ARCHITECTURE.md               # Module architecture (contract-owner declaration)
│   ├── DEV_LOG.md                    # Dev log (reverse chronological, newest on top) — history of completed actions
│   ├── DEV_LOG.archive/              # Auto-archive (only when DEV_LOG > 500 lines)
│   │   └── YYYY-MM.md
│   ├── CURRENT.md                    # Current breakpoint (< 30 lines; includes external-request inbox) — long-term stage view
│   └── HANDOFF.md                    # Optional, one-shot: end-of-session "scene snapshot" (incomplete / validation / review needed)
└── <ModuleB>/...
```

**Three info layers, distinct semantics** (don't mix):
- **DEV_LOG.md** = **history** of completed actions (reverse chronological, accumulates)
- **CURRENT.md** = the module's **long-term stage** (continuous, cross-session)
- **HANDOFF.md** = the **scene** where the last session stopped (one-shot; file exists = there's a half-finished work waiting; deleted once picked up)

`OpenLog/` follows the project's git repository, so teams / multiple worktrees / different agents share the same journal.

**Two conventions split**:
- **"Cross-module collaboration rules" at the top of INDEX.md** — govern *how modules interact* (contract ownership, request inbox, etc.) — universal meta-rules.
- **CONVENTIONS.md** — governs *how to actually build inside a single module* (project-specific tech practices: Unity asmdef templates, unit-test layout, naming, etc.). This is an **optional file**, only created when the project needs to record cross-cutting tech conventions.

---

## Startup phase (the most token-efficient entry)

Execute in order:

### Step 1: Make sure `OpenLog/INDEX.md` exists

- If the project root has no `OpenLog/`, create it.
- If there is no `INDEX.md`, write the empty template below:

  ```markdown
  # OpenLog Module Index

  > Engineering conventions (cross-cutting) see [CONVENTIONS.md](CONVENTIONS.md) (optional; ignore if absent).

  ## Cross-module collaboration rules
  1. A boundary = a named contract (interface / data shape); depend on the contract, not on the internals.
  2. The more foundational (depended-upon) module owns the contract — keep it stable, prefer additive changes; downstreams write adapters and only file requests.
  3. Contracts live in the owner's ARCHITECTURE → Key Interfaces; consumers cite them in their Dependencies. If mismatch is found, record a TODO locally first and discuss with the owner before changing the contract.
  4. Filing cross-module requests: append to the owner's CURRENT "External Requests (inbox)" (with date + source module), but **never modify the other module's contract/decisions** — contracts only change when the owner triages them into their ARCHITECTURE.

  | Module | One-line definition | Status |
  |--------|---------------------|--------|
  ```

  These "Cross-module collaboration rules" are the skill's default meta-rules, universal across projects. **Do not remove them** unless the user explicitly asks for a different collaboration model.

### Step 2: Read `OpenLog/INDEX.md` (once, in full)

Load the entire module catalogue + the "Cross-module collaboration rules" at the top into context. This is the only "full read" allowed in this phase.

**Conditional read of CONVENTIONS.md**: if INDEX has a pointer to `CONVENTIONS.md` at the top, AND the current task involves any of the following, **also read CONVENTIONS.md** (full read); otherwise skip:
- Creating a **foundational / shallow-dependency** sub-module (may need the project's assembly / test template)
- Touching a **cross-module contract** (interface / data shape / naming)
- The user explicitly says "per the engineering conventions" or "per project standard"
- The user asks "what's our project convention for X?"

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

Read files in this **strict order** — **do not parallelise, do not read whole files**:

1. **First check whether `OpenLog/<Module>/HANDOFF.md` exists** (Bash `test -f`)
   - **Exists** → read it in full (usually < 30 lines). This is the **scene where the last session stopped**, newer than CURRENT, highest priority.
   - **Doesn't exist** → skip; proceed to standard handoff.
2. `Read OpenLog/<Module>/ARCHITECTURE.md` (full, usually 1-2 KB)
3. `Read OpenLog/<Module>/DEV_LOG.md`, **first 40 lines only** (use Read's `limit: 40`) — covers the most recent ~5 entries
4. `Read OpenLog/<Module>/CURRENT.md` (full, fixed < 30 lines)

**Architecture-drift check** (lightweight, optional):

- Use Bash `stat -f "%m" OpenLog/<Module>/ARCHITECTURE.md` (macOS) or `stat -c %Y ...` (Linux) to get ARCHITECTURE's mtime.
- If > 30 days old AND the DEV_LOG head you just read shows ≥ 5 log entries newer than that mtime (compare entry timestamps to mtime), append to the handoff summary:

  > ⚠️ ARCHITECTURE hasn't been updated for N days, with M new log entries since — consider reviewing the architecture doc.

- Don't force the update; just hint.

Then output the **handoff summary** (with an extra section when HANDOFF exists):

> **Module purpose**: <one line>
> **Recent progress**: <synthesised from the 1-2 newest log entries>
> **Current breakpoint / next step**: <synthesised from CURRENT's "In Progress" + "Next">
> [⚠️ drift hint, optional]
>
> 🔴 **Open handoff** (YYYY-MM-DD, left in HANDOFF.md) — only output this block when the file exists:
> - Session goal: <verbatim from HANDOFF "Session goal">
> - Incomplete: N items (<list briefly>)
> - Needs review: M items (<list briefly>)
> - Validation: <one-line summary>
>
> **About closing the handoff**: After you pick up the handoff, ANY wrap-up trigger (including "done" / "finished" / `/openlog commit` etc. — any standard wrap-up phrase) will run the wrap-up step-0 pre-check, which proactively asks "should I close the handoff too?" — so you don't have to remember the specific phrase "handoff done"; it won't get silently forgotten.

Then start executing the user's instruction. **Note**: if the user immediately continues the handoff items, you already have the context — go. If the user pivots to something else, **don't** silently delete HANDOFF.md — it stays as a reminder until the next wrap-up triggers step 0's close flow.

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

#### Branch D: Leave a handoff (create / overwrite HANDOFF.md)

**Trigger phrases** (**must** be user-explicit — never proactively suggest this):
- "leave a handoff" / "write a handoff" / "leave handoff notes"
- "for the next agent..." / "things to watch out for next time..."
- User types `/openlog handoff`

Steps:
1. Use `AskUserQuestion` or a single conversational pass to collect 6 fields (if all info is already in the current conversation, organise it directly — don't re-ask):
   - **Session goal** (one line)
   - **Files changed** (list: path + one-line description)
   - **Key implementation** (decision points, 2-5 items)
   - **Validation status** (ran / didn't run / known failures)
   - **Incomplete** (`[ ]` checkbox list)
   - **Needs review** (flags for the next agent's attention)
2. Write to `OpenLog/<Module>/HANDOFF.md` using the template in the file-templates section (**overwrites** any existing file — a module has at most one open handoff. If you find one already there that hasn't been closed, warn: "An open HANDOFF exists. Overwriting will lose the previous content. Confirm?")
3. **Also** run the standard wrap-up flow (DEV_LOG append + CURRENT update) — handoff supplements wrap-up, doesn't replace it
4. Output confirmation: "Handoff written to HANDOFF.md. Next agent picking up `<Module>` will read it first."

#### Branch E: Close handoff (delete HANDOFF.md + absorb info)

**Trigger phrases**:
- "handoff done" / "handoff closed" / "scene cleared"
- "those handoff items are resolved"
- "delete the handoff"
- User types `/openlog handoff close`

Steps:
1. Read `OpenLog/<Module>/HANDOFF.md` (if absent, tell the user "no open handoff for this module" and stop)
2. **Absorb the key info** — for each item in handoff, decide where it lands:
   - **Incomplete items now completed** → append to current DEV_LOG entry's "linked" field (or add a new DEV_LOG entry)
   - **Incomplete items still incomplete** → move into CURRENT's "In Progress" or "Next"
   - **Review items approved + landing as an interface/contract change** → write into ARCHITECTURE's "Key Interfaces"
   - **Review items rejected** → write a line in the current DEV_LOG entry: "Review decision: no change, reason: <...>"
   - **Validation ❌ known failures still unfixed** → write into CURRENT's "Known Issues / TODO"
3. **Delete HANDOFF.md**
4. Output confirmation: "Handoff closed; N items absorbed into DEV_LOG / CURRENT / ARCHITECTURE."

**Critical**: step 2 must finish before step 3 — deleting straight away loses info. If you're unsure where something belongs, ask the user instead of guessing.

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

#### 0. Pre-check: Does HANDOFF.md exist? (mandatory, before any other step)

Use Bash `test -f OpenLog/<Module>/HANDOFF.md` to check whether an open handoff exists for the current module.

- **Doesn't exist** → skip this step, go directly to step 1
- **Exists** → **you must ask the user first**:

  > Detected an open `OpenLog/<Module>/HANDOFF.md`.
  > Should this wrap-up also close the handoff (absorb HANDOFF content into DEV_LOG / CURRENT / ARCHITECTURE, then delete HANDOFF.md)?
  > - Yes → I'll run branch E first, then standard wrap-up
  > - No → I'll only do standard wrap-up, but I'll warn you at the end that the handoff is still open

  User says "yes" → run the full branch E flow first (read HANDOFF → absorb info → delete HANDOFF.md), then **come back here** and continue with steps 1-5. Note: branch E has already distributed info into the three files; the changes from this current task in steps 1-3 may **partially overlap** with what branch E absorbed — merge writes, avoid duplicates.

  User says "no" → continue with steps 1-5, but **append a warning to the final output**:
  > ⚠️ HANDOFF.md is still open; remember to close it later (just say "handoff done" to run branch E).

**Why this step exists**: "I'm done" / "finished" are the most natural wrap-up phrases users say, but they may ALSO mean "the handoff is done too" — you must ask proactively, never let the handoff get silently forgotten during wrap-up.

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

- Added / removed / renamed a public interface (method, component, event) — these go into "Key Interfaces"
- Added / removed / renamed a module dependency (upstream or downstream)
- Added / removed / renamed a key file
- Landed a design decision worth recording ("why we did it this way", 1-3 lines)

**Pure implementation details don't pollute ARCHITECTURE.** This is the key token-saving rule.

**Contract inbox triage**: if during this task the module's CURRENT "External Requests (inbox)" had any newly accepted requests, transcribe them into Key Interfaces (ARCHITECTURE) and delete from inbox; for rejected ones, note the reason and delete; for deferred ones, move into "Known Issues / TODO". **Do not let inbox accumulate.**

#### 3.5. Append a new cross-cutting convention to CONVENTIONS.md (only if you actually distilled a reusable rule)

**Trigger conditions** (must all hold):
- This task revisited a project-specific practice (asmdef setup / test template / naming rule / directory shape) **back and forth with the user**
- The practice **will be reused in other modules** (not a one-off for this module)
- The user **explicitly confirmed** the rule should be recorded ("write this down" / "we'll always do it this way")

When triggered:
1. If `OpenLog/CONVENTIONS.md` doesn't exist, create it from the file-template section, and prepend a pointer line to `OpenLog/INDEX.md`: `> Engineering conventions (cross-cutting) see [CONVENTIONS.md](CONVENTIONS.md).`
2. Append a new rule at the bottom of CONVENTIONS.md (next C-number in sequence: C1 / C2 / ...), filling in: when to apply / when not / template / "applied to" list.
3. In this DEV_LOG entry, link with one line: "(distilled as CONVENTIONS C<N>)".

**Don't proactively try to manufacture this step** — most wrap-ups don't involve cross-cutting conventions and should skip it.

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

## Key Interfaces (this module's contract declaration as owner)
- `<Interface>.<Method>(...)` — `<path/to/file>:<line>` — one-line responsibility
- ...

## Dependencies
- Upstream: <list of upstream modules> (consumed via their ARCHITECTURE → Key Interfaces contracts)
- Downstream: <list of downstream modules> (consume this module's Key Interfaces)

## File List
- `<path/to/file>` — one-line responsibility
- ...

## Design Decisions
(Append as needed, 1-3 lines each. Record "why", not "what".)
```

**"Key Interfaces" is the contract-owner declaration**: anything here is open to all downstreams. The owner is responsible for "additive-only" stability; downstreams that want new interfaces should append a request to the owner's CURRENT "External Requests (inbox)" and wait for triage — **never edit this section directly.**

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

## External Requests (inbox)
- YYYY-MM-DD [<source module>] <request description>
(Consumers append here; this module as contract owner triages — accepted → moved into ARCHITECTURE Key Interfaces, then deleted; rejected → noted with reason, then deleted; deferred → moved into Known Issues / TODO.)

## Known Issues / TODO
- [ ] <to-do>
```

### HANDOFF.md (optional, one-shot — created by branch D, deleted by branch E)

```markdown
# <Module> Open Handoff

**Left at** YYYY-MM-DD HH:MM
**Left by**: <optional: agent name / user, for traceability>

## Session goal
<one line: what this session aimed to achieve>

## Files changed
- `<path/to/file>` — <one-line description>
- ...

## Key implementation
- <decision point 1, 1-2 lines>
- <decision point 2, 1-2 lines>

## Validation status
- ✅ Ran: <test name / description> (passed)
- ⏳ Didn't run: <untested items / Play Mode walk-through / etc.>
- ❌ Known failures: <describe if any; otherwise "none">

## Incomplete
- [ ] <unfinished item 1>
- [ ] <unfinished item 2>

## Needs review
- <flag for next agent's attention, 1-2 lines>
- <flag for next agent's attention, 1-2 lines>
```

**HANDOFF.md discipline**:
- **One-shot**: a module has at most one open handoff; new writes overwrite the old one (warn first)
- **Closed on pickup**: branch E deletes the file after absorbing info into DEV_LOG / CURRENT / ARCHITECTURE
- **Don't accumulate history**: HANDOFF isn't a log; past handoffs are not kept (anything worth keeping should already be absorbed into DEV_LOG at close-time)
- **Keep it short**: the whole file < 30 lines; large explanations belong in ARCHITECTURE's "Design Decisions" with a reference from handoff

### CONVENTIONS.md (optional — create only when the project needs cross-cutting tech rules)

```markdown
# OpenLog Engineering Conventions (cross-cutting)

> Cross-module **engineering practices** (distinct from INDEX's "Cross-module collaboration rules" — those govern inter-module contracts; these govern how to actually build inside a single module).
> Consult on demand; no need to read every session. Append new conventions here; INDEX only keeps a one-line pointer.

---

## C1. <Convention name, e.g. "Module asmdef + unit-test template">
**Established** YYYY-MM-DD, originated in `<first sub-module that landed it>`.

### When to apply
<Trigger scenarios, 1-3 lines>

### When NOT to apply / defer
<Counter-examples / high-cost scenarios, 1-3 lines>

### Template / steps / key disciplines
(Concrete content — may include code / config snippets / commands)

### Applied to
- `<module path>` — YYYY-MM-DD
- ...
```

Each convention gets one H2 with a sequential number (C1 / C2 / ...). **The content of each convention is project-specific** (language / engine / toolchain decided by the project); the template only standardises the shell structure.

---

## Forbidden behaviours (written down so we don't forget)

- ❌ **Don't re-read the entire DEV_LOG** to "understand history" — the reverse-chronological first 40 lines is the designed boundary
- ❌ **Don't dump scratch discussions into ARCHITECTURE** — only commit settled interfaces / dependencies / decisions
- ❌ **Don't let CURRENT exceed 30 lines** — archive or delete (and don't let "External Requests (inbox)" accumulate > 5 unhandled items)
- ❌ **Don't paste full code diffs into any doc** — use `path/to/file:line` references
- ❌ **Don't trigger wrap-up unilaterally** — only when the user gives an explicit linguistic signal
- ❌ **Don't grep the codebase to rebuild module structure** — ARCHITECTURE is the source of truth; if it's stale, update it before acting
- ❌ **Don't write tech-stack filler into ARCHITECTURE** (Unity / C# / React / etc.) — unless that fact has architectural meaning
- ❌ **Don't put project-specific tech conventions into SKILL.md itself** (Unity asmdef templates, specific test-framework syntax, private naming rules, etc.) — those belong in the project's `CONVENTIONS.md`; the skill only standardises the shell
- ❌ **Don't modify another module's contract/decisions** — only append a request to the other module's CURRENT "External Requests (inbox)" and let the contract owner triage
- ❌ **Don't proactively create HANDOFF.md** — only when the user explicitly requests it via one of the trigger phrases (otherwise you pollute the dir with bogus handoffs)
- ❌ **Don't accumulate history in HANDOFF.md** — it's one-shot scene state; past handoffs must be closed (deleted) and absorbed into the other three files
- ❌ **Don't delete HANDOFF.md without absorbing the info first** — branch E must complete the absorption step or you lose information

---

## Expected token cost per invocation

- Startup + pick up existing module: ~700-1000 tokens (INDEX + full ARCHITECTURE + first 40 lines of DEV_LOG + full CURRENT + summary output)
- Startup + **HANDOFF.md exists**: +150-400 tokens (extra HANDOFF read + extra summary section)
- Startup + **conditional CONVENTIONS read**: +200-600 tokens (only when creating a foundational sub-module / touching contracts / user says "per project conventions")
- Startup + new module: ~300 tokens (INDEX + template fill + one-row index update)
- Status change: ~100 tokens (just modify one INDEX row)
- Wrap-up: ~150-400 tokens (DEV_LOG prepend + CURRENT local edit; ARCHITECTURE usually skipped)
- Wrap-up + inbox triage: +50-150 tokens (per request, just edit/delete)
- Wrap-up + appending a new CONVENTIONS entry: ~300-500 tokens (rare; requires explicit user confirmation)
- Leave handoff (branch D): ~200-500 tokens (collect + write file + run standard wrap-up)
- Close handoff (branch E): ~150-400 tokens (read HANDOFF + absorb into 3 files + delete)
- Archive (rare): ~500 tokens

If you exceed these budgets in practice, stop and check whether you violated a "forbidden behaviour".
