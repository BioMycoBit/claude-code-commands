# /audit-rust

Rust codebase audit: clippy pedantic, unwrap audit, dead code, test proportion, unused PyO3 binding detection, and algorithmic complexity.

**Target:** Rust crate directories (list your Cargo.toml locations)

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
ls work/*_Rust_Audit.md 2>/dev/null | sort -r | head -1
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

Run all 6 phases (or 4 if `--quick`), collecting findings. Each finding must include: File, Line, Issue, Tool, LOC Impact, Effort.

### Phase 1: Clippy Pedantic

**If `AUTO_FIX` is true:**
```bash
# Run for each crate in the workspace:
cargo clippy --manifest-path your-crate/Cargo.toml --fix --allow-dirty -- -W clippy::pedantic 2>&1
```
Record what was auto-fixed (capture output showing fixed files/rules).

**Then (always):**
```bash
# Run for each crate in the workspace:
cargo clippy --manifest-path your-crate/Cargo.toml -- -W clippy::pedantic 2>&1
```

- Runs clippy with pedantic warnings enabled on all crates in the workspace
- **Severity mapping (by clippy lint category):**
  - `error` = HIGH
  - `warning` (correctness, suspicious) = MEDIUM
  - `warning` (style, pedantic, complexity) = LOW
- Capture: warning/error category, file, line, message, clippy lint name
- **LOC Impact:** ~1–5 lines per warning (approximate)
- **Effort:** auto-fixable = `trivial`, correctness = `medium`, style/pedantic = `trivial`

**If `--fix` was used and changes were made:**
- Re-run `cargo clippy -- -W clippy::pedantic` to show remaining issues
- Populate Auto-Fixed section with what was fixed (before vs after counts)
- If fixes introduced new warnings, report both states

**Error handling:** If `cargo clippy` not available:
```
cargo clippy not found. Install: rustup component add clippy
Skipping Phase 1 (Clippy Pedantic).
```
Note in Tools section of output: `cargo clippy — SKIPPED (not installed)`

### Phase 2: Unwrap Audit

```bash
# Run across all crate src/ directories:
grep -rn "\.unwrap()" crate-a/src/ crate-b/src/ --include="*.rs"
grep -rn "\.expect(" crate-a/src/ crate-b/src/ --include="*.rs"
```

- **Exclude `#[cfg(test)]` blocks and doc-test code** — test unwraps are acceptable. For each match, check if it's inside a `#[cfg(test)]` module or `#[test]` function. Exclude those from findings.
- **Severity mapping:**
  - Production `.unwrap()` (no context on panic) = HIGH
  - `.expect()` without useful context message (e.g., `expect("")`, `expect("unwrap")`) = MEDIUM
  - `.expect("descriptive message")` in production = INFO (acceptable pattern)
- Capture: file, line, surrounding context (function name), whether production or test
- **LOC Impact:** ~1 line per call
- **Effort:** `.unwrap()` → `.expect("context")` = `trivial`, `.unwrap()` → proper error handling = `small`

### Phase 3: Dead Code (clippy dead_code + agent)

**Basic clippy dead_code (always runs):**
```bash
# Run for each crate in the workspace:
cargo clippy --manifest-path your-crate/Cargo.toml 2>&1 | grep -E "dead_code|unused"
```

- Capture: item name, type (function/struct/module/field/import), file, line
- **Severity mapping (by estimated LOC of dead item):**
  - \>200 LOC = HIGH
  - 50–200 LOC = MEDIUM
  - <50 LOC = LOW
- **Effort:** unused imports = `trivial`, small functions/fields = `small`, large modules = `medium`

**Agent cross-reference (skip if `--quick`):**

If `AGENT_MODE` is true, use a Task agent (subagent_type: Plan) to analyze all crates for:

1. **Public functions only used in tests** — `pub fn` that are never called from non-test code
2. **PyO3-exported functions** — overlap with Phase 5, agent deduplicates (only report in Phase 5)
3. **Modules imported but only partially used** — `use module::*` where most items are unused
4. **Structs with unused fields** — fields set but never read

**Error handling:** If agent times out or fails, set `Agent: None (timeout)` in header and continue with tool-only results.

### Phase 4: Test Proportion (tokei + grep)

**Skip if `--quick` mode.**

```bash
# Total LOC per crate (list all crate src/ directories):
tokei crate-a/src/ crate-b/src/
# Test LOC — count lines inside #[cfg(test)] modules
grep -rn "#\[cfg(test)\]" crate-a/src/ crate-b/src/ --include="*.rs"
```

For each `#[cfg(test)]` module found, estimate its LOC by reading the file and counting lines from the `#[cfg(test)]` to the closing brace.

- Calculate test:production LOC ratio per crate
- **Severity mapping:**
  - <10% test ratio = HIGH
  - 10–25% = MEDIUM
  - \>25% = INFO (healthy)
- Capture: crate name, production LOC, test LOC, ratio
- **LOC Impact:** N/A (metric, not a code issue)
- **Effort:** writing tests = `large` per crate

**Error handling:** If `tokei` not installed:
```
tokei not found. Install: cargo install tokei
Skipping Phase 4 (Test Proportion).
```
Note in Tools: `tokei — SKIPPED (not installed)`

### Phase 5: Unused PyO3 Bindings (cross-reference)

**Skip if `--quick` mode.**

Cross-reference PyO3-exported functions in your Rust crates with Python imports in your Python backend:

```bash
# Find all #[pyfunction] and #[pymethods] exports in PyO3 crates
grep -rn "#\[pyfunction\]" crate-a/src/ crate-b/src/ --include="*.rs"
grep -rn "#\[pymethods\]" crate-a/src/ crate-b/src/ --include="*.rs"
# Find Python imports from the corresponding PyO3 packages
grep -rn "from your_rust_package\|import your_rust_package" python-backend/ --include="*.py"
```

Use a Task agent (subagent_type: Plan) to verify each exported function is actually called from Python:

1. For each `#[pyfunction]`, check if the function name appears in any Python import and is called
2. For each `#[pymethods]` impl block, check if the struct's methods are used from Python
3. Identify the primary Python bridge files for each PyO3 crate and cross-reference actual usage

- **Severity mapping:**
  - Exported but never imported from Python = HIGH
  - Imported but never called (imported name unused) = MEDIUM
- Capture: function name, Rust file, Python consumer (or "none")
- **LOC Impact:** estimated LOC of the unused exported function/method
- **Effort:** remove export + function = `small`, remove export but keep internal = `trivial`

**Note:** Only scan crates that have PyO3 bindings (i.e., `#[pyfunction]` or `#[pymethods]` exports). Skip pure Rust service crates.

**Error handling:** If agent times out, fall back to grep-only cross-reference (report unmatched exports without deep call verification). Set `Agent: None (timeout)` in header.

### Phase 6: Algorithmic Complexity (grep + agent)

Scan all crate `src/` directories for suboptimal algorithmic complexity patterns. **Grep patterns run in all modes; agent verification runs in full mode only.**

**Grep patterns (always run):**

Use the Grep tool (NOT bash grep) for all pattern scans, issuing parallel calls:

1. **Linear search on Vec** — `.contains()`, `.iter().find()`, `.iter().position()` where HashSet would be O(1)
   - Pattern: `\.contains\(|\.iter\(\)\.find\(|\.iter\(\)\.position\(` in `*.rs` files
2. **Clone inside iterator** — `.clone()` per element when borrow would suffice
   - Pattern: `\.clone\(\)` inside `.iter()`, `.map()`, `.for_each()` chains
3. **Vec without pre-allocation** — `Vec::new()` followed by `.push()` without `with_capacity()`
   - Pattern: `Vec::new\(\)` — agent checks if size is known at construction
4. **Nested iteration** — `.iter()` inside `.iter()` or nested `for` loops on collections
   - Pattern: nested `.iter()` or `.for_each(` at increased indentation
5. **Repeated allocation in hot loop** — `String::from()`, `.to_string()`, `format!()` per iteration
   - Pattern: `String::from\(|\.to_string\(\)|format!\(` inside `for`/`.for_each`/`.map`
6. **HashMap get+insert** — `.get()` then `.insert()` where `.entry().or_insert()` is O(1) amortized
   - Pattern: `\.get\(` and `\.insert\(` in same function — agent verifies if sequential on same key

**Full mode (agent-driven):**

If `AGENT_MODE` is true, use a Plan agent to verify grep hits:
1. Read surrounding context for each match (function scope + call sites)
2. Classify: is the collection size **bounded** (config, small fixed sets) or **unbounded** (per-message, per-request)?
3. Is the code in a **hot path** (request handlers, data processing pipelines, broadcast loops) or **cold path** (startup, config)?
4. For `.clone()` hits: is the clone necessary (ownership transfer across threads) or avoidable (borrow would work)?
5. For `Vec::new()`: is the final size known or estimable at construction?

**Severity mapping:**
- O(n²)+ in hot path (request handlers, data processing pipelines) = HIGH
- O(n²)+ in cold path (startup, one-time) = MEDIUM
- O(n) where O(1) available (Vec `.contains()` vs HashSet) in hot path = MEDIUM
- Unnecessary allocation in hot path (`.clone()`, `String::from()`) = MEDIUM
- Bounded input or cold path = LOW

**LOC Impact:** ~1–10 lines per finding
**Effort:** data structure swap (Vec→HashSet) = `small`, borrow refactor = `small`, algorithm redesign = `large`

---

## Step 6: Assemble Findings Document

Write to `work/MMDDYY_Rust_Audit.md` following shared.md Output Format exactly:

### 6a. Header

```markdown
# Rust Audit

**Date:** MMDDYY
**Prior:** work/MMDDYY_Rust_Audit.md (or "None")
**Mode:** Full | Quick (--quick)
**Agent:** Plan | None (--quick) | None (timeout)
**Tools:** cargo clippy X.X.X, tokei X.X.X (list versions, note any SKIPPED)
```

Get tool versions:
```bash
cargo clippy --version 2>/dev/null || echo "not installed"
tokei --version 2>/dev/null || echo "not installed"
rustc --version 2>/dev/null || echo "not installed"
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
| Clippy warnings (pedantic) | N | — | — |
| Production .unwrap() calls | N | — | — |
| Dead code items | N | — | — |
| Test:production ratio (per crate) | X% | — | — |
| Unused PyO3 exports | N | — | — |
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

Number findings sequentially across all severity levels (R-1, R-2, R-3... for Rust findings).

### 6e. Auto-Fixed Section

Only if `AUTO_FIX` is true. List what `cargo clippy --fix` changed:

```markdown
## Auto-Fixed

| # | File | What Changed | Tool |
|---|------|-------------|------|
| 1 | path/to/file.rs | Fixed pedantic lint (lint-name) | cargo clippy --fix |
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
1. CRITICAL findings first (if any)
2. HIGH findings (production `.unwrap()`, large dead code, unused PyO3 exports)
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
git add work/MMDDYY_Rust_Audit.md
git add docs/archive/audits/  # if priors archived
git add .claude/rules/tech-debt-patterns.md  # if updated

git commit -m "$(cat <<'EOF'
docs(audit): rust audit — N findings (X critical, Y high)

Tools: cargo clippy X.X.X, tokei X.X.X
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
cargo clippy --version 2>/dev/null || echo "not installed"
tokei --version 2>/dev/null || echo "not installed"
rustc --version 2>/dev/null || echo "not installed"
```

---

## Execution Checklist

Before completing, verify all items from shared.md Execution Checklist are met. Key Rust-specific items:

- [ ] All 6 phases ran (or 4 in quick mode) — each phase either produced findings or was skipped with install note
- [ ] Phase 1 (clippy pedantic): all crates scanned (every Cargo.toml in the workspace)
- [ ] Phase 1 severity: error HIGH, correctness/suspicious warning MEDIUM, style/pedantic warning LOW
- [ ] Phase 2 (unwrap audit): `#[cfg(test)]` blocks excluded — test unwraps are acceptable
- [ ] Phase 2 severity: production `.unwrap()` HIGH, weak `.expect()` MEDIUM, descriptive `.expect()` INFO
- [ ] Phase 3 (dead code): basic clippy always runs, agent cross-ref only in full mode
- [ ] Phase 3 deduplicates with Phase 5 (PyO3 exports reported in Phase 5, not Phase 3)
- [ ] Phase 4 (test proportion): skipped in `--quick`, severity by ratio — <10% HIGH, 10–25% MEDIUM, >25% INFO
- [ ] Phase 5 (PyO3 bindings): skipped in `--quick`, scans only crates with PyO3 exports (skips pure Rust service crates)
- [ ] Phase 5 cross-references primary Python bridge files as consumers for each PyO3 crate
- [ ] Phase 6 (algorithmic complexity): grep patterns ran, agent verified in full mode
- [ ] Phase 6 severity: O(n²)+ hot path HIGH, O(n²)+ cold path MEDIUM, O(n) vs O(1) hot path MEDIUM, bounded/cold LOW
- [ ] `--fix` runs `cargo clippy --fix --allow-dirty` and reports before/after state
- [ ] Findings numbered sequentially (R-1, R-2, ...)
- [ ] Tool versions listed in header (or "SKIPPED (not installed)")

---

## See Also

- `/audit-python` — Python equivalent (radon, vulture, ruff)
- `/audit-typescript` — TypeScript equivalent (circular deps, dead exports, oxlint)
