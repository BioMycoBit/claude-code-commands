# /audit-secrets

Secret scanning audit: git history scanning, working tree secrets, .env file hygiene, and hardcoded credential detection across all ecosystems.

**Target:** Full repository (git history + working tree), `.env*` files, `backend/`, `src/`, `your-rust-service/`, `your-rust-crate/`, `your-rust-crate/`

---

## Scope Boundary

**This audit covers:** Secret detection — scanning git history for leaked secrets, auditing .env files for proper exclusion, and grepping source code for hardcoded credentials/tokens/API keys.

**NOT covered here (see /audit-security):** Authentication flow analysis, authorization checks, IDOR, OWASP Top 10 code patterns, session management, CORS/CSRF configuration. This audit finds secrets; `/audit-security` analyzes whether auth/authz logic is correct.

**NOT covered here (see /audit-privacy):** PII data flow analysis, GDPR compliance, consent management. This audit finds leaked credentials; `/audit-privacy` traces personal data handling.

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
   - `AUTO_FIX` — `false` always (secrets require manual rotation, never auto-fix)
   - `SARIF_OUTPUT` — `true` if `--sarif`
   - `SPRINT_FOLDER` — extracted if `--sprint v*_*` pattern provided
2. Report mode to user:
   ```
   Mode: Quick (always — no agent analysis)
   Flags: [active flags, or "none"]
   Note: --quick and --fix have no effect (audit is always tool-only, secrets cannot be auto-fixed).
   ```

---

## Step 3: Find Prior Audit

```bash
ls work/*_Secrets_Audit.md 2>/dev/null | sort -r | head -1
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

**Tool rule:** Use the built-in **Grep tool** (NOT bash grep/rg) for ALL pattern scans per shared.md Tool Rules. Issue multiple Grep calls in a **single parallel message** where patterns are independent.

### Phase 1: gitleaks Scan

Scan git history and working tree for secrets:

```bash
gitleaks detect --source . --report-format json --report-path /tmp/gitleaks_report.json 2>&1
echo "Exit code: $?"
```

If gitleaks is not installed:
```
Tool not installed: gitleaks
Install: brew install gitleaks (Mac) | sudo apt install gitleaks (Linux) | go install github.com/gitleaks/gitleaks/v8@latest
```
Note in output under Tools and skip to Phase 2.

Parse JSON report:
- Extract: file, line, rule, match (redacted), commit, author, date
- Group by rule (aws-access-key, generic-api-key, private-key, etc.)
- Flag severity: private keys and cloud credentials = CRITICAL, API keys = HIGH, generic patterns = MEDIUM

**Working tree scan** (catches uncommitted secrets):

```bash
gitleaks detect --source . --no-git --report-format json --report-path /tmp/gitleaks_worktree.json 2>&1
echo "Exit code: $?"
```

### Phase 2: .env File Audit

**2a. Gitignore coverage** — verify .env files are excluded from version control:

```bash
git ls-files --cached -- '*.env' '.env*' '**/.env' '**/.env.*' 2>/dev/null
```

Any tracked .env file = CRITICAL finding.

**2b. .env file inventory** — find all .env files (tracked and untracked):

Use Glob to find all `.env*` files:
```
Glob("**/.env*")
Glob("**/*.env")
```

For each .env file found, check:

| Check | How | Severity |
|-------|-----|----------|
| Tracked in git? | `git ls-files --error-unmatch <file>` | CRITICAL if tracked |
| In .gitignore? | `grep -q '<pattern>' .gitignore` | HIGH if not in .gitignore |
| Contains real secrets? | Read file, check for non-placeholder values | MEDIUM (informational) |
| Has .env.example? | Check for corresponding .env.example or .env.template | LOW if missing |

**2c. .gitignore completeness** — verify .gitignore covers common secret file patterns:

Use Grep on `.gitignore`:
```
Grep(".env", path=".gitignore")
Grep("*.pem", path=".gitignore")
Grep("*.key", path=".gitignore")
Grep("credentials", path=".gitignore")
```

Flag missing patterns as LOW findings.

### Phase 3: Hardcoded Credential Grep

**TOOL RULE:** Use the built-in **Grep tool** directly for ALL pattern scans below.
Issue multiple Grep calls in a **single parallel message**.

**Scan Group 1 — Hardcoded passwords/secrets (Python):** 1 Grep call, *.py
Pattern: `(?i)(password|passwd|secret|api_key|apikey|access_key|private_key|token)\s*=\s*["'][^"']{8,}["']`
Exclude: test files, fixtures, .env.example patterns

**Scan Group 2 — Hardcoded passwords/secrets (TypeScript):** 1 Grep call, *.ts + *.tsx
Pattern: `(?i)(password|passwd|secret|apiKey|accessKey|privateKey|token)\s*[:=]\s*["'][^"']{8,}["']`
Exclude: test files, type definitions, config templates

**Scan Group 3 — Hardcoded passwords/secrets (Rust):** 1 Grep call, *.rs
Pattern: `(?i)(password|secret|api_key|token)\s*=\s*"[^"]{8,}"`

**Scan Group 4 — Private keys in source:** 1 Grep call, all files
Pattern: `-----BEGIN (RSA |EC |DSA |OPENSSH )?PRIVATE KEY-----`

**Scan Group 5 — Base64-encoded secrets:** 1 Grep call, *.py + *.ts + *.rs
Pattern: `(?i)(secret|key|token|password).*[A-Za-z0-9+/]{40,}={0,2}`

**Scan Group 6 — Connection strings with credentials:** 1 Grep call, all files
Pattern: `(?i)(mysql|postgres|mongodb|redis|amqp)://[^:]+:[^@]+@`

**Scan Group 7 — AWS/cloud credentials:** 1 Grep call, all files
Pattern: `AKIA[0-9A-Z]{16}|(?i)aws_secret_access_key\s*=`

### Phase 4: Verification (shared.md Phase 5.5)

For each finding from Phases 1-3:
1. **Read the file** at the reported line number
2. **Confirm the finding is real** — not a placeholder (`CHANGEME`, `xxx`, `your-key-here`), test fixture, example value, or environment variable reference (`os.environ`, `process.env`)
3. **Assign confidence:** HIGH (real secret), MEDIUM (ambiguous), LOW (false positive)
4. **Drop LOW confidence findings**

**Redaction rule:** NEVER include actual secret values in the audit output. Use `[REDACTED]` or show only first 4 characters + `****`.

---

## Step 6: Assemble Findings Document

Write to `work/MMDDYY_Secrets_Audit.md` following shared.md Output Format exactly:

### 6a. Header

```markdown
# Secrets Audit

**Date:** MMDDYY
**Prior:** work/MMDDYY_Secrets_Audit.md (or "None")
**Mode:** Quick (always — no agent analysis)
**Agent:** None
**Tools:** gitleaks (git history + working tree), Grep (hardcoded credentials), Read (.env audit)
```

### 6b. Executive Summary

2-3 sentences covering:
- Total findings count and severity distribution
- Whether any secrets were found in git history (most serious)
- .env file hygiene status
- Trend vs prior (if `COMPARE` is true)

### 6c. Metrics Dashboard

| Metric | Value | Prior | Trend |
|--------|-------|-------|-------|
| Total findings | N | — | — |
| Critical | N | — | — |
| High | N | — | — |
| Medium | N | — | — |
| Low | N | — | — |
| Secrets in git history | N | — | — |
| Secrets in working tree | N | — | — |
| .env files tracked in git | N | — | — |
| Hardcoded credentials | N | — | — |
| Private keys in source | N | — | — |

Prior and Trend columns populated only when `COMPARE` is true.

### 6d. Findings Tables

Group all findings by severity. Each table uses shared.md format:

```markdown
### Critical

| # | File | Line | Issue | Tool | LOC Impact | Effort |
|---|------|------|-------|------|------------|--------|
```

Repeat for High, Medium, Low/Informational.

Number findings sequentially with **SE-** prefix (SE-1, SE-2, SE-3...).

**CRITICAL:** Never include actual secret values in findings tables. Use `[REDACTED]`.

### 6e. Auto-Fixed Section

```markdown
## Auto-Fixed

N/A — Secrets cannot be auto-fixed. Leaked secrets require manual rotation:
1. Rotate the credential immediately
2. Remove from git history if in commits (use `git filter-repo` or BFG Repo-Cleaner)
3. Add to .gitignore if file-based
```

### 6f. Remediation Guide

Include a remediation guide section specific to secret types found:

```markdown
## Remediation Guide

### For secrets found in git history
1. **Rotate immediately** — the secret is compromised regardless of current branch state
2. **Remove from history** — `git filter-repo --invert-paths --path <file>` or BFG Repo-Cleaner
3. **Force push** — coordinate with team, all clones must re-fetch

### For hardcoded secrets in source
1. **Move to environment variable** — `os.environ["KEY"]` / `process.env.KEY`
2. **Use secrets manager** — Infisical (already deployed at :8080)
3. **Add .env to .gitignore** if not already

### For .env files tracked in git
1. **Remove from tracking** — `git rm --cached .env`
2. **Add to .gitignore** — ensure pattern covers all .env variants
3. **Rotate all values** — they're in git history now
```

### 6g. Delta Section

Only if `COMPARE` is true. Follow shared.md Delta Tracking rules.

### 6h. Historical Section

Per shared.md format.

### 6i. Recommendations

Top 3 highest-impact recommendations. Prioritize:
1. Secrets in git history (requires rotation + history rewrite)
2. Tracked .env files (requires removal + rotation)
3. Hardcoded credentials in source (requires refactor to env vars)

---

## Step 7: Post-Processing

Follow shared.md for all of these:

### 7a. SARIF Output (if --sarif)

If `SARIF_OUTPUT` is true, produce `work/MMDDYY_Secrets_Audit.sarif` per shared.md SARIF Writer Specification:

- **tool.driver.name:** `"audit-secrets"`
- **SARIF categories:** `"secrets"`
- **ruleId mapping:** Use `SE-N` prefix from Markdown findings
- **gitleaks native SARIF:** gitleaks supports `--report-format sarif` natively — use this output directly when available. For Grep-based findings, map via shared.md field mapping.
- **Redaction:** SARIF `message.text` must also use `[REDACTED]` — never include actual secret values
- Output path: `work/MMDDYY_Secrets_Audit.sarif`

### 7b. Tech Debt Integration

Per shared.md Tech Debt Integration rules.

### 7c. Git Commit

```bash
git add work/MMDDYY_Secrets_Audit.md
git add work/MMDDYY_Secrets_Audit.sarif  # if --sarif
git add docs/archive/audits/  # if priors archived
git add .claude/rules/tech-debt-patterns.md  # if updated

git commit -m "$(cat <<'EOF'
docs(audit): secrets audit — N findings (X critical, Y high)

Tools: gitleaks, Grep, Read
Mode: Quick (no agent)
[Delta: +N new, -N resolved]  (only if --since last)
[SARIF: work/MMDDYY_Secrets_Audit.sarif]  (only if --sarif)

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
| CRITICAL | Secret in git history (committed, even if since removed) | AWS key in old commit |
| CRITICAL | .env file tracked in git with real credentials | `.env` with DB password checked in |
| CRITICAL | Private key in source tree | RSA private key in config directory |
| HIGH | Hardcoded credential in source code (not in git history) | `password = "hunter2"` in Python |
| HIGH | .env file not in .gitignore | `.env.production` missing from .gitignore |
| HIGH | Connection string with embedded credentials | `postgres://user:pass@host/db` |
| MEDIUM | Generic secret pattern match (ambiguous) | `token = "abc123..."` — may be test fixture |
| MEDIUM | Missing .env.example for existing .env | No template for environment setup |
| LOW | .gitignore missing common secret patterns | No `*.pem` or `*.key` exclusion |
| LOW | Base64-encoded string matching secret pattern | Long base64 near keyword — may be data, not secret |
| INFO | Placeholder/example secret detected | `password = "CHANGEME"` — correct pattern |

---

## Execution Checklist

Before completing, verify all items from shared.md Execution Checklist are met. Key secrets-specific items:

- [ ] Phase 1: gitleaks ran on git history AND working tree (or install command noted)
- [ ] Phase 2: All .env files inventoried, gitignore coverage checked
- [ ] Phase 3: All 7 scan groups ran via Grep tool (not bash)
- [ ] Phase 4: Every finding verified — no false positives from placeholders/tests
- [ ] No actual secret values appear in the audit output (all `[REDACTED]`)
- [ ] Remediation guide included with rotation instructions
- [ ] Findings numbered with SE- prefix
- [ ] No agent analysis attempted (this audit has no agent phase)

---

## See Also

- `/audit-security` — OWASP security audit (auth/authz logic, injection, session management)
- `/audit-privacy` — Privacy/GDPR audit (PII data flows, consent, data retention)
- `/audit-dependency` — Dependency audit (CVE scanning, version management)
- `/audit-docker` — Docker audit (container security, exposed ports, mounted secrets)
