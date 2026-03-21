# Security Audit

Standalone security audit with progressive summarization. Run anytime — not tied to eod-docs rotation.

Covers **authentication** (is the door locked?), **authorization** (can you open THIS door?),
**injection** (can you break the lock?), and **data protection** (what's behind the door?).

## Usage

```
/audit-security              # Full audit
/audit-security --no-agent   # Skip Explore/Plan agents, grep-only
/audit-security --delta      # Force delta mode (auto-detected by default)
/audit-security --full       # Force full scan
```

---

## Step 1: Find Prior Version

Search for most recent `*_Security_Audit.md` in both `work/` and `docs/eod-docs/`:

```
Glob("*_Security_Audit.md", path="work/")
Glob("*_Security_Audit.md", path="docs/eod-docs/")
```

Use the one with the newer MMDDYY date. Read full content and extract:
- PRIOR_DATE from filename
- All findings tables (carry-forward candidates)
- Executive summary (for agent prompts)
- Historical section (for progressive summarization)

If no prior found: MODE = "full", note "Prior Version: None".

---

## Step 2: Compute Delta Scope

If prior exists:

```
PRIOR_COMMIT = git log --since="PRIOR_DATE" --format="%H" | tail -1
CHANGED_FILES = git diff --name-only $PRIOR_COMMIT..HEAD \
  -- <frontend-dir>/ your-rust-service/ your-rust-crate/ your-cli/
DELTA_RATIO = len(CHANGED_FILES) / total source file count
MODE = "delta" if DELTA_RATIO < 0.20 else "full"
```

Override: `--delta` forces delta, `--full` forces full.

Report: `"Delta scope: [N] files changed since [PRIOR_DATE] ([DELTA_RATIO]%). Mode: [MODE]"`

### Import-Aware Invalidation

In delta mode, CHANGED_FILES alone can miss indirect impacts. If ANY of these **security-critical
shared files** appear in CHANGED_FILES, **force a full re-scan of the dependent sections** —
not just expanded spot-checking, but a complete re-run as if those sections were in full mode.

A single change to `auth.py` or `settings.py` can keep the delta ratio low (1 file out of hundreds)
while silently invalidating a large portion of carried-forward findings.

| Shared File | Forces Full Re-scan Of |
|-------------|------------------------|
| `api/routers/auth.py` | Section B3 (identity trust), Section C (session/token), all auth findings |
| `api/middleware/*.py` | Section B4 (rate limiting), CSRF findings, security header findings |
| `api/websocket/connection_manager.py` | ALL WebSocket findings — auth, identity trust, IDOR |
| `api/websocket/message_handlers.py` | ALL WS handler findings — dispatch + handler chain |
| `api/models/messages.py` | ALL WS message type findings — whitelist validation |
| `config/settings.py` | Secrets audit, rate limit inventory, CORS, all config findings |

**Rule:** If any invalidation-table file is in CHANGED_FILES, its dependent sections run in full mode
even if the overall MODE is "delta". Other sections can still use delta carry-forward.

### New Endpoint Detection

Delta mode can miss newly added endpoints in router files that are otherwise unchanged
(or that changed for unrelated reasons). To catch this:

1. Count endpoints in the prior Authorization Surface table
2. Grep current router files for `@router.(get|post|put|delete|patch|websocket)` decorators
3. Compare counts per router file — if current > prior, flag the delta
4. **Any net-new endpoint gets mandatory B1 (IDOR) + B3 (identity trust) review** regardless of mode

This prevents a new `DELETE /api/storage/sessions/{id}` from slipping through because
`storage.py` was already in the prior baseline and the file's other endpoints were unchanged.

---

## Step 3: Execute Audit

### Phase 1: Explore Agent (if not `--no-agent`)

Launch **one** Explore agent with `subagent_type=Explore`. Explore agents use Grep/Read/Glob
internally (all auto-approved). This is the only Task agent needed before Phase 2.

**Delta mode:** Pass prior executive summary + CHANGED_FILES list:
> "Prior analysis summary: [paste executive summary]
> Changed files since prior: [CHANGED_FILES list]
> Security-critical shared files changed: [list from invalidation table, if any]
> Focus analysis ONLY on changed files + invalidated areas."

**Full mode prompt:**
> "Trace security-critical paths in auth.py, routers/, middleware, websocket handlers:
>
> 1. **Auth Enforcement** — Which endpoints check auth? Which don't? Map decorator/middleware usage
> 2. **Authorization / Ownership** — For endpoints that READ by resource ID: is the authenticated user_id used as a filter in the query, or is the resource fetched by ID alone? For endpoints that WRITE/DELETE: does the system verify the user owns the target resource before executing?
> 3. **Identity Trust** — Is user_id always extracted from verified JWT? Check WebSocket message handlers: do they accept user_id from message payload (untrusted) or from the authenticated connection?
> 4. **Sensitive Data Flows** — PII from collection → processing → storage → output. Logged sensitive data?
> 5. **Attack Surfaces** — Entry points (WS, REST, file upload) → trace user input to dangerous operations
> 6. **CORS/Session** — Where configured? Access-Control headers, cookie handling
>
> **Return:** Security flow map with authenticated/unauthenticated paths marked, plus authorization status per data-access endpoint."

**On timeout/failure:** Continue with grep patterns. Note `Agent: None (timeout)` in header.

### Phase 2: Direct Pattern Scan (Semgrep + Grep)

**TOOL RULE:** Use **Semgrep** (via Bash tool) for AST-aware pattern scans that benefit
from comment/string exclusion (Scan Groups 1-3, 5, 7-8). Use the built-in **Grep tool**
for simple text patterns and patterns Semgrep can't express (Scan Groups 4, 6, CORS).
DO NOT delegate to Bash agents or Task agents.

#### Section A: Injection & Configuration Patterns

In delta mode, pass changed directories to Semgrep/Grep. In full mode, scan `backend/` and `src/`.

**Scan Groups 1-3, 5, 7-8 — Semgrep AST-aware scan:** 1 Bash call
```bash
semgrep --config .semgrep/security.yml --json backend/ src/ 2>/dev/null
```
Parse JSON output: `results[]` array contains `check_id`, `path`, `start.line`, `extra.message`, `extra.metadata.owasp`.

This replaces the following grep patterns with AST-aware matching (no false positives from comments, strings, or test fixtures):
- Scan Group 1 — Injection (A03): SQL injection f-strings, command injection, code execution, template injection
- Scan Group 2 — XSS (A03): dangerouslySetInnerHTML
- Scan Group 3 — Secrets & PII logging (A02+A09): hardcoded secrets, PII in logger calls
- Scan Group 5 — Network & config (A02+A05): TLS verify=False, debug mode
- Scan Group 7 — Verbose errors (A05): traceback exposure
- Scan Group 8 — Bare except: bare try/except blocks

**Scan Group 4 — Frontend secrets (A09):** 1 Grep call, *.ts + *.tsx
Pattern: `console\.log.*(password|token|secret)`
(Kept as Grep — simple text pattern, Semgrep adds no value here)

**Scan Group 6 — Access control (A01):** 1 Grep call, *.py
Pattern: `@app\.(get|post|put|delete)` — then manually check for `Depends|auth` nearby
(Kept as Grep — requires manual contextual review that Semgrep can't automate)

**CORS check (A05):** 1 Grep call, *.py
Pattern: `allow_origins.*\*|Access-Control-Allow-Origin.*\*`
(Kept as Grep — config string pattern, not a code construct)

Also check (via Bash — 1 command, chain with `&&`):
`grep -q '\.env' .gitignore && echo ".env in .gitignore: YES" || echo ".env in .gitignore: NO"`

#### Section B: Authorization & Ownership Patterns (IDOR / Mass Assignment)

**Purpose:** Authentication confirms WHO you are. Authorization confirms WHAT you're allowed to access. This section checks whether authenticated users can access or modify data that isn't theirs.

**Tool rule:** Use Grep + Read tools directly for IDOR checks. Semgrep was evaluated for
ownership patterns but can't express decorator + function body + absence-of-check structures.
No Bash/Task agents.

Run these checks on ALL routers and WebSocket handlers. In delta mode, restrict to CHANGED_FILES (plus invalidated files from Step 2).

##### B1: IDOR (Insecure Direct Object Reference)

For every endpoint that fetches a resource by ID (path param like `/{session_id}`, `/{user_id}`, `/{invite_id}`):

| Check | How | OWASP |
|-------|-----|-------|
| Ownership on READ | Does the DB query include `user_id` from JWT as a filter, or fetch by resource ID alone? | A01 |
| Ownership on DELETE | Does the system verify the user created/owns the resource before deleting? | A01 |
| Ownership on UPDATE | Does the system verify the user owns the resource before mutating? | A01 |
| Session scoping | For session-scoped data (telemetry, storage, sensitive data), is session membership verified? | A01 |
| **WS resource references** | Do WebSocket message handlers accept resource IDs (`session_id`, `device_id`) from payload that belong to other users? A valid authenticated connection can still reference resources it doesn't own. This is **IDOR over WebSocket**, distinct from identity spoofing (B3). | A01 |

**Output table per endpoint:**
```
| Endpoint/Handler | Method/MsgType | Auth | Ownership Check | Query Filter | Verdict |
```
Verdict: `SAFE` (user_id in query) | `IDOR_RISK` (fetches by ID alone) | `N/A` (global resource)

##### B2: Mass Assignment

For every POST/PUT/PATCH endpoint that accepts a request body:

| Check | How | OWASP |
|-------|-----|-------|
| Pydantic model used? | Is body typed as a Pydantic model or raw `dict`? | A04 |
| Extra fields policy | Does model use `extra='forbid'`? Default Pydantic v2 is `ignore` (safe but implicit) | A04 |
| Privilege field injection | Can user submit `role`, `is_admin`, `user_id`, `status` in body? | A01 |
| Direct DB write | Is `model_dump()` or `**body.dict()` passed directly to DB insert/update? | A04 |

**Flag:** Any endpoint accepting raw `dict` instead of Pydantic model. Any model without `extra='forbid'`.

##### B3: Identity Trust (JWT Claims)

| Check | How | OWASP |
|-------|-----|-------|
| JWT extraction | Is user_id always from `get_current_user()` (JWT `sub` claim), never from request body/params? | A07 |
| WebSocket identity | Do WS handlers use `data.get("user_id")` from message payload (untrusted) or authenticated connection identity? | A01 |
| Header trust | Is any HTTP header (X-User-ID, etc.) trusted for identity without JWT validation? | A07 |
| URL identity | Are URL path params like `/ws/{client_id}` treated as verified identity? | A01 |

**Flag:** Any handler that reads user_id from message payload instead of the authenticated session. Any header used for identity without JWT verification.

**Note:** B3 checks WHO the message claims to be from. B1's WS resource references check WHAT the message claims to access. Both must be verified — a user can be correctly identified but still reference someone else's session_id.

##### B4: Rate Limiting Inventory

For each endpoint in the Authorization Surface table, document rate limiting:

| Check | How | OWASP |
|-------|-----|-------|
| Rate limit present | Does the endpoint have `@limiter.limit()` decorator? | A04 |
| Rate limit config | What's the limit? (e.g., 3/min, 5/min, 60/min) | A04 |
| WebSocket rate limiting | Are WS connections and/or messages rate limited? | A04 |

**Output:** Add a `Rate Limit` column to the Authorization Surface table.

#### Section C: Session & Token Management

Inventory JWT/session lifecycle (carry forward — only re-check when auth.py changes):

| Property | What to Check | OWASP |
|----------|---------------|-------|
| Token lifetime | How long before JWT expires? Is it configurable? | A07 |
| Refresh mechanism | Is there token refresh? Or must user re-login? | A07 |
| Revocation | Can issued tokens be invalidated? (Redis blacklist, DB check, etc.) | A07 |
| Logout behavior | Does logout clear cookie AND invalidate token server-side? | A07 |
| Cookie flags | httpOnly, Secure, SameSite — all set correctly? | A02 |
| CSRF protection | Double-submit cookie? Token in header? | A01 |

#### Section D: Threat Model (carry forward — update when deployment changes)

Include a 5-line threat model at the top of the output document:
```
## Threat Model
- **Deployment:** [Local network / Cloud / Hybrid] — [public internet exposure?]
- **Primary threats:** [Who would attack? Curious LAN user? Drive-by scanner? Targeted?]
- **Data sensitivity:** [What's the worst data exposed? Biometric = GDPR Art. 9 special category]
- **Trust boundaries:** [Frontend ↔ Backend (auth?), Backend ↔ Relay (signed?), Backend ↔ DB (local?)]
- **Out of scope:** [Physical access, supply chain, social engineering — unless relevant]
```

### Phase 3: Plan Agent (if not `--no-agent`)

Launch **one** Plan agent with `subagent_type=Plan` AFTER Phase 2 grep patterns complete.
Feed it the Explore output + Grep results. This is the only Task agent in Phase 3.

**Full mode prompt:**
> "Prioritize security findings by exploitability:
>
> **Findings:** [Insert grep results + Explore agent output here]
>
> **Analyze:**
> 1. **Exploitability Score** — severity(CRIT=4,HIGH=3,MED=2,LOW=1) × reachability(Public=3,Auth=2,Admin=1) × data_sensitivity(PII=3,Session=2,Public=1)
> 2. **OWASP Grouping** — Categorize by OWASP Top 10
> 3. **Remediation Order** — Fix dependencies (auth before IDOR, ownership before injection)
> 4. **Quick Wins vs Major Refactors** — Separate low-effort fixes from redesigns
> 5. **Attack Chains** — For each CRITICAL finding, write a 3-5 step exploit path
>
> **Return:** Scored table (File | Issue | OWASP | Score | Fix Order | Effort) grouped by severity, attack chains for CRITICALs, plus quick wins checklist."

**On timeout/failure:** Output raw grep findings with Explore context if available.

---

## Step 4: Apply Progressive Summarization

Compare findings with prior version:

- **New finding** (not in prior) → Full detail in Current Findings
- **Unchanged** (exists in prior, still present) → Keep, mark `CARRIED`
- **Changed** (in prior but details differ) → Update, mark `CHANGED`
- **Fixed/resolved** (in prior, no longer present) → Move to Historical as single bullet:
  `- [MMDDYY] Fixed: [brief description]`
- **Old historical** (>2 cycles old) → Compress or remove

---

## Step 5: Update Tech Debt Memory

If new security debt found → append to `.claude/rules/tech-debt-patterns.md`
If debt resolved → remove from `tech-debt-patterns.md`
Skip if no changes.

---

## Step 6: Write Output

Output file: `work/MMDDYY_Security_Audit.md`

### Header

```markdown
# Security Audit

**Date:** MMDDYY
**Prior Version:** [path] (or "None")
**Agent:** [Explore | Plan | Explore + Plan | None (--no-agent) | None (timeout)]
**Delta:** [N] files changed ([X]%), Mode: [delta|full]
```

### Output Sections (in order)

1. **Executive Summary** — 2-3 sentences: posture change, new findings count, resolved count
2. **Threat Model** — 5-line deployment context (Section D)
3. **Critical/High Findings** — Table with exploitability scores + attack chains for CRITICALs
4. **Medium/Low Findings** — Standard table
5. **Security Improvements** — What got better since last audit
6. **XSS / SQL Injection / Command Injection / Network Checks** — Pattern scan results
7. **Authorization Surface** — Endpoint | Method | Auth | Ownership | Rate Limit | Status
8. **Session Management** — Token lifecycle table (Section C)
9. **Input Validation Summary** — Pydantic model coverage, raw dict endpoints, extra field policies
10. **Secrets Audit** — Type | Location | Status
11. **OWASP Top 10 Coverage** — Category | Status | Findings | Change
12. **Historical (Resolved)** — Compressed bullets from prior cycles
13. **Recommendations** — Numbered by exploitability score (highest first)

### Finding Tables

**Critical/High:**
`| File | Line | Vulnerability | OWASP | Score | Status |`

Score = Severity × Reachability × Data_Sensitivity (see Scoring section below)

**Medium/Low:**
`| File | Line | Vulnerability | OWASP | Remediation | Status |`

### Attack Chains (CRITICAL findings only)

```markdown
### Attack Path: [Finding Name]
1. [Precondition — what attacker needs]
2. [Action — what attacker does]
3. [Exploit — what happens]
4. [Impact — what data/state is compromised]
**Detection:** [How would you know? Or: None — no logging]
```

---

## Reference

### OWASP Top 10 Checklist

| # | Category | What to Check |
|---|----------|---------------|
| A01 | Broken Access Control | Missing auth, **IDOR**, **ownership checks**, **WS resource spoofing** |
| A02 | Cryptographic Failures | Weak hashing, exposed secrets, **cookie flags** |
| A03 | Injection | SQL, Command, Template injection |
| A04 | Insecure Design | Rate limits, **mass assignment**, input validation |
| A05 | Security Misconfiguration | Debug mode, default creds, verbose errors |
| A06 | Vulnerable Components | Cross-ref Dependency_Audit — list CRITICAL/HIGH CVE count |
| A07 | Auth Failures | Session fixation, **token lifecycle**, **untrusted identity sources** |
| A08 | Data Integrity Failures | Missing signature verification |
| A09 | Logging Failures | PII in logs, missing audit trail |
| A10 | SSRF | Unvalidated URLs in requests |

### Exploitability Scoring

```
Score = Severity × Reachability × Data_Sensitivity

Severity:     CRITICAL=4, HIGH=3, MEDIUM=2, LOW=1
Reachability: Public/Unauth=3, Authenticated=2, Admin=1
Data_Sensitivity: Biometric/PII=3, Session/State=2, Public/Config=1

Max score: 36 (CRITICAL + Public + Biometric)
```

**Triage note:** The formula weights all three dimensions equally, but reachability often
dominates in practice. A public/unauthenticated MEDIUM (score 18) can be more dangerous than
an authenticated CRITICAL (score 24) because the attacker pool is orders of magnitude larger.
When findings score similarly on paper, break ties by reachability first.

### Severity Levels

| Severity | Criteria | Example |
|----------|----------|---------|
| CRITICAL | Active exploit risk, no auth needed | SQL injection, RCE, **unauth data access** |
| CRITICAL | Secrets exposed in repo/logs | Hardcoded API key, PII in plaintext logs |
| HIGH | Auth bypass or **authorization bypass** | Missing auth on admin endpoint, **IDOR** |
| HIGH | Data exposure across user boundaries | **User reads another user's data** |
| MEDIUM | Defense in depth gap | Missing rate limiting, **raw dict body params** |
| MEDIUM | Potential escalation path | Permissive CORS, **untrusted WS payload identity** |
| LOW | Best practice gap | HTTP internal, **Pydantic extra='ignore' not 'forbid'** |
| LOW | Informational | Debug endpoint exists |

---

## Step 7: Promote & Commit

```bash
# Promote prior from work/ to docs/eod-docs/ (if prior was in work/)
git rm docs/eod-docs/OLDDATE_Security_Audit.md   # if exists
git mv work/PRIORDATE_Security_Audit.md docs/eod-docs/  # if was in work/

# Stage new + promotions + tech debt
git add work/MMDDYY_Security_Audit.md
git add docs/eod-docs/
git add .claude/rules/tech-debt-patterns.md

git commit -m "docs(security): Security Audit MMDDYY

Generated: work/MMDDYY_Security_Audit.md
Promoted: PRIORDATE_Security_Audit.md → docs/eod-docs/

[Generated with Claude Code]

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

## Step 8: Completion Report

```
## Security Audit Complete

**Date:** MMDDYY
**Mode:** [delta (N files, X%) | full]
**Agent:** [Explore + Plan | None]

### Summary
- **New findings:** N
- **Carried:** N
- **Resolved:** N
- **Critical:** N | **High:** N | **Medium:** N | **Low:** N

### Key Changes
- [1-2 sentence per notable change]

### Commit
[hash] docs(security): Security Audit MMDDYY
```
