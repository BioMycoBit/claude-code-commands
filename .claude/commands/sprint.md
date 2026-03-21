# Sprint Planning Skill

Generate a polished sprint overview with progress tracker, first handoff, amplification, and cleanup.

**Features:** Architectural analysis via Plan agent (default ON) for better session decomposition and file identification.

## Usage

```
/sprint v2.0_auth [focus topic]
/sprint [focus topic]
/sprint --no-architect [focus topic]
```

**Folder shorthand:** When the first argument matches `v*_*` pattern (e.g., `v2.0_auth`, `v1.5b_CacheLayer`):
- **Folder:** `work/{identifier}` → `work/v2.0_auth`
- **Overview:** Extract name after first `_` → `00_{name}_overview.md` → `00_auth_overview.md`
- Remaining arguments become the focus/prompt

**Flags:**
- `--no-architect` - Skip Plan agent analysis (faster, less thorough)
- Default behavior uses Plan agent for architectural analysis

**Examples:**
- `/sprint v2.0_auth contacts and invites` → folder `work/v2.0_auth/`, overview `00_auth_overview.md`, focus "contacts and invites"
- `/sprint v1.5b_CacheLayer` → folder `work/v1.5b_CacheLayer/`, overview `00_CacheLayer_overview.md`, prompts for focus
- `/sprint` → Prompts for focus, then generates plan (with architect)
- `/sprint contacts and invites` → Uses provided focus directly (no folder shorthand)
- `/sprint --no-architect quick fix` → Skip architect for simple sprints

## Output

Creates:
1. `work/[folder]/00_[topic]_overview_v4.md` (polished overview with progress tracker)
2. `work/[folder]/01_[first_session]_handoff.md` (first session handoff document)

---

## Step 1: Parse Arguments & Get Focus

### 1a: Detect Folder Shorthand

Check if the first argument (after any flags) matches the pattern `v{version}_{name}` (e.g., `v2.0_auth`, `v1.5b_CacheLayer`, `v1.3_memoryleak`).

**If shorthand detected:**
1. Extract the full identifier (e.g., `v2.0_auth`)
2. Set **folder** = `work/{identifier}` (e.g., `work/v2.0_auth`)
3. Extract the **name** = everything after the first `_` (e.g., `auth` from `v2.0_auth`, `CacheLayer` from `v1.5b_CacheLayer`)
4. Set **overview** = `00_{name}_overview.md` (e.g., `00_auth_overview.md`)
5. Remaining arguments (after the identifier) become the focus topic

**If no shorthand detected:**
- All arguments are the focus topic
- Folder and overview will use defaults from Step 3

### 1b: Get Focus

**If no focus topic provided** (either no arguments at all, or only a folder shorthand with nothing after), ask the user:

```
What is the focus of this week's sprint?

Examples:
- "Continue Users system buildout"
- "LiveKit integration"
- "Refactoring and technical debt"
- "New feature: group sessions"
```

**If focus topic provided**, use it as the focus.

---

## Step 2: Research Phase

Gather context by reading these sources:

### 2a: Active Work Directory

Check `work/v1_*` directories for:
- Most recent session documents
- Pending/incomplete sessions
- Audit documents

### 2b: Relevant Design Docs

Search for design documents related to the focus:
- `work/*ConnectionLogic*.md`
- `work/*Roadmap*.md`
- `work/*Architecture*.md`

### 2c: Current Technical Debt

Check recent audit docs:
- `work/*Audit*.md`
- `work/*TechnicalDebt*.md`

### 2c-migration: Parity Audit (Migration/Replacement Sprints)

**Trigger:** If the sprint involves **replacing, migrating, or rewriting** existing code (e.g., Python→JS, REST→WebSocket, library swap), this step is MANDATORY.

**Why:** A past migration sprint built a JS SDK replacement across 5 sessions before auditing what the Python SDK actually did. Session 05b (parity audit) discovered 7 gaps — spawning 4 sidequests and inflating a 7-session sprint to 11. The audit should have been Session 01.

**Process:**
1. **Inventory the existing code** being replaced — list every file, count lines, identify all behaviors:
   ```bash
   # Example: find all files in the module being replaced
   find src/old-module/ -name "*.py" -exec wc -l {} +
   ```
2. **Build a parity table** — side-by-side comparison of what the old code does vs what the new code must do:
   - Auth flow (login/logout/re-login)
   - Data flow (what data types, what consumers)
   - Lifecycle (connect/disconnect/reconnect/status monitoring)
   - Cleanup (what happens on logout, page close, error)
   - Edge cases (dual paths, race conditions, stale state)
3. **Number the gaps** — each missing behavior becomes a candidate session
4. **Design sessions around the parity table**, not around the new technology

**Output:** Include a "Parity Audit" section in the overview with the gap table. Every gap must map to a session or be explicitly marked "defer with rationale."

**Lesson:** Audit existing behavior BEFORE designing the replacement. Build-first-audit-later causes scope creep.

### 2d: Architectural Analysis (Plan Agent) - DEFAULT ON

**Skip this step if `--no-architect` flag is present.**

Launch a Plan agent to analyze the codebase for the sprint focus:

```
Task tool with subagent_type=Plan:

"Analyze the codebase architecture for implementing: [FOCUS TOPIC]

Please identify:
1. **Critical files** - Which files will need modification?
2. **Dependencies** - What modules/services are interconnected?
3. **Session decomposition** - Suggest logical breakdown into 3-7 sessions
4. **Blast radius** - What could break if we change these areas?
5. **Architectural concerns** - Any patterns to follow or pitfalls to avoid?
6. **Recommended order** - Which changes should come first?

Focus areas to explore:
- Key source directories identified from project structure
- Configuration and infrastructure files
- Test directories

Return a structured analysis that can inform sprint planning."
```

**What the Plan agent provides:**
- File paths with specific line ranges
- Dependency graphs between modules
- Risk assessment for each proposed change
- Suggested session ordering based on dependencies

**How to use the output:**
- Session breakdown → Progress Tracker sessions
- Critical files → Files tables in handoffs
- Dependencies → Sprint Map ASCII diagram
- Architectural concerns → Risks table
- Recommended order → Session numbering

### 2e: Codesweep Report Discovery

Check for a recent codesweep report and incorporate duplication data into sprint planning:

```bash
# Find the latest codesweep report (date-stamped in work/)
ls -1 work/*_CodeSweep.md 2>/dev/null | sort | tail -1
```

**If a `work/*_CodeSweep.md` exists:**

1. Read the report and extract:
   - **Executive Summary** — total duplication %, clone count, duplicated lines
   - **Top 5 offenders** — highest-scoring clone groups with scores and actions
   - **Refactoring tickets** — the top 15 ticket summaries
   - **Delta section** (if present) — trend direction, new/resolved counts

2. Use this data in sprint planning:
   - Include duplication hotspots as candidate sessions (especially high-score groups touching sprint-focus files)
   - Factor refactoring tickets into session decomposition when they overlap with the sprint focus
   - Note duplication trends in the Context/Velocity section of the overview
   - If the sprint was invoked via `/audit-codesweep --sprint`, make refactoring the **primary** sprint theme — build sessions around the top offender groups

3. Add a **Duplication Context** section to the overview:
   ```markdown
   ### Duplication Context (from /audit-codesweep)

   | Metric | Value |
   |--------|-------|
   | Duplication | {percentage}% |
   | Clone groups | {group_count} |
   | Top offender | {fileA} ↔ {fileB} (score {score}) |
   | Trend | {arrow} {trend_word} (if delta available) |

   *Source: work/MMDDYY_CodeSweep.md*
   ```

**If no report exists**, skip silently — codesweep data is supplementary, not required.

### 2f: Memory: Velocity

Read `memory/velocity.md` if it exists. Use historical velocity data (sessions/day, effort accuracy, completion rates) to size sessions realistically. If the file is empty or missing, skip — velocity is supplementary.

---

## Step 3: Generate Overview Document

Create `work/[folder]/00_[topic]_overview.md` with this structure.

**If Plan agent was used (default):**
- Use agent's session decomposition for Progress Tracker
- Use agent's critical files for Files tables
- Use agent's dependencies for Sprint Map
- Use agent's architectural concerns for Risks table
- Use agent's recommended order for session numbering

**CRITICAL:** Progress Tracker MUST be immediately after the header, before any other content.

```markdown
# [Topic] Overview

**Sprint:** v1.5x_[topic]
**Status:** IN PROGRESS
**Date:** [Date]
**Location:** [UI path if applicable]

---

## Progress Tracker

| Session | Status | Date | Notes |
|---------|--------|------|-------|
| 01 - [Title] | ⏳ NEXT | | [Brief description] |
| 02 - [Title] | PENDING | | [Brief description] |
| 03 - [Title] | PENDING | | [Brief description] |
| 04 - [Title] | PENDING | | [Brief description] |
| 05 - [Title] | PENDING | | [Brief description] |

**Legend:** ✅ COMPLETE | ⏳ NEXT | PENDING | ❌ BLOCKED

---

## Executive Summary

[2-3 sentences: what this sprint addresses, root cause, recommended fix]

---

## Context

[Recent velocity from memory/velocity.md — sprints closed, sessions completed, themes]

This week's focus:
1. [Goal 1]
2. [Goal 2]
3. [Goal 3]

---

## Sprint Map

[ASCII box diagram showing phases and sessions]

---

## Phase A: [Name]

[Session table with Goal, Effort, Files]

### [Session]: [Title]

[Details, code snippets, blast radius]

> ### ELI5: [Concept]
> [Simple analogy]

---

[Repeat for Phases B, C, D as needed]

---

## Success Criteria

| Milestone | Validation |
|-----------|------------|
| [Goal] | [How to verify] |

---

## Handoff Documents

[File tree of planned session docs]

---

## Risks

| Risk | Prob | Impact | Mitigation |
|------|------|--------|------------|

---

## Dependencies

[ASCII diagram showing session dependencies]

---

## Go/No-Go

- [ ] Plan approved
- [ ] Launch scripts work
- [ ] [Other prerequisites]

---

## Keywords

[3-4 column table of 15-25 terms]

---

*[Source attribution]*
```

---

## Step 4: Amplify (Internal)

Run 4 amplification passes on the generated plan:

### Pass 1: Initial
Copy to `_v1.md`

### Pass 2: Print-Worthy
Apply clarity, flow, examples, gap-filling, formatting consistency.

### Pass 3: ELI5 + Deep Dive
Add ELI5 boxes for key concepts (2-3 minimum).
Add Deep Dive boxes for technical details (2-3 minimum).

### Pass 4: Tighten + Keywords
- Reduce word count 15-20%
- Add keywords table
- Final polish

---

## Step 5: Cleanup

After amplification completes:

1. Delete intermediate files:
   - `work/MMDDYY_WeekSprint.md` (source)
   - `work/MMDDYY_WeekSprint_v1.md`
   - `work/MMDDYY_WeekSprint_v2.md`
   - `work/MMDDYY_WeekSprint_v3.md`

2. Keep only:
   - `work/MMDDYY_WeekSprint_v4.md`

3. Rename final to remove version suffix (optional, based on preference):
   - Keep as `_v4.md` to indicate it's polished

---

## Step 6: Create First Handoff Document

After overview is complete, create the first session handoff:

**File:** `work/[folder]/01_[first_session_title]_handoff.md`

```markdown
# Session 01: [Title]

**Sprint:** v1.5x_[topic]
**Status:** ⏳ NEXT
**Prev:** 00_[topic]_overview_v4.md
**Next:** 02_[next_session]_handoff.md

---

## Goal

[One sentence: what this session accomplishes]

---

## Context

[Brief background from overview - why this session matters]

---

## Tasks

- [ ] [Specific task 1]
- [ ] [Specific task 2]
- [ ] [Specific task 3]

---

## Files

| File | Change |
|------|--------|
| `path/to/file.ts` | [What changes] |

---

## Success Criteria

- [ ] [Measurable outcome 1]
- [ ] [Measurable outcome 2]

---

## Notes

[Any additional context, gotchas, or dependencies]

---

*[Session tagline]*
```

**Naming Convention:**
- Use lowercase with underscores: `01_speed_audit_handoff.md`
- Extract title from Progress Tracker first session
- Keep filename under 40 chars

---

## Step 7: Commit

Commit the overview and first handoff:

```bash
git add work/[folder]/00_[topic]_overview_v4.md work/[folder]/01_[session]_handoff.md
git commit -m "docs([folder]): add overview and first handoff

Overview: 00_[topic]_overview_v4.md
First handoff: 01_[session]_handoff.md
Sessions: [count] planned

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

## Step 8: Summary

Output summary to user:

```
## Sprint Setup Complete

**Folder:** work/[folder]/
**Overview:** 00_[topic]_overview_v4.md
**First Handoff:** 01_[session]_handoff.md

### Sessions Planned
| # | Title | Status |
|---|-------|--------|
| 01 | [Title] | ⏳ NEXT |
| 02 | [Title] | PENDING |
| ... | ... | ... |

### Files Created
- `work/[folder]/00_[topic]_overview_v4.md` (amplified)
- `work/[folder]/01_[session]_handoff.md`

Ready for GO/NO-GO on Session 01?
```

---

## Execution Notes

- **Architect ON by default**: Plan agent runs unless `--no-architect` flag present
- **Architect adds value**: Better session decomposition, accurate file lists, dependency awareness
- **Skip architect when**: Simple sprints, docs-only work, time-sensitive situations
- **Progress Tracker first**: ALWAYS place Progress Tracker immediately after header in overview
- **Research first**: Gather context from code/docs before generating
- **Create folder**: If folder doesn't exist, create it
- **ELI5 required**: Every overview needs 2-3 ELI5 boxes minimum
- **Deep Dive required**: Every overview needs 2-3 Deep Dive boxes minimum
- **Cleanup automatic**: Only v4 survives (delete source + v1, v2, v3)
- **First handoff**: Always create 01_ handoff after overview
- **Commit both**: Overview and first handoff committed together

---

## Error Handling

| Error | Action |
|-------|--------|
| No git history | Skip velocity analysis, note in plan |
| No work directory | Create `work/` |
| Focus unclear | Ask clarifying questions |
| Amplify fails | Keep best version available |
| Plan agent fails | Continue without architect findings, note in summary |
| Plan agent timeout | Fall back to manual research, proceed |

---

## See Also

**Workflow chain:** `/sprint` → `/implement` → `/handoff-end` → `/sprint-end`

- `/implement` — Execute a handoff document created by this skill
- `/handoff-end` — Close a session and create the next handoff
- `/sprint-end` — Close the entire sprint with retrospective and archive
- `/random-sprint` — Collect ad-hoc items then delegate to this skill
- `/audit-codesweep --sprint` — Chain duplication scan into sprint planning

---

## Example Execution

### With Architect (Default)

```
User: /sprint v1.5c_screensavers Fix speed issues

Claude: [Parses shorthand: folder=work/v1.5c_screensavers, overview=00_screensavers_overview.md]
Claude: [Creates folder work/v1.5c_screensavers/]
Claude: [Research phase - git history, work docs, audits]
Claude: [Launches Plan agent for architectural analysis...]
  Plan agent: "Analyzed screensaver codebase. Found:
    - Critical files: renderer.ts, particles.ts, starfield.ts
    - Dependencies: All renderers share AnimationLoop base
    - Suggested sessions: 1) Audit 2) AnimationLoop fix 3) Renderer-specific
    - Risk: Changing AnimationLoop affects all screensavers"
Claude: [Generates overview using architect findings]
Claude: [Runs 4 amplification passes]
Claude: [Deletes v1, v2, v3, source]
Claude: [Creates 01_speed_audit_handoff.md with accurate file list]
Claude: [Commits both files]
Claude:
## Sprint Setup Complete

**Folder:** work/v1.5c_screensavers/
**Overview:** 00_screensavers_overview_v4.md
**First Handoff:** 01_speed_audit_handoff.md
**Architect:** Plan agent analysis included

### Sessions Planned
| # | Title | Status |
|---|-------|--------|
| 01 | Speed Audit & Quick Fixes | ⏳ NEXT |
| 02 | AnimationLoop Optimization | PENDING |
| 03 | Starfield Renderer Fix | PENDING |
| 04 | Particles Velocity System | PENDING |

Ready for GO/NO-GO on Session 01?
```

### Without Architect (Fast Mode)

```
User: /sprint --no-architect quick README update

Claude: [Skips Plan agent - fast mode]
Claude: [Research phase only - git history, work docs]
Claude: [Generates overview based on research]
...
```
