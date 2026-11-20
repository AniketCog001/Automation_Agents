---
description: >
  Token-minimal POM generation: drive the live browser via MCP snapshot_dom to
  extract a compact interactive-element JSON per page, then convert that JSON into
  XPath locators — saving ~90 % of the tokens a raw-HTML approach would consume.
  Includes deduplication against existing POM files and organized, framework-matching
  POM output. Works in both VS Code (Copilot agent) and Antigravity.
name: dom_extract_pom
version: "1.1.0"
---

# Skill — DOM-Extract → POM (90 % Token Savings)

## Why this skill exists

The classic approach feeds raw page HTML (50–100 KB, ≈12 000–25 000 tokens) into the
XPath prompt. This skill replaces that with a **four-phase pipeline**:

| Phase | What happens | AI tokens consumed |
|-------|-------------|-------------------|
| 1 – Context + Dedup scan | Read story, feature, KG, ALL existing POMs in module | ~500 tokens |
| 2 – DOM Extraction | MCP `snapshot_dom` returns compact JSON per page (≈1–3 KB) | **0 — browser does the work** |
| 3 – POM Generation | XPath derivation from compact JSON; write organized file | ~1 500 tokens |
| 4 – Validate & report | Self cross-check, KG update | ~200 tokens |

Compared with the raw-HTML approach (~14 000–27 000 tokens) that is **≈90 % fewer tokens**.

---

## 0. Prerequisites

| Requirement | Check |
|-------------|-------|
| MCP server running | `MCP/selenium_mcp_launch.py` in `.vscode/mcp.json` |
| Story file exists | `MCP/stories/<Module>/<STORY-ID>/story.md` (run `/fetch-userstory` first) |
| Reference POM exists | `object_repository/pages/Apps/<Module>/` |
| Credentials in env | `MCP/.env.mcp` → `QAAdminEmail` / `QAAdminPassword` |

Feature file is optional — if missing, derive element needs from story AC only.
If any other prerequisite is missing, **stop and report which one**.

---

## 1. Phase 1 — Context gathering + Deduplication scan (zero browser, minimal tokens)

Read ALL of these before touching the browser.

### 1a. Story file → extract element names and pages

Read `MCP/stories/<Module>/<STORY-ID>/story.md`.
Extract: module name, AC list, screen/page names mentioned, element names mentioned.

### 1b. Feature file → build "elements needed" list

Find `tests/features/Apps/<Module>/` and read any `.feature` for this story.
From each Gherkin step referencing a UI action, extract the element name.

```
Elements needed list
--------------------
1. "Audit Trail" page heading      | /then step 1
2. Search input                    | /when step 3
3. Date Range picker               | /when step 4
4. Results table                   | /then step 5
5. Export button                   | /when step 6
```

If no feature file exists: derive the list from the story AC only.

### 1c. KG context pack → mark already-verified locators

```
kg_context_pack(task="<STORY-ID>", module="<Module>")
```

Mark `USE` locators → **already verified, skip extraction**.
Mark `VERIFY` locators → **gate-check during walk**.
Mark `PROBE` / missing → **extract from DOM**.

### 1d. ★ DEDUPLICATION SCAN — read ALL existing POM files in the module ★

This step is **mandatory** — it prevents duplicate locators and wasteful re-extraction.

1. List every `*.py` file under `object_repository/pages/Apps/<Module>/` (recurse).
2. Read each file completely.
3. From each file, extract:
   - **Class names** already defined (e.g. `class CMOLandingPage`)
   - **Locator variable names** (all `locator*` attributes)
   - **XPath strings** (the selector inside `By.XPATH, "..."`)
4. Build the **Existing Locators Inventory**:

```
Existing Locators Inventory
============================
File: CmoLandingPage.py
  Classes: CMOLandingPage, CMOTemplateLandingPage
  Locators (CMOLandingPage):
    locatorCMOHeading          → '//*[contains(@class,"h6") and text()="CMO"]'
    locatorCMOTabTemplates     → '//button[@data-testid="cmo-tab-templates"]'
    locatorBreadcrumbs         → '//li[contains(@class,"inline-flex")]'
    locatorHelpButton          → '(//*[text()="Help"])[2]'
    ... (ALL locator names)

File: NewAuditPage.py
  Classes: NewAuditPage
  Locators: locatorNewAuditDialogOpen, locatorNewAuditHeading, ...
```

5. During Phase 3, for any new locator candidate:
   - **If the exact XPath already exists** anywhere in the module POMs → skip, note "reuse `<file>.<class>.<locatorName>`".
   - **If a locator with the same semantic name already exists** in the target class → skip.
   - **Only generate** locators that are genuinely new.

6. Determine **where the new locators go**:
   - If a POM class for this page already exists → **add a new section** to that class.
   - If no POM class exists for this page → **create a new class** in either a new file or an existing file (follow the module's convention).
   - Never create a duplicate class.

### 1e. POM Style Ruleset — extract from reference POM

Read the most relevant existing `*Page.py` completely. Lock in the style ruleset:

```
POM Style Ruleset (MUST follow exactly):
  Import block    : # pylint: disable=all  (line 1)
                    from selenium.webdriver.common.by import By
                    from utilities.genericUtils import CustomPyAutoWeb
  Base class      : CustomPyAutoWeb
  Class naming    : PascalCase + "Page" suffix (e.g. AuditTrailLandingPage)
  Locator prefix  : camelCase, starts with "locator" (e.g. locatorAuditTrailHeading)
  Locator shape   : locatorXxx = [(By.XPATH, "...")]   ← list of ONE tuple
  Multi-line XPath: use when XPath > ~80 chars (see §3b format rules)
  State suffixes  : Active/Inactive  Selected/NotSelected  Open/Closed
  Dynamic style   : locatorXxxByName = lambda name: [(By.XPATH, f"...{name}...")]
  Section divider : match the reference — either:
                    # ----------------- section name ------------------#
                    or:
                    # ===========================Section Title=========================#
  Element group   : # --- Group Name ---  above each logical group
  __init__        : [match the reference — present or absent, same signature]
  Multi-class file: allowed — follow the module's convention
```

### 1f. Output the page walk plan

```
Page walk plan
--------------
Page 1 : <name>   navigate via: <URL or click sequence>
Page 2 : <name>   navigate via: <action on page 1>
Sub-states:
  - <modal/dialog/panel name>  (open via: <action>)
  - <tab name>                 (click tab: <label>)
```

---

## 2. Phase 2 — DOM extraction (MCP, ZERO generation tokens)

### 2a. Browser setup (once per session)

```
start_browser({ "browser": "chrome", "options": { "headless": false } })
login_sso({ "url": "<APP_URL>", "username": "<from .env>", "password": "<from .env>" })
wait_for_idle({ "timeout_ms": 30000 })
```

### 2b. Per-page capture loop

For each page in the walk plan:

```
# 1. Navigate
<click_sequence or navigate tool>
wait_for_idle({ "timeout_ms": 30000 })

# 2. Capture compact interactive DOM
snapshot_dom({ "max_elements": 300 })
→ Write JSON to: MCP/fixtures/<STORY-ID>/dom/<page-slug>_snapshot.json

# 3. Sub-states (modal / dialog / open tab / side panel)
<open sub-state>
wait_for_idle({ "timeout_ms": 15000 })
snapshot_dom({ "max_elements": 200 })
→ Write to: MCP/fixtures/<STORY-ID>/dom/<page-slug>_<substate>_snapshot.json

# 4. Real-time KG update — IMMEDIATELY per element
kg_upsert(kind="locator", element_name="<name>", testid="<testid/xpath>",
          module="<Module>", page="<page-slug>", tag="snapshot_dom")
```

### 2c. Coverage table (print after each page)

| Element needed | Found in snapshot | Anchor type | Verdict |
|---------------|------------------|-------------|---------|
| Audit Trail heading | YES | h2 text | COVERED |
| Search input | YES | testid="audit-search" | COVERED |
| Date range picker | YES | role=button + name | COVERED |
| Export button | NO — ⚠️ GAP | — | RETRY |

For any GAP: open sub-states, retry `snapshot_dom` once.
After one retry, if still missing: `kg_add_knowledge` note, continue — **never invent**.

---

## 3. Phase 3 — Organized POM generation (compact JSON → XPaths)

### 3a. Deduplication filter (apply FIRST)

Before writing any locator, cross-check the Existing Locators Inventory from Phase 1d:
- Same XPath string already in any module POM → **skip**, note "reuse `<file>.<locatorName>`"
- Same semantic name already in target class → **skip**
- Genuinely new → **generate**

### 3b. XPath derivation priority (apply in order, stop at first match)

```
Priority 1 — testid present
  XPath: //tag[@data-testid='testid']

Priority 2 — unique button / link / tab / heading text (name field)
  XPath: //tag[normalize-space()='name']

Priority 3 — role + name
  XPath: //*[@role='role'][normalize-space()='name']

Priority 4 — input: type + name (placeholder / aria-label)
  XPath: //input[@type='type' and @placeholder='name']
     OR: //input[@type='type' and @aria-label='name']

Priority 5 — heading tag + text
  XPath: //h2[normalize-space()='name']

Priority 6 — state variant (only when attribute seen in snapshot)
  Append: and @aria-selected='true'  /  and @aria-selected='false'
          and @data-state='open'     /  and @data-state='closed'
          and @disabled
```

**Forbidden**: hashed classes (`css-*`, `sc-*`), bare utility classes (`flex` alone,
`mt-4`), absolute paths (`/html/body/...`), bare positional (`(//div)[3]`).

### 3c. Locator formatting rules (match existing files exactly)

**Single-line** — for XPaths that fit comfortably ≤ ~100 chars:
```python
locatorAuditTrailHeading = [(By.XPATH, "//h2[normalize-space()='Audit Trail']")]
```

**Multi-line** — for XPaths > ~80 chars or with class-chain conditions:
```python
locatorAuditTrailTableRows = [
    (
        By.XPATH,
        "//table[@data-testid='audit-trail-table']//tbody/tr",
    )
]
```

**Multiple fallback selectors** (two XPaths in one list):
```python
locatorExportButton = [
    (By.XPATH, "//button[@data-testid='audit-export-btn']"),
    (By.XPATH, "//button[normalize-space()='Export']"),
]
```

**Lambda**:
```python
locatorAuditTrailRowByUser = lambda user: [
    (By.XPATH, f"//tr[.//td[normalize-space()='{user}']]")
]
```

### 3d. Section organisation (CRITICAL — produces a clean, readable POM)

Always group locators into named sections using this order:

```
1.  # ----------------- header locators ------------------#
2.  # ----------------- breadcrumb locators ------------------#
3.  # ===========================Navigation Tabs=========================#
4.  # ===========================Search / Filter Panel=========================#
5.  # ===========================Results Table=========================#
6.  # ===========================Action Buttons=========================#
7.  # ===========================Modals / Dialogs=========================#
8.  # ===========================Toast / Alerts=========================#
```

Within each section, group elements logically with `# --- Group Name ---` comments:

```python
# ===========================Search / Filter Panel=========================#

# --- Panel toggle ---
locatorSearchPanelToggle = [(By.XPATH, "//*[@data-testid='audit-trail-search-panel-toggle']")]

# --- Date range fields ---
locatorDateFromInput = [(By.XPATH, "//input[@data-testid='date-from']")]
locatorDateToInput   = [(By.XPATH, "//input[@data-testid='date-to']")]

# --- Dropdown filters ---
locatorStatusDropdown = [(By.XPATH, "//button[@data-testid='status-filter']")]

# --- Action buttons ---
locatorSearchButton = [(By.XPATH, "//*[@data-testid='search-button']")]
locatorResetButton  = [(By.XPATH, "//button[@aria-label='Reset filters']")]
```

If the reference POM uses `# =========================#` style for major sections and
`# --- ---` for sub-groups, match that exactly.

### 3e. Locator naming conventions

| Element type | Pattern | Example |
|-------------|---------|---------|
| Page heading | `locator<Screen>Heading` | `locatorAuditTrailHeading` |
| Nav tab | `locator<Screen>Tab<Name>` | `locatorAuditTrailTabAll` |
| Tab — active | `locator<Screen>Tab<Name>Active` | `locatorAuditTrailTabAllActive` |
| Tab — inactive | `locator<Screen>Tab<Name>Inactive` | `locatorAuditTrailTabAllInactive` |
| Tab by label | `locator<Screen>NavTabByName` (lambda) | `locatorAuditTrailNavTabByName` |
| Input | `locator<Name>Input` | `locatorSearchInput` |
| Dropdown | `locator<Name>Dropdown` | `locatorStatusDropdown` |
| Button | `locator<Name>Button` | `locatorExportButton` |
| Breadcrumb | `locator<Name>BreadCrumb` | `locatorAuditTrailBreadCrumb` |
| Table | `locator<Screen>Table` | `locatorAuditTrailTable` |
| Table rows | `locator<Screen>TableRows` | `locatorAuditTrailTableRows` |
| Row by value | `locator<Entity>RowBy<Field>` (lambda) | `locatorAuditTrailRowByUser` |
| Column header | `locatorColumnHeader<Name>` | `locatorColumnHeaderUser` |
| Column data | `locator<Name>TableRowsData` | `locatorUserTableRowsData` |
| Modal open | `locator<Name>DialogOpen` | `locatorFilterDialogOpen` |
| Modal closed | `locator<Name>DialogClosed` | `locatorFilterDialogClosed` |
| Toast | `locatorToast<Name>` | `locatorToastSuccessMessage` |
| Panel toggle | match existing name or `locatorSearchPanelToggle` | — |

---

## 4. Deciding: new file vs. add to existing file

| Condition | Decision |
|-----------|----------|
| Module folder does not exist | Create `object_repository/pages/Apps/<Module>/` + `<Module>LandingPage.py` |
| Module has one multi-class file | Add new class to the **end** of that file |
| Module uses separate files per screen | Create `<ScreenName>Page.py` for each new screen |
| Class for this screen already exists | Add new section block inside the existing class |
| **Never** | Create two classes with the same name |

---

## 5. Phase 4 — Validation & cleanup

### 5a. Self cross-check (run BEFORE writing)

- [ ] Every element in "Elements needed" → locator OR documented `⚠️ GAP`
- [ ] Deduplication applied — no XPath or name duplicates any existing module POM
- [ ] No hashed class names (`css-*`, `sc-*`) in any XPath
- [ ] No absolute XPaths (`/html/body/...`)
- [ ] Every XPath grounded in snapshot data — never invented
- [ ] State variants only where snapshot showed the attribute
- [ ] Lambda for all data-driven names
- [ ] Section headers match the reference POM's divider style exactly
- [ ] Multi-line format for XPaths > ~80 chars
- [ ] `__init__` matches the reference POM (present/absent/same signature)
- [ ] No assertions, no waits, no test data, no cross-page logic in the POM

### 5b. KG finalize

```
kg_reindex()
kg_session_save(summary="...", decisions=[...], todos=[...])
```

### 5c. Final report

```
DOM-Extract → POM Report
========================
Story    : <STORY-ID>   Module   : <Module>
Pages walked  : N       Sub-states: M
Snapshots     : MCP/fixtures/<STORY-ID>/dom/ (N files)

POM output:
  File          : <path>   (new / extended)
  Classes added : <list>
  Sections added: <list>

Locators generated : <total>
  testid-anchor : N    text-anchor  : N
  role+name     : N    lambda (dyn) : N

Skipped (already in module POMs):
  <locatorName>  → reuse from <file>.<class>

Gaps (need data-testid in HTML):
  ⚠️ [element] — reason

Token saving : ~90 % vs raw-HTML approach
```

---

## 6. DOM snapshot → XPath quick-reference

```
Snapshot element                                    → XPath produced
────────────────────────────────────────────────────────────────────────────
{ tag:"button", testid:"audit-export" }             → //button[@data-testid='audit-export']
{ tag:"button", name:"Export" }                     → //button[normalize-space()='Export']
{ tag:"*",  role:"tab", name:"All" }                → //*[@role='tab'][normalize-space()='All']
{ tag:"input", type:"search" }                      → //input[@type='search']
{ tag:"input", type:"text", name:"From date" }      → //input[@type='text' and @placeholder='From date']
{ tag:"h2",  name:"Audit Trail" }                   → //h2[normalize-space()='Audit Trail']
{ tag:"button", testid:"t-x", ariaSelected:"true" } → //button[@data-testid='t-x' and @aria-selected='true']
{ tag:"button", name:"Delete", disabled:true }      → //button[normalize-space()='Delete' and @disabled]
```

---

## 7. Integration with the rest of the pipeline

This skill replaces the "attach HTML" step of `01_xpath_pom.prompt.md`.
After this skill runs, the output POM feeds directly into:

```
/dom-extract-pom  →  POM file  →  /03_step_definitions  →  step def
                              →  /web_component_prompt  →  web_component
```

The `story-to-tests` coordinator can delegate to `@dom-extractor` instead of the
full Explorer walk when only POM locators are needed (not flow verification).
