# /audit-python

Python codebase audit: cyclomatic complexity, maintainability, dead code, linting, pattern analysis, hotspot correlation, and algorithmic complexity.

**Target:** Python source directories (e.g., `src/backend/`, `app/`, `backend/`)

---

## Scope Boundary

**This audit covers:** Holistic Python code quality — cyclomatic complexity (radon cc), maintainability index (radon mi), dead code (vulture), linting (ruff), god objects, import chains, hotspot correlation, and algorithmic complexity.

**NOT covered here (see /audit-deadcode):** Cross-language dead code view — orphan file detection, dead component scanning, unused TS export cross-referencing, and Plan agent safe-removal planning. Phase 3 (vulture) is intentionally shared with `/audit-deadcode` — both audits run it from different perspectives.

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
   - `AUTO_FIX` — `true` if `--fix`
   - `SPRINT_FOLDER` — extracted if `--sprint v*_*` pattern provided
2. Report mode to user:
   ```
   Mode: Full | Quick
   Flags: [active flags, or "none"]
   ```

---

## Step 3: Find Prior Audit

```bash
ls work/*_Python_Audit.md 2>/dev/null | sort -r | head -1
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

Run all 7 phases (or 5 if `--quick`), collecting findings. Each finding must include: File, Line, Issue, Tool, LOC Impact, Effort.

### Phase 1: Cyclomatic Complexity (radon cc)

```bash
radon cc <python-source-dir>/ -a -s -n C
```

- Finds functions with CC >= 11 (grade C or worse)
- **Severity mapping:**
  - CC 30+ = HIGH
  - CC 15–29 = MEDIUM
  - CC 11–14 = LOW
- Capture: function name, file, line number, CC score, grade
- **LOC Impact:** estimate from function length (read file if needed for HIGH/CRITICAL)
- **Effort:** CC 30+ = `medium`, CC 15–29 = `small`, CC 11–14 = `trivial`

**Error handling:** If `radon` not installed:
```
radon not found. Install: pip install radon (or uv add --dev radon)
Skipping Phase 1 (Cyclomatic Complexity).
```
Note in Tools section of output: `radon cc — SKIPPED (not installed)`

### Phase 2: Maintainability Index (radon mi)

```bash
radon mi <python-source-dir>/ -s -n B
```

- Finds files with maintainability grade B or worse
- **Severity mapping:**
  - Grade D or F = HIGH
  - Grade C = MEDIUM
  - Grade B = LOW
- Capture: file path, MI score, grade
- **LOC Impact:** total lines in file (from radon output or wc -l)
- **Effort:** Grade D/F = `large` (major refactor), Grade C = `medium`, Grade B = `small`

**Error handling:** Same as Phase 1 (radon shared install).
Note in Tools: `radon mi — SKIPPED (not installed)`

### Phase 3: Dead Code (vulture)

```bash
vulture <python-source-dir>/ --min-confidence 80 --exclude <python-source-dir>/.venv
```

- Finds unused functions, imports, variables, classes at ≥80% confidence
- **Severity mapping (by estimated LOC of dead item):**
  - \>200 LOC = HIGH
  - 50–200 LOC = MEDIUM
  - <50 LOC = LOW
- Capture: file, line, item name, item type (function/import/variable/class), confidence %
- **LOC Impact:** estimate from item type (functions: read to measure, imports: ~1 line, variables: ~1–5 lines)
- **Effort:** imports/variables = `trivial`, small functions = `small`, large functions/classes = `medium`

**Error handling:** If `vulture` not installed:
```
vulture not found. Install: pip install vulture (or uv add --dev vulture)
Skipping Phase 3 (Dead Code).
```
Note in Tools: `vulture — SKIPPED (not installed)`

### Phase 4: Linting (ruff)

**If `AUTO_FIX` is true:**
```bash
ruff check <python-source-dir>/ --fix
```
Record what was auto-fixed (capture output showing fixed files/rules).

**Then (always):**
```bash
ruff check <python-source-dir>/ --statistics
```

- Captures rule violations grouped by rule code
- **Severity mapping (by ruff rule prefix):**
  - E (error), F (pyflakes), S (security/bandit), B (bugbear) = HIGH
  - W (warning), C (convention), N (naming) = MEDIUM
  - I (isort), D (docstring), UP (pyupgrade) = LOW
  - Auto-fixed issues (if `--fix` was used) = INFO
- Capture: rule code, count, description
- **LOC Impact:** count × ~1 line per violation (approximate)
- **Effort:** auto-fixable = `trivial`, naming/convention = `small`, errors/security = `medium`

**If `--fix` was used and changes were made:**
- Re-run `ruff check --statistics` to show remaining issues
- Populate Auto-Fixed section with what was fixed (before vs after counts)
- If fixes introduced new lint issues, report both states

**Error handling:** If `ruff` not installed:
```
ruff not found. Install: pip install ruff (or uv add --dev ruff)
Skipping Phase 4 (Linting).
```
Note in Tools: `ruff — SKIPPED (not installed)`

### Phase 5: Pattern Analysis (agent-driven)

**Skip if `--quick` mode.** Set `Agent: None (--quick)` in header.

Use a Task agent (subagent_type: Plan) to analyze your Python source directories for:

1. **God objects** — files >500 LOC with many public methods or mixed responsibilities
2. **Message/handler deduplication** — message types or handlers handled in multiple places without clear ownership
3. **Facade evaluation** — thin wrappers that add no value, just forward calls
4. **Import chain complexity** — circular or deep import chains that complicate testing

Cross-reference with known tech debt from `tech-debt-patterns.md` if it exists in the project.

**Severity mapping:**
- God objects >800 LOC or >20 public methods = HIGH
- God objects 500–800 LOC = MEDIUM
- Facade/deduplication issues = LOW
- Import chain issues = LOW

**Effort:** God objects = `large`, facades = `small`, deduplication = `medium`, imports = `medium`

**Error handling:** If agent times out or fails, set `Agent: None (timeout)` in header and continue with tool-only results.

### Phase 6: Hotspot Correlation (git log × radon)

**Skip if `--quick` mode.**

```bash
git log --since="$PRIOR_DATE" --name-only --pretty=format: -- <python-source-dir>/ | sort | uniq -c | sort -rn | head -20
```

Cross-reference the top 20 highest-churn files with Phase 1 radon CC findings:

| Churn | CC | Severity | Rationale |
|-------|-----|----------|-----------|
| High (top 5) | High (CC 15+) | **CRITICAL** | Actively editing complex code — highest defect risk |
| High (top 5) | Low (CC <11) | INFO | Healthy development — frequent changes, manageable complexity |
| Low (bottom 15) | High (CC 15+) | MEDIUM | Complex but stable — lower urgency |
| Low | Low | — | No finding (healthy) |

- **LOC Impact:** from radon CC phase data
- **Effort:** CRITICAL hotspots = `large` (refactor + test), MEDIUM = `medium`

**Note:** This is the only phase that can produce CRITICAL findings. A file must be both high-churn AND high-complexity to be CRITICAL.

### Phase 7: Algorithmic Complexity (grep + agent)

Scan your Python source directories for suboptimal algorithmic complexity patterns. **Grep patterns run in all modes; agent verification runs in full mode only.**

**Grep patterns (always run):**

Use the Grep tool (NOT bash grep) for all pattern scans, issuing parallel calls:

1. **Linear search in loop** — `.index()`, `.count()` called on lists
   - Pattern: `\.(index|count)\(` in `*.py` files
2. **Nested for loops** — potential O(n²) when iterating unbounded collections
   - Pattern: `^\s{8,}for\s+\w+\s+in\s+` (double-indented for loops)
3. **Sort/max/min inside loop** — O(n log n) or O(n) per iteration
   - Pattern: `sorted\(|\.sort\(` — flag when inside function bodies containing `for`/`while`
4. **String concatenation in loop** — O(n²) total due to immutable strings
   - Pattern: `\+=\s*["'f]` inside `for`/`while` blocks (should use `"".join()`)
5. **Quadratic list operations** — `.remove()`, `.insert(0,` on large lists
   - Pattern: `\.remove\(|\.insert\(\s*0`
6. **List where set/dict would be O(1)** — `if x in list_var` patterns
   - Pattern: `if\s+\w+\s+(not\s+)?in\s+\w+` — agent verifies container type

**Full mode (agent-driven):**

If `AGENT_MODE` is true, use a Plan agent to verify grep hits:
1. Read surrounding context for each match (function scope + call sites)
2. Classify: is the collection size **bounded** (n ≤ 16, hardcoded config) or **unbounded** (per-request, per-user, per-sample)?
3. Is the code in a **hot path** (per-request, per-message, per-frame, high-frequency callback) or **cold path** (startup, one-time init)?
4. Determine actual complexity class: O(n²), O(n log n), O(n) where O(1) available
5. Filter false positives: dict/set `in` checks are O(1) and not findings

**Severity mapping:**
- O(n²)+ in hot path (per-request/per-message/per-frame handler) = HIGH
- O(n²)+ in cold path (startup, config load, one-time) = MEDIUM
- O(n) where O(1) available (list `in` vs set/dict lookup) in hot path = MEDIUM
- O(n) where O(1) available in cold path, or bounded input (n ≤ 16) = LOW

**LOC Impact:** ~1–10 lines per finding (the loop + operation)
**Effort:** data structure swap (list→set/dict) = `small`, nested loop refactor = `medium`, algorithm redesign = `large`

---

## Step 6: Assemble Findings Document

Write to `work/MMDDYY_Python_Audit.md` following shared.md Output Format exactly:

### 6a. Header

```markdown
# Python Audit

**Date:** MMDDYY
**Prior:** work/MMDDYY_Python_Audit.md (or "None")
**Mode:** Full | Quick (--quick)
**Agent:** Plan | None (--quick) | None (timeout)
**Tools:** radon cc X.X.X, radon mi X.X.X, vulture X.X.X, ruff X.X.X (list versions, note any SKIPPED)
```

Get tool versions:
```bash
radon --version 2>/dev/null || echo "not installed"
vulture --version 2>/dev/null || echo "not installed"
ruff --version 2>/dev/null || echo "not installed"
```

### 6b. Executive Summary

2–3 sentences covering:
- Total findings count and severity distribution
- Highest-impact finding
- Trend vs prior (if `COMPARE` is true)

### 6c. Metrics Dashboard

| Metric | Value | Prior | Trend |
|--------|-------|-------|-------|
| Total findings | N | — | — |
| Critical | N | — | — |
| High | N | — | — |
| Medium | N | — | — |
| Low | N | — | — |
| Avg cyclomatic complexity | X.X | — | — |
| Avg maintainability index | X.X | — | — |
| Dead code (est. LOC) | N | — | — |
| Lint violations | N | — | — |
| Algorithmic complexity findings | N | — | — |

Prior and Trend columns populated only when `COMPARE` is true.

### 6d. Findings Tables

Group all findings by severity. Each table uses shared.md format:

```markdown
### Critical

| # | File | Line | Issue | Tool | LOC Impact | Effort |
|---|------|------|-------|------|------------|--------|
```

Repeat for High, Medium, Low/Informational.

Number findings sequentially across all severity levels (P-1, P-2, P-3... for Python findings).

### 6e. Auto-Fixed Section

Only if `AUTO_FIX` is true. List what ruff `--fix` changed:

```markdown
## Auto-Fixed

| # | File | What Changed | Tool |
|---|------|-------------|------|
| 1 | path/to/file.py | Removed unused import (F401) | ruff --fix |
```

### 6f. Delta Section

Only if `COMPARE` is true. Follow shared.md Delta Tracking rules:
- New Findings (not in prior)
- Resolved (was in prior, now gone)
- Unchanged (count only)
- Trend Summary table

### 6g. Historical Section

If prior existed, list resolved items:
```markdown
## Historical (Resolved)

- [MMDDYY] Fixed: [brief description from prior findings no longer present]
```

### 6h. Recommendations

Top 3 highest-impact recommendations with estimated effort. Prioritize:
1. CRITICAL findings first (hotspot correlation issues)
2. HIGH findings (CC 30+, dead code >200 LOC, security lint)
3. Quick wins with highest LOC-to-effort ratio

---

## Step 7: Post-Processing

Follow shared.md for all of these:

### 7a. Tech Debt Integration

1. Read `.claude/rules/tech-debt-patterns.md`
2. For each new CRITICAL or HIGH finding: check if already has an entry, if not append per shared format
3. For resolved findings: mark existing entries as `~~RESOLVED~~` with date
4. Respect 100-line cap — evaluate oldest entries for removal if near cap

### 7b. Git Commit

```bash
git add work/MMDDYY_Python_Audit.md
git add docs/archive/audits/  # if priors archived
git add .claude/rules/tech-debt-patterns.md  # if updated

git commit -m "$(cat <<'EOF'
docs(audit): python audit — N findings (X critical, Y high)

Tools: radon cc X.X.X, radon mi X.X.X, vulture X.X.X, ruff X.X.X
Mode: Full | Quick
[Delta: +N new, -N resolved]  (only if --since last)
[Auto-fixed: N issues]  (only if --fix)

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
```

### 7c. Completion Report

Output to user per shared.md Completion Report format — findings summary table, top 3 findings, output file paths.

### 7d. Sprint Chain

Only if `SPRINT_CHAIN` is true:
- If 0 critical/high findings: skip chain, show "No critical/high findings — sprint chain skipped"
- Otherwise: build sprint context per shared.md Sprint Chain rules and invoke `/sprint`

---

## Tool Version Discovery

When reporting tool versions in the header, run:
```bash
python3 -c "import radon; print(radon.__version__)" 2>/dev/null || echo "not installed"
python3 -c "import vulture; print(vulture.__version__)" 2>/dev/null || echo "not installed"
ruff version 2>/dev/null || echo "not installed"
```

---

## Execution Checklist

Before completing, verify all items from shared.md Execution Checklist are met. Key Python-specific items:

- [ ] All 7 phases ran (or 5 in quick mode) — each phase either produced findings or was skipped with install note
- [ ] radon CC severity: 30+ HIGH, 15–29 MEDIUM, 11–14 LOW
- [ ] radon MI severity: D/F HIGH, C MEDIUM, B LOW
- [ ] vulture severity by LOC: >200 HIGH, 50–200 MEDIUM, <50 LOW
- [ ] ruff severity by rule prefix: E/F/S/B HIGH, W/C/N MEDIUM, I/D/UP LOW
- [ ] Hotspot correlation: high churn + high CC = CRITICAL
- [ ] Pattern analysis cross-referenced known tech debt (if tech-debt-patterns.md exists)
- [ ] Phase 7 (algorithmic complexity): grep patterns ran, agent verified in full mode
- [ ] Phase 7 severity: O(n²)+ hot path HIGH, O(n²)+ cold path MEDIUM, O(n) vs O(1) hot path MEDIUM, bounded/cold LOW
- [ ] Findings numbered sequentially (P-1, P-2, ...)
- [ ] Tool versions listed in header (or "SKIPPED (not installed)")

---

## See Also

- `/audit-typescript` — TypeScript equivalent (circular deps, dead exports, oxlint)
- `/audit-rust` — Rust equivalent (clippy, unwrap audit)
- `/audit-codesweep` — Code duplication analysis (complementary to linting)
