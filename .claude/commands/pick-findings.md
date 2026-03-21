# Pick Findings Skill

Target excluded items from a previous `/sprint-from-findings` manifest. Lets you circle back to recommendations that were initially skipped.

## Usage

```
/pick-findings
/pick-findings 030726
/pick-findings v2.1a_fixes 030726
```

**Arguments:**
- **Date prefix** (optional): `MMDDYY` format. If omitted, finds the most recent manifest.
- **Folder shorthand** (optional): `v{version}_{name}` pattern for the new sprint. If omitted, auto-generates.

---

## Step 1: Find and Load Manifest

### 1a: Locate Manifest File

Search for findings manifests in `work/`:

```
Glob("*_findings_manifest.md", path="work/")
```

- If a date prefix was provided, match `{MMDDYY}_findings_manifest.md`
- If no date, use the **most recently modified** manifest
- If zero manifests found, tell the user to run `/sprint-from-findings` first

### 1b: Read and Parse Manifest

Read the manifest file. Parse the `## All Recommendations` table, extracting each row's:
- `#` (original row number)
- `Recommendation`
- `Source(s)`
- `Severity`
- `Effort`
- `Finding`
- `Status`

### 1c: Filter to Actionable Items

Separate items by status:
- **Available:** `EXCLUDED` items (these are what the user can pick from)
- **Already done:** `SELECTED → ...` or `DONE` items (show for context)
- **Declined:** `WONTFIX` items (show for context)

**If zero EXCLUDED items remain:** Report that all findings from this manifest have been addressed or declined. Suggest running new audits with `/eod-docs`.

---

## Step 2: Present Available Findings

**MANDATORY FORMAT:** Use a markdown table. Same format as `/sprint-from-findings` Step 4.

```
## Available Findings from {MMDDYY} Manifest

**Manifest:** {filename}
**Total findings:** {total}
**Already addressed:** {done_count} ({done_list})
**Available to pick:** {excluded_count}

### Previously Excluded Recommendations

| # | Recommendation | Source(s) | Severity | Effort | Finding |
|---|---------------|-----------|----------|--------|---------|
| 3 | ... | ... | MEDIUM | medium | ... |
| 4 | ... | ... | LOW | small | ... |
| 7 | ... | ... | INFO | — | ... |

Enter numbers to **include** (e.g., "3, 4") or say **all** to include everything.
Or say **wontfix 7** to permanently decline items.
```

**Note:** Use the **original row numbers** from the manifest so the user can cross-reference.

**Wait for user response.**

### Handle User Input

- **Numbers (e.g., "3, 4")** — Select those items, continue to Step 3
- **"all"** — Select all EXCLUDED items, continue to Step 3
- **"wontfix N,M"** — Mark those items as `WONTFIX` in the manifest, re-present remaining EXCLUDED items
- **"cancel"** — Exit cleanly

---

## Step 3: Update Manifest

After user selection, update the manifest file:

1. Change selected items from `EXCLUDED` → `SELECTED → {sprint_folder}`
2. Change any `wontfix` items from `EXCLUDED` → `WONTFIX`
3. Write the updated manifest back to the same file path

This keeps the manifest as the single source of truth.

---

## Step 4: Delegate to /sprint

Build the focus topic from selected recommendations and invoke `/sprint`, using the same format as `/sprint-from-findings` Step 5:

```
The user has selected the following recommendations from a previous findings manifest ({MMDDYY}).
Each recommendation comes from a specific audit finding with severity and effort estimates.
Use these as the primary input for session decomposition and sprint planning.

Source manifest: {manifest_filename}
Original source documents:
  - {doc1} ({type})
  - {doc2} ({type})
  ...

Selected recommendations (ranked by priority):
  [1] {recommendation text} (Source: {doc}, Severity: {sev}, Effort: {effort}, Finding: {id})
  [2] ...

Organize these into logical sessions. Related recommendations should be grouped
into single sessions when they touch the same files. Every recommendation must
appear in the final plan -- either as its own session, grouped with related items,
or explicitly deferred with rationale.

Effort estimates from audits:
  - trivial: < 10 LOC, single file
  - small: 10-30 LOC, 1-2 files
  - medium: 30-60 LOC, 2-4 files
  - large: 60+ LOC, 4+ files
```

Pass the folder shorthand to `/sprint` if provided.

---

## Error Handling

| Error | Action |
|-------|--------|
| No manifests found | Tell user to run `/sprint-from-findings` first |
| Manifest has no EXCLUDED items | Report all items addressed, suggest new audits |
| User selects zero items | Re-prompt: "Need at least one recommendation" |
| Manifest format unreadable | Surface parse error, show raw content |
| /sprint fails | Surface error, don't retry |

---

## Example Execution

```
User: /pick-findings 030726

Claude:
## Available Findings from 030726 Manifest

**Manifest:** 030726_findings_manifest.md
**Total findings:** 14
**Already addressed:** 8 (items 1-6, 9, 11 → v2.0e_sprint_from_findings)
**Available to pick:** 5
**Declined:** 1 (item 14 → WONTFIX)

### Previously Excluded Recommendations

| # | Recommendation | Source(s) | Severity | Effort | Finding |
|---|---------------|-----------|----------|--------|---------|
| 7 | Add structured logging to auth flow | Best Practices | MEDIUM | small | BP-3 |
| 8 | Frontend error boundary for canvas | Linting | MEDIUM | medium | L-5 |
| 10 | Rate limit WebSocket reconnects | Best Practices | LOW | small | BP-7 |
| 12 | Document retention policy in README | TechnicalDebt | LOW | trivial | TD-2 |
| 13 | Monitor hrv.rs LOC growth | Bottleneck | INFO | — | B-1 |

Enter numbers to include, say **all**, or **wontfix N** to decline.

User: 7, 8, 10

Claude: [Updates manifest: items 7, 8, 10 → SELECTED → v2.1a_fixes]
Launching `/sprint v2.1a_fixes` with 3 recommendations...

[/sprint takes over]
```

---

## See Also

- `/sprint-from-findings` -- Creates the manifests this skill reads
- `/sprint` -- Sprint planning (this skill delegates to it)
- `/eod-docs` -- Generates the audit documents that feed `/sprint-from-findings`
