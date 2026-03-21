# /audit-privacy

GDPR/privacy audit: PII lifecycle tracing, data retention analysis, consent tracking, cookie usage, deletion mechanisms, and compliance gap assessment against GDPR articles (Art. 5, 6, 9, 13, 15, 17, 32).

**Target:** `**/*.py`, `src/**/*.ts`, `src/**/*.tsx`, `docker-compose.yml`

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
   - `AUTO_FIX` — `false` always (`--fix` is a no-op — privacy compliance requires legal review)
   - `SPRINT_FOLDER` — extracted if `--sprint v*_*` pattern provided
2. Report mode to user:
   ```
   Mode: Full | Quick
   Flags: [active flags, or "none"]
   Note: --fix is ignored for privacy audit (compliance changes require legal review)
   ```

---

## Step 3: Find Prior Audit

```bash
ls work/*_Privacy_GDPR_Audit.md 2>/dev/null | sort -r | head -1
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

Run all phases (skip agent in Phase 1 if `--quick`), collecting findings. Each finding must include: File, Line, Issue, Tool, LOC Impact, Effort.

**Tool rule:** Use **Semgrep** (via Bash tool) for AST-aware PII detection in Phase 2
(eliminates false positives like `email_validator`, variable names containing "email" in
non-PII contexts). Use the built-in **Grep tool** for simple text patterns in Phases 3-5.
Issue independent tool calls in a **single parallel message**.

### Privacy Delta Logic

In delta mode (`--since last`):
1. Load PRIOR_BASELINE for this audit (PII inventory, data retention, compliance gaps)
2. CARRY FORWARD: Findings where source file NOT in CHANGED_FILES
3. RE-SCAN: PII/cookie/retention patterns ONLY on CHANGED_FILES
4. VERIFY: Spot-check 3-5 carried PII inventory items still accurate
5. MERGE + MARK: NEW | CARRIED | CHANGED | RESOLVED

If MODE == "full": Run all patterns on entire codebase.

### Phase 1: Agent Analysis (if AGENT_MODE = true)

Launch Explore agent with `subagent_type=Explore`.

**Delta mode:** Use Agent Scope Narrowing from shared.md — pass prior PII inventory + CHANGED_FILES list instead of full exploration prompt.

**Full mode prompt:**
> "Trace PII data flows for GDPR compliance in auth.py, routers/, models/, devices/, services/:
>
> 1. **Collection Points** — Forms, API endpoints, device data that could identify users
> 2. **Storage Locations** — Redis, databases, log files, cookies, localStorage
> 3. **Transmission Paths** — Relay server, third-party services, WebSocket messages
> 4. **Deletion Mechanisms** — TTL expiration, user deletion endpoints, log rotation
>
> Focus on: user models, session handling, sensitive data, device identifiers
>
> **Return:** PII lifecycle map with GDPR article references (Art. 5 minimization, Art. 6 lawful basis, Art. 17 erasure, Art. 32 security)."

**Use agent output to:** Add PII type column, mark GDPR articles per finding, highlight cross-boundary data flows, identify retention gaps.

**On timeout/failure:** Continue with Semgrep/grep patterns below without lifecycle mapping. Note `Agent: None (timeout)` in header.

**Skip agent if `--quick` mode.** Set `Agent: None (--quick)` in header.

### Phase 2: PII Field Detection

**Semgrep AST-aware scan:** 1 Bash call
```bash
semgrep --config .semgrep/privacy.yml --json backend/ src/ 2>/dev/null
```
Parse JSON output: `results[]` array. Group by `extra.metadata.pii-type`.

This replaces broad grep patterns with AST-aware matching that:
- Detects actual PII field access (`$VAR.email`, `password = $EXPR`) not just text containing "email"
- Catches PII in logger calls (GDPR Art. 5 violation)
- Flags PII in localStorage (Art. 32 risk)
- Excludes test files automatically (via Semgrep `paths.exclude`)

**Grep fallback — patterns Semgrep can't express (parallel):**

| Search | Pattern | Files | Tool |
|--------|---------|-------|------|
| Name fields | `name` in models/schemas context | `*.py`, `*.ts` | Grep |
| Phone fields | `phone` | `*.py`, `*.ts` | Grep |
| Address fields | `address` | `*.py`, `*.ts` | Grep |

(Kept as Grep — too generic for Semgrep AST matching, require manual contextual review)

**Analysis:**
1. Review Semgrep PII findings for actual data handling vs incidental name matches
2. Grep PII in database schemas (`model|schema|field` context)
3. Cross-reference with agent PII lifecycle map (if available)

### Phase 3: Data Storage & Retention

**Grep patterns (parallel):**

| Search | Pattern | Files | Tool |
|--------|---------|-------|------|
| Data storage | Database write patterns | `*.py` | Grep |
| TTL/expiration | `ttl`, `expire`, `retention` | `*.py`, `*.ts`, `*.yml` | Grep |
| Redis TTL | `setex\|expire\|pexpire` | `*.py` | Grep |
| Log retention | `rotate\|retention\|maxBytes` | `*.py`, `*.yml` | Grep |

### Phase 4: Consent & Cookie Tracking

**Grep patterns (parallel):**

| Search | Pattern | Files | Tool |
|--------|---------|-------|------|
| Consent tracking | `consent`, `gdpr`, `privacy` | `*.py`, `*.ts`, `*.tsx` | Grep |
| Cookie usage | `cookie`, `Cookie`, `Cookies.set` | `*.py`, `*.ts`, `*.tsx` | Grep |
| LocalStorage | `localStorage\|sessionStorage` | `*.ts`, `*.tsx` | Grep |

### Phase 5: Deletion & Erasure Mechanisms

**Grep patterns (parallel):**

| Search | Pattern | Files | Tool |
|--------|---------|-------|------|
| User deletion | `delete.*user\|remove.*user` | `*.py`, `*.ts` | Grep |
| Data removal | `remove.*data\|purge\|cleanup` | `*.py` | Grep |
| Account deletion | `delete.*account\|deactivate` | `*.py`, `*.ts` | Grep |

---

## Step 6: Assemble Findings Document

Write to `work/MMDDYY_Privacy_GDPR_Audit.md` following shared.md Output Format exactly:

### 6a. Header

```markdown
# Privacy GDPR Audit

**Date:** MMDDYY
**Prior:** work/MMDDYY_Privacy_GDPR_Audit.md (or "None")
**Mode:** Full | Quick (--quick)
**Agent:** Explore | None (--quick) | None (timeout)
**Tools:** Grep (built-in), Read (file analysis)
```

### 6b. Executive Summary

2–3 sentences covering:
- Total findings count and severity distribution
- Most significant compliance gap (e.g., missing consent for sensitive data)
- Trend vs prior (if `COMPARE` is true)

### 6c. Metrics Dashboard

| Metric | Value | Prior | Trend |
|--------|-------|-------|-------|
| Total findings | N | — | — |
| Critical | N | — | — |
| High | N | — | — |
| Medium | N | — | — |
| Low | N | — | — |
| PII fields identified | N | — | — |
| Storage locations | N | — | — |
| Consent mechanisms | N | — | — |
| Deletion endpoints | N | — | — |
| GDPR articles assessed | N/6 | — | — |

Prior and Trend columns populated only when `COMPARE` is true.

### 6d. Findings Tables

Group all findings by severity per shared.md format.

Number findings sequentially with **PR-** prefix (PR-1, PR-2, PR-3... for Privacy findings).

### 6e. Auto-Fixed Section

```markdown
## Auto-Fixed

N/A — Privacy audit is read-only. Compliance changes require legal review.
```

### 6f. Privacy-Specific Output Tables

#### PII Inventory

```markdown
### PII Inventory

| Field | Location | Purpose | Encrypted | GDPR Art. | Status |
|-------|----------|---------|-----------|-----------|--------|
| email | users.py | Auth | Yes | Art. 6 | CARRIED |
```

#### Data Retention

```markdown
### Data Retention

| Data Type | Retention | Cleanup | Status |
|-----------|-----------|---------|--------|
| Session data | 24h | Redis TTL | CARRIED |
```

#### GDPR Compliance Checklist

```markdown
### GDPR Compliance Checklist

| Requirement | GDPR Art. | Status | Notes |
|-------------|-----------|--------|-------|
| Right to access | Art. 15 | ? | User can export their data |
| Right to deletion | Art. 17 | ? | User can delete account |
| Data minimization | Art. 5 | ? | Only necessary data collected |
| Consent tracking | Art. 6/9 | ? | User consents recorded (Art. 9 for special categories) |
| Privacy policy | Art. 13 | ? | Policy document exists |
| Data encryption | Art. 32 | ? | PII encrypted at rest |
```

**Note:** Special category data (e.g., health, biometric, genetic) requires Art. 9 special category consent — explicit consent for processing of special category data for the purpose of uniquely identifying a natural person.

#### Compliance Gaps

```markdown
### Compliance Gaps

| Gap | Severity | GDPR Art. | Recommendation | Status |
|-----|----------|-----------|----------------|--------|
| No special category consent | HIGH | Art. 9 | Add consent form | CARRIED |
```

### 6g. Delta Section

Only if `COMPARE` is true. Follow shared.md Delta Tracking rules.

### 6h. Historical Section

Per shared.md format.

### 6i. Recommendations

```markdown
## Recommendations

1. Implement user data export endpoint (Art. 15)
2. Add special category consent form (Art. 9)
3. Document data retention in privacy policy (Art. 13)
```

Prioritize:
1. Missing consent mechanisms (especially special category Art. 9)
2. Missing deletion/erasure endpoints (Art. 17)
3. PII without encryption or retention policy (Art. 32)

---

## Step 7: Post-Processing

Follow shared.md for all of these:

### 7a. Tech Debt Integration

Per shared.md Tech Debt Integration rules.

### 7b. Git Commit

```bash
git add work/MMDDYY_Privacy_GDPR_Audit.md
git add docs/archive/audits/  # if priors archived
git add .claude/rules/tech-debt-patterns.md  # if updated

git commit -m "$(cat <<'EOF'
docs(audit): privacy audit — N findings (X critical, Y high)

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

| Severity | Criteria |
|----------|----------|
| CRITICAL | PII stored without encryption, sensitive data without Art. 9 consent |
| HIGH | No user deletion endpoint (Art. 17), PII in logs without rotation |
| HIGH | Missing consent mechanism for data collection (Art. 6) |
| MEDIUM | No data export endpoint (Art. 15), incomplete privacy policy |
| MEDIUM | Cookie set without consent banner, PII without documented purpose |
| LOW | Minor retention policy gap, informational privacy notice missing |
| INFO | Privacy-related code noted for documentation |

---

## Execution Checklist

Before completing, verify all items from shared.md Execution Checklist are met. Key privacy-specific items:

- [ ] Phase 1: Agent analysis ran (full mode) or skipped (quick mode) with correct header
- [ ] Phase 2: PII field grep ran across Python and TypeScript
- [ ] Phase 3: Data storage and retention patterns scanned
- [ ] Phase 4: Consent and cookie tracking scanned
- [ ] Phase 5: Deletion/erasure mechanisms scanned
- [ ] PII Inventory table populated
- [ ] Data Retention table populated
- [ ] GDPR Compliance Checklist populated (all 6 articles assessed)
- [ ] Compliance Gaps table populated
- [ ] Sensitive data Art. 9 explicitly assessed
- [ ] Findings numbered with PR- prefix
- [ ] `--fix` noted as N/A (compliance requires legal review)

---

## See Also

- `/audit-security` — OWASP security audit (overlaps: data encryption, auth)
- `/audit-licenses` — License compliance audit (legal compliance complement)
- `/audit-database` — Database audit (overlaps: data retention, schema security)
- `/audit-docker` — Docker audit (overlaps: secrets in containers, volume security)
