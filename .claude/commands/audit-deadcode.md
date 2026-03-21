# /audit-deadcode

Dead code audit: unused exports, orphan files, uncalled functions, and dead components across Python and TypeScript codebases.

**Target:** `src/**/*.ts`, `src/**/*.tsx`, `**/*.py`

---

## Scope Boundary

**This audit covers:** All dead code detection — unused exports, orphan files, uncalled functions, dead components, vulture (Python), TODO-delete markers, and Plan agent safe-removal planning.

**NOT covered here (see /audit-python):** Python cyclomatic complexity, maintainability index, linting (ruff), hotspot correlation, and algorithmic complexity. `/audit-python` Phase 3 (vulture) is intentionally duplicated — this audit owns the comprehensive dead code view; `/audit-python` includes vulture as part of its holistic Python quality assessment.

**NOT covered here (see /audit-typescript):** TypeScript circular dependencies, type duplication, linting (oxlint), component health, and algorithmic complexity. `/audit-typescript` Phase 2 (dead exports) and Phase 6 (orphans via madge) are intentionally duplicated — this audit owns the cross-language dead code view; `/audit-typescript` includes them as part of its holistic TS quality assessment.

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
   - `AUTO_FIX` — `false` always (dead code audit is read-only, `--fix` is a no-op — removal requires human review)
   - `SPRINT_FOLDER` — extracted if `--sprint v*_*` pattern provided
2. Report mode to user:
   ```
   Mode: Full | Quick
   Flags: [active flags, or "none"]
   Note: --fix is ignored for dead code audit (removal requires human review)
   ```

---

## Step 3: Find Prior Audit

```bash
ls work/*_DeadCode_Audit.md 2>/dev/null | sort -r | head -1
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

### Dead Code Delta Logic

In delta mode (`--since last`):
- Apply Delta Mode Protocol from `shared.md`
- Carry forward: unused export list + orphan file list + dead component list
- Compare current scan against prior findings using `(file, issue)` key

### Phase 1: Pattern Scans

Run all grep patterns in parallel:

| Search | Pattern | Files | Tool |
|--------|---------|-------|------|
| Unused exports | `export (const\|function\|class)` then verify imports | `*.ts`, `*.tsx` | Grep |
| Marked dead | `// TODO.*delete\|// UNUSED\|// DEAD` | `*.ts`, `*.tsx`, `*.py` | Grep |
| Orphan files | Files with 0 imports | all source files | Grep for filename |
| Uncalled functions | Function definitions not referenced elsewhere | `*.py` | Grep |
| Dead components | `.tsx` files imported only by test files (or not at all) | `*.tsx` | Glob + Grep |
| Python dead markers | `# TODO.*delete\|# UNUSED\|# DEAD` | `*.py` | Grep |

**Analysis process:**

1. Grep for all exports in `src/**/*.ts` and `src/**/*.tsx`
2. For each export, grep for its usage (import statements)
3. Flag exports with 0 imports as potentially dead
4. Check `**/*.py` for unused functions
5. Look for `# TODO.*delete`, `# UNUSED`, `# DEAD` markers
6. **Dead component scan:** Glob all `.tsx` files in `src/components/`. For each, extract the filename (without extension) and Grep for `from.*ComponentName` across `src/` excluding `__tests__/` directories. Files imported ONLY by test files (or not imported at all) are dead components. Report: file path, LOC, test file count, zero non-test importers.

### Phase 2: Agent Analysis (if AGENT_MODE = true)

Launch Plan agent with `subagent_type=Plan` AFTER Phase 1 grep patterns complete.

**Delta mode:** Use Agent Scope Narrowing from shared.md — pass prior executive summary + CHANGED_FILES list instead of full exploration prompt.

**Full mode prompt:**
> "Given dead code exports from grep, create a safe removal plan:
>
> **Findings:** [Insert grep results — unused exports, unreferenced functions, dead components]
>
> **Analyze:**
> 1. **Dependency Mapping** — Build removal graph, identify leaf nodes (safe first)
> 2. **Dynamic Import Check** — `import()`, string requires, test-only exports
> 3. **Dead Component Check** — For each flagged component, verify no non-test file imports it. Include associated test files in removal scope
> 4. **Categorize** — SAFE_TO_DELETE | NEEDS_INVESTIGATION | KEEP (test utils, public API)
> 5. **Removal Order** — Leaf deps first, group by file, estimate blast radius
>
> **Return:** Safe Removal Checklist with Phase 1 (leaf/zero-risk), Phase 2 (depends on Phase 1), Needs Investigation, Keep (Do Not Delete)"

**Use agent output to:** Replace raw grep with prioritized removal plan + blast radius + phased checklist.

**On timeout/failure:** Output raw grep findings. Note `Agent: None (timeout)` in header.

**Skip agent if `--quick` mode.** Set `Agent: None (--quick)` in header.

---

## Step 6: Assemble Findings Document

Write to `work/MMDDYY_DeadCode_Audit.md` following shared.md Output Format exactly:

### 6a. Header

```markdown
# DeadCode Audit

**Date:** MMDDYY
**Prior:** work/MMDDYY_DeadCode_Audit.md (or "None")
**Mode:** Full | Quick (--quick)
**Agent:** Plan | None (--quick) | None (timeout)
**Tools:** Grep (built-in), Glob (file scan), Read (file analysis)
```

### 6b. Executive Summary

2–3 sentences covering:
- Total dead code findings and severity distribution
- Highest-impact finding (e.g., large dead component, many unused exports in one file)
- Trend vs prior (if `COMPARE` is true)

### 6c. Metrics Dashboard

| Metric | Value | Prior | Trend |
|--------|-------|-------|-------|
| Total findings | N | — | — |
| Critical | N | — | — |
| High | N | — | — |
| Medium | N | — | — |
| Low | N | — | — |
| Unused exports (TS) | N | — | — |
| Unused functions (Python) | N | — | — |
| Dead components | N | — | — |
| Orphan files | N | — | — |
| Estimated removable LOC | N | — | — |

Prior and Trend columns populated only when `COMPARE` is true.

### 6d. Findings Tables

Group all findings by severity. Each table uses shared.md format:

```markdown
### Critical

| # | File | Line | Issue | Tool | LOC Impact | Effort |
|---|------|------|-------|------|------------|--------|
```

Repeat for High, Medium, Low/Informational.

Number findings sequentially with **DC-** prefix (DC-1, DC-2, DC-3... for DeadCode findings).

### 6e. Auto-Fixed Section

**Not applicable for dead code audit** — removal requires human review.
```markdown
## Auto-Fixed

N/A — Dead code audit is read-only. Use findings to plan safe removal sessions.
```

### 6f. Delta Section

Only if `COMPARE` is true. Follow shared.md Delta Tracking rules.

### 6g. Historical Section

If prior existed, list resolved items per shared.md format.

### 6h. Recommendations

Top 3 highest-impact recommendations with estimated effort. Prioritize:
1. SAFE_TO_DELETE items with highest LOC (quick wins)
2. Dead components with zero non-test importers
3. Files with multiple unused exports (batch cleanup)

---

## Step 7: Post-Processing

Follow shared.md for all of these:

### 7a. Tech Debt Integration

Per shared.md Tech Debt Integration rules — append new CRITICAL/HIGH findings to `tech-debt-patterns.md`, mark resolved entries.

### 7b. Git Commit

```bash
git add work/MMDDYY_DeadCode_Audit.md
git add docs/archive/audits/  # if priors archived
git add .claude/rules/tech-debt-patterns.md  # if updated

git commit -m "$(cat <<'EOF'
docs(audit): deadcode audit — N findings (X critical, Y high)

Tools: Grep, Glob, Read
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
| HIGH | Dead component with >200 LOC, file with 5+ unused exports |
| HIGH | Orphan file (zero imports anywhere) with >100 LOC |
| MEDIUM | Dead component 50-200 LOC, file with 2-4 unused exports |
| MEDIUM | Orphan file 50-100 LOC |
| LOW | Single unused export, dead marker (TODO delete), small orphan <50 LOC |
| INFO | Test-only exports (KEEP category), intentionally unused (public API) |

---

## Execution Checklist

Before completing, verify all items from shared.md Execution Checklist are met. Key dead code-specific items:

- [ ] Phase 1: All grep patterns ran (exports, markers, orphans, components)
- [ ] Phase 1: Each flagged export verified against import grep (not just definition scan)
- [ ] Phase 1: Dead component scan checked non-test importers specifically
- [ ] Phase 2: Agent analysis ran (full mode) or skipped (quick mode) with correct header
- [ ] Agent output categorized findings: SAFE_TO_DELETE / NEEDS_INVESTIGATION / KEEP
- [ ] Findings numbered with DC- prefix
- [ ] Estimated removable LOC metric populated
- [ ] `--fix` noted as N/A (read-only audit)

---

## See Also

- `/audit-techdebt` — Technical debt audit (TODO/FIXME/HACK markers, `:any` usage)
- `/audit-standards` — Linting + coding standards audit (naming, typing, exceptions)
- `/audit-python` — Python-specific code quality audit
- `/audit-typescript` — TypeScript-specific audit (circular deps, dead exports)
- `/audit-codesweep` — Code duplication analysis (cross-reference: files in both dead code and clone groups are high-priority)
