# Slop Check

Scan recently changed files for AI code anti-patterns that linters don't catch. Session-scoped — only checks files in the current blast radius, not the entire codebase.

## Usage

```
/slop-check                          # Auto-detect changed files from git diff
/slop-check @work/.../handoff.md     # Scope to handoff's blast radius
```

---

## Step 1: Determine Scope

### 1a: From Handoff (if provided)

Read the handoff doc, extract the **Blast Radius** file list. Then intersect with `git diff --name-only HEAD` to find files that were actually changed.

### 1b: From Git (no handoff)

```bash
git diff --name-only HEAD
```

If no uncommitted changes, use last commit:
```bash
git diff --name-only HEAD~1
```

### 1c: Validate

If 0 files in scope → output "Nothing to check" and stop.

---

## Step 2: Run Checks (Parallel)

Run all 7 checks in parallel using Grep across the scoped files. Report findings in a table.

### Check 1: Placeholder / Dummy Data (CRITICAL)

**BANNED PATTERN** — see user-preferences.md.

Grep scoped files for:
- `placeholder` (case-insensitive, skip comments that discuss the check itself)
- `dummy` (case-insensitive, exclude test fixtures and variable names like `dummy_device` in test files)
- `FIXME`
- `xxx` (as standalone token, not in hex/variable names)
- `todo.*hack` (case-insensitive)
- `"fallback"` or `"default"` used as literal string values for user-facing data
- `fake_` or `mock_` in non-test files

**Severity:** CRITICAL
**Rationale:** Missing state = show nothing or redirect. Never invent placeholder data.

### Check 2: Over-Abstraction (HIGH)

For each **new function/method** defined in scoped files (Grep for `def `, `function `, `const .* = (`, `=> {`):
1. Get the function name
2. Grep the entire codebase for call sites
3. If only 1 call site exists (the definition doesn't count) → flag it

**Severity:** HIGH
**Rationale:** Three similar lines > premature abstraction. Helpers should have 2+ consumers.

**Skip:** React components (single-use is fine), test helpers, `__init__` / constructors, lifecycle hooks (`on_startup`, `on_shutdown`).

### Check 3: Comment Noise (MEDIUM)

Grep scoped files for:
- `// increment` or `# increment` (obvious operation comments)
- `// set ` or `# set ` followed by variable name (e.g., `// set count`)
- `// removed` or `# removed` (stale markers — just delete the code)
- `// unused` or `# unused`
- `// old` or `# old`
- `// TODO` without a ticket/issue reference (bare TODOs are noise)
- `// eslint-disable` or `# noqa` added in this diff (suppressing instead of fixing)

**Severity:** MEDIUM
**Rationale:** Good code doesn't need narration. Stale markers are clutter.

### Check 4: Defensive Overkill (MEDIUM)

Grep scoped files for:
- `try:` / `try {` wrapping single function calls to internal code (not I/O, not external APIs)
- `?? "default"` or `?? ""` or `?? 0` on values that can't be nullish
- `|| "fallback"` or `|| []` or `|| {}` where the source is typed non-nullable
- `if (x !== null && x !== undefined)` on already-validated data
- `catch (e) { /* empty */ }` or `catch (e) { }` (swallowed errors)

**Severity:** MEDIUM
**Rationale:** Only validate at system boundaries. Trust internal code and framework guarantees.

### Check 5: Scope Creep (CRITICAL)

Compare the list of changed files (`git diff --name-only`) against the handoff's Blast Radius.

- Files changed that are NOT in the blast radius → flag each one
- Skip: lock files (`package-lock.json`, `uv.lock`), auto-generated files, `.gitignore`

**Severity:** CRITICAL (if handoff provided), skip (if no handoff)
**Rationale:** One handoff at a time. Don't scope-creep.

### Check 6: Backwards-Compat Shims (HIGH)

Grep scoped files for:
- `_unused` or `_old` variable names
- `// @deprecated` or `# @deprecated` on new code (old code is fine)
- Re-export patterns: `export { X } from` where X was removed/renamed
- `// for backwards` or `// backward compat` (case-insensitive)
- `_legacy` in new function/variable names

**Severity:** HIGH
**Rationale:** If code is unused, delete it. Don't keep it around "just in case."

### Check 7: Feature Creep (LOW — Manual Flag)

List all new function parameters, config options, and feature flags added in scoped files that are NOT mentioned in the handoff spec.

This check can't be fully automated — output the list for human review.

**Severity:** LOW (informational)
**Rationale:** Only make changes that are directly requested or clearly necessary.

---

## Step 3: Report

### 3a: Findings Table

Output all findings in one table, sorted by severity:

```
## Slop Check Results

**Scope:** [N] files checked
**Handoff:** [handoff name or "git diff"]

| # | Severity | Check | File | Line | Finding |
|---|----------|-------|------|------|---------|
| 1 | CRITICAL | Placeholder | `path/file.py` | 42 | `placeholder_user_id = "xxx"` |
| 2 | HIGH | Over-Abstraction | `path/util.ts` | 15 | `formatLabel()` — 1 call site |
| ...| | | | | |
```

### 3b: Verdict

| Findings | Verdict |
|----------|---------|
| 0 findings | **CLEAN** — no slop detected |
| MEDIUM/LOW only | **PASS** — [N] minor findings noted |
| Any HIGH | **WARN** — [N] findings need attention. Review before proceeding |
| Any CRITICAL | **STOP** — [N] critical findings. Fix before continuing |

### 3c: Gate Behavior (when called from /implement)

- **CLEAN / PASS** → proceed to Step 6 (Test)
- **WARN** → show findings, proceed unless user says stop
- **STOP** → show findings, **do not proceed**. Ask user before continuing.

---

## Design Notes

- **Session-scoped, not codebase-wide.** This is a fast quality gate, not an audit.
- **No `--fix` flag.** Slop requires human judgment — auto-fix would create different slop.
- **No agents.** All checks use direct Grep/Read. Keeps it fast (<30s).
- **Severity is slop-specific.** A "HIGH" here (single-use helper) might be "LOW" in a general audit. The severity reflects how much AI-generated code quality suffers from the pattern.

---

## See Also

- `/implement` — Calls this skill as Step 5b between Lint and Test
- `/audit-python`, `/audit-typescript` — Full codebase audits (structural, not session-scoped)
- `.claude/rules/user-preferences.md` — BANNED patterns (placeholder data, bypassing abstractions)

$ARGUMENTS
