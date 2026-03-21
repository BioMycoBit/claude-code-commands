# /audit-licenses

License compliance audit: scan Python, NPM, and Rust dependencies for GPL/AGPL/copyleft licenses that may impose obligations on the project. Flags incompatible licenses for commercial use.

**Target:** `pyproject.toml`, `package.json`, `your-rust-service/Cargo.toml`, `your-rust-crate/Cargo.toml`, `your-rust-crate/Cargo.toml`

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
   - `SPRINT_FOLDER` — extracted if `--sprint v*_*` pattern provided
   - `AUTO_FIX` — `false` always (license audit is read-only, `--fix` is a no-op — license changes require human review)
2. Report mode to user:
   ```
   Mode: Full | Quick
   Flags: [active flags, or "none"]
   Note: --fix is ignored for license audit (requires human review)
   ```

---

## Step 3: Find Prior Audit

```bash
ls work/*_License_Audit.md 2>/dev/null | sort -r | head -1
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

Run all 4 phases, collecting findings. Each finding must include: Package, Version, License, Ecosystem, Severity, Status.

**Tool rule:** Use the built-in **Grep tool** (NOT bash grep/rg) for ALL pattern scans per shared.md Tool Rules. Use Bash for license-checking tools (pip-licenses, npx license-checker, cargo license) where installed.

### License-Specific Delta Logic

In delta mode (`--since last`):
- If `pyproject.toml` or `uv.lock` NOT in CHANGED_FILES → carry forward all Python findings
- If `package.json` or `package-lock.json` NOT in CHANGED_FILES → carry forward all NPM findings
- If `Cargo.toml` or `Cargo.lock` NOT in CHANGED_FILES → carry forward all Rust findings
- Only re-scan ecosystems where lockfiles changed

### Phase 1: Python Dependencies

**Tool check:**
```bash
pip-licenses --version 2>/dev/null
```

**If pip-licenses is installed:**
```bash
cd <backend-dir> && pip-licenses --format=json --with-urls
```

Parse output for copyleft licenses.

**If pip-licenses is NOT installed:**
1. Print: `pip-licenses not installed. Install with: uv add --dev pip-licenses`
2. Fall back to grep-based analysis:
   - Read `pyproject.toml` — extract all dependency names
   - For each dependency, check known license from common knowledge or PyPI metadata
   - Flag any unknown licenses for manual review

**Grep patterns (always run as supplement):**

1. **Dependencies (pyproject.toml)** — `dependencies` section in `pyproject.toml`
2. **Dev dependencies** — `dev-dependencies|optional-dependencies` in `pyproject.toml`
3. **License headers in vendored code** — `GPL|GNU General Public|AGPL|Affero` in `**/*.py`

**Severity mapping:**
- AGPL dependency (network copyleft — SaaS trigger) = CRITICAL
- GPL dependency (strong copyleft) = HIGH
- LGPL dependency (weak copyleft — OK for dynamic linking, risky for static) = MEDIUM
- MPL dependency (file-level copyleft) = LOW
- MIT, Apache-2.0, BSD, ISC = no finding (permissive)
- Unknown license = MEDIUM (requires manual review)

**LOC Impact:** ~1 line per finding (the dependency declaration)
**Effort:** `trivial` (swap for alternative) to `large` (rewrite functionality if no alternative exists)

### Phase 2: NPM Dependencies

**Tool check:**
```bash
cd <frontend-dir> && npx license-checker --version 2>/dev/null
```

**If license-checker is available:**
```bash
cd <frontend-dir> && npx license-checker --summary --json
```

Parse output. Flag GPL, AGPL, and unknown licenses.

**If license-checker is NOT available:**
1. Print: `license-checker not available. Install with: npm install -g license-checker`
2. Fall back to grep-based analysis:
   - Read `package.json` — extract all dependency names
   - Grep `node_modules/*/package.json` for `"license":` fields if node_modules exists
   - Flag any GPL/AGPL/unknown

**Grep patterns (always run as supplement):**

1. **Dependencies (package.json)** — `"dependencies"|"devDependencies"` in `package.json`
2. **License field in node_modules** — `"license".*GPL|"license".*AGPL` in `node_modules/*/package.json` (if exists)
3. **License headers in vendored code** — `GPL|GNU General Public|AGPL|Affero` in `src/**/*.{ts,tsx,js}`

**Severity mapping:** Same as Phase 1 (AGPL=CRITICAL, GPL=HIGH, LGPL=MEDIUM, MPL=LOW, permissive=no finding)

**LOC Impact:** ~1 line per finding
**Effort:** `trivial` (swap for alternative) to `large` (rewrite)

### Phase 3: Rust Dependencies

**Tool check:**
```bash
cargo license --version 2>/dev/null
```

**If cargo-license is installed:**
```bash
cd your-rust-service && cargo license --json
cd your-rust-crate && cargo license --json
cd your-rust-crate && cargo license --json
```

Parse output for copyleft licenses.

**If cargo-license is NOT installed:**
1. Print: `cargo-license not installed. Install with: cargo install cargo-license`
2. Fall back to grep-based analysis:
   - Read `Cargo.toml` for each Rust crate (your Rust crates)
   - Check `Cargo.lock` for resolved dependency list
   - Grep for known copyleft crate names

**Grep patterns (always run as supplement):**

1. **Dependencies (Cargo.toml)** — `\[dependencies\]` sections in `your-rust-service/Cargo.toml`, `your-rust-crate/Cargo.toml`, `your-rust-crate/Cargo.toml`
2. **License field in Cargo.toml** — `license\s*=` in `**/Cargo.toml`
3. **License headers in Rust source** — `GPL|GNU General Public|AGPL|Affero` in `your-rust-service/src/**/*.rs`, `your-rust-crate/src/**/*.rs`, `your-rust-crate/src/**/*.rs`

**Severity mapping:** Same as Phase 1 (AGPL=CRITICAL, GPL=HIGH, LGPL=MEDIUM, MPL=LOW, permissive=no finding)

**Note:** Rust statically links by default — LGPL becomes HIGH for Rust (static linking triggers LGPL obligations unlike Python/JS dynamic linking).

**LOC Impact:** ~1 line per finding
**Effort:** `trivial` (swap for alternative) to `large` (rewrite)

### Phase 4: Compliance Matrix

Aggregate findings from Phases 1-3 into a cross-ecosystem compliance summary.

**Analysis:**
- List every dependency with a non-permissive license
- Check license compatibility with each other (e.g., Apache-2.0 + GPL-2.0 = incompatible)
- Check compatibility with project's intended use (commercial, SaaS, open-source)
- Flag dual-licensed packages where one option is permissive
- Note any NOTICE file requirements (Apache-2.0 requires attribution in NOTICE)

**This phase is always run (no agent needed — it's aggregation of Phases 1-3).**

---

## Step 6: Assemble Findings Document

Write to `work/MMDDYY_License_Audit.md` following shared.md Output Format exactly:

### 6a. Header

```markdown
# License Audit

**Date:** MMDDYY
**Prior:** work/MMDDYY_License_Audit.md (or "None")
**Mode:** Full | Quick (--quick)
**Agent:** None (license audit is tool-based, no agent analysis needed)
**Tools:** pip-licenses | Grep (fallback), license-checker | Grep (fallback), cargo-license | Grep (fallback)
```

### 6b. Executive Summary

2-3 sentences covering:
- Total non-permissive dependencies found across all ecosystems
- Highest-risk finding (AGPL? GPL in Rust static linking?)
- Trend vs prior (if `COMPARE` is true)

### 6c. Metrics Dashboard

| Metric | Value | Prior | Trend |
|--------|-------|-------|-------|
| Total findings | N | — | — |
| Critical (AGPL) | N | — | — |
| High (GPL) | N | — | — |
| Medium (LGPL/Unknown) | N | — | — |
| Low (MPL) | N | — | — |
| Python deps scanned | N | — | — |
| NPM deps scanned | N | — | — |
| Rust deps scanned | N | — | — |
| Permissive deps | N | — | — |
| Copyleft deps | N | — | — |
| Unknown license deps | N | — | — |

Prior and Trend columns populated only when `COMPARE` is true.

### 6d. Findings Tables

Group all findings by severity. Each table uses this license-specific format:

```markdown
### Critical (AGPL)

| # | Package | Version | License | Ecosystem | File | Effort |
|---|---------|---------|---------|-----------|------|--------|
```

Repeat for High (GPL), Medium (LGPL/Unknown), Low (MPL).

Number findings sequentially with **L-** prefix (L-1, L-2, L-3... for License findings).

### 6e. Auto-Fixed Section

**Not applicable for license audit** — license changes require human review. If `--fix` was passed, note:
```markdown
## Auto-Fixed

N/A — License audit is read-only. License changes require human and legal review.
```

### 6f. Delta Section

Only if `COMPARE` is true. Follow shared.md Delta Tracking rules.

### 6g. Compliance Matrix

```markdown
## Compliance Matrix

| Package | License | Ecosystem | Commercial Use | SaaS Use | Distribution | Static Link | Action Needed |
|---------|---------|-----------|---------------|----------|-------------|-------------|---------------|
| example | GPL-3.0 | Python | ❌ | ❌ | ❌ copyleft | N/A (Python) | Replace or isolate |
```

### 6h. Recommendations

Top 3 highest-impact recommendations with estimated effort. Prioritize:
1. CRITICAL findings first (AGPL deps — SaaS exposure)
2. HIGH findings (GPL deps — copyleft contamination risk)
3. Unknown licenses (need manual verification)

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
git add work/MMDDYY_License_Audit.md
git add docs/archive/audits/  # if priors archived
git add .claude/rules/tech-debt-patterns.md  # if updated

git commit -m "$(cat <<'EOF'
docs(audit): license audit — N findings (X critical, Y high)

Tools: pip-licenses, license-checker, cargo-license [or Grep fallback]
Mode: Full | Quick
[Delta: +N new, -N resolved]  (only if --since last)

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

## Execution Checklist

Before completing, verify all items from shared.md Execution Checklist are met. Key license-specific items:

- [ ] All 4 phases ran — each phase produced findings or confirmed clean
- [ ] Phase 1: Python deps scanned via pip-licenses or grep fallback
- [ ] Phase 2: NPM deps scanned via license-checker or grep fallback
- [ ] Phase 3: Rust deps scanned via cargo-license or grep fallback (all 3 crates: relay, signals, timeseries)
- [ ] Phase 4: Compliance matrix aggregated from all ecosystems
- [ ] Severity mapping applied: AGPL=CRITICAL, GPL=HIGH, LGPL=MEDIUM (HIGH for Rust static linking), MPL=LOW
- [ ] License-specific metrics populated (deps scanned per ecosystem, permissive vs copyleft count)
- [ ] Findings numbered sequentially with L- prefix (L-1, L-2, ...)
- [ ] `--fix` noted as N/A (requires human review)
- [ ] Delta uses lockfile-change logic (unchanged lockfile = carry forward ecosystem)
- [ ] Missing tools handled gracefully (print install command, fall back to grep)
- [ ] Rust LGPL treated as HIGH (static linking)
- [ ] Compliance matrix includes commercial/SaaS/distribution compatibility columns

---

## See Also

- `/audit-docker` — Docker infrastructure audit (container security, resource limits, network isolation)
- `/audit-resilience` — Error handling & resilience audit (error swallowing, timeouts, retry logic)
- `/audit-python` — Python code quality audit (radon, vulture, ruff)
- `/audit-typescript` — TypeScript audit (circular deps, dead exports, oxlint)
- `/audit-rust` — Rust audit (clippy, unwrap audit, PyO3 bindings)
- `/audit-security` — OWASP security audit (dependency vulnerabilities overlap)
- `/eod-docs day4` — Day 4 Production Readiness includes Dependency Vulnerability Audit (complementary — safety vs legality)
