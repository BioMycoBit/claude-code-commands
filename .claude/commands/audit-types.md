# /audit-types

Type checking audit: Python type coverage via ty + pyright (cross-referenced), TypeScript strict type checking via tsc --noEmit, type safety pattern scanning, and type coverage metrics across both ecosystems.

**Target:** `backend/` (Python — ty + pyright), `src/` (TypeScript — tsc --noEmit)

---

## Scope Boundary

**This audit covers:** Type system correctness — running type checkers (ty for Python, tsc for TypeScript), measuring type coverage, and scanning for type safety anti-patterns (`any`, `type: ignore`, untyped functions).

**NOT covered here (see /audit-python):** Python code quality beyond types — ruff linting, complexity analysis, dead code detection, docstring coverage, import ordering. This audit runs ty; `/audit-python` runs ruff.

**NOT covered here (see /audit-typescript):** TypeScript code quality beyond types — oxlint linting, React-specific patterns, bundle analysis, unused imports. This audit runs tsc --noEmit; `/audit-typescript` runs oxlint.

**NOT covered here (see /audit-linting):** Linter configuration, rule tuning, auto-fix via ruff/oxlint. This audit checks types; `/audit-linting` checks style and lint rules.

---

## Step 1: Load Shared Infrastructure

1. Read `.claude/commands/audit/shared.md`
2. All output format, severity levels, delta tracking, archival, tech debt integration, commit format, sprint chain, error handling, and completion report rules come from shared.md — do NOT deviate

---

## Step 2: Parse Flags

Parse `$ARGUMENTS` per shared Flag Parsing rules:

1. Strip flags from arguments, set booleans:
   - `AGENT_MODE` — `true` unless `--quick` is set
   - `COMPARE` — `true` if `--since last`
   - `SPRINT_CHAIN` — `true` if `--sprint`
   - `AUTO_FIX` — `false` always (type fixes require context, cannot be auto-applied)
   - `SARIF_OUTPUT` — `true` if `--sarif`
   - `SPRINT_FOLDER` — extracted if `--sprint v*_*` pattern provided
2. Report mode to user:
   ```
   Mode: Full | Quick
   Flags: [active flags, or "none"]
   Note: --fix has no effect (type errors require contextual fixes).
   ```

---

## Step 3: Find Prior Audit

```bash
ls work/*_Types_Audit.md 2>/dev/null | sort -r | head -1
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

Each finding must include: File, Line, Issue, Tool, LOC Impact, Effort.

**Tool rule:** Use the built-in **Grep tool** (NOT bash grep/rg) for ALL pattern scans per shared.md Tool Rules. Issue multiple Grep calls in a **single parallel message** where patterns are independent.

### Phase 1: Python Type Checking (ty + pyright)

Run **both** type checkers on the Python backend. Cross-reference results for confidence scoring.

#### Phase 1a: ty

```bash
cd <backend-dir> && ty check . 2>&1
echo "Exit code: $?"
```

If ty is not installed:
```
Tool not installed: ty
Install: uv tool install ty (recommended) | pip install ty
```
Note in output under Tools and continue to Phase 1b.

Parse ty output:
- Extract: file, line, error code, message
- Group by error category (unresolved-import, invalid-type, missing-return-type, etc.)
- Map to severity: unresolved imports in production code = HIGH, missing annotations = MEDIUM, strict mode warnings = LOW

**ty configuration check** — if no `ty.toml` or `[tool.ty]` in `pyproject.toml` exists, note it:
```
No ty configuration found. Using defaults.
Recommendation: Add [tool.ty] to pyproject.toml for project-specific settings.
```

#### Phase 1b: pyright

```bash
cd <backend-dir> && pyright . 2>&1
echo "Exit code: $?"
```

If pyright is not installed:
```
Tool not installed: pyright
Install: uv tool install pyright (recommended) | npm install -g pyright
```
Note in output under Tools and continue to Phase 1c.

Parse pyright output:
- Extract: file, line, error code (reportXxx), message
- Group by error category (reportMissingImports, reportGeneralClassIssues, reportMissingTypeStubs, etc.)
- Map to severity: same rules as ty (production import errors = HIGH, missing annotations = MEDIUM, strict warnings = LOW)

**pyright configuration check** — if no `pyrightconfig.json` or `[tool.pyright]` in `pyproject.toml` exists, note it:
```
No pyright configuration found. Using defaults.
Recommendation: Add [tool.pyright] to pyproject.toml for project-specific settings.
```

#### Phase 1c: Cross-Reference

Compare ty and pyright results by `(file, line, issue-category)`:

| Overlap | Confidence | Action |
|---------|------------|--------|
| Both report same error at same location | **HIGH** — convergence confirms real issue | Report once, note `Tool: ty + pyright` |
| Only ty reports | **MEDIUM** — single-checker finding | Report with `Tool: ty only` |
| Only pyright reports | **MEDIUM** — single-checker finding | Report with `Tool: pyright only` |
| Both report but different severity | **HIGH** — use higher severity | Note both assessments |

**Metrics from cross-reference:**
```
ty errors: N | pyright errors: N | Convergent: N | ty-only: N | pyright-only: N
```

### Phase 2: TypeScript Type Checking (tsc --noEmit)

Run tsc on the frontend:

```bash
cd <frontend-dir> && npx tsc --noEmit 2>&1
echo "Exit code: $?"
```

Parse tsc output:
- Extract: file, line, column, error code (TSxxxx), message
- Group by error code
- Map to severity: type assertion failures = HIGH, implicit any = MEDIUM, unused locals = LOW

**Note:** tsc --noEmit uses the existing tsconfig.json. Do NOT modify tsconfig for the audit.

### Phase 3: Type Safety Pattern Scan

Reference **Type Safety Patterns** from shared.md. Run all Grep calls in a single parallel message:

**Scan Group 1 — Explicit any (TypeScript):** 1 Grep call, *.ts + *.tsx
Pattern: `:\s*any\b`
Exclude test files from severity escalation (test `any` = LOW, production `any` = MEDIUM)

**Scan Group 2 — Type assertions to any (TypeScript):** 1 Grep call, *.ts + *.tsx
Pattern: `as any`

**Scan Group 3 — Type ignore comments (Python):** 1 Grep call, *.py
Pattern: `# type:\s*ignore`

**Scan Group 4 — Untyped function definitions (Python):** 1 Grep call, *.py
Pattern: `def \w+\([^)]*\)\s*:` (functions without `->` return annotation)
Cross-reference with: `def \w+\([^)]*\)\s*->` to compute typed vs untyped ratio

**Scan Group 5 — @ts-ignore / @ts-expect-error (TypeScript):** 1 Grep call, *.ts + *.tsx
Pattern: `@ts-ignore|@ts-expect-error`

**Scan Group 6 — Non-null assertions (TypeScript):** 1 Grep call, *.ts + *.tsx
Pattern: `\w+!\.|\w+!\[`

### Phase 4: Type Coverage Metrics

Compute type coverage metrics for the audit dashboard:

**Python:**
1. Count total function definitions: `def \w+` in `backend/`
2. Count typed function definitions: `def \w+.*->` in `backend/`
3. Coverage = typed / total × 100

**TypeScript:**
1. Count `any` occurrences (from Phase 3 Scan Groups 1-2)
2. Count `@ts-ignore` + `@ts-expect-error` (from Phase 3 Scan Group 5)
3. Count `type: ignore` equivalent bypasses

**Combined metrics table:**
```markdown
| Metric | Python | TypeScript |
|--------|--------|------------|
| Total functions | N | — |
| Typed functions | N | — |
| Type coverage % | N% | — |
| type: ignore / @ts-ignore | N | N |
| Explicit any | — | N |
| as any assertions | — | N |
| Non-null assertions | — | N |
| ty errors | N | — |
| pyright errors | N | — |
| Convergent errors (ty ∩ pyright) | N | — |
| tsc errors | — | N |
```

### Phase 5: Plan Agent Analysis (if AGENT_MODE)

Launch **one** Plan agent with `subagent_type=Plan` AFTER Phases 1-4 complete.

**Prompt:**
> "Prioritize type error remediation for a multi-service application:
>
> **ty errors (Python):** [paste Phase 1a output — first 50 errors max]
> **pyright errors (Python):** [paste Phase 1b output — first 50 errors max]
> **Cross-reference:** [paste Phase 1c convergence summary]
> **tsc errors (TypeScript):** [paste Phase 2 output — first 50 errors max]
> **Type safety patterns:** [paste Phase 3 summary counts]
>
> **Analyze:**
> 1. **Categorize errors** — group by root cause (missing stubs, incorrect types, unsafe casts)
> 2. **Prioritize by risk** — which type errors could cause runtime failures vs cosmetic issues?
> 3. **Identify quick wins** — errors fixable in <5 min each (missing return types, obvious annotations)
> 4. **Identify structural issues** — errors requiring refactoring (wrong abstractions, missing generics)
> 5. **Recommend type coverage targets** — realistic % improvement per sprint
>
> **Return:** Prioritized remediation plan with effort estimates, grouped by quick wins vs structural."

**On timeout/failure:** Continue with tool output only. Note `Agent: None (timeout)` in header.

### Phase 6: Verification (shared.md Phase 5.5)

For each CRITICAL/HIGH finding from Phases 1-4:
1. **Read the file** at the reported line number
2. **Confirm the finding is real** — not inside a comment, string literal, or already-fixed code
3. **Assign confidence:** HIGH, MEDIUM, LOW
4. **Drop LOW confidence findings**

Skip this step if `--quick` is set.

---

## Step 6: Assemble Findings Document

Write to `work/MMDDYY_Types_Audit.md` following shared.md Output Format exactly:

### 6a. Header

```markdown
# Types Audit

**Date:** MMDDYY
**Prior:** work/MMDDYY_Types_Audit.md (or "None")
**Mode:** Full | Quick (--quick)
**Agent:** Plan | None (--quick) | None (timeout)
**Tools:** ty (Python type checking), pyright (Python type checking), tsc --noEmit (TypeScript type checking), Grep (type safety patterns)
```

### 6b. Executive Summary

2-3 sentences covering:
- ty error count and top categories
- tsc error count and top categories
- Type coverage percentage (Python function annotations)
- Trend vs prior (if `COMPARE` is true)

### 6c. Metrics Dashboard

| Metric | Value | Prior | Trend |
|--------|-------|-------|-------|
| Total findings | N | — | — |
| Critical | N | — | — |
| High | N | — | — |
| Medium | N | — | — |
| Low | N | — | — |
| ty errors (Python) | N | — | — |
| pyright errors (Python) | N | — | — |
| Convergent (ty ∩ pyright) | N | — | — |
| tsc errors (TypeScript) | N | — | — |
| Python type coverage % | N% | — | — |
| Explicit `any` (TS) | N | — | — |
| `as any` assertions (TS) | N | — | — |
| `type: ignore` (Python) | N | — | — |
| `@ts-ignore/@ts-expect-error` | N | — | — |
| Non-null assertions (TS) | N | — | — |

Prior and Trend columns populated only when `COMPARE` is true.

### 6d. Findings Tables

Group all findings by severity. Each table uses shared.md format:

```markdown
### Critical

| # | File | Line | Issue | Tool | LOC Impact | Effort |
|---|------|------|-------|------|------------|--------|
```

Repeat for High, Medium, Low/Informational.

Number findings sequentially with **TY-** prefix (TY-1, TY-2, TY-3...).

**Grouping rule:** When a tool reports 50+ errors of the same type, group them into a single finding with count:
`TY-N | (multiple files) | — | 47 unresolved-import errors (see appendix) | ty | ~47 LOC | medium`

### 6e. Auto-Fixed Section

```markdown
## Auto-Fixed

N/A — Type errors require contextual fixes. Run `ty` or `tsc` locally to see inline suggestions.
```

### 6f. Delta Section

Only if `COMPARE` is true. Follow shared.md Delta Tracking rules.

### 6g. Historical Section

Per shared.md format.

### 6h. Recommendations

Top 3 highest-impact recommendations. Prioritize:
1. Type errors that could cause runtime failures (wrong types on API boundaries)
2. Quick wins: batch-fixable annotation gaps
3. Type coverage targets for next sprint

---

## Step 7: Post-Processing

Follow shared.md for all of these:

### 7a. SARIF Output (if --sarif)

If `SARIF_OUTPUT` is true, produce `work/MMDDYY_Types_Audit.sarif` per shared.md SARIF Writer Specification:

- **tool.driver.name:** `"audit-types"`
- **SARIF categories:** `"types-python-ty"` (ty findings), `"types-python-pyright"` (pyright findings), `"types-python-convergent"` (both checkers agree), `"types-typescript"` (tsc findings)
- **ruleId mapping:** Use `TY-N` prefix from Markdown findings
- **ty/tsc output → SARIF:** Parse ty and tsc error output. Each type error becomes a result with `ruleId` = error code (e.g., `unresolved-import`, `TS2345`), `level` per severity mapping.
- Output path: `work/MMDDYY_Types_Audit.sarif`

### 7b. Tech Debt Integration

Per shared.md Tech Debt Integration rules.

### 7c. Git Commit

```bash
git add work/MMDDYY_Types_Audit.md
git add work/MMDDYY_Types_Audit.sarif  # if --sarif
git add docs/archive/audits/  # if priors archived
git add .claude/rules/tech-debt-patterns.md  # if updated

git commit -m "$(cat <<'EOF'
docs(audit): types audit — N findings (X critical, Y high)

Tools: ty, tsc --noEmit, Grep
Mode: Full | Quick
[Delta: +N new, -N resolved]  (only if --since last)
[SARIF: work/MMDDYY_Types_Audit.sarif]  (only if --sarif)

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
| CRITICAL | Type error on API boundary that could cause runtime crash | Wrong return type on WebSocket handler causing silent data loss |
| CRITICAL | Type error in data pipeline that corrupts application data | Incorrect data shape type causing array index error |
| HIGH | `as any` on data flowing to/from backend | Casting data payload to any before processing |
| HIGH | Missing return type on public API function | `def connect_device(...)` without `-> DeviceInfo` |
| HIGH | `type: ignore` suppressing real error (not stub issue) | Ignoring type mismatch on function call |
| MEDIUM | Explicit `any` in production code | `const data: any = response.json()` |
| MEDIUM | Untyped function in production code | `def process_data(data):` without annotations |
| MEDIUM | Non-null assertion on potentially null value | `user!.id` without null check |
| LOW | `any` in test files | Test helper with `any` type |
| LOW | Missing type annotation on internal helper | Private utility function without full annotations |
| LOW | `@ts-expect-error` with valid reason comment | Suppression with documented justification |
| INFO | ty/tsc configuration suggestion | Missing strict mode flag |

---

## Execution Checklist

Before completing, verify all items from shared.md Execution Checklist are met. Key types-specific items:

- [ ] Phase 1a: ty ran on python-backend (or install command noted)
- [ ] Phase 1b: pyright ran on python-backend (or install command noted)
- [ ] Phase 1c: Cross-reference produced convergence metrics
- [ ] Phase 2: tsc --noEmit ran on src/ (or error noted)
- [ ] Phase 3: All 6 scan groups ran via Grep tool (not bash)
- [ ] Phase 4: Type coverage metrics computed for both ecosystems
- [ ] Phase 5: Plan agent ran (if AGENT_MODE, or timeout noted)
- [ ] Phase 6: CRITICAL/HIGH findings verified via Read tool
- [ ] Grouped findings when tool reports 50+ of same error type
- [ ] Findings numbered with TY- prefix
- [ ] Agent header field set correctly

---

## See Also

- `/audit-python` — Python code quality audit (ruff linting, complexity, dead code)
- `/audit-typescript` — TypeScript code quality audit (oxlint, React patterns, bundle)
- `/audit-linting` — Linter configuration audit (ruff/oxlint rule tuning)
- `/audit-modules` — Module boundary audit (tach for Python, knip for TypeScript)
