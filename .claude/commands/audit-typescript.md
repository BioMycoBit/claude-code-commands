# /audit-typescript

TypeScript codebase audit: circular dependencies, dead code, type duplication, linting, component health, orphan detection, and algorithmic complexity.

**Target:** TypeScript/React source directories (e.g., `src/`)

---

## Scope Boundary

**This audit covers:** Holistic TypeScript code quality — circular dependencies (madge), dead exports, type duplication, linting (oxlint), component health (props, hooks, prop drilling, state architecture), orphan detection, and algorithmic complexity.

**NOT covered here (see /audit-deadcode):** Cross-language dead code view — Python dead code (vulture), uncalled Python functions, and Plan agent safe-removal planning. Phase 2 (dead exports) and Phase 6 (orphans) are intentionally shared with `/audit-deadcode` — both audits run them from different perspectives.

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
ls work/*_TypeScript_Audit.md 2>/dev/null | sort -r | head -1
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

Run all 7 phases (or 4 tool phases + basic grep in `--quick`), collecting findings. Each finding must include: File, Line, Issue, Tool, LOC Impact, Effort.

### Phase 1: Circular Dependencies (madge)

```bash
npx madge --circular --extensions ts,tsx src/
```

- Finds circular import chains in TypeScript/TSX files
- **Severity mapping (by cycle node count):**
  - 4+ nodes in cycle = HIGH
  - 3 nodes = MEDIUM
  - 2 nodes = LOW
- Capture: cycle members (file paths), node count per cycle
- **LOC Impact:** sum of LOC across files in the cycle (estimate or `wc -l`)
- **Effort:** 4+ nodes = `medium`, 3 nodes = `small`, 2 nodes = `trivial`

**Error handling:** If `madge` not available:
```
madge not found. Install: npm install -D madge
Skipping Phase 1 (Circular Dependencies).
```
Note in Tools section of output: `madge --circular — SKIPPED (not installed)`

### Phase 2: Dead Code (export → import cross-reference)

**Skip basic grep if `--quick` mode** — export/import cross-referencing produces too many results without agent verification. Set `Dead exports: 0 (skipped in --quick)` in metrics.

**Full mode (agent-driven):**

If `AGENT_MODE` is true, use a Task agent (subagent_type: Plan) to:
1. Find all named exports: `grep -rn "export " src/ --include="*.ts" --include="*.tsx" | grep -v node_modules | grep -v "\.d\.ts"`
2. For each export, check if it has at least one import consumer elsewhere
3. Systematically verify each unmatched export has no dynamic imports, re-exports, or barrel references
4. Check for usage via `React.lazy()`, dynamic `import()`, or test files
5. Distinguish truly dead exports from entry points and public API surfaces

- **Severity mapping (by estimated LOC of dead export):**
  - \>200 LOC = HIGH
  - 50–200 LOC = MEDIUM
  - <50 LOC = LOW
- Capture: file, line, export name, type (function/component/type/constant)
- **LOC Impact:** estimate from exported item size
- **Effort:** types/constants = `trivial`, small functions = `small`, components/large functions = `medium`

### Phase 3: Type Duplication (grep + agent)

**Skip entirely if `--quick` mode.**

Use a Task agent (subagent_type: Plan) to analyze `src/` for:

1. **Interfaces/types defined in multiple files** with overlapping fields (e.g., same data shape defined in both services and components)
2. **Schema definitions (e.g., Zod, Valibot)** that duplicate interface shapes — a runtime schema and a TypeScript `interface` describing the same data
3. **Config types** that could share a base type (e.g., multiple config objects with shared fields)
4. **Prop types** duplicated across components that should be extracted to a shared types file

- **Severity mapping (by duplicate count):**
  - \>3 duplicates of same shape = HIGH
  - 2–3 duplicates = MEDIUM
  - Cosmetic overlap (1–2 shared fields) = LOW
- Capture: type/interface names, file locations, overlapping fields
- **LOC Impact:** total duplicated LOC across all instances
- **Effort:** shared base extraction = `small`, Valibot/interface consolidation = `medium`, config refactor = `medium`

### Phase 4: Linting (oxlint)

**If `AUTO_FIX` is true:**
```bash
npx oxlint src/ --fix
```
Record what was auto-fixed (capture output showing fixed files/rules).

**Then (always):**
```bash
npx oxlint src/
```

- Captures warnings and errors by rule
- **Severity mapping (by oxlint rule category):**
  - `correctness` rules (errors, logic bugs) = HIGH
  - `suspicious`, `nursery` rules (potential bugs) = MEDIUM
  - `pedantic`, `style` rules = LOW
  - Auto-fixed issues (if `--fix` was used) = INFO
- Capture: rule name, category, file, line, message
- **LOC Impact:** ~1 line per violation (approximate)
- **Effort:** auto-fixable = `trivial`, correctness = `medium`, style = `trivial`

**If `--fix` was used and changes were made:**
- Re-run `npx oxlint src/` to show remaining issues
- Populate Auto-Fixed section with what was fixed (before vs after counts)
- If fixes introduced new lint issues, report both states

**Error handling:** If `oxlint` not available:
```
oxlint not found. Install: npm install -D oxlint
Skipping Phase 4 (Linting).
```
Note in Tools: `oxlint — SKIPPED (not installed)`

### Phase 5: Component Health (grep + agent)

**Skip entirely if `--quick` mode.**

**Step 1 — Grep-based analysis (parallel):**

Run all grep patterns in parallel via Grep tool:

| Metric | Pattern | Files | Tool |
|--------|---------|-------|------|
| Props interfaces | `interface.*Props` | `*.tsx` | Grep |
| Props count | Count properties within Props interfaces | `*.tsx` | Read (follow-up) |
| useEffect hooks | `useEffect\(` | `*.tsx` | Grep |
| useState hooks | `useState\(` | `*.tsx` | Grep |
| useRef hooks | `useRef\(` | `*.tsx` | Grep |
| useCallback hooks | `useCallback\(` | `*.tsx` | Grep |
| useMemo hooks | `useMemo\(` | `*.tsx` | Grep |
| Zustand store usage | `useStore\|zustand\|create\(` | `*.ts`, `*.tsx` | Grep |
| Component size | Lines per `.tsx` component file | `*.tsx` | Bash (wc -l) |
| Inline styles | `style=\{\{` | `*.tsx` | Grep |

Analysis:
1. List all React components with line counts
2. Count useEffect hooks per component — flag components with >3 useEffect hooks
3. Identify components with >10 props
4. Note any prop drilling patterns (same prop passed through 3+ component levels)
5. Document Zustand store usage patterns
6. Flag components with >10 inline `style={{}}` instances

**Step 2 — Agent analysis (if AGENT_MODE = true):**

Launch Plan agent (subagent_type: Plan) with grep results.

**Full mode prompt:**
> "Analyze React component architecture in src/components/:
>
> 1. **Component Tree** — Build import hierarchy, find max depth
> 2. **Prop Drilling** — Props passed through >3 levels (trace origin -> consumer)
> 3. **Effect Density** — Components with >3 useEffect hooks + their dependencies
> 4. **State Lifting** — Same state in siblings? Props that should be context?
> 5. **God Components** — Components >400 LOC doing too much
> 6. **Inline Style Abuse** — Components with >10 inline style instances
>
> **Return:** Component health report with prop drilling chains, effect counts, god component splits, and state lifting recommendations."

**Use agent output to:** Add prop drilling chains, flag excessive effects (>3), recommend state lifting, identify god component split targets.

**On timeout/failure:** Continue with grep patterns from Step 1, skip prop drilling analysis. Note `Agent: None (timeout)` in header.

- **Severity mapping:**
  - \>600 LOC or >8 useEffects or >20 props = HIGH
  - Prop drilling through >5 levels = HIGH
  - \>400 LOC or >5 useEffects or >15 props = MEDIUM
  - Prop drilling through 3-5 levels = MEDIUM
  - 10+ inline styles or 3 useEffects or slightly oversized (250-300 lines) = LOW
  - Zustand store imported but only using 1 selector (could be prop) = LOW
- Capture: component name, file, LOC count, useEffect count, props count, specific concern
- **LOC Impact:** component file size
- **Effort:** God component split = `large`, useEffect consolidation = `medium`, CSS module extraction = `small`, context replacement = `medium`

### Phase 6: Orphan Detection (madge)

```bash
npx madge --orphans --extensions ts,tsx src/
```

- Finds files not imported by any other file
- **Exclude known entry points:** `App.tsx`, `main.tsx`, `vite-env.d.ts`, `*.test.*`, `*.spec.*`, `*.stories.*`
- **Severity mapping (by orphan file LOC):**
  - \>200 LOC orphan = HIGH
  - 50–200 LOC = MEDIUM
  - <50 LOC = LOW
- Capture: file path, LOC count
- **LOC Impact:** full file size (entire file is potentially removable)
- **Effort:** verify unused + delete = `trivial` for small files, `small` for large files

**Error handling:** If `madge` not available (same as Phase 1):
```
madge not found. Install: npm install -D madge
Skipping Phase 6 (Orphan Detection).
```
Note in Tools: `madge --orphans — SKIPPED (not installed)`

### Phase 7: Algorithmic Complexity (grep + agent)

Scan `src/` for suboptimal algorithmic complexity patterns. **Grep patterns run in all modes; agent verification runs in full mode only.**

**Grep patterns (always run):**

Use the Grep tool (NOT bash grep) for all pattern scans, issuing parallel calls:

1. **Linear search in loop** — `Array.includes()`, `Array.indexOf()` inside iteration
   - Pattern: `\.(includes|indexOf)\(` in `*.ts,*.tsx` files
2. **Nested iteration** — `.forEach`/`for...of`/`.map` inside another iteration
   - Pattern: `\.forEach\(` or `for\s*\(.*\bof\b` at increased indentation levels
3. **Filter-then-check chains** — `.filter().length`, `.filter()[0]` (should be `.some()` or `.find()`)
   - Pattern: `\.filter\(.*\)\s*\.\s*(length|\[0\])`
4. **Repeated array creation in loop** — `Object.keys()`, `Object.values()`, `Object.entries()` per-iteration
   - Pattern: `Object\.(keys|values|entries)\(` inside `for`/`.forEach`/`.map` bodies
5. **N+1 lookups** — `.find()` or `.filter()` inside `.map()`/`.forEach()`
   - Pattern: `\.(map|forEach)\(` with nested `\.(find|filter|includes)\(` (multiline check)
6. **Spread in reduce accumulator** — `[...acc, item]` in `.reduce()` creates new array per iteration (O(n²) total)
   - Pattern: `\.reduce\(.*\[\.\.\.`

**Full mode (agent-driven):**

If `AGENT_MODE` is true, use a Plan agent to verify grep hits:
1. Read surrounding context for each match (function scope + call sites)
2. Classify: is the collection size **bounded** (UI elements, config, small fixed lists) or **unbounded** (per-message, per-record, server data)?
3. Is the code in a **hot path** (render function, animation frame, WebSocket handler, store selector) or **cold path** (component mount, one-time setup)?
4. Determine actual complexity class: O(n²), O(n log n), O(n) where O(1) available
5. Filter false positives: `Set.has()` / `Map.get()` are O(1) and not findings

**Severity mapping:**
- O(n²)+ in hot path (render/animation/WS handler/store selector) = HIGH
- O(n²)+ in cold path (mount, config, one-time) = MEDIUM
- O(n) where O(1) available (`Array.includes` vs `Set.has()`) in hot path = MEDIUM
- O(n) where O(1) available in cold path, or bounded input (small fixed n) = LOW

**LOC Impact:** ~1–10 lines per finding (the loop + operation)
**Effort:** data structure swap (Array→Set/Map) = `small`, loop refactor = `medium`, algorithm redesign = `large`

---

## Step 6: Assemble Findings Document

Write to `work/MMDDYY_TypeScript_Audit.md` following shared.md Output Format exactly:

### 6a. Header

```markdown
# TypeScript Audit

**Date:** MMDDYY
**Prior:** work/MMDDYY_TypeScript_Audit.md (or "None")
**Mode:** Full | Quick (--quick)
**Agent:** Plan | Explore | None (--quick) | None (timeout)
**Tools:** madge X.X.X, oxlint X.X.X (list versions, note any SKIPPED)
```

Get tool versions:
```bash
npx madge --version 2>/dev/null || echo "not installed"
npx oxlint --version 2>/dev/null || echo "not installed"
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
| Circular dependency chains | N | — | — |
| Dead exports (est. LOC) | N | — | — |
| Lint violations (oxlint) | N | — | — |
| Orphan files | N | — | — |
| Total components | N | — | — |
| Components > 300 lines | N | — | — |
| Components with >3 useEffect | N | — | — |
| Components with >10 props | N | — | — |
| Prop drilling chains detected | N | — | — |
| Zustand store connections | N | — | — |
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

Number findings sequentially across all severity levels (T-1, T-2, T-3... for TypeScript findings).

### 6e. Auto-Fixed Section

Only if `AUTO_FIX` is true. List what oxlint `--fix` changed:

```markdown
## Auto-Fixed

| # | File | What Changed | Tool |
|---|------|-------------|------|
| 1 | path/to/file.tsx | Fixed correctness rule (rule-name) | oxlint --fix |
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

### 6h. Component Summary Table

Include a per-component summary:

```markdown
## Component Summary

| Component | Lines | Props | Effects | Stores | Status |
|-----------|-------|-------|---------|--------|--------|
| [name] | N | N | N | N | OK / WARN / ALERT |
```

Status thresholds:
- OK: <300 lines, ≤10 props, ≤3 effects
- WARN: 300-500 lines, 11-15 props, 4-5 effects
- ALERT: >500 lines, >15 props, >5 effects

### 6i. Recommendations

Top 3 highest-impact recommendations with estimated effort. Prioritize:
1. CRITICAL findings first (if any — requires both high churn + circular deps or similar compound risk)
2. HIGH findings (large circular chains, dead exports >200 LOC, correctness lint errors)
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
git add work/MMDDYY_TypeScript_Audit.md
git add docs/archive/audits/  # if priors archived
git add .claude/rules/tech-debt-patterns.md  # if updated

git commit -m "$(cat <<'EOF'
docs(audit): typescript audit — N findings (X critical, Y high)

Tools: madge X.X.X, oxlint X.X.X
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
npx madge --version 2>/dev/null || echo "not installed"
npx oxlint --version 2>/dev/null || echo "not installed"
```

---

## Execution Checklist

Before completing, verify all items from shared.md Execution Checklist are met. Key TypeScript-specific items:

- [ ] All 7 phases ran (or 4 tool + basic grep in quick mode) — each phase either produced findings or was skipped with install note
- [ ] Phase 1 (madge --circular): severity by node count — 4+ HIGH, 3 MEDIUM, 2 LOW
- [ ] Phase 2 (dead code): skipped in `--quick`, full agent cross-ref in full mode
- [ ] Phase 3 (type duplication): skipped in `--quick`, severity by duplicate count
- [ ] Phase 4 (oxlint): severity by rule category — correctness HIGH, suspicious MEDIUM, style LOW
- [ ] Phase 5 (component health): skipped in `--quick`, severity by LOC/useEffect/props count
- [ ] Phase 5: All grep patterns ran (Props, useEffect, useState, useRef, useCallback, useMemo, Zustand, inline styles)
- [ ] Phase 5: Component line counts collected
- [ ] Phase 5: Components with >3 useEffect flagged
- [ ] Phase 5: Components with >10 props flagged
- [ ] Phase 5: Component Summary table populated with OK/WARN/ALERT thresholds
- [ ] Phase 6 (madge --orphans): severity by orphan LOC — >200 HIGH, 50–200 MEDIUM, <50 LOW
- [ ] Known entry points excluded from orphan detection (App.tsx, main.tsx, vite-env.d.ts, test/spec/stories files)
- [ ] Phase 7 (algorithmic complexity): grep patterns ran, agent verified in full mode
- [ ] Phase 7 severity: O(n²)+ hot path HIGH, O(n²)+ cold path MEDIUM, O(n) vs O(1) hot path MEDIUM, bounded/cold LOW
- [ ] `--fix` runs oxlint auto-fix and reports before/after state
- [ ] Findings numbered sequentially (T-1, T-2, ...)
- [ ] Tool versions listed in header (or "SKIPPED (not installed)")

---

## See Also

- `/audit-python` — Python equivalent (radon, vulture, ruff)
- `/audit-rust` — Rust equivalent (clippy, unwrap audit, PyO3 bindings)
- `/audit-standards` — Linting trend + coding standards (Phase 4 runs same oxlint)
- `/audit-accessibility` — WCAG accessibility (component-level a11y attributes)
- `/audit-browser-compat` — Browser compatibility (component-level API usage)
