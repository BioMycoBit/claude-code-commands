# /audit-sbom

Software Bill of Materials (SBOM) audit: generate CycloneDX/SPDX SBOMs across all ecosystems, check for completeness, flag unlicensed dependencies, validate SBOM freshness, and scan for known vulnerabilities via grype.

**Target:** `backend/` (Python), `<frontend-dir>/` (NPM), `your-rust-crate/` + `your-rust-crate/` + `your-rust-service/` (Rust), Docker images

---

## Scope Boundary

**This audit covers:** SBOM generation and validation — producing machine-readable dependency inventories (CycloneDX JSON), checking completeness against lockfiles, and flagging dependencies with missing license metadata.

**NOT covered here (see /audit-dependency):** Dependency version management, CVE scanning, outdated packages. This audit generates the *inventory*; `/audit-dependency` checks *currency and security*.

**NOT covered here (see /audit-licenses):** Detailed license compliance analysis — GPL/AGPL detection, copyleft risk assessment. This audit flags *missing* license metadata; `/audit-licenses` analyzes *compatibility and risk*.

---

## Step 1: Load Shared Infrastructure

1. Read `.claude/commands/audit/shared.md`
2. All output format, severity levels, delta tracking, archival, tech debt integration, commit format, sprint chain, error handling, and completion report rules come from shared.md — do NOT deviate

---

## Step 2: Parse Flags

Parse `$ARGUMENTS` per shared Flag Parsing rules:

1. Strip flags from arguments, set booleans:
   - `AGENT_MODE` — `false` always (no agent analysis — tool output is definitive)
   - `COMPARE` — `true` if `--since last`
   - `SPRINT_CHAIN` — `true` if `--sprint`
   - `AUTO_FIX` — `false` always (SBOM generation is read-only)
   - `SPRINT_FOLDER` — extracted if `--sprint v*_*` pattern provided
2. Report mode to user:
   ```
   Mode: Quick (always — no agent analysis)
   Flags: [active flags, or "none"]
   Note: --quick and --fix have no effect (audit is always tool-only, SBOM generation is read-only).
   ```

---

## Step 3: Find Prior Audit

```bash
ls work/*_SBOM_Audit.md 2>/dev/null | sort -r | head -1
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

No agent analysis — tool output is definitive. Each finding must include: File, Line, Issue, Tool, LOC Impact, Effort.

### Phase 1: Generate SBOMs

Generate CycloneDX SBOMs for each ecosystem using syft:

**1a. Python SBOM:**

```bash
cd <backend-dir> && syft dir:. --output cyclonedx-json=/tmp/sbom_python.json 2>&1
echo "Exit code: $?"
```

**1b. NPM SBOM:**

```bash
cd <frontend-dir> && syft dir:. --output cyclonedx-json=/tmp/sbom_npm.json 2>&1
echo "Exit code: $?"
```

**1c. Rust SBOMs:**

```bash
syft dir:your-rust-crate --output cyclonedx-json=/tmp/sbom_rust_signals.json 2>&1
syft dir:your-rust-crate --output cyclonedx-json=/tmp/sbom_rust_timeseries.json 2>&1
syft dir:your-rust-service --output cyclonedx-json=/tmp/sbom_rust_relay.json 2>&1
echo "Exit code: $?"
```

**1d. Docker SBOM (if images available):**

```bash
syft your-image:latest --output cyclonedx-json=/tmp/sbom_docker_relay.json 2>&1
echo "Exit code: $?"
```

If syft is not installed:
```
Tool not installed: syft
Install: curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b /usr/local/bin
Alternative: brew install syft (Mac) | go install github.com/anchore/syft/cmd/syft@latest
```
Note in output under Tools and skip to Phase 2 with lockfile-only analysis.

### Phase 2: Completeness Check

For each generated SBOM, validate completeness against lockfiles:

**2a. Python completeness:**

1. Read `pyproject.toml` — count declared dependencies
2. Parse `/tmp/sbom_python.json` — count components
3. Compare: SBOM components should be >= lockfile entries (transitive deps add more)
4. Flag any declared dependency missing from SBOM = HIGH

**2b. NPM completeness:**

1. Read `package.json` — count dependencies + devDependencies
2. Parse `/tmp/sbom_npm.json` — count components
3. Compare declared vs SBOM components
4. Flag missing packages = HIGH

**2c. Rust completeness:**

1. Read each `Cargo.toml` — count declared dependencies
2. Parse corresponding SBOM JSON — count components
3. Compare declared vs SBOM components
4. Flag missing crates = HIGH

**Lockfile-only fallback** (if syft not installed):

1. Read lockfiles directly (uv.lock, package-lock.json, Cargo.lock)
2. Count total resolved dependencies per ecosystem
3. Report counts without SBOM validation (note reduced coverage)

### Phase 3: License Metadata Check

For each SBOM, check for dependencies with missing license metadata:

```bash
# Parse SBOM JSON for components without license info
cat /tmp/sbom_python.json | python3 -c "
import json, sys
data = json.load(sys.stdin)
components = data.get('components', [])
missing = [c['name'] for c in components if not c.get('licenses')]
print(f'Total: {len(components)}, Missing license: {len(missing)}')
for m in missing[:20]:
    print(f'  - {m}')
" 2>&1
```

Repeat for each SBOM file.

Map to severity:
- Production dependency with no license metadata = MEDIUM (compliance risk)
- Dev dependency with no license metadata = LOW
- Docker base image with no license metadata = MEDIUM

### Phase 4: SBOM Freshness

Check if stored SBOMs exist and are current:

```bash
ls docs/sbom/*.json 2>/dev/null
```

If stored SBOMs exist:
1. Compare component counts: stored vs freshly generated
2. Flag significant divergence (>5% difference) = MEDIUM (SBOM is stale)
3. Report last modification date of stored SBOMs

If no stored SBOMs:
```
No stored SBOMs found in docs/sbom/.
Recommendation: Store generated SBOMs in docs/sbom/ and regenerate on each release.
```
= LOW finding

### Phase 5: Summary Statistics

Produce aggregate statistics:

```markdown
## SBOM Summary

| Ecosystem | Declared Deps | SBOM Components | Missing License | Coverage |
|-----------|---------------|-----------------|-----------------|----------|
| Python | N | N | N | N% |
| NPM | N | N | N | N% |
| Rust (signals) | N | N | N | N% |
| Rust (timeseries) | N | N | N | N% |
| Rust (relay) | N | N | N | N% |
| Docker | — | N | N | — |
| **Total** | **N** | **N** | **N** | **N%** |
```

### Phase 6: Grype Vulnerability Scanning

Scan syft-generated SBOMs for known vulnerabilities using grype. This cross-references SBOM components against vulnerability databases (NVD, GitHub Advisory, etc.).

**Tool check:**

```bash
grype --version 2>&1
```

If grype is not installed:
```
Tool not installed: grype
Install: curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sh -s -- -b /usr/local/bin
Alternative: brew install grype (Mac) | go install github.com/anchore/grype/cmd/grype@latest
```
Note in output under Tools and skip Phase 6 entirely.

**6a. Scan SBOMs:**

Use the SBOMs generated in Phase 1 as input to grype:

```bash
# Scan Python SBOM
grype sbom:/tmp/sbom_python.json --output json 2>&1 | python3 -c "
import json, sys
data = json.load(sys.stdin)
matches = data.get('matches', [])
by_sev = {}
for m in matches:
    sev = m.get('vulnerability', {}).get('severity', 'Unknown')
    by_sev[sev] = by_sev.get(sev, 0) + 1
print(f'Total: {len(matches)}')
for sev in ['Critical', 'High', 'Medium', 'Low', 'Negligible']:
    print(f'  {sev}: {by_sev.get(sev, 0)}')
# Show top 10 critical/high
for m in matches[:10]:
    v = m.get('vulnerability', {})
    a = m.get('artifact', {})
    if v.get('severity') in ('Critical', 'High'):
        fix = v.get('fix', {}).get('versions', ['no fix'])
        print(f'  {v.get(\"id\")} ({v.get(\"severity\")}) — {a.get(\"name\")}@{a.get(\"version\")} → fix: {fix}')
"
```

Repeat for each SBOM generated in Phase 1:

```bash
grype sbom:/tmp/sbom_npm.json --output json 2>&1
grype sbom:/tmp/sbom_rust_signals.json --output json 2>&1
grype sbom:/tmp/sbom_rust_timeseries.json --output json 2>&1
grype sbom:/tmp/sbom_rust_relay.json --output json 2>&1
```

If Phase 1 SBOMs were not generated (syft unavailable), scan lockfiles directly:

```bash
# Fallback: scan directories directly
grype dir:<frontend-dir>/python-backend --output json 2>&1
grype dir:<frontend-dir> --output json 2>&1
grype dir:your-rust-service --output json 2>&1
```

**6b. Parse Grype Output:**

For JSON output, extract:
- `matches[]` — array of vulnerability matches, each with:
  - `vulnerability.id` — CVE ID (e.g., `CVE-2024-1234`)
  - `vulnerability.severity` — `Critical`, `High`, `Medium`, `Low`, `Negligible`
  - `vulnerability.description` — human-readable description
  - `vulnerability.fix.versions[]` — available fix versions (if any)
  - `artifact.name` — package name
  - `artifact.version` — installed version
  - `artifact.type` — ecosystem (`python`, `npm`, `rust-crate`)

**6c. Cross-Reference with audit-dependency:**

If a recent `work/*_Dependency_Audit.md` exists, note overlapping CVEs:
- CVEs found by both grype and audit-dependency → mark as `(confirmed)`
- CVEs found only by grype → mark as `NEW` (grype has broader database coverage)
- Do NOT deduplicate — each audit reports from its own perspective

**Severity mapping:**

| grype Severity | Audit Severity | Condition |
|----------------|----------------|-----------|
| Critical | CRITICAL | Fix available |
| Critical | HIGH | No fix available (awareness only) |
| High | HIGH | Fix available |
| High | MEDIUM | No fix available |
| Medium | LOW | Any |
| Low/Negligible | INFO | Count only |

**Output table:**

```markdown
### Grype Vulnerability Scan

| Ecosystem | Critical | High | Medium | Low | Total | Fixable |
|-----------|----------|------|--------|-----|-------|---------|
| Python | N | N | N | N | N | N/N |
| NPM | N | N | N | N | N | N/N |
| Rust (signals) | N | N | N | N | N | N/N |
| Rust (timeseries) | N | N | N | N | N | N/N |
| Rust (relay) | N | N | N | N | N | N/N |
| **Total** | **N** | **N** | **N** | **N** | **N** | **N/N** |

### Top Vulnerabilities

| CVE | Severity | Package | Installed | Fix Available | Ecosystem |
|-----|----------|---------|-----------|---------------|-----------|
| CVE-YYYY-NNNN | Critical | pkg | 1.2.3 | 1.2.4 | Python |
```

List up to 10 Critical/High CVEs with fix versions. Remaining findings as counts only.

**LOC Impact:** ~1 line per finding (version bump in lockfile/manifest)
**Effort:** `trivial` (version bump) to `medium` (major version upgrade with breaking changes)

---

## Step 6: Assemble Findings Document

Write to `work/MMDDYY_SBOM_Audit.md` following shared.md Output Format exactly:

### 6a. Header

```markdown
# SBOM Audit

**Date:** MMDDYY
**Prior:** work/MMDDYY_SBOM_Audit.md (or "None")
**Mode:** Quick (always — no agent analysis)
**Agent:** None
**Tools:** syft (SBOM generation), grype (vulnerability scanning), Read (lockfile analysis), Bash (JSON parsing) [note if syft/grype unavailable]
```

### 6b. Executive Summary

2-3 sentences covering:
- Total components across all ecosystems
- Missing license metadata count
- SBOM completeness status
- Trend vs prior (if `COMPARE` is true)

### 6c. Metrics Dashboard

| Metric | Value | Prior | Trend |
|--------|-------|-------|-------|
| Total findings | N | — | — |
| Critical | N | — | — |
| High | N | — | — |
| Medium | N | — | — |
| Low | N | — | — |
| Total SBOM components | N | — | — |
| Python components | N | — | — |
| NPM components | N | — | — |
| Rust components | N | — | — |
| Missing license metadata | N | — | — |
| SBOM completeness % | N% | — | — |
| Grype CVEs (critical) | N | — | — |
| Grype CVEs (high) | N | — | — |
| Grype CVEs (fixable) | N | — | — |

Prior and Trend columns populated only when `COMPARE` is true.

### 6d. Findings Tables

Group all findings by severity. Each table uses shared.md format.

Number findings sequentially with **SB-** prefix (SB-1, SB-2, SB-3...).

### 6e. Auto-Fixed Section

```markdown
## Auto-Fixed

N/A — SBOM generation is read-only. Missing license metadata must be resolved upstream or documented.
```

### 6f. SBOM Summary Table

Include the summary table from Phase 5.

### 6g. Delta Section

Only if `COMPARE` is true. Follow shared.md Delta Tracking rules.

### 6h. Historical Section

Per shared.md format.

### 6i. Recommendations

Top 3 highest-impact recommendations. Prioritize:
1. Dependencies missing from SBOM (incomplete inventory)
2. Production dependencies with missing license metadata (compliance risk)
3. SBOM storage and regeneration workflow (CI integration)

---

## Step 7: Post-Processing

Follow shared.md for all of these:

### 7a. Tech Debt Integration

Per shared.md Tech Debt Integration rules.

### 7b. Git Commit

```bash
git add work/MMDDYY_SBOM_Audit.md
git add docs/archive/audits/  # if priors archived
git add .claude/rules/tech-debt-patterns.md  # if updated

git commit -m "$(cat <<'EOF'
docs(audit): sbom audit — N findings (X critical, Y high)

Tools: syft, grype, Read, Bash
Mode: Quick (no agent)
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
| CRITICAL | SBOM generation fails entirely for an ecosystem | syft crashes on Python backend |
| HIGH | Declared dependency missing from SBOM (incomplete inventory) | `fastapi` in pyproject.toml but not in SBOM |
| HIGH | Production dependency with no license AND no upstream metadata | Unknown license on production package |
| MEDIUM | Production dependency with missing license metadata | Package without license field in SBOM |
| MEDIUM | SBOM significantly stale (>5% component divergence from current) | Stored SBOM missing 20 new dependencies |
| LOW | Dev dependency with missing license metadata | Test utility without license field |
| LOW | No stored SBOMs in repository | docs/sbom/ directory absent |
| INFO | SBOM format or tool version note | syft version, CycloneDX spec version |

---

## Execution Checklist

Before completing, verify all items from shared.md Execution Checklist are met. Key SBOM-specific items:

- [ ] Phase 1: SBOMs generated for all 3 ecosystems (or install command noted)
- [ ] Phase 2: Completeness validated against lockfiles
- [ ] Phase 3: License metadata checked for all components
- [ ] Phase 4: SBOM freshness assessed (stored vs current)
- [ ] Phase 5: Summary statistics table produced
- [ ] Phase 6: Grype vulnerability scan ran (or install command noted if unavailable)
- [ ] Grype findings cross-referenced with audit-dependency (if prior exists)
- [ ] Findings numbered with SB- prefix
- [ ] No agent analysis attempted (this audit has no agent phase)

---

## See Also

- `/audit-dependency` — Dependency version management (CVE scanning, outdated packages)
- `/audit-licenses` — License compliance (GPL/AGPL detection, copyleft risk)
- `/audit-security` — Security audit (code-level vulnerabilities)
- `/audit-docker` — Docker audit (container images, base image analysis)
