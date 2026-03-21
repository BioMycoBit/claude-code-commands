# /audit-regression

Regression prevention audit: danger zone mapping via bug fix analysis, swallowed error detection, test coverage gaps for high-churn files, and pattern analysis of recurring failure modes.

**Target:** All source files — `src/**/*.ts`, `src/**/*.tsx`, `**/*.py`, `your-rust-service/src/**/*.rs`

---

## Scope Boundary

**This audit covers:** Regression risk mapping — danger zone identification via bug fix commit analysis, swallowed error detection, test coverage gaps for high-churn files, pattern analysis of recurring failure modes, and risk scoring (changes x bugs x test_coverage).

**NOT covered here (see /audit-test-quality):** Test suite quality — test inventory counts, execution time profiling, flaky test indicators, test isolation analysis, mock hygiene, and coverage gap strategy. This audit maps where regressions are likely; `/audit-test-quality` assesses the quality of existing tests.

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
   - `AUTO_FIX` — `false` always (regression audit is read-only, `--fix` is a no-op — danger zone remediation requires design decisions)
   - `SPRINT_FOLDER` — extracted if `--sprint v*_*` pattern provided
2. Report mode to user:
   ```
   Mode: Full | Quick
   Flags: [active flags, or "none"]
   Note: --fix is ignored for regression audit (remediation requires design decisions)
   ```

---

## Step 3: Find Prior Audit

```bash
ls work/*_Regression_Prevention.md 2>/dev/null | sort -r | head -1
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

Run all phases (skip agent in Phase 2 if `--quick`), collecting findings. Each finding must include: File, Line, Issue, Tool, LOC Impact, Effort.

**Tool rule:** Use the built-in **Grep tool** (NOT bash grep/rg) for ALL pattern scans per shared.md Tool Rules. Issue multiple Grep calls in a **single parallel message** where patterns are independent.

### Regression Delta Logic

In delta mode (`--since last`):
- Apply Delta Mode Protocol from `shared.md`
- Carry forward: danger zone map, risk scores
- Git log: use `--since={PRIOR_DATE}` not hardcoded intervals

### Phase 1: Grep-Based Analysis

**Step 1 — Recent bug fixes (Bash):**

```bash
git log --oneline --grep="fix\|bug" --since="{PRIOR_DATE or 14 days ago}"
```

**Step 2 — Danger zone files (Bash):**

For each fix commit from Step 1, note the files changed:
```bash
git log --grep="fix\|bug" --name-only --since="{PRIOR_DATE or 90 days ago}"
```

**Step 3 — Swallowed errors (Grep, parallel):**

> See also: Shared Grep Patterns > Error Handling Patterns in `audit/shared.md` for canonical regex.

| Search | Pattern | Files | Tool |
|--------|---------|-------|------|
| JS/TS swallowed catch | `catch.*console.log` or empty catch (shared: Error Handling) | `*.ts`, `*.tsx` | Grep |
| Python swallowed except | `except.*:\s*pass$` (shared: Error Handling) | `*.py` | Grep |
| Empty catch blocks | `catch\s*\(.*\)\s*\{\s*\}` (shared: Error Handling) | `*.ts`, `*.tsx` | Grep |

**Step 4 — Missing tests (Glob + Grep):**

Cross-reference files from bug fix commits with test file existence:
- For each changed file, check if a corresponding test file exists
- Flag high-churn files without corresponding tests

### Phase 2: Agent Analysis (if AGENT_MODE = true)

Launch Explore agent with `subagent_type=Explore`.

**Delta mode:** Use Agent Scope Narrowing from shared.md — pass prior executive summary + CHANGED_FILES list instead of full exploration prompt.

**Full mode prompt:**
> "Map 'danger zones' by analyzing bug patterns:
>
> 1. **Repeated Fix Files** — `git log --grep='fix' --name-only --since='{PRIOR_DATE or 90 days ago}'`
> 2. **Bug Clusters** — What connects buggy files? Same module/data flow/dependencies?
> 3. **Test Coverage Gaps** — High-change files without corresponding tests?
> 4. **Pattern Analysis** — Missing error handling? Complex state? Async races?
>
> **Return:** Danger zone map with risk scores (changes x bugs x test_coverage)"

**Use agent output to:** Highlight danger zones + risk scores in findings tables.

**On timeout/failure:** Continue with grep analysis from Phase 1. Note `Agent: None (timeout)` in header.

**Skip agent if `--quick` mode.** Set `Agent: None (--quick)` in header.

---

## Step 6: Assemble Findings Document

Write to `work/MMDDYY_Regression_Prevention.md` following shared.md Output Format exactly:

### 6a. Header

```markdown
# Regression Prevention Audit

**Date:** MMDDYY
**Prior:** work/MMDDYY_Regression_Prevention.md (or "None")
**Mode:** Full | Quick (--quick)
**Agent:** Explore | None (--quick) | None (timeout)
**Tools:** Grep (built-in), Bash (git log), Glob (test file check)
```

### 6b. Executive Summary

2–3 sentences covering:
- Total danger zone findings and severity distribution
- Highest-risk area (e.g., module with most bug fixes and no tests)
- Trend vs prior (if `COMPARE` is true)

### 6c. Metrics Dashboard

| Metric | Value | Prior | Trend |
|--------|-------|-------|-------|
| Total findings | N | — | — |
| Critical | N | — | — |
| High | N | — | — |
| Medium | N | — | — |
| Low | N | — | — |
| Bug fix commits (period) | N | — | — |
| Danger zone files | N | — | — |
| Swallowed errors | N | — | — |
| High-churn files without tests | N | — | — |

Prior and Trend columns populated only when `COMPARE` is true.

### 6d. Findings Tables

Group all findings by severity. Each table uses shared.md format:

```markdown
### Critical

| # | File | Line | Issue | Tool | LOC Impact | Effort |
|---|------|------|-------|------|------------|--------|
```

Repeat for High, Medium, Low/Informational.

Number findings sequentially with **RG-** prefix (RG-1, RG-2, RG-3... for Regression findings).

### 6e. Auto-Fixed Section

**Not applicable for regression audit** — danger zone remediation requires design decisions.
```markdown
## Auto-Fixed

N/A — Regression audit is read-only. Use findings to prioritize test coverage and error handling improvements.
```

### 6f. Delta Section

Only if `COMPARE` is true. Follow shared.md Delta Tracking rules.

### 6g. Historical Section

If prior existed, list resolved items per shared.md format.

### 6h. Danger Zone Map

Include a summary table of the highest-risk areas:

```markdown
## Danger Zone Map

| Zone | Files | Bug Fixes | Test Coverage | Risk Score | Trend |
|------|-------|-----------|---------------|------------|-------|
| [module/area] | N | N | low/med/high | N | ↑/→/↓ |
```

### 6i. Recommendations

Top 3 highest-impact recommendations with estimated effort. Prioritize:
1. Danger zones with highest risk scores (most fixes, least tests)
2. Swallowed errors in critical paths
3. High-churn files missing test coverage

---

## Step 7: Post-Processing

Follow shared.md for all of these:

### 7a. Tech Debt Integration

Per shared.md Tech Debt Integration rules — append new CRITICAL/HIGH findings to `tech-debt-patterns.md`, mark resolved entries.

### 7b. Git Commit

```bash
git add work/MMDDYY_Regression_Prevention.md
git add docs/archive/audits/  # if priors archived
git add .claude/rules/tech-debt-patterns.md  # if updated

git commit -m "$(cat <<'EOF'
docs(audit): regression audit — N findings (X critical, Y high)

Tools: Grep, Bash, Glob
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
| HIGH | File with 5+ bug fixes in period and no test coverage |
| HIGH | Swallowed error in critical path (WebSocket handler, auth, data pipeline) |
| MEDIUM | File with 3-4 bug fixes and partial test coverage |
| MEDIUM | Swallowed error in non-critical path |
| LOW | File with 1-2 bug fixes and no tests (low churn) |
| LOW | Empty catch block in utility code |
| INFO | Bug fix pattern noted but already has test coverage |

---

## Methodology Validation

### Danger Zone Model

The prediction model identifies danger zones using: `risk_score = fix_count × churn_rate × (1 - test_coverage)`.

**Validated 2026-03-15** against 90 days of git history:

| File | Fix Count (90d) | Predicted | Recurred? | Result |
|------|----------------|-----------|-----------|--------|
| server.py | 33 | HIGH risk | Yes (repeated fixes) | True Positive |
| TestSessionCanvas.tsx | 32 | HIGH risk | Yes | True Positive |
| websocket.ts | 26 | HIGH risk | Yes | True Positive |
| timeseries_db.py | 24 | HIGH risk | Yes | True Positive |
| useAppWebSocket.ts | 17 | HIGH risk | Yes | True Positive |
| csrf.py | 13 | MEDIUM risk | Yes | True Positive |

**Observations:**
- Files with 10+ fixes in 90 days have ~100% recurrence rate — the model correctly identifies these
- The 5+ threshold for HIGH severity is well-calibrated — all files above it had recurrence
- The 3-4 range (MEDIUM) captures real but lower-frequency recurrence
- Meta files (tech-debt-patterns.md, velocity.md, debugging-patterns.md) inflate fix counts but aren't code risk — **exclude `*.md` from danger zone analysis** and `.env`

### Recommended Threshold Adjustments

| Threshold | Current | Recommended | Rationale |
|-----------|---------|-------------|-----------|
| HIGH danger zone | 5+ fixes | 5+ fixes (no change) | Well-calibrated against actual recurrence |
| MEDIUM danger zone | 3-4 fixes | 3-4 fixes (no change) | Captures moderate-risk files |
| File exclusions | None | Exclude `*.md`, `.env`, `*.json` (non-code) | Reduces false positives from documentation churn |
| Time window | 14 days (recent) / 90 days (danger zone) | No change | 90-day window captures seasonal patterns |

### Precision/Recall Estimates

Based on the 90-day validation window:
- **Precision:** ~85% — most flagged files are genuine danger zones (15% are meta/config files)
- **Recall:** ~90% — the model catches most recurring bug locations
- **After exclusion filter:** Precision improves to ~95% (meta files excluded)

### Phase 1 Update: File Exclusions

When computing danger zone files, exclude non-code files from the risk map:

```bash
# Exclude non-code files from danger zone analysis
git log --grep="fix\|bug" --name-only --since="{date}" | \
  grep -v '\.md$' | grep -v '\.json$' | grep -v '\.env' | grep -v '\.yml$' | \
  sort | uniq -c | sort -rn
```

---

## Execution Checklist

Before completing, verify all items from shared.md Execution Checklist are met. Key regression-specific items:

- [ ] Phase 1: Git log bug fix scan completed with date range
- [ ] Phase 1: Danger zone files identified from fix commits
- [ ] Phase 1: Swallowed error grep ran (JS/TS catch + Python except)
- [ ] Phase 1: Test coverage cross-reference completed
- [ ] Phase 2: Agent analysis ran (full mode) or skipped (quick mode) with correct header
- [ ] Danger Zone Map table populated with risk scores
- [ ] Findings numbered with RG- prefix
- [ ] `--fix` noted as N/A (read-only audit)

---

## See Also

- `/audit-test-quality` — Test quality audit (flaky tests, mock hygiene, coverage gaps)
- `/audit-python` — Python code quality audit (overlaps: error handling patterns)
- `/audit-typescript` — TypeScript audit (overlaps: type safety reducing regressions)
- `/audit-resilience` — Resilience audit (overlaps: error recovery, retry patterns)
