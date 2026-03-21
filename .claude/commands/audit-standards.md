# /audit-standards

Linting trend audit + coding standards: run ruff and oxlint, track error counts over time, flag regressions, categorize by fixability, and scan for design-level coding standards (naming conventions, type annotations, exception handling, debug output, docstring coverage) via grep patterns.

> **Overlap note:** The linter commands below (`ruff check`, `npm run lint`) are the same as `/audit-python` Phase 4 and `/audit-typescript` Phase 4. This audit adds **trend annotations** (up/down/stable vs prior counts) and **historical comparison** that standalone audits don't track. Do not skip this audit in favor of standalone — the trend data is the unique value.

> **Merge note (2026-03-15):** Absorbed `/audit-bestpractices` (266 LOC, 15 grep patterns). The bestpractices scope boundary ("patterns linters can't enforce") was weak — most patterns ARE linter-enforceable. Phase 8 contains all absorbed patterns. BP- prefix retired.

**Target:** `backend/` (ruff), `<frontend-dir>/` (oxlint), `src/**/*.ts`, `src/**/*.tsx`, `**/*.py` (standards patterns)

---

## Scope Boundary

**This audit covers:** Tool-enforced lint rules with trend tracking — ruff error counts by rule, oxlint warning counts, per-rule delta vs prior (up/down/stable), regression detection, fixability categorization, ruff configuration reference, and auto-fix support. **Also covers** design-level coding standards — naming conventions (camelCase/snake_case), type annotations, exception handling patterns (bare except, broad except), debug output (print/console.log), docstring coverage, import order, and hardcoded values.

**NOT covered here:** Dead code (see /audit-deadcode), technical debt markers (see /audit-techdebt), component health (see /audit-typescript Phase 5).

---

## Step 1: Load Shared Infrastructure

1. Read `.claude/commands/audit/shared.md`
2. All output format, severity levels, delta tracking, archival, tech debt integration, commit format, sprint chain, error handling, and completion report rules come from shared.md — do NOT deviate

---

## Step 2: Parse Flags

Parse `$ARGUMENTS` per shared Flag Parsing rules:

1. Strip flags from arguments, set booleans:
   - `AGENT_MODE` — `false` always (no agent analysis — linting + standards are fully automated)
   - `COMPARE` — `true` if `--since last`
   - `SPRINT_CHAIN` — `true` if `--sprint`
   - `AUTO_FIX` — `true` if `--fix` (meaningful for this audit: runs `ruff check . --fix`)
   - `SARIF_OUTPUT` — `true` if `--sarif`
   - `SPRINT_FOLDER` — extracted if `--sprint v*_*` pattern provided
2. Report mode to user:
   ```
   Mode: Quick (always — no agent analysis)
   Flags: [active flags, or "none"]
   ```

---

## Step 3: Find Prior Audit

```bash
ls work/*_Standards_Audit.md work/*_Linting_Audit.md 2>/dev/null | sort -r | head -1
```

If found, read it for delta comparison later (extract error counts by rule). If not, set `Prior: None`.

---

## Step 4: Archive Prior (if exists)

Follow shared.md Archival rules:
- `git mv` prior to `docs/archive/audits/MMDDYY/`
- Handle collision with `_2` suffix
- Run BEFORE writing new audit document

---

## Step 5: Run Analysis Phases

No agent analysis — linting + standards are fully automated via tool output and Grep patterns. Always runs full scan (not delta-restricted). Delta is on **results** — compare current counts vs prior counts per rule.

**Tool rule:** Use **Semgrep** (via Bash tool) for AST-aware pattern scans in Phase 8
(standards patterns — naming, exceptions, debug output, type annotations). Use the
built-in **Grep tool** for patterns Semgrep can't express and for linter output parsing.
Issue independent tool calls in a **single parallel message**.

### Phase 1: Run Python Linter (ruff)

```bash
cd <backend-dir> && source .venv/bin/activate  # adjust path to your virtualenv

# Summary by rule
ruff check . --statistics 2>&1

# Full error list with file:line
ruff check . 2>&1

# Total count
ruff check . 2>&1 | grep "^Found"
```

### Phase 2: Run TypeScript Linter (oxlint)

```bash
cd <frontend-dir>

# Full lint output (includes warning/error counts in footer)
npm run lint 2>&1
```

### Phase 3: Compare with Prior Audit

If prior exists:
1. Load prior error counts by rule
2. For each rule: calculate delta (current - prior)
3. Classify each rule's trend:
   - **up NEW** — rule not in prior (first appearance)
   - **up n** — count increased by n since prior
   - **stable** — count unchanged
   - **down n** — count decreased by n since prior
   - **RESOLVED** — was in prior, now zero → move to Historical

### Phase 4: Categorize Errors by Fixability

| Category | Criteria | Action |
|----------|----------|--------|
| Auto-fixable | `ruff check . --fix --diff` shows changes | Can fix with `ruff check . --fix` |
| Structural | E402 (import order), F821 (undefined name) | Requires manual refactoring |
| Intentional | Suppressed with `# noqa` or config ignore | Verify suppression is justified |

### Phase 5: Auto-Fix (if --fix)

Only when `AUTO_FIX` is true:

```bash
cd <backend-dir> && source .venv/bin/activate  # adjust path to your virtualenv

# Show what would change
ruff check . --fix --diff 2>&1

# Apply fixes
ruff check . --fix 2>&1

# Re-run to get post-fix counts
ruff check . --statistics 2>&1
```

Report both before and after counts.

### Phase 6: Flag Regressions

- Any rule with **up** trend = regression, severity HIGH
- New rules appearing = regression, severity HIGH
- Growing total count = overall regression warning in Executive Summary

### Phase 7: Ruff Configuration Reference

Read `pyproject.toml` and note:
- `[tool.ruff]` `select` / `ignore` rules (explains why some rules don't appear)
- `per-file-ignores` (intentional suppressions)
- `target-version` (affects available rules)

Include relevant ruff config in the audit for context.

### Phase 8: Standards Patterns (absorbed from audit-bestpractices)

In delta mode, restrict file scope to CHANGED_FILES.

**Semgrep AST-aware scan (Batches 1-4):** 1 Bash call
```bash
semgrep --config .semgrep/standards.yml --json backend/ src/ 2>/dev/null
```
Parse JSON output: `results[]` array. Group by `extra.metadata.category`.

This replaces grep patterns for:
- **Batch 1 — Naming:** camelCase in Python functions (rule: `python-camelcase-function`)
- **Batch 2 — Type annotations:** TS `: any` usage, `as any` casts (rules: `ts-explicit-any`, `ts-as-any-cast`)
- **Batch 3 — Exceptions:** bare except, broad except without re-raise, empty catch blocks (rules: `python-bare-except`, `python-broad-except-no-reraise`, `ts-empty-catch-block`)
- **Batch 4 — Debug output:** print() in Python non-test code, console.log() in TS non-test code (rules: `python-print-statement`, `ts-console-log`). Test files excluded via Semgrep `paths.exclude`.

Semgrep advantage: won't flag `print()` calls in comments/strings, won't match `except:` inside docstrings, won't match `: any` inside string literals.

**Grep fallback — patterns Semgrep can't express (parallel):**

| Category | Pattern | Files | Check |
|----------|---------|-------|-------|
| snake_case in TS | `const [a-z]*_[a-z]` | `*.ts`, `*.tsx` | Should be camelCase |
| Missing return type (Python) | `def \w+\([^)]*\)\s*:` without `->` (shared: Type Safety) | `*.py` | Add `-> type` annotation |
| Missing param type (Python) | `def .*[^:]\):` | `*.py` | Add param annotations |

(Kept as Grep — regex-only patterns that match text structure, not AST constructs)

**Batch 5 — Documentation & style (Grep, parallel):**

| Category | Pattern | Files | Check |
|----------|---------|-------|-------|
| Missing docstring | `def [^_].*:$` without `"""` | `*.py` | Add for public API |
| Missing JSDoc | `export function` without `/**` | `*.ts` | Add JSDoc |
| Import order | stdlib -> third-party -> local | `*.py` | Reorder (manual) |
| Unused variables | No `_` prefix on intentionally unused | `*.py`, `*.ts` | Add `_` prefix |
| Hardcoded values | String literals, magic numbers in logic | all | Extract to constants |

(Kept as Grep — documentation/style patterns require contextual proximity checks)

---

## Step 6: Assemble Findings Document

Write to `work/MMDDYY_Standards_Audit.md` following shared.md Output Format, with these audit-specific sections:

### 6a. Header

```markdown
# Standards Audit

**Date:** MMDDYY
**Prior:** work/MMDDYY_Standards_Audit.md (or "None")
**Mode:** Quick (always — no agent analysis)
**Agent:** None
**Tools:** ruff [version], oxlint [version], Grep (built-in)
```

### 6b. Executive Summary

2-3 sentences covering:
- Total Python errors, total TS warnings, total standards findings
- Trend vs prior (regressions? improvements?)
- Auto-fixable count (if applicable)

### 6c. Metrics Dashboard

| Metric | Value | Prior | Trend |
|--------|-------|-------|-------|
| Python errors (ruff) | N | — | — |
| TS warnings (oxlint) | N | — | — |
| Total lint issues | N | — | — |
| Auto-fixable | N | — | — |
| Regressions (rules with up) | N | — | — |
| New rules (first appearance) | N | — | — |
| Resolved rules (now zero) | N | — | — |
| camelCase in Python | N | — | — |
| snake_case in TS | N | — | — |
| Missing type annotations | N | — | — |
| `: any` usage | N | — | — |
| Bare/broad except | N | — | — |
| print()/console.log() | N | — | — |
| Missing docstrings | N | — | — |

Prior and Trend columns populated only when `COMPARE` is true.

### 6d. Python (ruff) Section

**Error Summary table:**

| Rule | Description | Count | Prior | Trend | Fixability |
|------|-------------|-------|-------|-------|------------|
| E501 | Line too long | N | — | — | Auto-fixable |

**Errors by File table** (top 10 files by error count)

**Auto-fixable count:** N errors can be fixed with `ruff check . --fix`

### 6e. TypeScript (oxlint) Section

**Warning Summary table:**

| Rule | Description | Count | Prior | Trend |
|------|-------------|-------|-------|-------|

**Warnings by File table** (top 10 files by warning count)

### 6f. Standards Patterns Section

Group by category (naming, type annotations, exceptions, debug output, documentation). Number findings with **LT-** prefix (continuing from lint findings).

One table per category, columns: File | Line | Issue | Convention/Recommendation | Status

### 6g. Regressions

Rules where count increased since prior (or "No regressions detected").

### 6h. Auto-Fixed Section

If `AUTO_FIX` was true:
```markdown
## Auto-Fixed

| # | File | What Changed | Tool |
|---|------|-------------|------|
| ... | ... | ... | ruff --fix |

**Before:** N errors | **After:** M errors | **Fixed:** N-M
```

If not: `N/A — Run with --fix to auto-fix [N] issues.`

### 6i. Delta Section

Only if `COMPARE` is true. Follow shared.md Delta Tracking rules. Use trend annotations (up NEW, up n, stable, down n, RESOLVED) as finding status.

### 6j. Historical Section

Per shared.md format.

### 6k. Ruff Configuration

```markdown
## Ruff Configuration

From `pyproject.toml`:
- Selected rules: [list]
- Ignored rules: [list]
- Per-file ignores: [list]
- Target version: [version]
```

### 6l. Recommendations

Top 3 highest-impact fixes. Prioritize:
1. F-code errors (runtime crash risk — CRITICAL)
2. Regression rules (count increased — HIGH)
3. Security-risk standards patterns (bare except hiding errors — HIGH)
4. Auto-fixable batch (quick wins)

---

## Step 7: Post-Processing

Follow shared.md for all of these:

### 7a. SARIF Output (if --sarif)

If `SARIF_OUTPUT` is true, produce `work/MMDDYY_Standards_Audit.sarif` per shared.md SARIF Writer Specification:

- **tool.driver.name:** `"audit-standards"`
- **SARIF categories:** `"lint-python"` (ruff findings), `"lint-typescript"` (oxlint findings), `"standards"` (Phase 8 findings)
- **ruleId mapping:** Use `LT-N` prefix from Markdown findings
- **ruff native SARIF:** ruff supports `--output-format sarif` natively — use this for ruff findings and merge into the combined SARIF. Semgrep findings and oxlint findings map via shared.md field mapping.
- Output path: `work/MMDDYY_Standards_Audit.sarif`

### 7b. Tech Debt Integration

Per shared.md Tech Debt Integration rules.

### 7c. Git Commit

```bash
git add work/MMDDYY_Standards_Audit.md
git add work/MMDDYY_Standards_Audit.sarif  # if --sarif
git add docs/archive/audits/  # if priors archived
git add .claude/rules/tech-debt-patterns.md  # if updated

git commit -m "$(cat <<'EOF'
docs(audit): standards audit — N Python errors, M TS warnings, P standards findings

Tools: ruff [version], oxlint [version], Grep
Mode: Quick (no agent)
[Delta: +N new, -N resolved]  (only if --since last)
[Auto-fixed: N issues]  (only if --fix)
[SARIF: work/MMDDYY_Standards_Audit.sarif]  (only if --sarif)

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
```

### 7d. Completion Report

Output to user per shared.md Completion Report format.

### 7e. Sprint Chain

Only if `SPRINT_CHAIN` is true — per shared.md Sprint Chain rules.

---

## Severity Mapping

| Severity | Criteria | Example |
|----------|----------|---------|
| CRITICAL | Error-class rules (F-codes) | F821 undefined name — runtime crash risk |
| HIGH | New errors since last audit (any rule) | Regression introduced by recent work |
| HIGH | Security-risk standards pattern | Bare except hiding errors |
| HIGH | Bug-prone standards pattern | Missing type on public API |
| MEDIUM | Pre-existing errors, stable count | E402 import order — cosmetic but noisy |
| MEDIUM | Maintainability standards issue | Inconsistent naming |
| MEDIUM | Readability standards issue | Missing docstring on complex function |
| LOW | Warnings only (oxlint warnings) | `no-new-array` style preference |
| LOW | Style standards issue | Import order, print in test files |
| INFO | Auto-fixable errors | Can be resolved with `ruff check . --fix` |

---

## Execution Checklist

Before completing, verify all items from shared.md Execution Checklist are met. Key audit-specific items:

- [ ] Phase 1: ruff ran with both `--statistics` and full output
- [ ] Phase 2: oxlint ran via `npm run lint`
- [ ] Phase 3: Prior comparison completed (if prior exists) with per-rule trend
- [ ] Phase 4: Errors categorized by fixability (auto-fixable, structural, intentional)
- [ ] Phase 5: Auto-fix ran and before/after counts reported (if `--fix`)
- [ ] Phase 6: Regressions flagged with HIGH severity
- [ ] Phase 7: Ruff config from pyproject.toml documented
- [ ] Phase 8: All 15 standards grep patterns ran (naming, typing, exceptions, debug, docs)
- [ ] Phase 8: Each pattern targeted correct file types (*.py vs *.ts/*.tsx)
- [ ] Phase 8: Standards findings grouped by category with per-category tables
- [ ] Trend annotations applied (up NEW, up n, stable, down n, RESOLVED)
- [ ] No agent analysis attempted (this audit has no agent phase)

---

## See Also

- `/audit-python` — Python code quality audit (Phase 4 runs ruff — this audit adds trend tracking)
- `/audit-typescript` — TypeScript audit (Phase 4 runs oxlint — this audit adds trend tracking)
- `/audit-techdebt` — Technical debt audit (TODO/FIXME markers)
