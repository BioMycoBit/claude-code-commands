# /audit-database

Database integrity audit: schema consistency, query safety, connection patterns, data retention, backup/recovery, and migration safety across your databases (e.g., DuckDB, PostgreSQL, SQLite, MongoDB).

**Target:** Database schema files, query modules, migration scripts, ORM models

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
   - `AUTO_FIX` — `false` always (database audit is read-only, `--fix` is a no-op — schema and query changes require human review)
2. Report mode to user:
   ```
   Mode: Full | Quick
   Flags: [active flags, or "none"]
   Note: --fix is ignored for database audit (read-only by design)
   ```

---

## Step 3: Find Prior Audit

```bash
ls work/*_Database_Audit.md 2>/dev/null | sort -r | head -1
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

Run all 6 phases (or 4 if `--quick` — skip agent analysis in Phase 5 and Phase 6), collecting findings. Each finding must include: File, Line, Issue, Tool, LOC Impact, Effort.

**Tool rule:** Use the built-in **Grep tool** (NOT bash grep/rg) for ALL pattern scans per shared.md Tool Rules. Issue multiple Grep calls in a **single parallel message** where patterns are independent.

### Database-Specific Delta Logic

In delta mode (`--since last`):
- Filter CHANGED_FILES to `.py`, `.rs`, `.toml`, `.sql` extensions
- Re-scan only changed files for Phases 1-4 (grep-based)
- Agent phases (5, 6) always run full scope in full mode (backup/migration gaps don't localize to single files)
- Carry forward findings from unchanged files
- If dependency files changed (database driver version bump) → re-scan all phases

### Phase 1: Schema Consistency

Scan table definitions and model definitions across all databases for alignment.

**Grep patterns (parallel):**

1. **SQL CREATE TABLE** — `CREATE TABLE|CREATE.*IF NOT EXISTS` in `**/*.py`, `**/*.rs`, `**/*.sql`
2. **SQL column definitions** — `(INTEGER|DOUBLE|VARCHAR|TIMESTAMP|BIGINT|BOOLEAN|FLOAT|REAL|TEXT|SERIAL)` in `**/*.py`, `**/*.sql`
3. **ORM/Document model definitions** — `class.*Document|class.*Model|Collection|collection_name|db\[|Base.*Model|declarative_base` in `**/*.py`
4. **ORM field definitions** — `Field\(|Column\(|StringField|IntField|FloatField|DateTimeField|BooleanField|ListField|EmbeddedDocument` in `**/*.py`
5. **SQL table references** — `INSERT INTO|FROM\s+\w+|UPDATE\s+\w+|DROP TABLE` in `**/*.py`, `**/*.sql`
6. **Rust schema definitions** — `CREATE|INSERT|SELECT|execute|prepare` in `**/*.rs`

**Analysis:**
- Map SQL table schemas (columns, types) vs ORM/document model schemas (fields, types)
- Flag type mismatches between systems (e.g., DOUBLE vs FloatField with different precision)
- Flag naming inconsistencies (e.g., `device_id` in SQL vs `deviceId` in document models)
- Verify shared entities have consistent field naming across all databases
- If multiple languages access the same database, check that schemas match across language boundaries

**Severity mapping:**
- Type mismatch on shared field that could cause data loss (DOUBLE → INTEGER truncation) = HIGH
- Naming inconsistency on shared entity field (camelCase vs snake_case mismatch) = MEDIUM
- Missing field in one system that exists in the other = MEDIUM
- Schema defined in code but no migration mechanism = MEDIUM
- Minor type variance on non-critical field = LOW
- Consistent schema across systems = no finding (good pattern)

**LOC Impact:** ~1-5 lines per finding (type correction, field rename)
**Effort:** `trivial` (rename field) to `medium` (migrate data with type change)

### Phase 2: Query Safety

Scan for dangerous query patterns: unbounded queries, SQL injection vectors, N+1 patterns.

**Grep patterns (parallel):**

1. **Unbounded SELECT (no LIMIT)** — `SELECT.*FROM(?!.*LIMIT)` in `**/*.py`, `**/*.sql` (multiline context needed)
2. **Missing WHERE clause** — `SELECT.*FROM\s+\w+\s*$|SELECT.*FROM\s+\w+\s*ORDER` in `**/*.py`, `**/*.sql`
3. **String formatting in SQL** — `f".*SELECT|f".*INSERT|f".*UPDATE|f".*DELETE|\.format\(.*SELECT|%.*SELECT` in `**/*.py`
4. **String concatenation in SQL** — `\+.*SELECT|\+.*INSERT|\+.*WHERE|sql.*\+.*str` in `**/*.py`
5. **Parameterized queries (good pattern)** — `\?\s*,|\$\d|%s.*params|execute\(.*,\s*\[|execute\(.*,\s*\(` in `**/*.py`
6. **N+1 pattern (query in loop)** — `for.*:\s*\n.*execute|for.*:\s*\n.*query|for.*:\s*\n.*find` in `**/*.py` (multiline)
7. **Document DB unindexed queries** — `find\(|find_one\(|aggregate\(` in `**/*.py`
8. **Rust SQL queries** — `execute|query_row|prepare` in `**/*.rs`

**Analysis:**
- Flag `SELECT *` without `LIMIT` — could return unbounded rows (OOM risk on large tables)
- Flag string formatting/concatenation in SQL — SQL injection vector
- Flag queries inside loops — N+1 pattern, should be batched
- Cross-reference with query builders — are they using parameterized queries?
- Note any intentional inline params (e.g., numeric-only inlining for type compatibility) — these are NOT injection vectors if properly validated

**Severity mapping:**
- String formatting/concatenation in SQL with user-controlled input = CRITICAL
- `SELECT` without `LIMIT` on large/growing table (can be millions of rows) = HIGH
- N+1 query pattern (query in loop) = HIGH
- Missing WHERE clause on DELETE/UPDATE = HIGH
- String formatting in SQL with only internal values (no user input) = MEDIUM
- `SELECT` without `LIMIT` on small/bounded collection = LOW
- Parameterized queries used consistently = no finding (good pattern)

**LOC Impact:** ~1-10 lines per finding (add LIMIT, parameterize query, batch queries)
**Effort:** `trivial` (add LIMIT) to `medium` (refactor N+1 to batch query)

### Phase 3: Connection Patterns

Scan for concurrent access issues, connection lifecycle, and pooling patterns.

**Grep patterns (parallel):**

1. **SQL database connect (Python)** — `\.connect\(|Connection\(|create_engine|sessionmaker` in `**/*.py`
2. **SQL database connect (Rust)** — `Connection::open|Pool::new|connect\(` in `**/*.rs`
3. **Document DB connect** — `MongoClient|motor\.motor_asyncio|AsyncIOMotorClient|pymongo` in `**/*.py`
4. **Connection close** — `\.close\(\)|conn\.close|connection\.close|disconnect` in `**/*.py`
5. **Context manager for DB** — `with.*connect|async with.*connect|with.*Connection|with.*Session` in `**/*.py`
6. **Connection pooling** — `pool|Pool|max_connections|maxPoolSize|pool_size|create_engine` in `**/*.py`
7. **Concurrent DB access** — `threading|Thread|asyncio.*connect|lock.*connect|Lock.*connect` in `**/*.py`
8. **Query timeout/interrupt** — `interrupt\(\)|Timer.*interrupt|threading\.Timer|statement_timeout|lock_timeout` in `**/*.py`

**Analysis:**
- Check for database-specific concurrency limitations (e.g., embedded databases like DuckDB/SQLite may have file locking constraints that differ across platforms)
- Check if long-running queries have timeout guards (some databases lack built-in statement timeouts)
- Check connection pooling configuration (pool size, connection lifecycle)
- Verify connections are closed in error paths (context managers or try/finally)
- Check for multiple connections opened to same database file (file lock contention for embedded databases)

**Severity mapping:**
- Multiple connections to same embedded database file from different processes/libraries = CRITICAL
- Database connection without close/context manager in error path = HIGH
- No connection pooling configuration for client-server databases = HIGH
- Long-running query without timeout guard = HIGH
- Connection opened in loop without reuse = MEDIUM
- Connection pooling configured but with default values = LOW
- Context managers used consistently = no finding (good pattern)

**LOC Impact:** ~3-10 lines per finding (add context manager, add timeout, configure pool)
**Effort:** `small` (add context manager) to `medium` (refactor connection lifecycle)

### Phase 4: Data Retention

Scan for data lifecycle policies, TTL settings, and compliance gaps.

**Grep patterns (parallel):**

1. **TTL settings** — `TTL|ttl|expire|expireAfterSeconds|createIndex.*expire` in `**/*.py`
2. **Data deletion** — `delete_many|delete_one|drop_collection|DELETE FROM|TRUNCATE|remove\(` in `**/*.py`
3. **User data deletion** — `delete_user|delete_user_data|cascade.*delete|purge` in `**/*.py`
4. **Data retention/cleanup** — `DELETE.*WHERE.*timestamp|DELETE.*WHERE.*created|VACUUM|cleanup|retention` in `**/*.py`
5. **Indefinite storage markers** — No TTL or cleanup for collection/table — search for table/collection creation without corresponding cleanup code
6. **Privacy/compliance references** — `gdpr|GDPR|consent|right.*delete|data.*subject|personal.*data|ccpa|CCPA` in `**/*.py`
7. **Sensitive data storage** — Grep for application-specific sensitive data types (PII, credentials, tokens, etc.) in database service files
8. **Cascade delete coverage** — `refresh_token|invite|session|telemetry|user` in user management files

**Analysis:**
- Check if growing tables (timeseries, logs, events) have any retention policy (auto-delete after N days?)
- Check if document collections have TTL indexes
- Verify `delete_user_data()` (or equivalent) covers ALL data stores
- Flag any sensitive data stored without explicit retention period
- Cross-reference with any existing privacy audit findings

**Severity mapping:**
- Sensitive data stored indefinitely without retention policy = CRITICAL
- `delete_user_data()` missing data stores (incomplete cascade) = HIGH
- Database collection/table without TTL or cleanup mechanism = HIGH
- No data export capability for user data portability = MEDIUM
- Growing tables without periodic cleanup/maintenance = MEDIUM
- Short-lived data (session state) without explicit TTL = LOW
- Proper TTL + cleanup + cascade deletion = no finding (good pattern)

**LOC Impact:** ~5-20 lines per finding (add TTL index, extend cascade delete, add retention policy)
**Effort:** `small` (add TTL index) to `large` (implement full data lifecycle compliance)

### Phase 5: Backup & Recovery (agent-driven)

**Grep patterns (always run):**

1. **Backup scripts** — `backup|dump|export|mongodump|pg_dump|snapshot` in project-wide search
2. **Backup configuration** — `backup|cron|scheduled|periodic` in `docker-compose.yml`, CI/CD configs, and scripts
3. **Recovery scripts** — `restore|recover|import|mongorestore|pg_restore` in project-wide search
4. **Data export** — `EXPORT|COPY.*TO|parquet|csv.*export|dump` in `**/*.py`, `**/*.sql`
5. **Data directory mounts** — `volumes:` entries with data paths in `docker-compose.yml`

**Full mode (agent-driven):**

If `AGENT_MODE` is true, launch Plan agent with `subagent_type=Plan`:

> "Analyze backup and recovery readiness for this project's databases:
>
> **Findings:** [Insert backup/recovery grep results, volume mounts, export capabilities]
>
> **Analyze:**
> 1. **Per-Database Backup** — For each database discovered, is it backed up? How? What's the RPO (recovery point objective)?
> 2. **Backup Scheduling** — Are backups automated (cron, CI/CD)? Are they tested? Where are backups stored?
> 3. **Recovery Testing** — Has recovery ever been tested? Is there a runbook?
> 4. **Data Export** — Can users export their data (portability)? What formats?
> 5. **Disaster Recovery** — If the host dies, what's the recovery procedure? How long (RTO)?
>
> **Return:** Backup coverage matrix (database × backup method × frequency × tested? × RPO × RTO)"

**Skip agent if `--quick` mode.** Set `Agent: None (--quick)` in header.

**Severity mapping:**
- No backup mechanism for any database = CRITICAL
- Backups exist but never tested (recovery untested) = HIGH
- Database file on single disk with no backup = HIGH
- Database without automated backup schedule = HIGH
- No documented recovery procedure = MEDIUM
- Backups exist but RPO > 24 hours for active data = MEDIUM
- No data export capability for users = MEDIUM
- Comprehensive backup + tested recovery = no finding (good pattern)

**LOC Impact:** ~10-50 lines per finding (backup scripts, cron jobs, recovery docs)
**Effort:** `medium` (add backup script) to `large` (implement full backup/recovery pipeline)

**On timeout/failure:** Output raw backup/recovery grep results without agent analysis. Note `Agent: None (timeout)` in header.

### Phase 6: Migration Safety (agent-driven)

**Grep patterns (always run):**

1. **Schema migration tools** — `alembic|migrate|migration|flyway|dbmate|knex|prisma` in `**/*.py`, `**/*.rs`, `**/*.ts`, `**/*.js`, and dependency files
2. **Schema version tracking** — `schema_version|migration_version|__version__|revision` in `**/*.py`, `**/*.sql`
3. **ALTER TABLE/CREATE INDEX** — `ALTER TABLE|ADD COLUMN|DROP COLUMN|CREATE INDEX|DROP INDEX` in `**/*.py`, `**/*.rs`, `**/*.sql`
4. **Conditional schema creation** — `IF NOT EXISTS|IF EXISTS|try.*CREATE` in `**/*.py`, `**/*.rs`
5. **Index creation** — `create_index|ensure_index|createIndex|CREATE INDEX` in `**/*.py`, `**/*.sql`
6. **Rust schema definitions** — `CREATE.*IF NOT EXISTS|ALTER|migrate` in `**/*.rs`

**Full mode (agent-driven):**

If `AGENT_MODE` is true, launch Plan agent with `subagent_type=Plan`:

> "Analyze schema migration safety for this project:
>
> **Findings:** [Insert migration grep results, schema creation patterns, version tracking]
>
> **Analyze:**
> 1. **Migration Framework** — Is there a formal migration tool (Alembic, dbmate)? Or are schemas created ad-hoc in code?
> 2. **Schema Versioning** — How are schema changes tracked? Can you tell what version a database is at?
> 3. **Rollback Capability** — If a migration fails, can you roll back? Or is it one-way?
> 4. **Data Loss Risk** — Any `DROP COLUMN`, `ALTER TYPE`, or `TRUNCATE` that could lose data?
> 5. **Zero-Downtime** — Can schema changes deploy without stopping the application?
>
> **Return:** Migration safety matrix (database × has migration tool × versioned × rollback capable × zero-downtime)"

**Skip agent if `--quick` mode.** Set `Agent: None (--quick)` in header.

**Severity mapping:**
- No migration framework (ad-hoc schema changes in code) = HIGH
- Schema changes that could lose data without migration (DROP COLUMN, ALTER TYPE) = HIGH
- No schema versioning (can't determine current schema state) = HIGH
- No rollback capability on migration failure = MEDIUM
- Schema created with `IF NOT EXISTS` (safe creation but no evolution path) = MEDIUM
- Formal migration tool with versioning and rollback = no finding (good pattern)

**LOC Impact:** ~10-30 lines per finding (add migration framework, version tracking)
**Effort:** `medium` (add schema versioning) to `large` (implement migration framework)

**On timeout/failure:** Output raw migration grep results without agent analysis. Note `Agent: None (timeout)` in header.

---

## Step 6: Assemble Findings Document

Write to `work/MMDDYY_Database_Audit.md` following shared.md Output Format exactly:

### 6a. Header

```markdown
# Database Audit

**Date:** MMDDYY
**Prior:** work/MMDDYY_Database_Audit.md (or "None")
**Mode:** Full | Quick (--quick)
**Agent:** Plan | None (--quick) | None (timeout)
**Tools:** Grep (built-in), Read (file analysis) [list any additional tools used]
```

### 6b. Executive Summary

2-3 sentences covering:
- Total findings count and severity distribution
- Highest-impact finding (e.g., no backup, data retention gaps, SQL injection vector)
- Trend vs prior (if `COMPARE` is true)

### 6c. Metrics Dashboard

| Metric | Value | Prior | Trend |
|--------|-------|-------|-------|
| Total findings | N | — | — |
| Critical | N | — | — |
| High | N | — | — |
| Medium | N | — | — |
| Low | N | — | — |
| SQL tables scanned | N | — | — |
| Document collections scanned | N | — | — |
| Queries without LIMIT | N | — | — |
| String-formatted SQL | N | — | — |
| Collections without TTL | N | — | — |
| Backup mechanisms | N | — | — |
| Migration framework | Yes/No | — | — |

Prior and Trend columns populated only when `COMPARE` is true.

### 6d. Findings Tables

Group all findings by severity. Each table uses shared.md format:

```markdown
### Critical

| # | File | Line | Issue | Tool | LOC Impact | Effort |
|---|------|------|-------|------|------------|--------|
```

Repeat for High, Medium, Low/Informational.

Number findings sequentially with **DB-** prefix (DB-1, DB-2, DB-3... for Database findings).

### 6e. Auto-Fixed Section

**Not applicable for database audit** — schema and query changes require human review. If `--fix` was passed, note:
```markdown
## Auto-Fixed

N/A — Database audit is read-only. Schema and query changes require human review.
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

### 6h. Backup & Migration Matrix

Include the agent-generated matrices (or stub if `--quick`):
```markdown
## Backup Coverage Matrix

| Database | Backup Method | Frequency | Tested? | RPO | RTO | Status |
|----------|--------------|-----------|---------|-----|-----|--------|
| [database 1] | ? | ? | ? | ? | ? | ? |
| [database 2] | ? | ? | ? | ? | ? | ? |

## Migration Safety Matrix

| Database | Migration Tool | Versioned | Rollback | Zero-Downtime | Status |
|----------|---------------|-----------|----------|---------------|--------|
| [database 1] | ? | ? | ? | ? | ? |
| [database 2] | ? | ? | ? | ? | ? |
```

### 6i. Recommendations

Top 3 highest-impact recommendations with estimated effort. Prioritize:
1. CRITICAL findings first (data retention gaps, SQL injection, no backups)
2. HIGH findings (unbounded queries, incomplete cascade delete, no migration framework)
3. Quick wins with highest data-safety-to-effort ratio

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
git add work/MMDDYY_Database_Audit.md
git add docs/archive/audits/  # if priors archived
git add .claude/rules/tech-debt-patterns.md  # if updated

git commit -m "$(cat <<'EOF'
docs(audit): database audit — N findings (X critical, Y high)

Tools: Grep, Read
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

Before completing, verify all items from shared.md Execution Checklist are met. Key database-specific items:

- [ ] All 6 phases ran (or 4 in quick mode — skip Phases 5 and 6) — each phase produced findings or confirmed clean
- [ ] Phase 1: All database schemas compared for consistency across languages and data stores
- [ ] Phase 2: All SQL queries checked for LIMIT, parameterization, N+1 patterns
- [ ] Phase 3: Connection concurrency checked (platform-specific locking for embedded databases), connection lifecycle verified
- [ ] Phase 4: All data stores checked for TTL/retention policy, cascade delete completeness verified
- [ ] Phase 5: Backup mechanisms inventoried for all databases (agent in full mode)
- [ ] Phase 6: Migration framework presence and rollback capability assessed (agent in full mode)
- [ ] Severity mapping applied per phase-specific rules above
- [ ] Database-specific metrics populated (tables/collections scanned, queries without LIMIT, string-formatted SQL, TTL coverage, backup mechanisms)
- [ ] Findings numbered sequentially with DB- prefix (DB-1, DB-2, ...)
- [ ] `--fix` noted as N/A (read-only audit)
- [ ] Delta filters to `.py`, `.rs`, `.toml`, `.sql` files
- [ ] Backup Coverage Matrix and Migration Safety Matrix included (or stubbed in quick mode)
- [ ] Database-specific limitations noted (e.g., embedded DB file locking, missing statement timeout, platform differences)
- [ ] Data retention and compliance gaps cross-referenced with known tech debt

---

## See Also

- `/audit-docker` — Docker infrastructure audit (container security, resource limits, network isolation)
- `/audit-resilience` — Error handling & resilience audit (error swallowing, timeouts, retry logic)
- `/audit-observability` — Logging & monitoring audit (structlog, alerts, dashboards)
- `/audit-test-quality` — Test quality audit (flaky tests, isolation, mock hygiene, coverage gaps)
- `/audit-python` — Python code quality audit (radon, vulture, ruff)
- `/audit-rust` — Rust audit (clippy, unwrap audit, PyO3 bindings)
- `/audit-security` — OWASP security audit (SQL injection overlaps with Phase 2)
