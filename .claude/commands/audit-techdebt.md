# /audit-techdebt

Technical debt audit: TODO/FIXME/HACK markers, type-safety gaps (`: any`), large files, long functions, backward compatibility code, deprecation markers, and handoff-documented debt.

**Target:** `src/**/*.ts`, `src/**/*.tsx`, `**/*.py`, `work/**/*_handoff.md`

---

## Scope Boundary

**This audit covers:** Structural technical debt — TODO/FIXME/HACK markers, `: any` type gaps, large file detection, backward compatibility tracking, deprecation markers, handoff-documented debt, and Plan agent impact scoring.

**NOT covered here (see /audit-performance):** Runtime bottleneck analysis — git churn correlation, sequential await profiling, setTimeout/setInterval cataloging, and refactoring strategy planning. This audit identifies structural debt; `/audit-performance` identifies runtime performance debt.

**NOT covered here (see /audit-deadcode):** Dead code removal — unused exports, orphan files, vulture scanning. This audit flags TODO-delete markers; `/audit-deadcode` owns comprehensive dead code detection and safe-removal planning.

**NOT covered here (see /audit-standards):** Lint trend tracking — per-rule regression detection, historical count comparison, fixability categorization. This audit counts `: any` as a debt metric; `/audit-standards` tracks lint violations over time.

---

## Step 1: Load Shared Infrastructure

1. Read `.claude/commands/audit/shared.md`
2. All output format, severity levels, delta tracking, archival, tech debt integration, commit format, sprint chain, error handling, and completion report rules come from shared.md — do NOT deviate

---

## Step 2: Parse Flags

Parse `$ARGUMENTS` per shared Flag Parsing rules:

1. Strip flags from arguments, set booleans:
   - `AGENT_MODE` — `true` unless `--quick`
   - `COMPARE` — `true` if `--since last`
   - `SPRINT_CHAIN` — `true` if `--sprint`
   - `AUTO_FIX` — `false` always (tech debt audit is read-only, `--fix` is a no-op — debt remediation requires design decisions)
   - `SPRINT_FOLDER` — extracted if `--sprint v*_*` pattern provided
2. Report mode to user:
   ```
   Mode: Full | Quick
   Flags: [active flags, or "none"]
   Note: --fix is ignored for tech debt audit (remediation requires design decisions)
   ```

---

## Step 3: Find Prior Audit

```bash
ls work/*_TechnicalDebt_Audit.md 2>/dev/null | sort -r | head -1
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

Run all phases (skip agent in Phase 3 if `--quick`), collecting findings. Each finding must include: File, Line, Issue, Tool, LOC Impact, Effort.

**Tool rule:** Use the built-in **Grep tool** (NOT bash grep/rg) for ALL pattern scans per shared.md Tool Rules. Issue multiple Grep calls in a **single parallel message** where patterns are independent.

### Tech Debt Delta Logic

In delta mode (`--since last`):
- Apply Delta Mode Protocol from `shared.md`
- Carry forward: TODO table, file sizes, compat items
- Git churn: use `--since={PRIOR_DATE}` not `--since=30 days`

### Phase 1: Marker Scan

Run all grep patterns in parallel:

| Search | Pattern | Files | Tool |
|--------|---------|-------|------|
| TODO markers | `TODO\|FIXME\|HACK\|XXX\|TEMP\|WORKAROUND` | `*.ts`, `*.tsx`, `*.py`, `*.rs` | Grep |
| TypeScript any | `:\s*any` (shared: Type Safety Patterns) | `*.ts`, `*.tsx` | Grep |
| Python Any | `-> Any\|: Any` | `*.py` | Grep |
| Backward compat | `backward.compat\|compat\|legacy.support\|# compat` | all source | Grep |
| Optional compat fields | `Optional\[.*\].*=.*None` with compat comments | `*.py` | Grep |
| Deprecation markers | `deprecated\|DEPRECATED\|@deprecated` | all source | Grep |

### Phase 2: Structural Analysis

1. **Large files** — Files > 500 lines:
   ```bash
   find src/ -name "*.ts" -o -name "*.tsx" | xargs wc -l | sort -rn | head -20
   find <backend-dir> -name "*.py" | xargs wc -l | sort -rn | head -20
   ```
2. **Note files that have grown since last audit** (if prior exists, compare line counts)
3. **Handoff tech debt** — Scan handoff docs for `## Tech Debt Incurred` sections:
   ```bash
   # Find handoffs since prior audit date (or all if no prior)
   find work -name "*_handoff.md" -newer {PRIOR_FILE} 2>/dev/null
   ```
   Grep each for `## Tech Debt` sections and aggregate open items.

### Backward Compatibility Tracking

Track temporary compat code (optional fields, old payload acceptance, conditional version logic, deprecated endpoints). Format:

| Item | Location | Added | Cleanup Criteria |
|------|----------|-------|------------------|
| Optional mediaQuality | POST /telemetry | 011926 | Both PCs updated |

Mark items RESOLVED when cleanup complete and both environments updated.

### Handoff Document Tech Debt

Scan handoffs since PRIOR_DATE (delta mode) or since prior audit date (full mode) for `## Tech Debt Incurred` sections. Format:

| Item | Source Handoff | Sprint | Status |
|------|----------------|--------|--------|
| Consider admin-only for DELETE | 04_users_router_auth.md | v1.5 | OPEN |

Mark RESOLVED when addressed. When debt moves to code TODO, note new location.

### Phase 3: Agent Analysis (if AGENT_MODE = true)

Launch Plan agent with `subagent_type=Plan` AFTER Phase 1-2 complete.

**Delta mode:** Use Agent Scope Narrowing from shared.md — pass prior executive summary + CHANGED_FILES list instead of full exploration prompt.

**Full mode prompt:**
> "Prioritize technical debt by impact and dependencies:
>
> **Findings:** [Insert grep results — TODO/FIXME/HACK, type issues, compat debt]
>
> **Analyze:**
> 1. **Git Churn** — `git log --since='{PRIOR_DATE or 30 days ago}' --name-only | sort | uniq -c | sort -rn`. High-churn + debt = high impact
> 2. **Dependencies** — Fixing X enables fixing Y? Technical blockers?
> 3. **Impact Score** — (severity x commits x lines). HACK=3, FIXME=2, TODO=1
> 4. **Remediation Order** — Quick wins first, group by file for PRs
>
> **Return:** Table with File | Debt | Score | Depends On | Effort, grouped: High (>10), Medium (5-10), Low (<5), Blocked"

**Use agent output to:** Replace raw grep with impact-scored table + remediation order + quick wins.

**On timeout/failure:** Output raw grep findings sorted by file. Note `Agent: None (timeout)` in header.

**Skip agent if `--quick` mode.** Set `Agent: None (--quick)` in header.

---

## Step 6: Assemble Findings Document

Write to `work/MMDDYY_TechnicalDebt_Audit.md` following shared.md Output Format exactly:

### 6a. Header

```markdown
# TechnicalDebt Audit

**Date:** MMDDYY
**Prior:** work/MMDDYY_TechnicalDebt_Audit.md (or "None")
**Mode:** Full | Quick (--quick)
**Agent:** Plan | None (--quick) | None (timeout)
**Tools:** Grep (built-in), Bash (wc -l, find), Read (file analysis)
```

### 6b. Executive Summary

2–3 sentences covering:
- Total debt findings and severity distribution
- Highest-impact area (e.g., `: any` proliferation, large file growth, compat code aging)
- Trend vs prior (if `COMPARE` is true)

### 6c. Metrics Dashboard

| Metric | Value | Prior | Trend |
|--------|-------|-------|-------|
| Total findings | N | — | — |
| Critical | N | — | — |
| High | N | — | — |
| Medium | N | — | — |
| Low | N | — | — |
| TODO/FIXME/HACK markers | N | — | — |
| `: any` usage (TS) | N | — | — |
| `Any` usage (Python) | N | — | — |
| Files > 500 lines | N | — | — |
| Backward compat items | N | — | — |
| Handoff tech debt (open) | N | — | — |

Prior and Trend columns populated only when `COMPARE` is true.

### 6d. Findings Tables

Group by severity per shared.md format. Number findings with **TD-** prefix (TD-1, TD-2, ...).

### 6e. Auto-Fixed Section

```markdown
## Auto-Fixed

N/A — Tech debt audit is read-only. Use findings to plan remediation sessions.
```

### 6f. Delta Section

Only if `COMPARE` is true. Follow shared.md Delta Tracking rules.

### 6g. Historical Section

Per shared.md format.

### 6h. Recommendations

Top 3 highest-impact recommendations. Prioritize:
1. HACK markers (highest severity weight)
2. High-churn files with debt (agent analysis)
3. Aging compat code past cleanup criteria

### 6i. Cross-References

```markdown
## Cross-References

See also: `/audit-codesweep` for duplication analysis. Files appearing in both TODO/FIXME findings and top clone groups are high-priority refactoring targets.
```

---

## Step 7: Post-Processing

Follow shared.md for all of these:

### 7a. Tech Debt Integration

Per shared.md Tech Debt Integration rules.

### 7b. Git Commit

```bash
git add work/MMDDYY_TechnicalDebt_Audit.md
git add docs/archive/audits/  # if priors archived
git add .claude/rules/tech-debt-patterns.md  # if updated

git commit -m "$(cat <<'EOF'
docs(audit): techdebt audit — N findings (X critical, Y high)

Tools: Grep, Bash, Read
Mode: Full | Quick
[Delta: +N new, -N resolved]  (only if --since last)

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
```

### 7c. Completion Report

Output to user per shared.md Completion Report format.

### 7d. Sprint Chain

Only if `SPRINT_CHAIN` is true — per shared.md Sprint Chain rules.

---

## Severity Mapping

| Severity | Criteria |
|----------|----------|
| HIGH | HACK marker (severity weight 3), file >1000 lines, 10+ `: any` in one file |
| HIGH | Compat code past cleanup criteria date |
| MEDIUM | FIXME marker (weight 2), file 500-1000 lines, 5-9 `: any` in one file |
| MEDIUM | Compat code within cleanup window |
| LOW | TODO marker (weight 1), single `: any`, deprecation marker |
| INFO | Handoff tech debt documented but not yet actionable |

---

## Execution Checklist

Before completing, verify all items from shared.md Execution Checklist are met. Key tech debt-specific items:

- [ ] Phase 1: All marker patterns ran (TODO/FIXME/HACK, any, compat, deprecated)
- [ ] Phase 2: Large file scan completed with line counts
- [ ] Phase 2: Handoff tech debt sections scanned and aggregated
- [ ] Phase 2: Backward compat items tracked with cleanup criteria
- [ ] Phase 3: Agent analysis ran (full mode) or skipped (quick mode)
- [ ] Findings numbered with TD- prefix
- [ ] Cross-reference note for `/audit-codesweep` included
- [ ] `--fix` noted as N/A (read-only audit)

---

## See Also

- `/audit-deadcode` — Dead code audit (unused exports, orphan files, dead components)
- `/audit-standards` — Linting + coding standards audit (naming, typing, exceptions, trend tracking)
- `/audit-codesweep` — Code duplication analysis (cross-reference with TODO/FIXME findings)
- `/audit-python` — Python-specific code quality audit
- `/audit-typescript` — TypeScript-specific audit
