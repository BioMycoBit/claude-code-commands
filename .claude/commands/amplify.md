# Amplify Skill

Iterative content amplification through progressive refinement passes.

## Usage

```
/amplify <iterations> <file_path>
```

**Examples:**
- `/amplify 4 work/guide.md` → Creates guide_v1.md through guide_v4.md
- `/amplify 2 work/notes.md` → Creates notes_v1.md and notes_v2.md
- `/amplify 3 work/doc_v2.md` → Detects v2, creates doc_v3.md through doc_v5.md

## Arguments

- `$ARGUMENTS` - Expected format: `<iterations> <file_path>`
  - iterations: 1-4 (capped at 4)
  - file_path: Path to source file

---

## Step 1: Parse Arguments

Extract from `$ARGUMENTS`:

1. **Iterations**: First argument (number 1-4, cap at 4)
2. **File path**: Remaining argument(s) joined

**Validation:**
- If iterations > 4, cap at 4 and inform user
- If iterations < 1, default to 4
- If file doesn't exist, error and stop

---

## Step 2: Detect Version Number

Check if the source file already has a version suffix:

| Pattern | Detection |
|---------|-----------|
| `guide_v2.md` | Existing v2, start at v3 |
| `guide_v12.md` | Existing v12, start at v13 |
| `guide.md` | No version, start at v1 |

**Regex:** `_v(\d+)\.md$`

Calculate:
- `start_version` = detected version + 1 (or 1 if none)
- `end_version` = start_version + iterations - 1

---

## Step 3: Read Source Content

Read the source file completely. This is the input for Pass 1.

---

## Step 4: Execute Passes

Run passes sequentially. Each pass reads the previous output and writes a new version.

### Pass 1: Initial

**If source has no version number:**
- Copy source to `<name>_v1.md` as-is (this IS the v1)
- OR if source already has content to amplify, proceed to Pass 2 logic

**If source already has a version:**
- Read it, prepare for amplification in next pass

### Pass 2: Print-Worthy Quality

**Prompt to apply:**
```
Elevate this content to publication quality. Apply these enhancements:

CLARITY: Replace vague statements with specific claims. If something "improves performance," state by how much or why.

FLOW: Ensure each paragraph leads logically to the next. Add transition sentences where ideas jump.

EXAMPLES: Where concepts are abstract, add a concrete example or analogy that makes it tangible.

GAPS: Read as a newcomer would. Fill any "wait, how does that work?" moments.

FORMATTING: Use consistent heading hierarchy, bullet style, and emphasis patterns throughout.

Write the complete enhanced version.
```

**Output:** `<name>_v<N>.md`

### Pass 3: ELI5 + High-Level Depth

**Prompt to apply:**
```
Add depth layers to the most important concepts. Select 1-2 core ideas that would benefit from both simplified and advanced treatment.

For each selected concept, add:

**ELI5 Box:** A simple analogy using everyday objects or experiences. No jargon. A 10-year-old should nod along.

**Deep Dive Box:** Technical depth for practitioners. Include implementation considerations, edge cases, or architectural implications.

Format as clearly marked callout sections (use blockquotes or headers like "### ELI5: [Concept]").

Write the complete version with these depth layers integrated naturally into the document flow.
```

**Output:** `<name>_v<N>.md`

### Pass 4: Tighten + Keywords Index

**Prompt to apply:**
```
Final editorial pass. Target: reduce word count by 15-20% while preserving all substance.

TIGHTEN:
- Cut filler phrases ("it should be noted that," "in order to," "the fact that")
- Replace weak verbs with strong ones ("utilize" → "use," "facilitate" → "enable")
- Eliminate redundant adjectives and hedging language
- Convert passive voice to active where it improves clarity

CONSISTENCY:
- Verify heading levels follow logical hierarchy
- Ensure bullet/number style is uniform
- Check that emphasis (bold/italic) follows a consistent pattern

KEYWORDS SECTION:
Add a "Keywords" section at the document end with 15-25 key terms.
Format as a 3-4 column table. Terms only, no definitions.
Order alphabetically or by conceptual grouping.

Write the complete final version.
```

**Output:** `<name>_v<N>.md`

---

## Step 5: Write Each Version

For each pass:

1. Apply the pass prompt to the previous version's content
2. Write the result to `<name>_v<N>.md` in the same directory as source
3. Log what was done

**File naming:**
- Source: `work/guide.md` or `work/guide_v2.md`
- Base name: `guide` (strip `_v\d+` suffix)
- Output: `work/guide_v<N>.md`

---

## Step 6: Summary Report

After all passes complete, output:

```
## Amplify Complete

**Source:** <file_path>
**Iterations:** <N>
**Versions created:** v<start> through v<end>

### Final Output
- work/<name>_v4.md (polished + keywords)

### Cleaned Up
- work/<name>.md (source)
- work/<name>_v1.md, _v2.md, _v3.md (intermediates)

### Pass Summary
| Pass | Focus | Key Changes |
|------|-------|-------------|
| 1 | Initial | [baseline] |
| 2 | Print-worthy | [what was enhanced] |
| 3 | Depth layers | [concepts given ELI5/deep dive] |
| 4 | Polish + index | [tightening done, keyword count] |
```

---

## Step 7: Cleanup Intermediate Files

After all passes complete and v4 is written:

1. Delete the source file (if it has no version suffix)
2. Delete intermediate versions: v1, v2, v3
3. Keep only the final v4

```bash
rm work/<name>.md        # original source
rm work/<name>_v1.md     # intermediate
rm work/<name>_v2.md     # intermediate
rm work/<name>_v3.md     # intermediate
# keep work/<name>_v4.md  # final polished version
```

**Exception:** If source already had a version (e.g., `_v2.md`), only delete files created during this run.

---

## Execution Notes

- **No pauses**: Run all passes consecutively
- **Cap at 4**: If user requests >4, cap and inform
- **Auto-cleanup**: Delete source and intermediates, keep only v4
- **Same directory**: Output files go in same directory as source
- **Full content**: Each version is complete, not a diff

---

## Error Handling

| Error | Action |
|-------|--------|
| File not found | Stop, inform user |
| Invalid iterations | Default to 4, inform user |
| Write permission denied | Stop, inform user |
| Path is directory | Stop, inform user |

---

## Example Execution

```
User: /amplify 4 work/linux_tips.md

1. Parse: iterations=4, path=work/linux_tips.md
2. Detect: no version suffix, start at v1
3. Read: load linux_tips.md content
4. Pass 1: write work/linux_tips_v1.md (copy)
5. Pass 2: enhance → write work/linux_tips_v2.md
6. Pass 3: add ELI5/deep dive → write work/linux_tips_v3.md
7. Pass 4: tighten + keywords → write work/linux_tips_v4.md
8. Cleanup: delete linux_tips.md, _v1, _v2, _v3
9. Report: summary (only _v4 remains)
```
