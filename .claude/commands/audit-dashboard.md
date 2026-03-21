# /audit-dashboard

Unified health dashboard audit: reads all recent audit documents, computes composite health scores by domain, identifies cross-audit file hotspots, and derives DORA metrics from git history.

**Target:** `work/*_Audit.md`, `work/*_WCAG_Audit.md`, git history

---

## Scope Boundary

**This audit covers:** Aggregation and correlation of findings from all other audits — composite health scoring by domain (Security, Quality, Reliability, Operations), cross-audit file hotspot detection (files flagged by 3+ audits), and DORA metrics derived from git history and sprint velocity data.

**NOT covered here:** Individual audit findings (see each `/audit-*` command). This audit reads existing audit outputs — it does not re-scan code.

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
   - `AUTO_FIX` — `false` always (`--fix` is a no-op — dashboard is read-only aggregation)
   - `SPRINT_FOLDER` — extracted if `--sprint v*_*` pattern provided
2. Report mode to user:
   ```
   Mode: Full | Quick
   Flags: [active flags, or "none"]
   Note: --fix is ignored for dashboard audit (read-only aggregation)
   ```

---

## Step 3: Find Prior Audit

```bash
ls work/*_Dashboard_Audit.md 2>/dev/null | sort -r | head -1
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

Run all 4 phases (or 3 if `--quick` — skip agent analysis in Phase 4), collecting findings. Each finding must include: File, Line, Issue, Tool, LOC Impact, Effort.

**Tool rule:** Use the built-in **Grep tool** (NOT bash grep/rg) and **Read tool** for all analysis per shared.md Tool Rules.

### Dashboard Delta Logic

In delta mode (`--since last`):
1. Load PRIOR_BASELINE for this audit
2. Compare composite scores: prior vs current
3. Compare hotspot list: files added/removed from hotspot list
4. Compare DORA metrics: prior vs current

### Phase 1: Collect Audit Documents

Gather all recent audit documents from `work/`:

```bash
ls work/*_Audit.md work/*_WCAG_Audit.md 2>/dev/null | sort -r
```

**For each audit document found:**

1. Read the Metrics Dashboard section
2. Extract: Total findings, Critical, High, Medium, Low counts
3. Extract: Date of the audit
4. Map the audit to its domain (see Domain Mapping below)

**Domain Mapping:**

| Domain | Audits |
|--------|--------|
| Security | security, secrets, privacy, dependency, docker |
| Quality | python, typescript, rust, standards, codesweep, deadcode, types |
| Reliability | resilience, pipeline, regression, test-quality |
| Operations | observability, performance, sbom, iac |
| Frontend | accessibility, browser-compat |
| Architecture | modules, api-surface |

If no audit documents found:
```
No prior audit documents found in work/. Run individual audits first.
Dashboard requires at least 1 audit document to produce scores.
```
= HIGH finding (DA-1: No audit data available)

**Severity mapping:**
- Audit domain with 0 audits ever run = HIGH (blind spot)
- Audit document >30 days old = MEDIUM (stale data)
- Audit document >90 days old = HIGH (severely stale)
- All domains have recent audits = no finding

**LOC Impact:** N/A (meta-audit)
**Effort:** `small` per missing audit (run the audit)

### Phase 2: Composite Health Scoring

For each domain, compute a health score (0-100):

**Scoring Formula:**

```
domain_score = 100 - (critical_count * 25) - (high_count * 10) - (medium_count * 3) - (low_count * 1)
domain_score = max(0, min(100, domain_score))  # Clamp to 0-100
```

**Output table:**

```markdown
## Domain Health Scores

| Domain | Score | Critical | High | Medium | Low | Audits Included | Stalest Audit |
|--------|-------|----------|------|--------|-----|-----------------|---------------|
| Security | 72 | 0 | 2 | 3 | 5 | security, secrets | 031526 |
| Quality | 85 | 0 | 1 | 2 | 3 | python, typescript | 031426 |
| ... | ... | ... | ... | ... | ... | ... | ... |
| **Overall** | **78** | **0** | **5** | **12** | **20** | **N audits** | **MMDDYY** |
```

**Overall score** = weighted average of domain scores:
- Security: 30% weight (highest risk)
- Quality: 25% weight
- Reliability: 20% weight
- Operations: 15% weight
- Frontend: 5% weight
- Architecture: 5% weight

**Severity mapping:**
- Domain score <50 = CRITICAL (domain is in poor health)
- Domain score 50-69 = HIGH (domain needs attention)
- Domain score 70-84 = MEDIUM (domain is acceptable)
- Domain score 85-100 = no finding (domain is healthy)

### Phase 3: Cross-Audit File Hotspots

Identify files that appear in findings across multiple audits.

**Process:**

1. For each audit document, extract the Findings tables
2. Parse `File` column from each finding row
3. Count how many distinct audits flag each file
4. Rank files by audit count (descending)

**Output table:**

```markdown
## Cross-Audit File Hotspots

| File | Audits Flagging | Audit Names | Total Findings | Highest Severity |
|------|-----------------|-------------|----------------|------------------|
| server.py | 5 | python, security, resilience, performance, standards | 12 | HIGH |
| WebRTCService.ts | 4 | typescript, security, performance, browser-compat | 8 | MEDIUM |
| ... | ... | ... | ... | ... |
```

Only include files flagged by **3+ distinct audits** (hotspot threshold).

**Severity mapping:**
- File flagged by 5+ audits = CRITICAL (systemic issue, likely needs refactoring)
- File flagged by 4 audits = HIGH
- File flagged by 3 audits = MEDIUM

**LOC Impact:** Varies per file
**Effort:** `large` (hotspot files need focused refactoring sprints)

### Phase 4: DORA Metrics (agent-driven)

Derive DORA metrics from git history and sprint data.

**Full mode (agent-driven):**

If `AGENT_MODE` is true, launch Plan agent with `subagent_type=Plan`:

> "Analyze DORA metrics from the this project repository:
>
> 1. **Deployment Frequency** — Count git tags matching `v*` pattern in the last 90 days. Also count merges to master.
> 2. **Lead Time for Changes** — Average time from first commit on a branch to merge (use `git log --merges` for last 10 merges).
> 3. **Change Failure Rate** — Ratio of commits with `fix:` or `hotfix:` prefix to total commits in last 90 days.
>
> **Return:** DORA metrics table with values, industry benchmarks (Elite/High/Medium/Low per DORA report), and this project's tier for each metric."

**Skip agent if `--quick` mode.** Use git commands directly for basic counts:

```bash
# Deployment frequency (tags in last 90 days)
git tag --sort=-creatordate | head -20

# Change failure rate (fix commits / total commits, last 90 days)
git log --oneline --since="90 days ago" | wc -l
git log --oneline --since="90 days ago" --grep="^fix" | wc -l
```

**DORA Benchmarks (2023 State of DevOps Report):**

| Metric | Elite | High | Medium | Low |
|--------|-------|------|--------|-----|
| Deployment Frequency | On-demand (multiple/day) | Weekly-monthly | Monthly-biannually | Fewer than biannually |
| Lead Time | <1 day | 1 day - 1 week | 1 week - 1 month | >1 month |
| Change Failure Rate | <5% | 5-10% | 10-15% | >15% |

**Severity mapping:**
- DORA metric at "Low" tier = HIGH
- DORA metric at "Medium" tier = MEDIUM
- DORA metric at "High" or "Elite" tier = no finding

**On timeout/failure:** Output raw git counts without tier classification. Note `Agent: None (timeout)` in header.

---

## Step 6: Assemble Findings Document

Write to `work/MMDDYY_Dashboard_Audit.md` following shared.md Output Format exactly:

### 6a. Header

```markdown
# Dashboard Audit

**Date:** MMDDYY
**Prior:** work/MMDDYY_Dashboard_Audit.md (or "None")
**Mode:** Full | Quick (--quick)
**Agent:** Plan | None (--quick) | None (timeout)
**Tools:** Read (audit documents), Grep (finding extraction), Bash (git history)
```

### 6b. Executive Summary

2–3 sentences covering:
- Overall health score and domain distribution
- Number of hotspot files (flagged by 3+ audits)
- DORA tier summary
- Trend vs prior (if `COMPARE` is true)

### 6c. Metrics Dashboard

| Metric | Value | Prior | Trend |
|--------|-------|-------|-------|
| Total findings (across all audits) | N | — | — |
| Overall health score | N/100 | — | — |
| Domains below 70 | N/6 | — | — |
| Hotspot files (3+ audits) | N | — | — |
| Audit documents analyzed | N | — | — |
| Stalest audit (days old) | N days | — | — |
| Deployment frequency (90d) | N | — | — |
| Change failure rate | N% | — | — |

Prior and Trend columns populated only when `COMPARE` is true.

### 6d. Findings Tables

Group all findings by severity. Each table uses shared.md format:

```markdown
### Critical

| # | File | Line | Issue | Tool | LOC Impact | Effort |
|---|------|------|-------|------|------------|--------|
```

Repeat for High, Medium, Low/Informational.

Number findings sequentially with **DA-** prefix (DA-1, DA-2, DA-3...).

### 6e. Domain Health Scores

Include the domain health table from Phase 2.

### 6f. Cross-Audit Hotspots

Include the hotspot table from Phase 3.

### 6g. DORA Metrics

Include the DORA metrics table from Phase 4.

### 6h. Auto-Fixed Section

```markdown
## Auto-Fixed

N/A — Dashboard audit is read-only aggregation. Address findings via individual audits.
```

### 6i. Delta Section

Only if `COMPARE` is true. Follow shared.md Delta Tracking rules:
- Score changes per domain (improved/regressed)
- Hotspot files added/removed
- DORA metric tier changes

### 6j. Historical Section

If prior existed, list resolved items per shared.md format.

### 6k. Recommendations

Top 3 highest-impact recommendations. Prioritize:
1. Domains with score <50 (run those audits, fix critical findings)
2. Hotspot files flagged by 5+ audits (dedicated refactoring sprint)
3. DORA metrics at "Low" tier (process improvements)

---

## Step 7: Post-Processing

Follow shared.md for all of these:

### 7a. Tech Debt Integration

Per shared.md Tech Debt Integration rules — append new CRITICAL/HIGH findings to `tech-debt-patterns.md`, mark resolved entries.

### 7b. Git Commit

```bash
git add work/MMDDYY_Dashboard_Audit.md
git add docs/archive/audits/  # if priors archived
git add .claude/rules/tech-debt-patterns.md  # if updated

git commit -m "$(cat <<'EOF'
docs(audit): dashboard audit — N findings (X critical, Y high)

Tools: Read, Grep, Bash
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

## Execution Checklist

Before completing, verify all items from shared.md Execution Checklist are met. Key dashboard-specific items:

- [ ] Phase 1: All `work/*_Audit.md` files discovered and parsed
- [ ] Phase 2: Domain health scores computed with correct weighting
- [ ] Phase 3: Cross-audit hotspot table produced (3+ audit threshold)
- [ ] Phase 4: DORA metrics derived from git history (agent in full mode)
- [ ] Domain mapping matches current audit inventory
- [ ] Overall score uses weighted average (Security 30%, Quality 25%, etc.)
- [ ] Findings numbered with DA- prefix
- [ ] `--fix` noted as N/A (read-only aggregation)
- [ ] Stale audit detection applied (>30 days = MEDIUM, >90 days = HIGH)

---

## See Also

- `/audit-performance` — Performance patterns audit (DORA overlaps with deployment metrics)
- `/audit-techdebt` — Tech debt patterns audit (complementary to hotspot analysis)
- `/audit-regression` — Regression audit (change failure rate overlap)
- `/audit-project-overview` — Project overview (broader context)
