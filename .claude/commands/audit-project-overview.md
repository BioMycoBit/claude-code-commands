# /audit-project-overview

Project pulse check: current version status, recent accomplishments, open work items, and sprint priorities. This is a project summary, not a code scan — it provides context for other audits by capturing the project's current state and direction.

**Target:** `CLAUDE.md`, `work/**/*.md`, `git log`

---

## Step 1: Load Shared Infrastructure

1. Read `.claude/commands/audit/shared.md`
2. All output format, severity levels, delta tracking, archival, tech debt integration, commit format, sprint chain, error handling, and completion report rules come from shared.md — do NOT deviate

---

## Step 2: Parse Flags

Parse `$ARGUMENTS` per shared Flag Parsing rules:

1. Strip flags from arguments, set booleans:
   - `AGENT_MODE` — `false` always (no agent needed for project overview)
   - `COMPARE` — `true` if `--since last`
   - `SPRINT_CHAIN` — `true` if `--sprint`
   - `AUTO_FIX` — `false` always (project overview is read-only, `--fix` is a no-op)
   - `SPRINT_FOLDER` — extracted if `--sprint v*_*` pattern provided
2. Report mode to user:
   ```
   Mode: Full | Quick (no difference — this audit has no agent phase)
   Flags: [active flags, or "none"]
   Note: --quick and --fix are no-ops for project overview (no agent, read-only)
   ```

---

## Step 3: Find Prior Audit

```bash
ls work/*_Project_Overview.md 2>/dev/null | sort -r | head -1
```

If found, read it for delta comparison later. If not, set `Prior: None`.

---

## Step 4: Archive Prior (if exists)

Follow shared.md Archival rules:
- `git mv` prior to `docs/archive/audits/MMDDYY/`
- Handle collision with `_2` suffix
- Run BEFORE writing new audit document

---

## Step 5: Run Analysis Phases

Each section gathers project state information. No agent is needed — this audit uses Read, Bash (git log), and Glob only.

### Project Overview Delta Logic

In delta mode (`--since last`):
1. Load PRIOR_BASELINE for this audit (version, accomplishments, priorities)
2. Reference prior for trend comparison (velocity, debt direction)
3. Only re-scan git log since PRIOR_DATE (not full 7 days)
4. CARRY FORWARD: Open issues still open, update status if changed
5. MERGE + MARK: NEW | CARRIED | CHANGED | RESOLVED

If MODE == "full": Run all patterns as below.

### Phase 1: Current Version & Status

1. Read `CLAUDE.md` header for version and status
2. Note the current architecture version (v1.x, v2.0, etc.)
3. Note active stack components and their versions

### Phase 2: Recent Accomplishments

```bash
git log --since="7 days ago" --oneline
```

In delta mode, use `--since=PRIOR_DATE` instead.

Summarize commits into categories:
- Features added
- Bugs fixed
- Refactoring/cleanup
- Documentation
- Tests added

### Phase 3: Open Work Items

1. Glob `work/**/*_overview*.md` for active sprint overviews
2. Scan for status != COMPLETE
3. List open sprints with their current session progress

### Phase 4: Current Priorities

1. Identify in-progress handoffs in `work/` directories
2. Note any blocked sessions
3. Summarize current sprint focus areas

---

## Step 6: Assemble Findings Document

Write to `work/MMDDYY_Project_Overview.md` following shared.md Output Format exactly:

### 6a. Header

```markdown
# Project Overview

**Date:** MMDDYY
**Prior:** work/MMDDYY_Project_Overview.md (or "None")
**Mode:** Full | Quick
**Agent:** None (not applicable — project overview uses no agent)
**Tools:** Read (CLAUDE.md), Bash (git log), Glob (work items)
```

### 6b. Executive Summary

2–3 sentences covering:
- Current project version and phase
- Key accomplishments since last overview (or last 7 days)
- Current focus areas and any blockers

### 6c. Metrics Dashboard

| Metric | Value | Prior | Trend |
|--------|-------|-------|-------|
| Commits (period) | N | — | — |
| Features added | N | — | — |
| Bugs fixed | N | — | — |
| Active sprints | N | — | — |
| Completed sprints (period) | N | — | — |
| Open work items | N | — | — |
| Blocked sessions | N | — | — |

Prior and Trend columns populated only when `COMPARE` is true.

### 6d. Sections

Instead of severity-based findings tables, use project-oriented sections:

```markdown
## Current Version & Architecture

[Version info from CLAUDE.md]

## Recent Accomplishments

| Category | Count | Highlights |
|----------|-------|------------|
| Features | N | [key features] |
| Bug fixes | N | [key fixes] |
| Refactoring | N | [key refactors] |
| Documentation | N | [key docs] |
| Tests | N | [key test additions] |

## Open Work Items

| Sprint | Status | Current Session | Focus |
|--------|--------|-----------------|-------|
| [sprint name] | [status] | [session N of M] | [focus area] |

## Current Priorities

1. [highest priority]
2. [second priority]
3. [third priority]

## Blockers

- [blocker 1, or "None"]
```

### 6e. Auto-Fixed Section

```markdown
## Auto-Fixed

N/A — Project overview is read-only.
```

### 6f. Delta Section

Only if `COMPARE` is true. Follow shared.md Delta Tracking rules:
- New work items not in prior
- Resolved work items (completed since prior)
- Unchanged (still open)

### 6g. Historical Section

If prior existed, list completed items per shared.md format.

### 6h. Recommendations

Top 3 recommendations based on project state:
1. Highest-priority blocker resolution
2. Sprint velocity observations
3. Focus area suggestions

---

## Step 7: Post-Processing

Follow shared.md for all of these:

### 7a. Tech Debt Integration

Skip — project overview does not produce CRITICAL/HIGH code findings.

### 7b. Git Commit

```bash
git add work/MMDDYY_Project_Overview.md
git add docs/archive/audits/  # if priors archived

git commit -m "$(cat <<'EOF'
docs(audit): project overview — [summary of current state]

Tools: Read, Bash, Glob
Mode: Full | Quick
[Delta: +N new items, -N resolved]  (only if --since last)

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
```

### 7c. Completion Report

Output to user per shared.md Completion Report format.

### 7d. Sprint Chain

Only if `SPRINT_CHAIN` is true — per shared.md Sprint Chain rules. Note: project overview rarely produces actionable sprint targets — chain is most useful when combined with other audit findings.

---

## Severity Mapping

Project overview does not use traditional severity levels. Instead, work items are categorized by status:

| Status | Meaning |
|--------|---------|
| ACTIVE | Sprint in progress, on track |
| BLOCKED | Sprint or session blocked by dependency |
| STALE | Sprint with no commits in >7 days |
| COMPLETE | Sprint finished |

---

## Execution Checklist

Before completing, verify:

- [ ] CLAUDE.md read for version and architecture status
- [ ] Git log scanned for accomplishments (correct date range)
- [ ] Open work items listed from work/ directories
- [ ] Current priorities identified
- [ ] Blockers noted (or "None")
- [ ] Agent header set to `None (not applicable)`
- [ ] `--fix` noted as N/A (read-only)

---

## See Also

- `/audit-techdebt` — Technical debt audit (complements project overview with debt tracking)
- `/audit-test-quality` — Test quality audit (testing progress context)
- `/eod-docs` — Full end-of-day documentation (project overview is one component)
