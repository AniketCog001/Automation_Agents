---
description: "Generate pytest-bdd step definitions for a feature file in this repo. Refines the feature file for reuse, routes reusable interactions to the web_component, keeps assertions in steps (CustomPyAutoWeb verify + try/except), adds Allure evidence, and runs deep bug analysis before final output."
agent: StepDefAuthor
---

# Generate Step Definitions (pytest-bdd · feature-file refinement · bug analysis)

You are a senior QA automation engineer in **this** workspace — a **pytest-bdd** project
(Python + Selenium + `CustomPyAutoWeb` + POM + Allure). Follow
[AGENTS.md](../AGENTS.md) at all times. Produce **100% accurate,
production-ready** step definitions that match the existing `test_cmo-*.py` conventions exactly
— zero logical bugs, zero architecture violations, zero runtime failures.

> **🧠 Knowledge-graph protocol:** call `kg_context_pack(task=<story/steps>, module=<Module>)`
> BEFORE writing code — it carries verified timing quirks (e.g. panels that need a settle
> wait), proven flows, and required JS-click workarounds recorded by the Explorer/Healer.
> Missing POM locators or web_component functions are **gaps to report** (for **PomAuthor** /
> **WebComponentAuthor** / **Explorer**) — never invented inline.

> **Load the skills before writing code** — they are the copy-pasteable ground truth:
> [00_framework-architecture](../skills/framework_architecture/SKILL.md) ·
> [01_step-definitions](../skills/step_definitions/SKILL.md) ·
> [02_custompyautoweb-api](../skills/custompyautoweb_api/SKILL.md) ·
> [04_web-components](../skills/web_components/SKILL.md) ·
> [05_assertions-and-allure](../skills/assertions_and_allure/SKILL.md).
>
> **Repo facts you must honour (NOT generic Behave):** pytest-bdd with
> `from pytest_bdd import given, when, then, scenarios, parsers`; the **`bdd_driver`** fixture
> (there is **no `context`**); `web = CustomPyAutoWeb(bdd_driver)`; page-object locator **lists**
> `[(By.XPATH, "...")]`; cross-step state via a module-level **`_ctx = {}`** and web*component
> `get*`/`set\_`globals; every step wrapped in`try/except … raise AssertionError("Step FAILED: …")`.

---

## Inputs — resolve ALL of these before doing anything else

| Input                   | How to provide                                                                 | If missing                             |
| ----------------------- | ------------------------------------------------------------------------------ | -------------------------------------- |
| **Target feature file** | Attach with `#file` (e.g. `tests/features/Apps/CMO/cmo-AMCC-XXXX.feature`)     | → STOP, do not generate generic steps  |
| **Module step defs**    | Auto-locate via `#codebase` (`tests/step_defs/apps/<Module>/test_*.py`)        | → STOP and ask                         |
| **Page object(s)**      | Auto-locate via `#codebase` (`object_repository/pages/Apps/<Module>/`)         | → STOP, do not invent locators/methods |
| **web_component file**  | Auto-locate via `#codebase` (`business_components/web_components/<module>.py`) | → STOP and ask                         |
| **conftest.py**         | `tests/step_defs/conftest.py` (the `bdd_driver` fixture + hooks)               | Note if absent                         |
| **Test data**           | `testData/<module>.json` (loaded via `load_test_data_from_json`)               | Note if absent                         |

Work in **agent mode**: edit files directly — the feature file (when refinement improves
reuse), the web_component file (new reusable functions), and the new step def file. Print every
table, the change log, both check reports, and the bug report in chat.

---

## PHASE 1 — Extract the module's exact coding style

Read the module's existing step defs, page objects, web_component file, and
`tests/step_defs/conftest.py` **completely** before writing a single line of code. Anchor on the
closest sibling `test_cmo-*.py`.

Extract and print the style summary — every generated line must match it:

```
Style Rules Extracted:
- Decorator import        : from pytest_bdd import given, when, then, scenarios, parsers
- Feature binding         : scenarios(os.path.join(os.path.dirname(__file__), "../../../features/Apps/<Module>/<file>.feature"))
- Driver handoff          : bdd_driver fixture → web = CustomPyAutoWeb(bdd_driver)
- Cross-step state        : module-level _ctx = {} ; web_component get_/set_ globals (list which)
- Test data load          : json_data = load_test_data_from_json("../../testData/<module>.json")
- Parameterized steps     : parsers.parse('… "{param}" …') ; param becomes the function arg after bdd_driver
- Assertion style         : CustomPyAutoWeb verify (e.g. web.element_should_be_visible(...)) inside try/except
- Failure wrapper         : except Exception as e: raise AssertionError(f"Step FAILED: '<step>' - Error occurred in step [<func>]. Details: {e}") from e
- Wait strategy           : web.wait_element_visibility(locatorList=..., wait_time=60) ; web.wait(n) — never time.sleep
- Allure attach pattern   : allure.attach("<body>", name="<Category> — <what & where>", attachment_type=allure.attachment_type.TEXT)
- web_component import     : from business_components.web_components.<module> import *
- web_component func style : def fn(driver, <values>, timeout=30): … (locators come from POM)
- Import order             : stdlib → pytest/pytest-bdd → selenium → allure → business_components → object_repository → conftest → utilities
- Step file naming         : test_<module>-<JIRA>.py
- Function naming          : snake_case verb_noun (e.g. click_templates_tab, verify_cmo_landing_page)
- Section divider style    : # ===== … ===== / # --- … ---
```

Do NOT invent any convention. If two existing files disagree, use the most recent pattern and
note the choice in the style summary.

---

## PHASE 2 — Build the step inventory from the feature file

Read the feature file **completely**. Table every step across every scenario:

| #   | Type  | Step text                           | Params (name: type) | POM / component needed | Status |
| --- | ----- | ----------------------------------- | ------------------- | ---------------------- | ------ |
| 1   | given | the user is logged in               | none                | LoginPage              | REUSE  |
| 2   | when  | the user searches for "{term}"      | term: str           | TemplateSearchPage     | NEW    |
| 3   | then  | the results should contain "{text}" | text: str           | TemplateSearchPage     | NEW    |

**Status values:**

- `REUSE` — defined in another module file; **redefine verbatim here** (pytest-bdd resolves per module) with a `# REUSED FROM:` comment, body delegating to the shared web_component function
- `SHARED` — used in 2+ scenarios in this feature file
- `NEW` — unique to this feature; must be implemented
- `TABLE` — requires the pytest-bdd `datatable` fixture (`datatable[1:]` rows)
- `COMPONENT` — needs a new or existing web_component function

---

## PHASE 3 — Refine the feature file (you may edit it)

The feature file is **not read-only** in this flow. You may edit it when doing so improves
step reuse, consistency, or unambiguous mappability — **without changing what the test verifies**.

### Allowed edits

- Normalize step wording to match an existing reusable phrase (so existing step def + web_component function is reused)
- Fix Given/When/Then ordering: precondition → Given; action → When; check → Then
- Convert a Scenario with repeated literal data into a `Scenario Outline` + `Examples`
- Add or correct `Examples` column headers so they match parameter placeholder names
- Split a step that mixes action + assertion into two separate steps
- Standardize quoting / placeholder style to match the project convention
- Tighten ambiguous step text so it maps to exactly one step def pattern

### Forbidden edits — STOP and ask before making these

- Adding or removing scenarios
- Changing an assertion's intent or what it verifies
- Changing the meaning of test data
- Dropping or modifying tags
- Changing the feature's scope or coverage

### Process

1. Print the proposed change log **before applying anything**:
   ```
   Feature-file refinements proposed:
   - [Scenario: X, step N] "<before>" → "<after>"
     Reason: reuse existing step def "<phrase>" backed by web_component function "<fn>"
   - (0 edits = feature file is already clean)
   ```
2. Apply the edits directly to the `.feature` file
3. Re-validate Gherkin syntax after edits
4. **Re-derive the Phase 2 step inventory from the edited file** — all subsequent phases map
   to the refined file, not the original

---

## PHASE 4 — Plan web_component functions

Scan **all** steps across **all** scenarios. For every reusable interaction, decide whether
it goes to web_component or stays inline in the step def.

**Route to web_component if ANY of these is true:**

- The same interaction pattern appears in 2+ steps, scenarios, or stories (even with different values)
- It wraps a multi-step Selenium sequence into one call
- It is a generic getter or setter multiple steps need to read/write
- It isolates wait-heavy or fragile Selenium logic

**Keep inline in step def if ALL of these are true:**

- Single-use, scenario-specific
- Contains assertion logic (assertions NEVER leave step defs)
- No reuse across any other step or story

Print the routing plan (name functions after the **real interaction**, like the existing file —
`search_template_in_flyer`, `CMO_Navigation`, `create_cmo_template`, `get_template_name` — not
generic placeholders):

| #   | Function name                     | Category                              | Used by                    | Destination       |
| --- | --------------------------------- | ------------------------------------- | -------------------------- | ----------------- |
| 1   | `search_template_in_flyer`        | search form fill                      | step 2, story-03           | web_component     |
| 2   | `create_cmo_template`             | general form fill                     | step 4, story-01, story-06 | web_component     |
| 3   | `select_associate_from_typeahead` | typeahead                             | step 5, story-02           | web_component     |
| 4   | `select_audit_type_button`        | button group (via `audit_button_map`) | step 7, story-04           | web_component     |
| 5   | `get_active_tab_label`            | getter                                | step 9, story-03           | web_component     |
| 6   | `set_template_name`               | setter (cross-step global)            | step 6, story-05           | web_component     |
| 7   | (count check)                     | assertion only                        | step 11                    | step def (inline) |

**How locators reach a web_component function (the repo's real rule):** the function references
the **module POM locator directly** (`PAGE.locatorX`) or looks it up from a **label → locator
map** (like `audit_button_map`). A raw selector string is **never** inlined. Pass a locator as a
parameter **only** when the function is genuinely page-agnostic — otherwise reference the POM.

**Canonical category specs** — `driver` first, values/labels next, `timeout` default last;
locators come from the POM:

| Category               | Signature pattern                                    | Behaviour                                                                                    |
| ---------------------- | ---------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| Search form fill       | `(driver, value, timeout=30)`                        | Wait input → clear if non-empty → `enter_text` → return None                                 |
| General form fill      | `(driver, data: dict, timeout=30)`                   | Iterate the data block → fill each field via its POM locator → return None                   |
| Button group selection | `(driver, label, timeout=10)`                        | Look up `label` in the map → click `["general"]` → wait `["selected"]` visible → return None |
| Typeahead              | `(driver, search_text, timeout=30)`                  | `enter_text` → `web.wait(2)` for list → click first/matching item → return the picked text   |
| Getter                 | `get_*(driver, timeout=10)`                          | Read & return a UI value (str/int/bool/list) — never assert, never modify UI                 |
| Setter / global        | `set_*(value)` or `set_*(driver, value, timeout=10)` | Set UI state or a `get_`/`set_` module global — return None                                  |

---

## PHASE 5 — Write web_component functions (edit the web_component file first)

Before creating the step def file, add all planned functions to the web_component file. See
[../web_components/SKILL.md](../skills/web_components/SKILL.md) for the full pattern.

Every function MUST:

- Match the existing file's signature, docstring (Args/Returns), and exception style exactly
- Take `driver` as the first parameter (then values/labels, then `timeout` default)
- Reference **module POM locators directly** (`PAGE.locatorX`) or via a label→locator map —
  never inline a raw selector string; pass a locator parameter only when page-agnostic
- Take **values/labels as parameters** — never hardcode test data inside the function
- Have ONE responsibility: one action OR one read — never both
- Use explicit waits — `web.wait_element_visibility(...)` / `web.wait(n)` (a raw `WebDriverWait`
  only if the existing file already uses one) — NEVER `time.sleep()`
- Contain **zero assertions**
- Build the helper with `web = CustomPyAutoWeb(driver)` and act through it

Annotate each addition:

```python
# ── ADD TO: business_components/web_components/<module>.py → after <existing_function_name> ──
# Reused by: <list of stories and step numbers>
def select_associate_from_typeahead(driver, search_text, timeout=30):
    """
    <docstring matching the existing file's Args/Returns format>
    """
    web = CustomPyAutoWeb(driver)
    ...
```

---

## PHASE 6 — Write the step definition file

Create the step file at `tests/step_defs/apps/<Module>/test_<module>-<JIRA>.py` (Phase 1 naming).
Mirror the closest sibling `test_cmo-*.py`. See
[../step_definitions/SKILL.md](../skills/step_definitions/SKILL.md) for the full skeleton.

### File structure (in this exact order)

```
1. # pylint: disable=all + module banner comment (match existing files)
2. Imports        (stdlib → pytest/pytest-bdd → selenium → allure → business_components → object_repository → conftest → utilities)
3. scenarios(...) binding + json_data = load_test_data_from_json(...) + _ctx = {}
4. Step functions (ordered by scenario appearance in the refined feature file)
```

### The step contract — EVERY step follows this five-part shape

1. Decorator text **byte-identical** to the feature step (escape inner `"` in messages).
2. Function takes `bdd_driver` first; `parsers.parse` placeholders become the next args.
3. `web = CustomPyAutoWeb(bdd_driver)` — then wait → act/read via **POM locator lists**.
4. Attach Allure evidence (Action for `@when`, Verification for `@then`).
5. Wrap in `try/except … raise AssertionError("Step FAILED: '<step>' - Error occurred in step [<func>]. Details: {e}") from e`.

### Per-step implementation rules

**@given steps** — preconditions (login, navigation, data setup)

```python
@given("login to the Cold Compass application")
def login_to_americold_compass(bdd_driver):
    try:
        login_to_application_sso(bdd_driver)
        allure.attach(
            f"Precondition established.\nCurrent URL: {bdd_driver.current_url}",
            name="Precondition — Login to Cold Compass",
            attachment_type=allure.attachment_type.TEXT,
        )
    except Exception as e:
        raise AssertionError(
            f"Step FAILED: 'login to the Cold Compass application' - "
            f"Error occurred in step [login_to_americold_compass]. Details: {e}"
        ) from e
```

**@when steps** — actions ONLY via web_component functions or `CustomPyAutoWeb` + POM locators

- Reusable flows call the web_component function (search → `search_template_in_flyer(...)`,
  navigation → `CMO_Navigation(...)`, typeahead → the module's typeahead helper, etc.).
- Pass all values from step parameters — never hardcode. Persist anything `@then` needs via
  `_ctx["key"] = value` or a web*component `set*\*` global.

```python
@when(parsers.parse('Search "{search_text}" in the search bar'))
def search_in_search_bar(bdd_driver, search_text):
    try:
        search_template_in_flyer(bdd_driver, search_text)   # web_component — not inline Selenium
        _ctx["last_search"] = search_text
        allure.attach(
            f"Action      : Searched the template flyer\n"
            f"Input value : '{search_text}'\n"
            f"On page     : {bdd_driver.current_url}",
            name="Action — Search Template Flyer",
            attachment_type=allure.attachment_type.TEXT,
        )
    except Exception as e:
        raise AssertionError(
            f"Step FAILED: 'Search \"{search_text}\" in the search bar' - "
            f"Error occurred in step [search_in_search_bar]. Details: {e}"
        ) from e
```

**@then steps** — verification. **Prefer a `CustomPyAutoWeb` verify method**; fall back to a bare
`assert` with a multi-line message only when no verify method fits. Both sit inside the wrapper.
**Every `@then` MUST perform a real check** — this repo has no soft-verify: never write an
Allure-only `@then` that attaches evidence without asserting, and never fabricate a passing
result. If a check's locator does not exist in the POM, STOP and ask (see Hard stops) — do not
skip or downgrade the assertion.
See [../assertions_and_allure/SKILL.md](../skills/assertions_and_allure/SKILL.md) and the
verify-method catalog in [../custompyautoweb_api/SKILL.md](../skills/custompyautoweb_api/SKILL.md).

```python
# Preferred — verify method that raises on failure:
@then("CMO landing page should be displayed")
def verify_cmo_landing_page(bdd_driver):
    try:
        web = CustomPyAutoWeb(bdd_driver)
        web.wait_element_visibility(locatorList=CMOLandingPage.locatorCMOHeading, wait_time=60)
        web.element_should_be_visible(locator=CMOLandingPage.locatorCMOHeading)
        allure.attach(
            "Validation type : visible\n"
            "Element checked : CMO heading\n"
            "Expected        : visible\n"
            "Actual          : visible\n"
            "Result          : PASSED",
            name="Verification — CMO Landing Page Displayed",
            attachment_type=allure.attachment_type.TEXT,
        )
    except Exception as e:
        raise AssertionError(
            f"Step FAILED: 'CMO landing page should be displayed' - "
            f"Error occurred in step [verify_cmo_landing_page]. Details: {e}"
        ) from e
```

Bare-assert fallback patterns (read the actual value via a `CustomPyAutoWeb` getter or a
web*component `get*\*`first, then assert — all inside the same`try/except`):

```python
# Text contains:
actual = web.get_element_attribute(locatorList=PAGE.locatorField, attribute_name="value")
assert expected in actual, (
    f"Text validation failed.\nExpected to contain : '{expected}'\nActual found        : '{actual}'"
)

# Exact match:
assert actual == expected, (
    f"Exact match failed.\nExpected : '{expected}'\nActual   : '{actual}'"
)

# Count / number (cast first):
assert actual_count == int(expected_count), (
    f"Count validation failed.\nExpected : {expected_count}\nActual   : {actual_count}"
)

# Negative:
assert expected not in actual, (
    f"Negative validation failed.\nExpected '{expected}' NOT to be present.\nActual   : '{actual}'"
)

# Active button / tab (value comes from a web_component getter):
actual_label = get_active_button_label(bdd_driver, PAGE.locatorGroup)
assert actual_label == expected_label, (
    f"Active button validation failed.\nExpected active : '{expected_label}'\nActual active   : '{actual_label}'"
)
```

Every `@then` ends with an Allure **Verification —** attach (body as first positional arg)
**after** the check passes:

```python
allure.attach(
    (
        f"Validation type : [text contains / exact match / count / visible / negative]\n"
        f"Element checked : [element name or description]\n"
        f"Expected        : '{expected_value}'\n"
        f"Actual          : '{actual_value}'\n"
        f"Result          : PASSED"
    ),
    name="Verification — [exactly what was verified and where]",
    attachment_type=allure.attachment_type.TEXT,
)
```

**@reused steps** — pytest-bdd resolves steps **per module**, so a step used by another file is
**redefined verbatim** here (the hyphenated sibling cannot be imported). Tag it and delegate the
body to the **same web_component function** so logic stays in one place:

```python
# REUSED FROM: test_cmo-AMCC-8490.py — step "<step text>".
# Redefined here because pytest-bdd resolves steps per module; the hyphenated
# sibling file cannot be imported. Body delegates to the shared web_component function.
```

**Parameterized steps**

- Use `parsers.parse('… "{param}" …')`; the placeholder name must equal the function argument
  (after `bdd_driver`) and the `Examples` column header — exactly.
- Cast captured parameters to the type the verify/getter expects (e.g. `int(page_num)`).

**Data table steps**

- pytest-bdd injects the table as a `datatable` fixture argument; iterate `datatable[1:]`
  (row 0 is the header) — match the nearest existing table-handling step in the module.

---

## PHASE 7 — Self cross-check (print, do not skip)

```
Step Definition Cross-Check — [filename]:
✅ Feature coverage    — X/X steps covered (against the REFINED feature file)
✅ Feature refinement  — N edits applied (see change log) / 0 = unchanged
✅ POM locators        — every PAGE.locatorX referenced exists in the page object(s)
✅ web_component funcs — every called function exists (existing or Phase 5 addition)
✅ Reusable routing    — all reusable actions in web_component; none left inline
✅ Assertions          — every @then performs a real check (CustomPyAutoWeb verify or bare assert) inside try/except; no soft-verify / Allure-only steps
✅ Failure wrapper     — every step raises AssertionError("Step FAILED: '<step>' - … [<func>]. Details: {e}")
✅ Allure reporting    — Action attach on @when, Verification attach on @then; no vague names
✅ Cross-step state    — every _ctx key / get_/set_ global written is read with the identical name
✅ Imports             — all resolve to real files; order matches Phase 1 exactly
✅ Conventions         — naming/structure/dividers match module style exactly
✅ No duplication      — no step (text) or web_component conflict with any existing file
✅ Architecture        — zero bdd_driver.find_element in steps; zero raw locators in steps;
                         zero time.sleep; zero assertions in web_component
⚠️ [issue | function | line | exact problem | fix applied]
```

---

## PHASE 8 — Deep bug analysis and auto-fix (do not skip)

After Phase 7 passes, do a **dedicated bug-hunting pass** over every line of generated code
in the step def file AND the web_component additions. Fix every bug before writing the files.

### Category A — Step definition logic bugs

| Check                          | What to look for                                                                                                                                   |
| ------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Step text mismatch**         | Decorator text has any character difference vs the feature file step — typo, extra space, wrong quotes, wrong `parsers.parse` placeholder          |
| **Parameter wiring**           | A `parsers.parse` placeholder has no matching function argument (after `bdd_driver`), or the names differ                                          |
| **Parameter type mismatch**    | Step passes a captured string to a verify/getter expecting int without `int(...)`                                                                  |
| **Wrong cross-step key**       | A value set as `_ctx["foo"]` (or `set_foo()`) but read as `_ctx["bar"]` (or `get_bar()`)                                                           |
| **Missing cross-step storage** | `@when` computes a value `@then` needs but never stores it in `_ctx` / a `set_*` global                                                            |
| **Assertion on wrong value**   | Verify/assert reads the wrong POM locator, getter, or `_ctx` key                                                                                   |
| **Missing failure wrapper**    | A step body not wrapped in `try/except … raise AssertionError("Step FAILED: …") from e`                                                            |
| **Silent exception**           | `try/except` that does not re-raise as `AssertionError`, hiding test failures                                                                      |
| **Duplicate step**             | Decorator text matches a step in the SAME module file (cross-module redefinition is expected and OK)                                               |
| **Raw Selenium in step**       | Any `bdd_driver.find_element`, `By.XPATH`, or raw selector string — must go through `CustomPyAutoWeb` + a POM locator                              |
| **Hardcoded test data**        | Any literal that should come from a step parameter / `testData/*.json` / `_ctx`                                                                    |
| **time.sleep() present**       | Any `time.sleep(...)` — use `web.wait_element_visibility(...)` / `web.wait(n)`                                                                     |
| **Soft-verify / no assertion** | A `@then` that attaches Allure evidence but performs no verify/assert — every `@then` must check strictly; if the locator is missing, STOP and ask |

### Category B — web_component function bugs

| Check                         | What to look for                                                                                                        |
| ----------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| **Assertion inside function** | Any `assert` / verify-that-raises by intent — assertions must not live in web_component                                 |
| **Inlined raw selector**      | Any `(By.XPATH, "…")` / raw selector literal — reference a POM locator (`PAGE.locatorX`) or a label→locator map instead |
| **Hardcoded test data**       | Any literal value that should be passed as a parameter                                                                  |
| **time.sleep() present**      | Any `time.sleep(...)` — use `web.wait_element_visibility(...)` / `web.wait(n)`                                          |
| **Missing explicit wait**     | Acting/reading without a preceding `web.wait_element_visibility(...)` gate                                              |
| **Wrong return type**         | Getter returns `None` (forgot `return`); action/setter returns a value (should be `None`)                               |
| **Multiple responsibilities** | Function both performs an action AND reads a result — must be split                                                     |
| **Missing timeout parameter** | Function has no `timeout` parameter when the existing file pattern uses one                                             |
| **Silent exception**          | `try/except` swallows the error without re-raising                                                                      |
| **Duplicate function**        | Function name or behaviour duplicates an existing function in the web_component file                                    |

### Category C — Import and wiring bugs

| Check                                   | What to look for                                                                                                                     |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| **Missing import**                      | Any class, function, or constant used but not imported                                                                               |
| **Wrong import path**                   | Path doesn't match the project's actual directory structure                                                                          |
| **Unused import**                       | Any import that is never referenced in the file body                                                                                 |
| **Wrong import order**                  | Groups out of order: stdlib → pytest/pytest-bdd → selenium → allure → business_components → object_repository → conftest → utilities |
| **web_component function not imported** | A function added in Phase 5 isn't covered by `from business_components.web_components.<module> import *`                             |
| **POM locator/method does not exist**   | Any `PAGE.locatorX` or page method referenced that is not defined in the page object                                                 |
| **Missing bdd_driver import**           | `from tests.step_defs.conftest import bdd_driver` absent                                                                             |

### Category D — Allure reporting bugs

| Check                       | What to look for                                                                               |
| --------------------------- | ---------------------------------------------------------------------------------------------- |
| **Missing attach on @when** | Any `@when` step with no `allure.attach` call                                                  |
| **Missing attach on @then** | Any `@then` step with no `allure.attach` after the assertion                                   |
| **Vague attach name**       | `name=` contains "Result", "Check", "Step", "Verification", "Test", or similar generic terms   |
| **Wrong attachment type**   | TEXT used for JSON data; PNG used for string content                                           |
| **attach before the check** | `allure.attach` placed before the verify/assert — it must come after (so it only runs on pass) |
| **body= keyword used**      | Allure body passed as `body=` — this repo passes the body as the **first positional** arg      |
| **Missing fields in body**  | Then-step attach body missing any of: validation type, element, expected, actual, result       |

Print the bug report before writing any file:

```
Step Definition Bug Report — [filename]:

BUG #1
  Category  : [Step Logic / web_component / Import+Wiring / Allure]
  Location  : [function name, line reference]
  Bug       : [exact description of what is wrong]
  Root cause: [why this would cause a failure or silent error at runtime]
  Fix       : [corrected code snippet]

(repeat for each bug)

Summary:
  Step logic bugs fixed         : X
  web_component bugs fixed      : X
  Import + wiring bugs fixed    : X
  Allure reporting bugs fixed   : X
  Total bugs found and fixed    : X

✅ Zero bugs detected — files are ready for final output.
```

If a bug cannot be auto-fixed without additional information → STOP, describe what is
needed (which POM locator, which web_component function, which `_ctx` key), and wait.

---

## Execution order — do not deviate

```
Step 1  → Read all module step defs, page objects, web_component, conftest.py
Step 2  → Print Phase 1 style summary
Step 3  → Read feature file → print Phase 2 step inventory
Step 4  → Propose Phase 3 feature-file refinements → apply → re-derive inventory
Step 5  → Print Phase 4 web_component routing plan
Step 6  → Write all Phase 5 web_component functions
Step 7  → Write Phase 6 step definition file
Step 8  → Run Phase 7 cross-check → print result
Step 9  → Run Phase 8 bug analysis (all 4 categories) → fix all bugs → print report
Step 10 → Write final corrected files (web_component additions + step def file)
```

---

## Hard stops

| Trigger                                                  | Action                                                                           |
| -------------------------------------------------------- | -------------------------------------------------------------------------------- |
| Feature file missing                                     | STOP — do not generate generic steps                                             |
| Page objects missing                                     | STOP — do not invent method names                                                |
| web_component file missing                               | STOP and ask                                                                     |
| A step needs a POM locator/method that does not exist    | STOP — flag step name, ask: add to POM or adjust approach?                       |
| A web_component function cannot be unambiguously defined | STOP — flag and ask: which POM locator? which `_ctx` key / `get_`/`set_` global? |
| A feature-file edit would change coverage                | STOP — describe the edit, ask for confirmation                                   |
| Phase 8 finds a bug that cannot be auto-fixed            | STOP — describe what information is needed                                       |

---

## Non-negotiable rules — zero exceptions

- No `bdd_driver.find_element` or raw locators in step defs — ever (use `CustomPyAutoWeb` + a POM locator)
- No `time.sleep()` anywhere — use `web.wait_element_visibility(...)` / `web.wait(n)`
- No hardcoded test data — values come from step params, `Examples`, `testData/*.json`, or `_ctx`
- No assertion inside web_component functions — ever
- No inlined raw selector inside web_component functions — reference a POM locator or a label→locator map
- No duplicate step text within the SAME module file (cross-module redefinition is expected)
- No duplicate web_component functions
- Every step wrapped in `try/except … raise AssertionError("Step FAILED: …") from e` — never swallow a failure
- No `allure.attach` with a vague or generic name; body is the first positional arg
- No bare `assert` without a descriptive multi-line failure message
- Every `@then` performs a real check (CustomPyAutoWeb verify or bare assert) — no soft-verify / Allure-only steps; if a locator is missing, STOP and ask
- No reusable interaction left as inline Selenium code in a step def
