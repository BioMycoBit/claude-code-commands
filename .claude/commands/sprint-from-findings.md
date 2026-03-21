# Sprint from Findings Skill

Scan today's audit/overview/walkthrough documents in `work/`, extract top recommendations, and build a sprint from them.

## Usage

```
/sprint-from-findings
/sprint-from-findings v1.9y_recommendations
/sprint-from-findings 030426
/sprint-from-findings v1.9y_recommendations 030426
```

**Arguments:**
- **Folder shorthand** (optional): `v{version}_{name}` pattern. If omitted, auto-generates from date + `_recommendations` (e.g., `v1.9y_recommendations`).
- **Date prefix** (optional): `MMDDYY` format. If omitted, uses today's date.

---

## Step 1: Determine Date and Find Documents

### 1a: Parse Arguments

- If a `MMDDYY` date is provided, use it
- Otherwise, derive today's date in `MMDDYY` format

### 1b: Find Today's Documents

Search for all files in `work/` matching the date prefix:

```
Glob("{MMDDYY}_*.md", path="work/")
```

**Expected file types:**
- `{date}_*_Audit.md` (e.g., `030426_Resilience_Audit.md`)
- `{date}_*_Overview.md` (e.g., `030426_Project_Overview.md`)
- `{date}_*_Walkthrough.md` (e.g., `030426_System_Walkthrough.md`)

**If zero files found:** Ask the user which date prefix to use, or if they want to specify file paths directly.

### EOD Day Lookup

Use this table to tag each document with its EOD sprint day in the discovery table:

| Output File Pattern | EOD Day |
|---------------------|---------|
| DeadCode_Audit | Day 1: Code Hygiene |
| TechnicalDebt_Audit | Day 1: Code Hygiene |
| Best_Practices_Audit | Day 1: Code Hygiene |
| Linting_Audit | Day 1: Code Hygiene |
| CodeSweep | Day 1: Code Hygiene |
| Modules_Audit | Day 1: Code Hygiene |
| Python_Audit | Day 2: Python & Testing |
| TestQuality_Audit | Day 2: Python & Testing |
| Regression_Prevention | Day 2: Python & Testing |
| Project_Overview | Day 2: Python & Testing |
| TypeScript_Audit | Day 3: TS & Frontend |
| Component_Health | Day 3: TS & Frontend |
| Browser_Compatibility_Audit | Day 3: TS & Frontend |
| Accessibility_WCAG_Audit | Day 3: TS & Frontend |
| Rust_Audit | Day 4: Rust & Performance |
| Performance_Audit | Day 4: Rust & Performance |
| Bottleneck_Audit | Day 4: Rust & Performance |
| System_Walkthrough | Day 4: Rust & Performance |
| Docker_Audit | Day 5: Infrastructure |
| Observability_Audit | Day 5: Infrastructure |
| Pipeline_Audit | Day 5: Infrastructure |
| Database_Audit | Day 6: Data & Architecture |
| Resilience_Audit | Day 6: Data & Architecture |
| API_Surface | Day 6: Data & Architecture |
| Dependency_Audit | Day 6: Data & Architecture |
| Security_Audit | Day 7: Security & Compliance |
| Dependency_Vuln_Audit | Day 7: Security & Compliance |
| License_Audit | Day 7: Security & Compliance |
| Privacy_GDPR_Audit | Day 7: Security & Compliance |
| Secrets_Audit | Day 7: Security & Compliance |
| SBOM_Audit | Day 7: Security & Compliance |
| Types_Audit | Day 2 or Day 3 (check content) |

If a document doesn't match any pattern, use `—` for the EOD Day column.

### 1c: Report Discovery and Get Approval

Present what was found and **wait for user confirmation** before reading any files:

```
## Found {count} documents for {MMDDYY}

| # | Document | EOD Day | Type | Size |
|---|----------|---------|------|------|
| 1 | {filename} | Day 1: Code Hygiene | Audit | {line_count} lines |
| 2 | {filename} | Day 2: Python & Testing | Overview | {line_count} lines |
| 3 | {filename} | Day 4: Rust & Performance | Walkthrough | {line_count} lines |

**Total:** {total_lines} lines across {count} documents

Proceed with extraction? (**yes** / **no** / exclude specific numbers e.g. "skip 3,5")
```

**Wait for user response.** Do NOT read document contents until approved.

- **"yes"** / **"go"** / **"proceed"** — Continue to Step 2 with all documents
- **"skip N,M"** / **"exclude N,M"** — Remove those documents from the list, continue
- **"no"** / **"cancel"** — Exit cleanly

---

## Step 2: Extract Recommendations

Read the **Recommendations** section from each document. Every audit/overview/walkthrough ends with a `## Recommendations` section containing numbered items.

**For each document, read in parallel**, then extract:
1. The recommendation text
2. The finding ID (if referenced, e.g., `E-1`, `PL-5`)
3. The severity (CRITICAL, HIGH, MEDIUM, LOW — from the finding if referenced)
4. The effort estimate (trivial, small, medium, large)
5. The source document name

### Recommendation Format

Build a unified table:

```markdown
| # | Recommendation | Source | Severity | Effort | Finding |
|---|---------------|--------|----------|--------|---------|
| 1 | Database storage status notification | Resilience Audit | CRITICAL | medium | E-1 |
| 2 | Wire frontend error handlers | Resilience Audit | CRITICAL | medium | E-2+E-3 |
| ...
```

---

## Step 3: Rank and Deduplicate

### 3a: Deduplicate

Multiple documents may recommend the same fix (e.g., Pipeline Audit and Resilience Audit both flag missing counters). Merge duplicates, noting all source documents.

### 3b: Rank and Tag File Overlap

Sort by priority:
1. **CRITICAL** findings first
2. **HIGH** findings
3. **MEDIUM** findings with small effort (high ROI)
4. **MEDIUM** findings with medium+ effort
5. **LOW** findings
6. Monitoring/informational items last

**File overlap tagging:** For each recommendation, note the primary files it would touch.
When multiple recommendations (even from different audits) share the same files, tag them
as a cluster. This drives session grouping in Step 5 — sessions are organized by file
overlap, not audit source, to minimize context switching.

### 3c: Also Check Tech Debt

Read `.claude/rules/tech-debt-patterns.md` and cross-reference. If any tech debt item matches a recommendation, note it (avoids duplicating known debt).

---

## Step 4: Present Findings for User Selection

**MANDATORY FORMAT — TABLES ONLY:** The ENTIRE Step 4 output MUST use markdown tables. No bullet lists, numbered lists, or prose paragraphs anywhere in this step. Every piece of structured data — recommendations, file clusters, tech-debt cross-references, defaults — goes in a table. Sort recommendation rows by severity: CRITICAL → HIGH → MEDIUM → LOW → INFO. Within the same severity, sort by effort (smallest first = highest ROI).

```
## Recommendations Extracted

| Stat | Value |
|------|-------|
| Documents scanned | {count} |
| Total recommendations | {count} |
| After dedup | {count} |

### Ranked Recommendations (sorted CRITICAL → LOW)

| # | Recommendation | Source(s) | Severity | Effort | Include? |
|---|---------------|-----------|----------|--------|----------|
| 1 | ... | ... | CRITICAL | medium | Y |
| 2 | ... | ... | CRITICAL | large | Y |
| 3 | ... | ... | HIGH | small | Y |
| 4 | ... | ... | HIGH | medium | Y |
| 5 | ... | ... | MEDIUM | small | Y |
| 6 | ... | ... | MEDIUM | medium | Y |
| 7 | ... | ... | LOW | small | — |
| 8 | ... | ... | INFO | — | — |

### File Overlap Clusters

| Cluster | Items | Primary Files |
|---------|-------|---------------|
| Signal processing | #1, #5, #8 | lib/signals/src/ |
| Connection manager | #3, #4 | connection_manager.py |
| Test fixes | #2, #6 | Python tests + TS schemas |

### Tech-Debt Cross-References

| Item | Matching Tech-Debt Entry |
|------|--------------------------|
| #2 | "Test Coverage Below Industry Standard" |
| #4 | (none — new concern) |

### Defaults

| Rule | Items |
|------|-------|
| Auto-included (CRITICAL + HIGH) | #1–#4 |
| Auto-included (MEDIUM, sessions <= 7) | #5–#6 |
| Excluded (LOW / INFO) | #7–#8 |

Enter numbers to **exclude** (e.g., "exclude 5, 8") or say **all** to include everything.
Or say **only 1,2,3** to select specific items.
```

**NEVER** skip a table. Even if there is only 1 recommendation, use a table. Even if there are 0 tech-debt cross-references, show the table with a "(none)" row.

**Wait for user response before proceeding.**

---

## Step 4b: Write Findings Manifest

After the user selects which recommendations to include, **write a manifest file** to `work/` before delegating to `/sprint`. This manifest is the persistent record of all findings and their disposition.

**Filename:** `work/{MMDDYY}_findings_manifest.md`

**Format:**

```markdown
# Findings Manifest — {MMDDYY}

**Generated:** {ISO date}
**Source documents:** {count}
**Total recommendations:** {count} (after dedup)

## All Recommendations

| # | Recommendation | Source(s) | Severity | Effort | Finding | Status |
|---|---------------|-----------|----------|--------|---------|--------|
| 1 | ... | ... | CRITICAL | medium | E-1 | SELECTED → {sprint_folder} |
| 2 | ... | ... | HIGH | small | PL-5 | SELECTED → {sprint_folder} |
| 3 | ... | ... | MEDIUM | medium | ... | EXCLUDED |
| 4 | ... | ... | LOW | small | ... | EXCLUDED |

## Source Documents

- {filename1} ({type})
- {filename2} ({type})
```

**Status values:**
- `SELECTED → {sprint_folder}` — included in the sprint being created
- `EXCLUDED` — user chose not to include it (available for `/pick-findings` later)
- `DONE` — completed in a prior sprint (set by `/pick-findings` when re-selecting from manifest)
- `WONTFIX` — explicitly declined (set manually or via `/pick-findings`)

**IMPORTANT:** Always write the manifest. Even if the user selects "all", write it with all items as `SELECTED`. This creates the audit trail.

---

## Step 5: Delegate to /sprint

Build the focus topic from selected recommendations and invoke `/sprint`:

```
The user has selected the following recommendations from today's audits ({MMDDYY}).
Each recommendation comes from a specific audit finding with severity and effort estimates.
Use these as the primary input for session decomposition and sprint planning.

Source documents:
  - {doc1} ({type})
  - {doc2} ({type})
  ...

Selected recommendations (ranked by priority):
  [1] {recommendation text} (Source: {doc}, Severity: {sev}, Effort: {effort}, Finding: {id})
  [2] ...
  [3] ...

Organize these into logical sessions. **Group by file overlap, not by audit source.**
Recommendations from different audits that touch the same files belong in the same
session — this reduces context switching and avoids re-reading the same code in
multiple sessions. Every recommendation must appear in the final plan -- either as
its own session, grouped with related items, or explicitly deferred with rationale.

Effort estimates from audits:
  - trivial: < 10 LOC, single file
  - small: 10-30 LOC, 1-2 files
  - medium: 30-60 LOC, 2-4 files
  - large: 60+ LOC, 4+ files
```

Pass the folder shorthand to `/sprint` if provided (e.g., `/sprint v1.9y_recommendations`).

---

## Error Handling

| Error | Action |
|-------|--------|
| No documents for date | Ask user for correct date or file paths |
| Document has no Recommendations section | Skip it, note in summary |
| Zero recommendations total | Report finding, suggest running audits first |
| User excludes all items | Re-prompt: "Need at least one recommendation" |
| /sprint fails | Surface error, don't retry |

---

## Example Execution

```
User: /sprint-from-findings v1.9y_recommendations

Claude:
## Found 5 documents for 030426

| # | Document | EOD Day | Type |
|---|----------|---------|------|
| 1 | 030426_Pipeline_Audit.md | Day 5: Infrastructure | Audit |
| 2 | 030426_Resilience_Audit.md | Day 6: Data & Architecture | Audit |
| 3 | 030426_Bottleneck_Audit.md | Day 4: Rust & Performance | Audit |
| 4 | 030426_System_Walkthrough.md | Day 4: Rust & Performance | Walkthrough |
| 5 | 030426_Project_Overview.md | Day 2: Python & Testing | Overview |

[Reads Recommendations sections from all 5 documents...]

## Recommendations Extracted

**Documents scanned:** 5
**Total recommendations:** 18
**After dedup:** 14

### Ranked Recommendations

| # | Recommendation | Source(s) | Severity | Effort |
|---|---------------|-----------|----------|--------|
| 1 | Database storage status notification | Resilience | CRITICAL | medium |
| 2 | Wire frontend error handlers | Resilience | CRITICAL | medium |
| 3 | Database query timeout guard | Resilience | HIGH | small |
| 4 | Add pipeline stage counters | Pipeline | HIGH | medium |
| 5 | Port adaptive tick to all device types | Pipeline | MEDIUM | small |
| 6 | Handle Lagged receivers in relay | Pipeline | MEDIUM | small |
| 7 | Monitor hrv.rs growth | Bottleneck | INFO | — |
| ...

Default: Items 1-6 included (CRITICAL + HIGH + high-ROI MEDIUM).

Enter numbers to exclude, say **all**, or **only 1,2,3**.

User: all

Claude: Launching `/sprint v1.9y_recommendations` with 14 recommendations...

[/sprint takes over with full recommendation list as focus]
```

---

## See Also

- `/sprint` -- Sprint planning (this skill delegates to it)
- `/eod-docs` -- Generates the audit documents this skill reads
- `/pick-findings` -- Target excluded items from a previous manifest
- `/random-sprint` -- Similar pattern: collect items, delegate to /sprint
