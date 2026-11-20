---
description: "Generate a production-ready Selenium POM class with XPath locators for a new page, matching existing project conventions exactly. Includes deep bug analysis and auto-fix before final output."
agent: PomAuthor
---

# Generate POM + XPath Locators

You are a senior QA automation engineer in this workspace — a **pytest-bdd** project
(Python + Selenium + `CustomPyAutoWeb` + POM + Allure).
Follow [AGENTS.md](../AGENTS.md) at all times. Produce
**100% accurate, production-ready** XPath locators for a new page, strictly matching the
existing POM conventions.

> **🧠 Knowledge-graph protocol:** call `kg_context_pack(task=<page/screen>, module=<Module>)`
> BEFORE writing any locator — selectors already verified live (recommendation **USE**, or
> **VERIFY** after a gate) take priority over ones derived from static HTML. If a needed
> element is missing from both the pack and the provided HTML, **report the gap** so the
> coordinator sends the **Explorer** agent to walk the screen — never invent a selector.

> **Load the skills first** — they are the copy-pasteable ground truth:
> [00_framework-architecture](../skills/framework_architecture/SKILL.md) ·
> [03_page-objects](../skills/page_objects/SKILL.md).
>
> **Repo facts you must honour:** page classes live in
> object_repository/pages/Apps/`<Module>`/`<Name>`Page.py, are named in PascalCase, and
> **extend `CustomPyAutoWeb`**; every locator is a **class-level variable** named
> `locatorXxx` (camelCase, `locator` prefix) whose value is a **list of one By-tuple** —
> `locatorXxx = [(By.XPATH, "...")]`; runtime-variable locators are **`lambda` functions
> that return that same list** — `locatorXxxByName = lambda name: [(By.XPATH, f"...{name}...")]`;
> the `__init__(self, driver, url="")` (if present) sits **after** all locator declarations
> and calls `super().__init__(driver)`. There are **no page methods that assert** — a POM
> holds locators (and optional thin navigation helpers); assertions live in step defs.

---

## Inputs — resolve ALL of these before doing anything else

| Input                     | How to provide                                                                                       | If missing                                                 |
| ------------------------- | ---------------------------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| **Reference POM**         | Attach with `#file` or let me find it via `#codebase`                                                | → STOP and ask                                             |
| **Target HTML**           | Attach with `#file` or paste inline                                                                  | → STOP, never guess XPaths                                 |
| **Screenshot** (optional) | Attach with `#file`                                                                                  | Used only to confirm visible states — HTML is ground truth |
| **Target file**           | `${input:targetFile:path for the new POM, e.g. object_repository/pages/Apps/CMO/CmoTemplatePage.py}` | → STOP and ask                                             |

Work in **agent mode**: read all inputs in full, then **create/edit the target `.py` file
directly**. Print every analysis table and both check reports in chat.

---

## PHASE 1 — Extract the project's POM style from the reference file

Read the reference POM **completely** before writing a single line.
Extract, print, and lock in the following ruleset — every generated line must match it:

```
POM Style Ruleset:
- Class name format        : [e.g. CMOLandingPage — PascalCase, matches the page]
- Locator variable style   : [camelCase with `locator` prefix — e.g. locatorCMOHeading]
- Locator declaration type : [list of one By-tuple — locatorXxx = [(By.XPATH, "...")]]
- State suffix convention   : [Active/Inactive, Selected/NotSelected, Open/Closed — match the reference]
- Dynamic locator style     : [lambda returning the list — locatorXxxByName = lambda name: [(By.XPATH, f"...{name}...")]]
- File name format         : [e.g. CmoLandingPage.py — PascalCase + `Page.py`]
- Import block             : [# pylint: disable=all ; from selenium.webdriver.common.by import By ; from utilities.genericUtils import CustomPyAutoWeb]
- Base class               : [CustomPyAutoWeb — confirm from the reference]
- __init__ signature       : [present? `def __init__(self, driver, url="")` with super().__init__(driver) AFTER the locators — or no __init__ at all]
- Section divider style    : [both forms seen: `# ----------------- header locators --------------------` and `# ===== Title =====#` boxes — match the reference]
- Element group comment     : [e.g. `# --- Supervisors Tab ---` above each element's locators]
- Multiple classes per file : [note if the reference defines several page classes in one file]
```

Do NOT invent any convention. If the reference is ambiguous, note it and use the most
common pattern found.

---

## PHASE 2 — Build the element inventory from the HTML (HTML is ground truth)

Read the HTML **completely**. Cross-reference the screenshot only to confirm visual states.

**State detection protocol** — for every element, inspect the HTML for:
`aria-selected`, `aria-pressed`, `aria-disabled`, `aria-expanded`, `disabled`,
`class` tokens, `data-state`, `data-active`, `data-testid`, `role`.
List **only states that exist in the HTML**. Never assume a state.

Print the inventory table:

| #   | Element             | Type   | States in HTML                  | Best anchor                       |
| --- | ------------------- | ------ | ------------------------------- | --------------------------------- |
| 1   | Create Template Btn | button | DEFAULT                         | `text()="Create Template"`        |
| 2   | Filter Tab          | tab    | DEFAULT, SELECTED, NOT_SELECTED | `text()` + `@aria-selected`       |
| 3   | Delete Button       | button | DEFAULT, DISABLED               | `@data-testid` + `@aria-disabled` |

**⚠️ FLAG RULE**: if an element is visible in the screenshot but has no unique anchor in
the HTML → print `⚠️ WARNING: [element] — no unique anchor found in HTML`, propose an
approach, and **wait for confirmation** before writing its locator.

---

## PHASE 3 — Write ONE XPath per confirmed state

For every element in the inventory write **exactly one** best XPath per confirmed state.

### XPath anchor priority — follow this order strictly

Attempt each level in order. Move to the next level **only** if the current level cannot
produce a unique, stable XPath for this element. Print one decision-log line per element.

| Level | Anchor                                    | Use when                                   |
| ----- | ----------------------------------------- | ------------------------------------------ |
| 1     | `normalize-space()` / `text()`            | Visible text exists — **always try first** |
| 2     | `@aria-label`, `@role` + `text()`         | No visible text (icon buttons, inputs)     |
| 3     | `@data-testid`, `@data-cy`, `@data-qa`    | Levels 1–2 not unique                      |
| 4     | `@id`, `@name`                            | Levels 1–3 not available                   |
| 5     | Semantic `@class` token + `text()`        | **Last resort** — see class rules below    |
| 6     | Positional axis anchored to stable parent | **Absolute last resort** — flag it         |

**Decision log** — print one line per element before writing its XPath:

```
→ Create Button  : Level 1 — unique text 'Create Template' on button
→ Search Input   : Level 2 — no text; aria-label='Search templates'
→ Close Icon     : Level 3 — no text/aria; data-testid='close-btn'
→ Submit Button  : Level 5 — no text/aria/testid/id; class 'form__submit' is
                   unique and semantic; combined with tag + text ⚠️
```

### Class-based XPaths — strict rules (Level 5 only)

- ✗ NEVER auto-generated / hashed: `css-1x2y3z`, `sc-abc123`, `MuiButton-root-456`
- ✗ NEVER utility / layout: `flex`, `mt-4`, `d-flex`, `col-md-6`, `text-sm`
- ✗ NEVER a class appearing 5+ times in the DOM
- ✓ ONLY semantically named, component-unique tokens
- ✓ ALWAYS combine with tag + text: `//button[contains(@class,'form__submit') and normalize-space()='Submit']`

### State XPath rules

- Combine the element's primary text/aria anchor **with** the state attribute:
  `//div[@role='tab' and normalize-space()='Active' and @aria-selected='true']`
- Use ONLY the state attribute the HTML actually contains for that element
- DEFAULT state must NOT accidentally match SELECTED or DISABLED elements —
  add an exclusion predicate if needed

### Dynamic XPath rules

- Any XPath targeting a runtime-variable value (tab name, row label, item text) **MUST**
  be a **`lambda` that returns the locator list**, using an f-string — the project's exact pattern:
  ```python
  locatorCMONavTabByName = lambda tab_name: [
      (By.XPATH, f"//nav//button[normalize-space(text())='{tab_name}']")
  ]
  ```
- Static elements with genuinely fixed, never-changing labels stay as a plain locator list

### Strictly forbidden

- ✗ More than one XPath per state per element
- ✗ Absolute paths: `/html/body/div/...`
- ✗ Floating index-only: `(//div)[3]`
- ✗ Any attribute value not present in the HTML
- ✗ Speculative states — no state without HTML evidence
- ✗ Skipping a higher-priority level without logging why

---

## PHASE 4 — Generate the POM class (write the target file)

Create/edit `${input:targetFile}` with:

- `# pylint: disable=all` header, imports, class name, base class (`CustomPyAutoWeb`), and
  the `__init__` (if the reference has one) **matching the reference exactly** — the
  `__init__` goes **after** all locator declarations
- Locators grouped by UI section using the reference's exact section-divider style
- Every locator is a **list of one By-tuple**; one variable per confirmed state, with a
  state suffix matching the reference (`Active`/`Inactive`, `Selected`/`NotSelected`, etc.)
  and a `# --- Element ---` comment above each group:
  ```python
  # --- Filter Tab ---
  locatorFilterTab = [(By.XPATH, "//div[@role='tab' and normalize-space()='Filter']")]
  locatorFilterTabSelected = [
      (By.XPATH, "//div[@role='tab' and normalize-space()='Filter' and @aria-selected='true']")
  ]
  locatorFilterTabNotSelected = [
      (By.XPATH, "//div[@role='tab' and normalize-space()='Filter' and @aria-selected='false']")
  ]
  ```
- Elements with a single (DEFAULT) state only need one variable (no suffix):
  ```python
  # --- Create Template Button ---
  locatorCreateTemplateButton = [(By.XPATH, '//*[text()="Create Template"]/ancestor::button')]
  ```
- **Runtime-variable XPaths as `lambda` functions returning the list** — never inline strings:
  ```python
  locatorTemplateCardByName = lambda template_name: [
      (By.XPATH, f"//div[@data-testid='template-card' and .//*[normalize-space()='{template_name}']]")
  ]
  ```
- Level 5/6 locators get an inline comment explaining why the higher level was skipped:
  ```python
  # Level 5 — no text/aria/testid/id; 'form__submit' is unique and semantic
  locatorSubmitButton = [
      (By.XPATH, "//button[contains(@class,'form__submit') and normalize-space()='Submit']")
  ]
  ```
- Declare ONLY states confirmed in Phase 2 — zero speculative locators

---

## PHASE 5 — Self cross-check (print in chat, do not skip)

```
[ClassName] — POM Cross-Check:
✅ XPath accuracy      — every attribute/tag copied verbatim from the HTML
✅ State accuracy      — only confirmed states; one locator per state; no speculation
✅ Priority compliance — decision log printed; text-first; class only at Level 5+
✅ Element coverage    — X/X inventory elements covered; none invented
✅ Convention match    — class/imports/__init__/naming/sections match reference 100%
✅ Syntax valid        — every locator is a `[(By.XPATH, "...")]` list; each `lambda` returns that list
⚠️ [element | variable | exact problem | fix applied]

Level usage summary:
  Level 1 (text)       : X
  Level 2 (aria)       : X
  Level 3 (data-testid): X
  Level 4 (id/name)    : X
  Level 5 (class)      : X ⚠️
  Level 6 (positional) : X ⚠️
```

---

## PHASE 6 — Deep bug analysis and auto-fix (do not skip)

After Phase 5 passes, do a **dedicated bug-hunting pass** over every locator, method, and
line of the generated class. Fix every bug found before writing the file. This phase goes
deeper than the cross-check.

### Category A — XPath logic bugs

For every XPath string verify:

| Check                          | What to look for                                                                                                                                                                  |
| ------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Wrong axis direction**       | `ancestor::` where `descendant::` is needed, or reversed parent/child                                                                                                             |
| **Wrong predicate logic**      | `and`/`or` confusion; separate `[][]` predicates (both must be true) vs intended `or`; missing negation                                                                           |
| **normalize-space() misuse**   | `normalize-space(@text)` is wrong — use `normalize-space()`; text node in a child `span`/`label` needs `.//span[normalize-space()='x']` not `normalize-space()='x'` on the button |
| **Incomplete state predicate** | SELECTED XPath missing `@aria-selected='true'`; DEFAULT accidentally matches SELECTED because no exclusion exists                                                                 |
| **Dynamic format errors**      | Missing `f` prefix; wrong quote nesting; `.format()` key doesn't match the method parameter name; unescaped literal `{` or `}`                                                    |
| **Duplicate match risk**       | XPath matches multiple elements — verify uniqueness against the full HTML; if not unique, scope with an ancestor predicate                                                        |
| **Whitespace / encoding**      | Non-breaking space (char 160) inside text; HTML entity not decoded; extra spaces not wrapped in `normalize-space()`                                                               |

### Category B — Python code bugs

| Check                      | What to look for                                                                                                                                                                                                   |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Missing imports**        | `from selenium.webdriver.common.by import By` and `from utilities.genericUtils import CustomPyAutoWeb` present; nothing else needed for a pure locator class                                                       |
| **Wrong import path**      | Relative vs absolute path matches the reference exactly                                                                                                                                                            |
| **Unused imports**         | Remove anything not used in the class body                                                                                                                                                                         |
| **Constructor errors**     | If the reference has an `__init__(self, driver, url="")`: it calls `super().__init__(driver)`, sets `self.driver`/`self.url`, and is placed AFTER the locators; if the reference has no `__init__`, do not add one |
| **Dynamic locator errors** | Dynamic locator is a `lambda <param>:` returning the list (NO `self`); the f-string placeholder name matches the lambda parameter; the `f` prefix is present                                                       |
| **Locator type mismatch**  | Every locator is a LIST containing one tuple — `[(By.XPATH, "...")]`; tuple order is `(By.XPATH, xpath_string)` not reversed; the outer `[ ]` is never dropped                                                     |
| **String not closed**      | Multi-line XPath string missing parentheses or line continuation; mismatched quotes                                                                                                                                |
| **Naming violations**      | Wrong case for variable/method/class vs the extracted ruleset                                                                                                                                                      |

### Category C — logical correctness bugs

| Check                                 | What to look for                                                                                        |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| **Locator targets wrong element**     | Trace each XPath against the HTML — does it uniquely select the intended element and ONLY that element? |
| **State locator matches wrong state** | SELECTED locator matches NOT_SELECTED (wrong aria value); DISABLED matches enabled (predicate inverted) |
| **Missing inventory element**         | An element in the Phase 2 table has no variable in the class                                            |
| **Invented element**                  | A variable exists for an element not in the Phase 2 table                                               |

Print the bug report before finalizing:

```
[ClassName] — Phase 6 Bug Report:

BUG #1
  Category  : [XPath Logic / Python Code / Logical Correctness]
  Location  : [variable or method name]
  Bug       : [exact description]
  Root cause: [why this would fail at runtime]
  Fix       : [corrected code]

(repeat for each bug)

Summary:
  XPath Logic bugs fixed        : X
  Python Code bugs fixed        : X
  Logical Correctness bugs fixed: X
  Total                         : X

✅ Zero bugs detected — class is ready for final output.
```

If a bug cannot be auto-fixed without additional information → STOP, describe what is
needed, and wait.

---

## Execution order — do not deviate

```
Step 1  → Read reference POM → print style ruleset
Step 2  → Read HTML → extract all real attributes and state indicators
Step 3  → Read screenshot → confirm visible elements and states
Step 4  → Print element inventory (confirmed states only)
Step 5  → Print decision log (one line per element — which level and why)
Step 6  → Write ONE XPath per confirmed state following priority levels
Step 7  → Generate POM class with inline comments on Level 5/6 locators
Step 8  → Run Phase 5 cross-check → print with level usage summary
Step 9  → Run Phase 6 bug analysis → fix all bugs → print bug report
Step 10 → Write final corrected file to ${input:targetFile}
```

---

## Hard stops

| Trigger                                                | Action                                                      |
| ------------------------------------------------------ | ----------------------------------------------------------- |
| HTML missing or unreadable                             | STOP — do not guess any XPath                               |
| Reference POM missing                                  | STOP — do not invent conventions                            |
| Element has no unique anchor of any kind               | STOP — flag, propose approach, wait                         |
| State requested with no HTML evidence                  | STOP — flag, ask whether to skip                            |
| Only available anchor is a hashed/auto-generated class | STOP — flag and recommend adding `@data-testid` to the HTML |
| Phase 6 bug cannot be auto-fixed                       | STOP — describe what information is needed                  |
