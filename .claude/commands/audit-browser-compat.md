# /audit-browser-compat

Browser compatibility audit: CSS prefix usage, modern JS API detection, browser API support verification, polyfill checks, and feature detection gaps across target browsers (Chrome 90+, Firefox 88+, Safari 14+, Edge 90+).

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
   - `AUTO_FIX` — `false` always (`--fix` is a no-op — browser compat fixes require manual verification)
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
ls work/*_Browser_Compatibility_Audit.md 2>/dev/null | sort -r | head -1
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

### Browser Compat Delta Logic

In delta mode (`--since last`):
1. Load PRIOR_BASELINE for this audit (API usage table, feature detection status)
2. CARRY FORWARD: Findings where source `.tsx`/`.ts`/`.css` file NOT in CHANGED_FILES
3. RE-SCAN: Grep patterns below ONLY on changed frontend files
4. VERIFY: Spot-check 3-5 carried API usage findings still accurate
5. MERGE + MARK: NEW | CARRIED | CHANGED | RESOLVED

If MODE == "full": Run all patterns on entire codebase.

### Phase 1: CSS Prefix Usage

**Grep patterns (parallel):**

| Check | Pattern | Files | Purpose |
|-------|---------|-------|---------|
| Webkit prefixes | `-webkit-` | `*.css`, `*.tsx` | Vendor prefix usage |
| Mozilla prefixes | `-moz-` | `*.css`, `*.tsx` | Vendor prefix usage |
| MS prefixes | `-ms-` | `*.css`, `*.tsx` | Vendor prefix usage |

**Analysis:**
- Identify CSS properties using vendor prefixes
- Check if standard (unprefixed) property is also present
- Flag prefixed-only usage without standard fallback

**Severity mapping:**
- Prefixed property without standard fallback on critical UI = MEDIUM
- Prefixed property with standard fallback = INFO
- Unnecessary prefix (well-supported property) = LOW

**LOC Impact:** ~1 line per finding
**Effort:** `trivial` (add standard property alongside prefix)

### Phase 2: Modern JS API Detection

**Grep patterns (parallel):**

| Check | Pattern | Files | Purpose |
|-------|---------|-------|---------|
| Optional chaining | `\?\.\w` | `*.ts`, `*.tsx` | ES2020 feature usage |
| Nullish coalescing | `\?\?` | `*.ts`, `*.tsx` | ES2020 feature usage |
| Object.fromEntries | `Object\.fromEntries` | `*.ts`, `*.tsx` | ES2019 feature usage |

**Analysis:**
- Count usage of modern JS features
- Verify target browsers support these features (Chrome 90+, Firefox 88+, Safari 14+, Edge 90+ all support ES2020)
- Flag any features that need polyfills for target browser range

**Severity mapping:**
- Feature unsupported in any target browser = HIGH
- Feature supported in all target browsers = INFO (no action needed)

**LOC Impact:** ~1 line per finding
**Effort:** `small` (add polyfill or transpile target)

### Phase 3: Browser API Usage

**Grep patterns (parallel):**

| Check | Pattern | Files | Purpose |
|-------|---------|-------|---------|
| MediaDevices | `navigator\.mediaDevices` | `*.ts`, `*.tsx` | WebRTC camera/mic |
| RTCPeerConnection | `RTCPeerConnection` | `*.ts`, `*.tsx` | WebRTC connections |
| getUserMedia | `getUserMedia` | `*.ts`, `*.tsx` | Media access |
| Bluetooth | `navigator\.bluetooth` | `*.ts`, `*.tsx` | Web Bluetooth API |
| Permissions | `navigator\.permissions` | `*.ts`, `*.tsx` | Permissions API |

**Analysis:**
1. Grep WebRTC APIs (`RTCPeerConnection`, `getUserMedia`, `mediaDevices`) in `.ts`/`.tsx`
2. Verify all browser APIs have Safari/Firefox support or feature detection
3. Flag APIs with limited browser support without feature detection guards

**Severity mapping:**
- Browser API without feature detection on critical path = HIGH
- Browser API with partial support + no detection = MEDIUM
- Browser API with full target browser support = INFO

**LOC Impact:** ~3-10 lines per finding (add feature detection guard)
**Effort:** `small` (add `if (navigator.X)` guard)

### Phase 4: Modern CSS Features

**Grep patterns (parallel):**

| Check | Pattern | Files | Purpose |
|-------|---------|-------|---------|
| Backdrop filter | `backdrop-filter` | `*.css`, `*.tsx` | Safari prefix needed |
| CSS gap | `gap:` | `*.css`, `*.tsx` | Flexbox gap support |
| Aspect ratio | `aspect-ratio` | `*.css`, `*.tsx` | Modern CSS property |

**Analysis:**
- Check if modern CSS properties have fallbacks for target browsers
- Safari 14 may need `-webkit-backdrop-filter`
- Verify `gap` usage is in flex/grid context (grid gap well-supported, flex gap needs Safari 14.1+)

**Severity mapping:**
- Modern CSS without fallback breaking layout in target browser = HIGH
- Modern CSS with limited support but non-critical visual = MEDIUM
- Modern CSS with full target support = INFO

**LOC Impact:** ~1-3 lines per finding
**Effort:** `trivial` to `small` (add fallback or prefix)

### Phase 5: Feature Detection

**Grep patterns (parallel):**

| Check | Pattern | Files | Purpose |
|-------|---------|-------|---------|
| Navigator checks | `if.*navigator\.` | `*.ts`, `*.tsx` | Feature detection guards |
| Window checks | `if.*window\.` | `*.ts`, `*.tsx` | Feature detection guards |
| Polyfill imports | `polyfill` | `*.ts`, `*.tsx` | Polyfill presence |

**Analysis:**
1. Cross-reference browser APIs from Phase 3 with feature detection guards from this phase
2. Flag APIs used without corresponding feature detection
3. Check for polyfill imports

**Severity mapping:**
- Critical API (WebRTC, MediaDevices) without any detection = HIGH
- Non-critical API without detection = MEDIUM
- All APIs have proper detection = no finding

**LOC Impact:** ~3-5 lines per finding (add if guard)
**Effort:** `small` (add feature detection wrapper)

---

## Step 6: Assemble Findings Document

Write to `work/MMDDYY_Browser_Compatibility_Audit.md` following shared.md Output Format exactly:

### 6a. Header

```markdown
# Browser Compatibility Audit

**Date:** MMDDYY
**Prior:** work/MMDDYY_Browser_Compatibility_Audit.md (or "None")
**Mode:** Full (grep-only)
**Agent:** None (grep-only audit)
**Tools:** Grep (built-in)
**Target Browsers:** Chrome 90+, Firefox 88+, Safari 14+, Edge 90+
```

### 6b. Executive Summary

2–3 sentences covering:
- Total findings count and severity distribution
- Highest-impact finding (e.g., unguarded WebRTC API, missing prefix)
- Trend vs prior (if `COMPARE` is true)

### 6c. Metrics Dashboard

| Metric | Value | Prior | Trend |
|--------|-------|-------|-------|
| Total findings | N | — | — |
| Critical | N | — | — |
| High | N | — | — |
| Medium | N | — | — |
| Low | N | — | — |
| Browser APIs detected | N | — | — |
| APIs with feature detection | N | — | — |
| CSS vendor prefixes | N | — | — |
| Polyfill imports | N | — | — |

Prior and Trend columns populated only when `COMPARE` is true.

### 6d. Findings Tables

Group all findings by severity. Each table uses shared.md format:

```markdown
### Critical

| # | File | Line | Issue | Tool | LOC Impact | Effort |
|---|------|------|-------|------|------------|--------|
```

Repeat for High, Medium, Low/Informational.

Number findings sequentially with **BC-** prefix (BC-1, BC-2, BC-3... for Browser Compat findings).

### 6e. Browser API Usage Table

```markdown
## Browser API Usage

| API | Files | Safari | Firefox | Detection | Status |
|-----|-------|--------|---------|-----------|--------|
| RTCPeerConnection | N | ? | ? | Yes/No | NEW/CARRIED |
| navigator.mediaDevices | N | ? | ? | Yes/No | NEW/CARRIED |
```

### 6f. Feature Detection Gaps

```markdown
## Feature Detection Gaps

| Feature | Files | Has Detection | Status |
|---------|-------|---------------|--------|
| Bluetooth | devices/*.ts | No | NEW |
```

### 6g. Auto-Fixed Section

**Not applicable for browser compat audit** — fixes require manual browser testing.
```markdown
## Auto-Fixed

N/A — Browser compatibility audit is read-only. Use findings to add feature detection guards and polyfills.
```

### 6h. Delta Section

Only if `COMPARE` is true. Follow shared.md Delta Tracking rules.

### 6i. Historical Section

If prior existed, list resolved items per shared.md format.

### 6j. Recommendations

Top 3 highest-impact recommendations with estimated effort. Prioritize:
1. Unguarded critical browser APIs (WebRTC, MediaDevices)
2. Modern CSS without fallbacks breaking target browsers
3. Quick wins: adding feature detection to existing API usage

---

## Step 7: Post-Processing

Follow shared.md for all of these:

### 7a. Tech Debt Integration

Per shared.md Tech Debt Integration rules — append new CRITICAL/HIGH findings to `tech-debt-patterns.md`, mark resolved entries.

### 7b. Git Commit

```bash
git add work/MMDDYY_Browser_Compatibility_Audit.md
git add docs/archive/audits/  # if priors archived
git add .claude/rules/tech-debt-patterns.md  # if updated

git commit -m "$(cat <<'EOF'
docs(audit): browser compat audit — N findings (X critical, Y high)

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

Before completing, verify all items from shared.md Execution Checklist are met. Key browser-compat-specific items:

- [ ] Phase 1: CSS vendor prefixes scanned (`-webkit-`, `-moz-`, `-ms-`)
- [ ] Phase 2: Modern JS API usage counted (optional chaining, nullish coalescing)
- [ ] Phase 3: Browser APIs detected and cross-referenced with target browser support
- [ ] Phase 4: Modern CSS properties checked for fallbacks
- [ ] Phase 5: Feature detection gaps identified (APIs without `if` guards)
- [ ] Browser API Usage table populated
- [ ] Feature Detection Gaps table populated
- [ ] Findings numbered with BC- prefix
- [ ] `--fix` noted as N/A (manual verification required)

---

## See Also

- `/audit-accessibility` — WCAG 2.1 AA accessibility audit (complementary frontend audit)
- `/audit-typescript` — TypeScript audit (overlaps: frontend code quality)
- `/audit-typescript` — TypeScript audit (Phase 5: component health — overlaps: component-level analysis)
- `/audit-resilience` — Resilience audit (overlaps: graceful degradation patterns)
