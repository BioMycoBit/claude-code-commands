# /audit-accessibility

WCAG 2.1 Level AA accessibility audit: images without alt text, non-semantic clickable elements, ARIA label coverage, role attribute usage, keyboard handler gaps, focus style verification, and axe-core runtime DOM testing.

**Target:** `src/**/*.ts`, `src/**/*.tsx`, `src/**/*.css`

---

## Step 1: Load Shared Infrastructure

1. Read `.claude/commands/audit/shared.md`
2. All output format, severity levels, delta tracking, archival, tech debt integration, commit format, sprint chain, error handling, and completion report rules come from shared.md — do NOT deviate

---

## Step 2: Parse Flags

Parse `$ARGUMENTS` per shared Flag Parsing rules:

1. Strip flags from arguments, set booleans:
   - `AGENT_MODE` — `false` always (this audit is grep-only, no agent analysis)
   - `COMPARE` — `true` if `--since last`
   - `SPRINT_CHAIN` — `true` if `--sprint`
   - `AUTO_FIX` — `false` always (`--fix` is a no-op — accessibility fixes require manual verification and screen reader testing)
   - `SPRINT_FOLDER` — extracted if `--sprint v*_*` pattern provided
2. Report mode to user:
   ```
   Mode: Full (grep-only, no agent)
   Flags: [active flags, or "none"]
   Note: --quick has no effect (no agent to skip). --fix is ignored (manual verification required).
   ```

---

## Step 3: Find Prior Audit

```bash
ls work/*_Accessibility_WCAG_Audit.md 2>/dev/null | sort -r | head -1
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

Run all phases, collecting findings. Each finding must include: File, Line, Issue, Tool, LOC Impact, Effort.

**Tool rule:** Use the built-in **Grep tool** (NOT bash grep/rg) for ALL pattern scans per shared.md Tool Rules. Issue multiple Grep calls in a **single parallel message** where patterns are independent.

### Accessibility Delta Logic

In delta mode (`--since last`):
1. Load PRIOR_BASELINE for this audit (a11y findings by WCAG criterion)
2. CARRY FORWARD: Findings where source `.tsx`/`.jsx`/`.css` file NOT in CHANGED_FILES
3. RE-SCAN: ARIA/a11y grep patterns ONLY on changed frontend files
4. VERIFY: Spot-check 3-5 carried a11y findings still present
5. MERGE + MARK: NEW | CARRIED | CHANGED | RESOLVED

If MODE == "full": Run all patterns on entire codebase.

### Phase 1: Images Without Alt Text (WCAG 1.1.1)

**Grep patterns:**

| Check | Pattern | Files | WCAG |
|-------|---------|-------|------|
| Images without alt | `<img` without `alt=` | `*.tsx`, `*.jsx` | 1.1.1 |

**Analysis:**
1. Grep `<img` in `.tsx`/`.jsx`
2. Filter those missing `alt=` attribute
3. Decorative images should use `alt=""` (empty alt), not missing alt

**Severity mapping:**
- Content image without any `alt` attribute = HIGH (WCAG 1.1.1 failure)
- Decorative image without `alt=""` = MEDIUM
- All images have appropriate alt = no finding

**LOC Impact:** ~1 line per finding
**Effort:** `trivial` (add alt attribute)

### Phase 2: Non-Semantic Clickable Elements (WCAG 4.1.2)

**Grep patterns (parallel):**

| Check | Pattern | Files | WCAG |
|-------|---------|-------|------|
| Clickable divs | `<div.*onClick` | `*.tsx`, `*.jsx` | 4.1.2 |
| Clickable spans | `<span.*onClick` | `*.tsx`, `*.jsx` | 4.1.2 |

**Analysis:**
1. Grep `<div.*onClick` and `<span.*onClick`
2. These should be `<button>` or `<a>` for proper semantics
3. Non-semantic clickables lack keyboard support and screen reader announcements

**Severity mapping:**
- Clickable div/span on primary interaction (navigation, form submit) = HIGH
- Clickable div/span on secondary interaction = MEDIUM
- All interactive elements use semantic HTML = no finding

**LOC Impact:** ~3-5 lines per finding (replace element + add keyboard handler)
**Effort:** `small` (change tag + add keyboard support)

### Phase 3: ARIA Label Coverage (WCAG 4.1.2)

**Grep patterns (parallel):**

| Check | Pattern | Files | WCAG |
|-------|---------|-------|------|
| ARIA labels | `aria-label` | `*.tsx`, `*.jsx` | 4.1.2 |
| ARIA labelledby | `aria-labelledby` | `*.tsx`, `*.jsx` | 4.1.2 |
| Role attributes | `role=` | `*.tsx`, `*.jsx` | 4.1.2 |

**Analysis:**
1. Count total `aria-label` and `aria-labelledby` usage
2. Grep `role=` assignments
3. Cross-reference interactive components with ARIA coverage
4. Custom widgets (modals, dropdowns, tabs) should have appropriate ARIA roles

**Severity mapping:**
- Custom widget without role or aria-label = HIGH
- Interactive component missing aria-label = MEDIUM
- Good ARIA coverage = INFO (positive finding)

**LOC Impact:** ~1-2 lines per finding
**Effort:** `trivial` to `small` (add aria-label or role)

### Phase 4: Keyboard Navigation (WCAG 2.1.1)

**Grep patterns (parallel):**

| Check | Pattern | Files | WCAG |
|-------|---------|-------|------|
| onKeyDown handlers | `onKeyDown` | `*.tsx`, `*.jsx` | 2.1.1 |
| onKeyUp handlers | `onKeyUp` | `*.tsx`, `*.jsx` | 2.1.1 |
| onKeyPress handlers | `onKeyPress` | `*.tsx`, `*.jsx` | 2.1.1 |

**Analysis:**
1. Grep keyboard handlers (`onKeyDown`, `onKeyUp`, `onKeyPress`)
2. Cross-reference with onClick handlers — every onClick should have a keyboard equivalent
3. Check for Enter/Space key handling on custom interactive elements

**Severity mapping:**
- Interactive element with onClick but no keyboard handler = HIGH
- Custom widget without arrow key navigation = MEDIUM
- Keyboard handlers present and comprehensive = INFO

**LOC Impact:** ~5-10 lines per finding (add keyboard event handler)
**Effort:** `small` to `medium` (add keyboard interaction)

### Phase 5: Focus Styles (WCAG 2.4.7)

**Grep patterns (parallel):**

| Check | Pattern | Files | WCAG |
|-------|---------|-------|------|
| Focus pseudo-class | `:focus` | `*.css`, `*.tsx` | 2.4.7 |
| Focus-visible | `:focus-visible` | `*.css`, `*.tsx` | 2.4.7 |
| Outline none | `outline:\s*none\|outline:\s*0` | `*.css`, `*.tsx` | 2.4.7 |

**Analysis:**
1. Grep focus styles (`:focus`, `:focus-visible`) in CSS
2. Flag `outline: none` or `outline: 0` without replacement focus indicator
3. Interactive elements must have visible focus indicators

**Severity mapping:**
- `outline: none` without alternative focus indicator = HIGH (WCAG 2.4.7 failure)
- No focus styles defined for interactive components = MEDIUM
- Focus-visible used appropriately = INFO

**LOC Impact:** ~2-5 lines per finding
**Effort:** `small` (add focus-visible styles)

### Phase 6: axe-core Runtime Testing (WCAG 2.x)

Runtime WCAG compliance testing against the rendered DOM using axe-core via Playwright. Complements the static grep checks in Phases 1-5 by catching dynamic content, CSS-generated content, and runtime state issues that static analysis misses.

**Prerequisites:**

axe-core runs via `@axe-core/playwright` in Playwright test context. Check installation:

```bash
cd <frontend-dir> && npm ls @axe-core/playwright 2>&1
```

If not installed:
```
Tool not installed: @axe-core/playwright
Install: cd <frontend-dir> && npm install --save-dev @axe-core/playwright
```
Note in output under Tools and skip Phase 6 entirely.

**Requires running application.** axe-core tests rendered DOM — the app must be served (dev server or production build). If the app is not running, note it and skip:
```
Application not running. Start with: cd <frontend-dir> && npm run dev
Phase 6 skipped — axe-core requires rendered DOM.
```

**6a. Page-Level Scan:**

Run axe-core against key application routes:

```javascript
// Playwright test pattern (reference for audit execution)
const { test, expect } = require('@playwright/test');
const AxeBuilder = require('@axe-core/playwright').default;

test('accessibility audit - main pages', async ({ page }) => {
  const pages = ['/', '/session', '/settings', '/devices'];

  for (const route of pages) {
    await page.goto(route);
    const results = await new AxeBuilder({ page })
      .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa'])
      .analyze();

    // results.violations[] — each with .id, .impact, .description, .nodes[]
  }
});
```

Execute via Playwright:

```bash
cd <frontend-dir> && npx playwright test --grep "accessibility" --reporter=json 2>&1
```

If no existing accessibility test exists, run axe-core inline via Playwright's `evaluate`:

```bash
cd <frontend-dir> && npx playwright test tests/a11y/ --reporter=json 2>&1
```

**6b. Parse axe-core Results:**

For each violation:
- `id` — axe rule ID (e.g., `color-contrast`, `image-alt`, `button-name`)
- `impact` — `critical`, `serious`, `moderate`, `minor`
- `description` — human-readable description
- `nodes[]` — DOM elements with `.html` (snippet) and `.target` (CSS selector)
- `tags[]` — WCAG criteria (e.g., `wcag2a`, `wcag412`)

**Severity mapping (axe impact → audit severity):**

| axe-core Impact | Audit Severity |
|-----------------|----------------|
| critical | CRITICAL |
| serious | HIGH |
| moderate | MEDIUM |
| minor | LOW |

**6c. Deduplicate with Static Findings:**

Cross-reference axe-core violations with Phases 1-5 grep findings:
- `image-alt` violation → likely already found in Phase 1 (mark as CONFIRMED)
- `button-name` → may overlap Phase 3 ARIA checks
- `color-contrast` → NOT covered by static analysis (NEW finding, unique to axe-core)
- `link-name` → may overlap Phase 3

For duplicate findings, add `(confirmed by axe-core)` annotation to the static finding. Do NOT create a second finding for the same issue.

**Output table:**

```markdown
### axe-core Runtime Findings

| Rule ID | Impact | WCAG | Description | Pages Affected | Static Overlap |
|---------|--------|------|-------------|----------------|----------------|
| color-contrast | serious | 1.4.3 | Elements must have sufficient color contrast | /session, /settings | None (new) |
| image-alt | critical | 1.1.1 | Images must have alt text | / | Phase 1 (confirmed) |
```

**LOC Impact:** Varies per finding
**Effort:** `trivial` to `medium` (depends on fix complexity)

---

## Step 6: Assemble Findings Document

Write to `work/MMDDYY_Accessibility_WCAG_Audit.md` following shared.md Output Format exactly:

### 6a. Header

```markdown
# Accessibility WCAG Audit

**Date:** MMDDYY
**Prior:** work/MMDDYY_Accessibility_WCAG_Audit.md (or "None")
**Mode:** Full (grep-only)
**Agent:** None (grep-only audit)
**Tools:** Grep (built-in), axe-core via @axe-core/playwright (runtime DOM testing) [note if unavailable]
**Standard:** WCAG 2.1 Level AA
```

### 6b. Executive Summary

2–3 sentences covering:
- Total findings count and severity distribution
- Highest-impact finding (e.g., images without alt, clickable divs)
- Trend vs prior (if `COMPARE` is true)

### 6c. Metrics Dashboard

| Metric | Value | Prior | Trend |
|--------|-------|-------|-------|
| Total findings | N | — | — |
| Critical | N | — | — |
| High | N | — | — |
| Medium | N | — | — |
| Low | N | — | — |
| Images without alt | N | — | — |
| Non-semantic clickables | N | — | — |
| ARIA labels count | N | — | — |
| Role attributes count | N | — | — |
| Keyboard handlers count | N | — | — |
| Focus styles count | N | — | — |
| outline:none without replacement | N | — | — |
| axe-core violations (critical+serious) | N | — | — |
| axe-core violations (moderate+minor) | N | — | — |
| Pages scanned by axe-core | N | — | — |

Prior and Trend columns populated only when `COMPARE` is true.

### 6d. Findings Tables

Group all findings by severity. Each table uses shared.md format:

```markdown
### Critical

| # | File | Line | Issue | Tool | LOC Impact | Effort |
|---|------|------|-------|------|------------|--------|
```

Repeat for High, Medium, Low/Informational.

Number findings sequentially with **A11Y-** prefix (A11Y-1, A11Y-2, A11Y-3... for Accessibility findings).

### 6e. WCAG 2.1 AA Checklist

```markdown
## WCAG 2.1 AA Checklist

| Criterion | Description | Status |
|-----------|-------------|--------|
| 1.1.1 | Non-text content has alt | Pass/Fail/Partial |
| 1.4.3 | Contrast ratio 4.5:1 | ? (manual check needed) |
| 2.1.1 | Keyboard accessible | Pass/Fail/Partial |
| 2.4.7 | Focus visible | Pass/Fail/Partial |
| 4.1.2 | Name, Role, Value | Pass/Fail/Partial |
```

### 6f. Images Without Alt Text

```markdown
## Images Without Alt Text

| File | Line | Element | Status |
|------|------|---------|--------|
| Banner.tsx | 45 | `<img src={logo}>` | NEW |
```

### 6g. Non-Semantic Clickables

```markdown
## Non-Semantic Clickables

| File | Line | Current | Should Be | Status |
|------|------|---------|-----------|--------|
| Card.tsx | 23 | `<div onClick>` | `<button>` | CARRIED |
```

### 6h. ARIA Coverage

```markdown
## ARIA Coverage

| Component | Has ARIA | Missing | Status |
|-----------|----------|---------|--------|
| Modal | Yes | — | CARRIED |
```

### 6i. Keyboard Navigation Gaps

```markdown
## Keyboard Navigation Gaps

| Component | Tab Order | Enter/Space | Arrow Keys | Status |
|-----------|-----------|-------------|------------|--------|
| Menu | ? | ? | ? | CARRIED |
```

### 6j. Auto-Fixed Section

**Not applicable for accessibility audit** — fixes require manual verification and screen reader testing.
```markdown
## Auto-Fixed

N/A — Accessibility audit is read-only. Use findings to improve WCAG compliance.
```

### 6k. Delta Section

Only if `COMPARE` is true. Follow shared.md Delta Tracking rules.

### 6l. Historical Section

If prior existed, list resolved items per shared.md format.

### 6m. Recommendations

Top 3 highest-impact recommendations with estimated effort. Prioritize:
1. WCAG 1.1.1 failures (images without alt — easiest to fix)
2. WCAG 4.1.2 failures (non-semantic clickables, missing ARIA)
3. WCAG 2.4.7 failures (outline:none without replacement)
4. Keyboard navigation gaps for custom interactive widgets

---

## Step 7: Post-Processing

Follow shared.md for all of these:

### 7a. Tech Debt Integration

Per shared.md Tech Debt Integration rules — append new CRITICAL/HIGH findings to `tech-debt-patterns.md`, mark resolved entries.

### 7b. Git Commit

```bash
git add work/MMDDYY_Accessibility_WCAG_Audit.md
git add docs/archive/audits/  # if priors archived
git add .claude/rules/tech-debt-patterns.md  # if updated

git commit -m "$(cat <<'EOF'
docs(audit): accessibility WCAG audit — N findings (X critical, Y high)

Tools: Grep
Mode: Full (grep-only)
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

## Execution Checklist

Before completing, verify all items from shared.md Execution Checklist are met. Key accessibility-specific items:

- [ ] Phase 1: Images without alt scanned (WCAG 1.1.1)
- [ ] Phase 2: Non-semantic clickables found (`<div onClick>`, `<span onClick>`)
- [ ] Phase 3: ARIA coverage assessed (aria-label, aria-labelledby, role)
- [ ] Phase 4: Keyboard handler coverage cross-referenced with onClick handlers
- [ ] Phase 5: Focus styles checked, outline:none flagged
- [ ] Phase 6: axe-core runtime scan ran (or install/app-not-running noted)
- [ ] axe-core findings deduplicated against static findings (Phases 1-5)
- [ ] WCAG 2.1 AA Checklist populated
- [ ] Custom output tables populated (Images, Clickables, ARIA, Keyboard)
- [ ] Findings numbered with A11Y- prefix
- [ ] `--fix` noted as N/A (manual verification required)

---

## See Also

- `/audit-browser-compat` — Browser compatibility audit (complementary frontend audit)
- `/audit-typescript` — TypeScript audit (Phase 5: component health — overlaps: component-level analysis)
- `/audit-standards` — Linting + coding standards audit (overlaps: coding standards)
