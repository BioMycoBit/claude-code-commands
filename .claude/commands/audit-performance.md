# /audit-performance

Performance patterns audit: hot code path identification, inefficient loop patterns, list membership lookups, sequential awaits, N+1 queries, string concatenation in loops, unbounded growth, and missing memoization across Python, TypeScript, and Rust.

**Target:** `**/*.py`, `src/**/*.ts`, `src/**/*.tsx`, `your-rust-service/src/**/*.rs`, `your-rust-crate/src/**/*.rs`

---

## Scope Boundary

**This audit covers:** Algorithmic complexity patterns — O(n²) loops, N+1 queries, list membership lookups, sequential awaits, missing memoization, unbounded growth, string concatenation in loops, global retention, expensive renders, hot-path identification via Explore agent. **Also covers** file-level metrics — line counts, git churn frequency, file size rankings, complex nesting depth, setTimeout/setInterval cataloging, and Plan agent refactoring strategy for large files.

**NOT covered here:** Component health (see /audit-typescript Phase 5), dead code (see /audit-deadcode), linting trends (see /audit-standards).

> **Merge note (2026-03-15):** Absorbed `/audit-bottleneck` (323 LOC). The scope split (file metrics vs algorithmic patterns) was artificial — users running either audit expected the other's findings. Phase 3 contains all absorbed file-level analysis. BN- prefix retired.

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
   - `AUTO_FIX` — `false` always (`--fix` is a no-op — performance fixes require benchmarking)
   - `SPRINT_FOLDER` — extracted if `--sprint v*_*` pattern provided
2. Report mode to user:
   ```
   Mode: Full | Quick
   Flags: [active flags, or "none"]
   Note: --fix is ignored for performance audit (fixes require benchmarking)
   ```

---

## Step 3: Find Prior Audit

```bash
ls work/*_Performance_Audit.md work/*_Bottleneck_Audit.md 2>/dev/null | sort -r | head -1
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

### Performance Delta Logic

In delta mode (`--since last`):
- Apply Delta Mode Protocol from `shared.md`
- Carry forward: performance findings, hot path annotations
- Re-scan only CHANGED_FILES for grep patterns

### Phase 1: Grep-Based Pattern Analysis

Run each pattern via Grep tool. In delta mode, restrict file scope to CHANGED_FILES.

**Grep patterns (parallel):**

> See also: Shared Grep Patterns > Performance Patterns in `audit/shared.md` for canonical regex (sequential awaits, list membership, string concat, unbounded query).

| Category | Pattern | Files | Flag |
|----------|---------|-------|------|
| Loop calculations | `for.*in\|while\|\.forEach\|\.map(` | `*.py`, `*.ts`, `*.tsx` | Math/object creation in loop body |
| List membership | `if .+ in \[` (shared: Performance) | `*.py`, `*.ts`, `*.tsx` | Should be set/dict — O(n) → O(1) |
| List index lookup | `\.index(\|\.indexOf(` | `*.py`, `*.ts` | Should be dict lookup |
| Sequential awaits | `await .*\n\s*await ` (shared: Performance) | `*.py`, `*.ts` | `asyncio.gather()` / `Promise.all` |
| N+1 queries | `await\|fetch\|query` inside for/while | `*.py`, `*.ts` | Batch query |
| String concat in loop | `\+=\s*["'f]` (shared: Performance) | `*.py`, `*.ts` | Use join/template |
| Unbounded growth | `\.append\|\.push\|\.add` without size check | `*.py`, `*.ts` | Add size limit |
| Global retention | `global\|self\.\w+\s*=\s*\[\]` | `*.py` | Memory leak risk |
| Missing memoization | `useEffect\|useState` without `useMemo\|useCallback` | `*.tsx` | Add memoization |
| Expensive render | `\.map(.*=>` in components | `*.tsx` | Check re-render frequency |

**Analysis per finding:**
- Identify if pattern is in a hot path (WS handler, signal processing, render loop)
- Check data volume flowing through the pattern
- Determine if O(n) lookup, sequential await, or loop allocation is on a hot path

### Phase 2: File-Level Metrics (absorbed from audit-bottleneck)

**Step 1 — Large files by line count (Bash):**

```bash
find <backend-dir> <frontend-dir>/src your-rust-service/src your-rust-crate/src -name '*.py' -o -name '*.ts' -o -name '*.tsx' -o -name '*.rs' | xargs wc -l | sort -rn | head -20
```

**Step 2 — Complex nesting (Grep, parallel):**

| Search | Pattern | Files | Tool |
|--------|---------|-------|------|
| Deep nesting (Python) | `if.*if.*if` or deep indentation | `*.py` | Grep |
| Deep nesting (TS) | `if.*if.*if` or deep indentation | `*.ts`, `*.tsx` | Grep |
| Deep nesting (Rust) | `if.*if.*if` or deep indentation | `*.rs` | Grep |

**Step 3 — Hot paths via git log (Bash):**

```bash
git log --since="{PRIOR_DATE or 30 days ago}" --name-only --pretty=format: | sort | uniq -c | sort -rn | head -20
```

**Step 4 — Async patterns (Grep, parallel):**

| Search | Pattern | Files | Tool |
|--------|---------|-------|------|
| setTimeout/setInterval | `setTimeout\|setInterval` | `*.ts`, `*.tsx` | Grep |

**Analysis per finding:**
1. List top 20 largest files by line count
2. Identify files with most commits in last 30 days
3. Cross-reference: files that are both large AND high-churn are highest priority
4. Grep for setTimeout/setInterval usage

### Phase 3: Agent Analysis (if AGENT_MODE = true)

Launch Explore agent with `subagent_type=Explore` for hot path analysis, then Plan agent for refactoring strategy.

**Delta mode:** Use Agent Scope Narrowing from shared.md — pass prior executive summary + CHANGED_FILES list instead of full exploration prompt.

**Explore agent (hot paths):**
> "Identify performance-critical code paths:
>
> 1. **Hot Paths** — Functions called every frame/message (WS handlers, signal processing, React renders)
> 2. **Data Volumes** — Sizes flowing through flagged patterns (data arrays, WS rates, state updates)
> 3. **Context Check** — Are O(n) lookups, sequential awaits, loop allocations on hot paths?
>
> Focus: signal processing, WebSocket handlers, React useEffect components
>
> **Return:** Hot paths with call frequency and data sizes"

**Plan agent (refactoring strategy — run after Phase 2 file metrics):**
> "Create a refactoring strategy for identified code bottlenecks:
>
> **Findings:** [Insert Phase 2 file-level findings — large files, high complexity, many imports]
>
> **Analyze:**
> 1. **Root Cause** — God object / feature creep / legitimate complexity?
> 2. **Dependency Map** — What imports the bottleneck? Circular risks?
> 3. **Extraction Targets** — Cohesive function groups, target files, lines moved
> 4. **Refactoring Order** — Phase 1 (prep), Phase 2 (utils), Phase 3 (core logic)
>
> **Return:** Per-file table: Phase | Extract | Target File | Lines | Risk. Include prerequisites and keep-in-place items."

**Use agent output to:** Filter grep findings to hot-path findings + add "Hot Path: Yes/No" column + impact priority. Add phased extraction plan per bottleneck file.

**On timeout/failure:** Note all findings without filtering, skip refactoring plan. Note `Agent: None (timeout)` in header.

**Skip agent if `--quick` mode.** Set `Agent: None (--quick)` in header.

### Phase 4: Benchmark Coverage

Check which hot paths have pytest-benchmark coverage. Cross-reference hot paths from Phases 1-3 against existing benchmarks.

**Step 1 — Find existing benchmarks (Glob + Grep, parallel):**

| Search | Pattern | Tool |
|--------|---------|------|
| Benchmark files | `**/bench_*.py`, `**/benchmark_*.py`, `**/test_*benchmark*.py` | Glob |
| pytest-benchmark usage | `benchmark\(` | Grep (`*.py`) |
| Speed-test generated | `Generated by /speed-test` | Grep (`*.py`) |

**Step 2 — Build hot path inventory:**

From Phase 1-3 findings, extract all functions/modules flagged as hot paths:
- Signal processing functions (high frequency+)
- WebSocket handlers (per-message rate)
- React render-path functions (60 Hz)
- Any function with severity HIGH or CRITICAL

**Step 3 — Cross-reference:**

For each hot path, check if any benchmark file imports or calls it. Classify:
- **Covered** — benchmark exists and tests the function
- **Partial** — benchmark exists for the module but not the specific function
- **Uncovered** — no benchmark found

**Analysis output:** Coverage ratio (covered / total hot paths) and list of uncovered hot paths with recommended `/speed-test` invocation.

---

## Step 6: Assemble Findings Document

Write to `work/MMDDYY_Performance_Audit.md` following shared.md Output Format exactly:

### 6a. Header

```markdown
# Performance Audit

**Date:** MMDDYY
**Prior:** work/MMDDYY_Performance_Audit.md (or "None")
**Mode:** Full | Quick (--quick)
**Agent:** Explore | None (--quick) | None (timeout)
**Tools:** Grep (built-in), Bash (wc -l, git log), Read (file analysis)
```

### 6b. Executive Summary

2–3 sentences covering:
- Total findings count and severity distribution
- Highest-impact finding (e.g., O(n²) in hot path, N+1 query)
- Trend vs prior (if `COMPARE` is true)

### 6c. Metrics Dashboard

| Metric | Value | Prior | Trend |
|--------|-------|-------|-------|
| Total findings | N | — | — |
| Critical | N | — | — |
| High | N | — | — |
| Medium | N | — | — |
| Low | N | — | — |
| Hot path findings | N | — | — |
| Loop inefficiencies | N | — | — |
| Sequential awaits | N | — | — |
| Unbounded growth patterns | N | — | — |
| Missing memoization | N | — | — |
| Files >500 LOC | N | — | — |
| Files >1000 LOC | N | — | — |
| High-churn files (top 20) | N | — | — |
| setTimeout/setInterval uses | N | — | — |
| Benchmark coverage (hot paths) | N/M (%) | — | — |

Prior and Trend columns populated only when `COMPARE` is true.

### 6d. Findings Tables

Group findings by category. Each table uses columns: File | Line | Pattern | Severity | Hot Path | Recommendation | Status

```markdown
### Loop Calculations

| File | Line | Pattern | Severity | Hot Path | Recommendation | Status |
|------|------|---------|----------|----------|----------------|--------|
```

Repeat for each category: Loop Calculations, List Membership, List Index Lookup, Sequential Awaits, N+1 Queries, String Concat in Loop, Unbounded Growth, Global Retention, Missing Memoization, Expensive Render.

Number findings sequentially with **PF-** prefix (PF-1, PF-2, PF-3... for Performance findings).

### 6e. File Size Rankings

Include the top 20 largest files:

```markdown
## File Size Rankings

| Rank | File | Lines | Commits (30d) | Growth | Status |
|------|------|-------|---------------|--------|--------|
| 1 | path/to/file | N | N | +/-N | NEW/CARRIED/CHANGED |
```

### 6f. Refactoring Plan (agent output)

If Plan agent ran successfully, include:

```markdown
## Refactoring Plan

| Phase | Extract | Source File | Target File | Lines | Risk | Prerequisites |
|-------|---------|-------------|-------------|-------|------|---------------|
```

If agent did not run: `## Refactoring Plan\n\nN/A — Agent skipped (--quick) or timed out.`

### 6g. Benchmark Coverage

```markdown
## Benchmark Coverage

**Coverage:** N/M hot paths benchmarked (X%)
**Benchmark files found:** N

| Hot Path | Frequency | Benchmark File | Status |
|----------|-----------|----------------|--------|
| your_module.process_data | high frequency | your benchmark scripts | Covered |
| websocket.broadcast | 2-30 Hz | — | **Uncovered** |
```

For each **Uncovered** hot path, include a recommended `/speed-test` invocation:

```markdown
### Recommended Benchmarks

| Priority | Target | Command |
|----------|--------|---------|
| 1 | module.function | `/speed-test module.function` |
```

If no hot paths were identified (unlikely), note: `No hot paths identified — benchmark coverage N/A.`

### 6h. Auto-Fixed Section

**Not applicable for performance audit** — fixes require benchmarking.
```markdown
## Auto-Fixed

N/A — Performance audit is read-only. Use findings to prioritize optimization work.
```

### 6i. Delta Section

Only if `COMPARE` is true. Follow shared.md Delta Tracking rules.

### 6j. Historical Section

If prior existed, list resolved items per shared.md format.

### 6k. Recommendations

Top 3 highest-impact recommendations with estimated effort. Prioritize:
1. O(n²) or worse in hot paths (signal processing, WS handlers)
2. N+1 query patterns (DB call in loop)
3. Sequential awaits that could be parallelized
4. Unbounded growth without size limits

---

## Step 7: Post-Processing

Follow shared.md for all of these:

### 7a. Tech Debt Integration

Per shared.md Tech Debt Integration rules — append new CRITICAL/HIGH findings to `tech-debt-patterns.md`, mark resolved entries.

### 7b. Git Commit

```bash
git add work/MMDDYY_Performance_Audit.md
git add docs/archive/audits/  # if priors archived
git add .claude/rules/tech-debt-patterns.md  # if updated

git commit -m "$(cat <<'EOF'
docs(audit): performance audit — N findings (X critical, Y high)

Tools: Grep, Read
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

| Severity | Criteria | Example |
|----------|----------|---------|
| HIGH | O(n²) or worse in hot path | Nested loops over large data |
| HIGH | N+1 query pattern | DB call in loop |
| HIGH | File >1000 LOC with 10+ commits in 30 days | God object + high churn |
| HIGH | Sequential await chain in hot path | WS handler, signal processing |
| MEDIUM | O(n) where O(1) possible | List lookup instead of set |
| MEDIUM | Repeated calculation | Same math in every iteration |
| MEDIUM | File >500 LOC with moderate churn (5-9 commits) | Growing complexity |
| MEDIUM | Deep nesting (3+ levels) in frequently modified code | Maintainability risk |
| LOW | Micro-optimization | String concat vs join |
| LOW | Missing memoization | useMemo opportunity |
| LOW | File >500 LOC with low churn (<5 commits) | Stable large file |
| LOW | setTimeout/setInterval in non-hot-path code | Timer usage catalog |
| INFO | Large file that is legitimately complex (stable, well-tested) | No action needed |

---

## Execution Checklist

Before completing, verify all items from shared.md Execution Checklist are met. Key performance-specific items:

- [ ] Phase 1: All 10 grep pattern categories scanned
- [ ] Phase 1: Loop calculations, list membership, sequential awaits all checked
- [ ] Phase 1: Unbounded growth and global retention patterns scanned
- [ ] Phase 1: Missing memoization and expensive render patterns checked (tsx)
- [ ] Phase 2: Top 20 largest files identified with line counts
- [ ] Phase 2: High-churn files identified via git log
- [ ] Phase 2: Complex nesting patterns scanned (Python, TS, Rust)
- [ ] Phase 2: setTimeout/setInterval usage scanned
- [ ] Phase 2: File Size Rankings table populated
- [ ] Phase 3: Agent analysis ran (full mode) or skipped (quick mode) with correct header
- [ ] Phase 3: Refactoring Plan populated (if agent ran) or marked N/A
- [ ] Phase 4: Existing benchmark files discovered via Glob
- [ ] Phase 4: Hot paths cross-referenced against benchmark coverage
- [ ] Phase 4: Benchmark Coverage table populated with Covered/Uncovered status
- [ ] Phase 4: Recommended `/speed-test` invocations listed for uncovered hot paths
- [ ] Findings include Hot Path column (Yes/No) when agent provided context
- [ ] Findings grouped by category with per-category tables
- [ ] Findings numbered with PF- prefix
- [ ] `--fix` noted as N/A (benchmarking required)

---

## See Also

- `/audit-python` — Python code quality (overlaps: Python-specific patterns)
- `/audit-rust` — Rust code quality (overlaps: Rust-specific optimizations)
- `/audit-typescript` — TypeScript audit (overlaps: frontend patterns, component health)
- `/audit-deadcode` — Dead code audit (overlaps: large files may contain dead code)
- `/audit-techdebt` — Technical debt audit (overlaps: god objects, complexity)
- `/speed-test` — Generate pytest-benchmark tests for uncovered hot paths identified in Phase 4
