# /audit-dependency

Dependency version audit: manifest reading, version pinning, major version gaps, and outdated package detection across Python (pyproject.toml/uv.lock), NPM (package.json/package-lock.json), and Rust (Cargo.toml/Cargo.lock) ecosystems.

**Target:** `pyproject.toml`, `uv.lock`, `package.json`, `package-lock.json`, `**/Cargo.toml`, `**/Cargo.lock`

---

## Scope Boundary

**This audit covers:** Dependency version management AND vulnerability scanning — manifest reading, version pinning, major version gaps, outdated packages, CVE scanning (npm audit, pip-audit), and auto-fix for npm vulnerabilities. Merged from former `/audit-dependency-vuln` (Session 03, v2.0y_audits).

**NOT covered here (see /audit-licenses):** Detailed license compliance analysis — GPL/AGPL detection, copyleft risk assessment, license-checker output. This audit notes license concerns only as informational findings.

---

## Step 1: Load Shared Infrastructure

1. Read `.claude/commands/audit/shared.md`
2. All output format, severity levels, delta tracking, archival, tech debt integration, commit format, sprint chain, error handling, and completion report rules come from shared.md — do NOT deviate

---

## Step 2: Parse Flags

Parse `$ARGUMENTS` per shared Flag Parsing rules:

1. Strip flags from arguments, set booleans:
   - `AGENT_MODE` — `false` always (no agent analysis — pure manifest/lockfile reading)
   - `COMPARE` — `true` if `--since last`
   - `SPRINT_CHAIN` — `true` if `--sprint`
   - `AUTO_FIX` — `true` if `--fix` (runs `npm audit fix` for auto-fixable vulnerabilities)
   - `SARIF_OUTPUT` — `true` if `--sarif`
   - `SPRINT_FOLDER` — extracted if `--sprint v*_*` pattern provided
2. Report mode to user:
   ```
   Mode: Quick (always — no agent analysis)
   Flags: [active flags, or "none"]
   Note: --quick has no effect (audit is always tool-only).
   ```

---

## Step 3: Find Prior Audit

```bash
ls work/*_Dependency_Audit.md 2>/dev/null | sort -r | head -1
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

No agent analysis — this audit reads manifests and lockfiles. Each finding must include: File, Line, Issue, Tool, LOC Impact, Effort.

**Tool rule:** Use the built-in **Grep tool** (NOT bash grep/rg) for ALL pattern scans per shared.md Tool Rules. Issue multiple Grep calls in a **single parallel message** where patterns are independent.

### Dependency Delta Logic

In delta mode (`--since last`):
1. **CVE scans ALWAYS fresh** — `npm audit` and `pip-audit` MUST run every cycle (CVE databases update daily)
2. **Short-circuit version check:** If NONE of `package.json`, `package-lock.json`, `pyproject.toml`, `uv.lock`, `Cargo.toml`, `Cargo.lock` are in CHANGED_FILES → carry forward version/pinning findings. Note "Lockfiles unchanged — version findings carried forward. CVE scans refreshed."
3. If lockfiles changed: RE-SCAN only the changed ecosystem (Python/NPM/Rust)
4. CARRY FORWARD: Unchanged ecosystem version findings
5. MERGE + MARK: NEW | CARRIED | CHANGED | RESOLVED

If MODE == "full": Run all checks as below.

### Phase 1: Python Dependencies

1. **Read** `pyproject.toml` — extract all dependencies with version constraints
2. **Read** `backend/uv.lock` — check resolved versions
3. List major dependencies with pinned versions
4. Flag unpinned or loosely pinned dependencies (e.g., `>=` without upper bound)
5. Note dependency groups (`[dependency-groups]`) and their purpose

### Phase 2: NPM Dependencies

1. **Read** `package.json` — extract `dependencies` and `devDependencies` with version ranges
2. **Run** `cd <frontend-dir> && npm outdated --json` (Bash) — check for outdated packages
3. Flag major version gaps (current vs latest differ by major version)
4. Note `^` vs `~` vs exact pinning distribution

### Phase 3: Rust Dependencies

1. **Read** all `Cargo.toml` files (Glob `**/Cargo.toml`) — extract dependencies with version constraints
2. List workspace dependencies and their versions
3. Flag loosely pinned dependencies
4. Note any `path` dependencies (local crates)

### Phase 4: Cross-Ecosystem Analysis

1. Flag any dependencies marked for update or with known deprecation
2. Note dependencies that appear in multiple ecosystems (e.g., same functionality in Python + Rust)

### Phase 5: Vulnerability Scanning

> Merged from former `/audit-dependency-vuln` (v2.0y_audits Session 03). DV- findings now use DEP- prefix.

**5a. NPM Vulnerability Scan:**

```bash
cd <frontend-dir> && npm audit --json
```

Parse output for critical/high vulnerabilities:
- Extract package name, version, CVE ID, severity, fix version
- Group by severity (critical, high, moderate, low)

**If `AUTO_FIX` is true:**
```bash
cd <frontend-dir> && npm audit fix
```
Record what was fixed in Auto-Fixed section.

**5b. Python Vulnerability Scan:**

```bash
cd <backend-dir> && pip-audit --format json
```

Parse output for vulnerabilities:
- Extract package name, version, CVE ID, fix version
- Note if `pip-audit` is not installed (provide install command: `uv add --dev pip-audit`)

**5c. Rust Vulnerability Scan:**

```bash
cargo audit 2>&1
cargo deny check advisories 2>&1
```

- Extract any advisories or vulnerabilities from Cargo ecosystem
- Note if tools are not installed (`cargo install cargo-audit cargo-deny`)

**Vulnerability output table:**

```markdown
### Vulnerabilities

| # | Package | Version | CVE | Severity | Fix Version | Ecosystem | Status |
|---|---------|---------|-----|----------|-------------|-----------|--------|
| DEP-N | example | 1.0.0 | CVE-XXXX-XXXXX | CRITICAL | 1.0.1 | npm | NEW |
```

---

## Step 6: Assemble Findings Document

Write to `work/MMDDYY_Dependency_Audit.md` following shared.md Output Format exactly:

### 6a. Header

```markdown
# Dependency Audit

**Date:** MMDDYY
**Prior:** work/MMDDYY_Dependency_Audit.md (or "None")
**Mode:** Quick (always — no agent analysis)
**Agent:** None
**Tools:** Read (manifest analysis), Bash (npm outdated, npm audit, pip-audit, cargo audit, cargo deny), Grep (pattern scan)
```

### 6b. Executive Summary

2–3 sentences covering:
- Total findings count and severity distribution
- Most significant version gaps or pinning issues
- Trend vs prior (if `COMPARE` is true)

### 6c. Metrics Dashboard

| Metric | Value | Prior | Trend |
|--------|-------|-------|-------|
| Total findings | N | — | — |
| Critical CVEs | N | — | — |
| High | N | — | — |
| Medium | N | — | — |
| Low | N | — | — |
| Python dependencies | N | — | — |
| NPM dependencies | N | — | — |
| NPM devDependencies | N | — | — |
| Rust dependencies | N | — | — |
| Major version behind | N | — | — |
| Unpinned dependencies | N | — | — |
| NPM vulnerabilities | N | — | — |
| Python vulnerabilities | N | — | — |
| Rust advisories | N | — | — |

Prior and Trend columns populated only when `COMPARE` is true.

### 6d. Findings Tables

Group all findings by severity. Each table uses shared.md format:

```markdown
### Critical

| # | File | Line | Issue | Tool | LOC Impact | Effort |
|---|------|------|-------|------|------------|--------|
```

Repeat for High, Medium, Low/Informational.

Number findings sequentially with **DEP-** prefix (DEP-1, DEP-2, DEP-3... for Dependency findings).

### 6e. Auto-Fixed Section

If `AUTO_FIX` was true and `npm audit fix` ran:
```markdown
## Auto-Fixed

| # | Package | From | To | Tool |
|---|---------|------|----|------|
| 1 | example | 1.0.0 | 1.0.1 | npm audit fix |
```

If `AUTO_FIX` was false:
```markdown
## Auto-Fixed

N/A — Run with `--fix` to auto-fix npm vulnerabilities via `npm audit fix`. Version bumps still require manual testing.
```

### 6f. Delta Section

Only if `COMPARE` is true. Follow shared.md Delta Tracking rules.

### 6g. Historical Section

Per shared.md format.

### 6h. Recommendations

Top 3 highest-impact recommendations. Prioritize:
1. Dependencies with known deprecation or EOL
2. Major version gaps with security implications
3. Loosely pinned dependencies that could break on update

---

## Step 7: Post-Processing

Follow shared.md for all of these:

### 7a. SARIF Output (if --sarif)

If `SARIF_OUTPUT` is true, produce `work/MMDDYY_Dependency_Audit.sarif` per shared.md SARIF Writer Specification:

- **tool.driver.name:** `"audit-dependency"`
- **SARIF categories:** `"dependency-python"`, `"dependency-npm"`, `"dependency-rust"`
- **ruleId mapping:** Use `DEP-N` prefix from Markdown findings
- **pip-audit JSON → SARIF:** Map `pip-audit --format json` output. Each vulnerability becomes a result with `ruleId` = CVE ID, `level` = severity mapping.
- **npm audit JSON → SARIF:** Map `npm audit --json` output similarly.
- **cargo audit JSON → SARIF:** Map `cargo audit --json` output similarly.
- Output path: `work/MMDDYY_Dependency_Audit.sarif`

### 7b. Tech Debt Integration

Per shared.md Tech Debt Integration rules.

### 7c. Git Commit

```bash
git add work/MMDDYY_Dependency_Audit.md
git add work/MMDDYY_Dependency_Audit.sarif  # if --sarif
git add docs/archive/audits/  # if priors archived
git add .claude/rules/tech-debt-patterns.md  # if updated

git commit -m "$(cat <<'EOF'
docs(audit): dependency audit — N findings (X critical, Y high)

Tools: Read, Bash (npm outdated, npm audit, pip-audit, cargo audit/deny), Grep
Mode: Quick (no agent)
[Delta: +N new, -N resolved]  (only if --since last)
[Auto-fixed: N issues]  (only if --fix)
[SARIF: work/MMDDYY_Dependency_Audit.sarif]  (only if --sarif)

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
| CRITICAL | CVE with CVSS 9.0+ or known active exploitation | Remote code execution in production dependency |
| HIGH | CVE with CVSS 7.0-8.9, fix version available | Known vulnerability with patch available |
| HIGH | Deprecated or EOL dependency still in use | Package with no maintenance for 2+ years |
| HIGH | Major version gap with known breaking changes | React 17 when 19 is current |
| MEDIUM | CVE with CVSS 4.0-6.9 | Moderate vulnerability |
| MEDIUM | Major version behind (1 major) | Package at v4 when v5 is current |
| MEDIUM | Unpinned dependency in production | `>=2.0` without upper bound |
| LOW | CVE with CVSS <4.0, or outdated without known CVE | Low-severity advisory |
| LOW | Minor/patch version behind | Package at 2.1 when 2.3 is current |
| LOW | Loose pinning on dev dependency | `^` range on devDependency |
| INFO | Dependency noted for future review, tool not installed | Path dependency, workspace member |

---

## Execution Checklist

Before completing, verify all items from shared.md Execution Checklist are met. Key dependency-specific items:

- [ ] Phase 1: pyproject.toml and uv.lock read and analyzed
- [ ] Phase 2: package.json read, npm outdated ran
- [ ] Phase 3: All Cargo.toml files read and analyzed
- [ ] Phase 4: Cross-ecosystem analysis completed
- [ ] Phase 5: npm audit, pip-audit, cargo audit/deny ran (or install commands noted)
- [ ] Phase 5: CVE scans ran fresh (never carried forward from prior)
- [ ] Phase 5: Vulnerability table populated with CVE IDs and fix versions
- [ ] Delta: CVE scans always fresh, version findings use short-circuit logic
- [ ] Findings numbered with DEP- prefix
- [ ] No agent analysis attempted (this audit has no agent phase)
- [ ] `--fix` runs `npm audit fix` if set, or noted as N/A

---

## See Also

- `/audit-licenses` — License compliance audit (GPL/AGPL detection, copyleft risk)
- `/audit-security` — OWASP security audit (code-level vulnerabilities, not dependency CVEs)
- `/audit-python` — Python code quality audit
- `/audit-rust` — Rust code quality audit (clippy, dependency audit)
- `/audit-typescript` — TypeScript audit

> **Note:** `/audit-dependency-vuln` was merged into this audit in v2.0y_audits Session 03. Former DV- findings now use DEP- prefix.
