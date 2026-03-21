# Implementation Skill

Execute a handoff document: load plan, research, get approval, implement, lint, test. Does NOT commit — `/handoff-end` owns the commit.

Please review CLAUDE.md for complete system instructions (Core/Workflow/Architecture sections).

## Usage

```
/implement @work/v1.5x_topic/03_feature_handoff.md
/implement @work/v1.5x_topic/03_feature_handoff.md Understood?
```

---

## Step 1: Parse & Load Handoff

### 1a: Find the Handoff Document

**Priority order:**

1. **Arguments** — User passed `@work/.../handoff.md`
2. **Conversation context** — If continuing from `/handoff-end`, use the "Next" doc it created
3. **Ask user** — If neither available:
   ```
   What handoff document are we working from?
   ```

### 1b: Validate

- Confirm the file exists (Read it)
- Extract: **Goal**, **Blast Radius**, **Tasks**, **Success Criteria**, **Stop Conditions**
- If any section is missing or empty, flag it before proceeding

### 1c: Find Sprint Context

Locate the overview doc in the same folder:
```
Glob("00_*.md", path="work/[sprint_folder]")
```

Read the Progress Tracker to understand where this session fits.

---

## Step 2: Research

Read these **in parallel** before writing any code:

### 2a: Memory Files

| File | Why |
|------|-----|
| `.claude/rules/debugging-patterns.md` | Known failure modes to watch for |
| `.claude/rules/codebase-quirks.md` | Import patterns, gotchas, workarounds |
| `.claude/rules/tech-debt-patterns.md` | Known debt in areas you're touching |

Apply this knowledge during implementation (lazy imports, CRLF fixes, pacing delays, etc.).

### 2b: Blast Radius Files

Read **every file** listed in the handoff's Files/Blast Radius table before modifying any of them. Understand existing code before changing it.

### 2c: Adjacent Context (if needed)

If the handoff references other modules, types, or APIs — read those too. Don't guess at interfaces.

---

## Step 3: Present GEAUX Gate

> **Customizable:** "GEAUX" is the default approval keyword — customize it to match your team's style (e.g., "GO", "LGTM", "SHIP IT").

Output this formatted summary for user approval:

```
## Ready to Implement

**Session:** [ID] — [Title]
**Sprint:** [sprint_name]
**Goal:** [one-line goal from handoff]

### Blast Radius
| File | Change |
|------|--------|
| `path/to/file` | [what changes] |

### Stop Conditions
- [stop condition 1]
- [stop condition 2]

### Edge Cases — Correct Behavior
For every edge case or null/missing state identified during research, state the **correct runtime behavior** — not how the types compile.
- [edge case] → [what should actually happen at runtime]
- Example: "no user on F5 refresh → show loading, not a placeholder userId"
- If you can't state the correct behavior, flag it as a question — don't default to a hack.

### Concerns
- [any issues found during research — missing files, unclear spec, debt in area]
- [or "None" if clean]

**GEAUX 🐯🐯🐯** to start | **NO-GEAUX** to adjust
```

**Wait for explicit approval before writing any code.**

---

## Step 4: Execute

### 4a: Task Tracking (3+ tasks)

If the handoff has 3 or more tasks, create a TaskCreate todo list for visibility. Mark each in_progress before starting, completed when done.

### 4b: Implement

Work through the handoff's Tasks section in order. For each task:
1. Read the target file (if not already read in Step 2)
2. **Pre-write verification (VISIBLE):** Before every Edit/Write, state what you verified in your message text — grepped method name, confirmed store API, checked window global exists, etc. If the user doesn't see a verification statement, you skipped it. This applies to implementation code AND test scripts. Never guess an API — grep it first.
3. Make the change
4. Verify the change is correct (re-read if complex)

### 4c: Watch Stop Conditions

If you find yourself drifting into work listed in the handoff's STOP CONDITIONS — **stop and ask the user**. Don't scope-creep.

### 4d: Track Tech Debt

If you take any shortcuts during implementation (inline styles, TODO comments, wrapper divs, hardcoded values, skipped edge cases), note them for Step 6.

---

## Step 5: Lint

Run the appropriate linters based on what was changed. Use whichever linters the project has configured (check CLAUDE.md, `package.json` scripts, `pyproject.toml`, `Makefile`, etc.).

**Examples:**
```bash
# Python projects might use: ruff check, flake8, mypy, pylint
# JS/TS projects might use: npm run lint, eslint, biome
# Rust projects might use: cargo clippy
# Go projects might use: golangci-lint run
```

Run linters for all affected languages in parallel.

**If lint fails:** Fix the issues and re-run. Do not skip linting.

---

## Step 5b: Slop Check

Scan implementation for AI code anti-patterns that linters miss. Run `/slop-check` with the handoff doc.

**Scope:** Only files in the handoff's Blast Radius that were actually changed (intersection of blast radius + `git diff --name-only`).

**Checks (run Grep patterns in parallel):**
1. Placeholder/dummy data (BANNED — CRITICAL if found)
2. Single-use abstractions (new helpers/utilities with only 1 call site)
3. Comment noise (obvious comments, stale markers)
4. Unnecessary defensive code (try/catch around safe internal calls)
5. Scope creep (files changed outside blast radius)
6. Backwards-compat shims (`_unused` vars, re-exports, `// removed`)
7. Feature creep (new params/flags not in handoff spec)

**Gate behavior:**
- 0 findings → proceed to Step 6
- MEDIUM/LOW only → note in Step 7 summary, proceed
- HIGH → warn user, proceed unless they say stop
- CRITICAL → **STOP**. Show findings, fix before continuing

See `/slop-check` for full check definitions and severity mapping.

---

## Step 6: Test

**Auto-trigger `/debug-bt`** when:
- The handoff has testable success criteria (API endpoints, components, state)
- The changes involve frontend services or REST APIs

**Skip `/debug-bt`** for:
- Pure refactoring (no new behavior)
- Docs-only changes
- Config/infrastructure changes

When triggered, `/debug-bt` derives the test script path from the handoff folder.

**STOP HERE** — wait for user to run the test script and report results. Do NOT proceed to Step 7 until user confirms tests pass. Fix any failures before moving on.

---

## Step 7: Summary & Hand Off

**Only after user confirms tests pass.** Do NOT commit — `/handoff-end` handles the commit.

Output to user:

```
## Implementation Complete

**Session:** [ID] — [Title]
**Sprint:** [sprint_name]

### Changes
| File | What Changed |
|------|-------------|
| `path/to/file` | [brief description] |

### Success Criteria
- [x] [criterion 1]
- [x] [criterion 2]
- [ ] [criterion needing user test — note why]

### Tech Debt
- None | [list any shortcuts taken]

### Memory Updates
- [any new debugging patterns or codebase quirks discovered, or "None"]

### Next
Ready for `/handoff-end` to close this session, commit, and create the next handoff.
```

**STOP.** Do not commit. Do not run git add. `/handoff-end` owns the commit.

---

## Error Handling

| Error | Action |
|-------|--------|
| Handoff doc not found | Ask user for correct path |
| Blast radius file missing | Flag it — file may have been moved/renamed. Check git log |
| Goal unclear or too broad | Ask user to clarify before starting |
| Lint fails repeatedly | Fix what you can, flag remaining for user |
| Success criterion can't be verified | Note in summary, defer to user testing |
| Stop condition hit | Stop immediately, show what happened, ask user |
| Memory file at cap | Evaluate oldest entries for removal before appending |
| Handoff missing sections | Flag missing sections, ask user if OK to proceed |

---

## Key Principles

1. **Read before write** — Never modify code you haven't read
2. **Approval before action** — GEAUX gate is mandatory, not decorative
3. **Stop conditions are real** — If you hit one, stop. Don't "just finish this one thing"
4. **Lint is a gate** — Not optional, not skippable
5. **Slop check is a gate** — AI anti-patterns that linters miss. CRITICAL findings = stop
6. **Test then stop** — Prove it works, then hand off. `/implement` never commits.
7. **Track debt honestly** — Shortcuts happen; hiding them makes them worse
8. **One handoff at a time** — Don't pull in work from other sessions

---

## See Also

**Workflow chain:** `/sprint` → `/implement` → `/handoff-end` → `/sprint-end`

- `/sprint` — Create the sprint plan and first handoff that this skill executes
- `/handoff-end` — Close the session after implementation and create the next handoff
- `/debug-bt` — Auto-triggered by this skill when testable success criteria exist
- `/park` — Temporarily set aside a session if blocked

$ARGUMENTS
