---
description: 'Cookbook for writing pytest-bdd step definitions: file skeleton, bdd_driver,
  scenarios() binding, parsers, try/except contract.'
name: step_definitions
---

# Skill 01 — Step Definitions (the test-def cookbook)

Read [../framework_architecture/SKILL.md](../framework_architecture/SKILL.md) first. This skill
shows the **exact, copy-pasteable shapes** for step-definition files in this repo. Every
snippet here mirrors real `test_cmo-*.py` files.

> **🧠 ANTI-HALLUCINATION CONTRACT (STRICT):**
> Step definitions MUST NOT contain raw or guessed XPaths. All UI locators MUST be referenced from Page Object classes (`[(By.XPATH, "...")]`). Every `@then` step assertion MUST perform empirical checks against live DOM content returned by `CustomPyAutoWeb` verification methods — **never mock or fake step passes**.

---

## 1. File skeleton (always this order)

```python
# pylint: disable=all
# =============================================================================
# <JIRA-ID> — <short scenario theme>
# =============================================================================
import pytest
from pytest_bdd import given, when, then, scenarios, parsers
from selenium.webdriver.common.by import By
import allure
import os

from business_components.web_components.login import login_to_application_sso
from business_components.web_components.cmo import *
from object_repository.pages.Apps.CMO.CmoLandingPage import CMOLandingPage
from tests.step_defs.conftest import bdd_driver
from utilities.genericUtils import CustomPyAutoWeb
from utilities.helper import load_test_data_from_json

# Bind this module to its feature file (relative to this file's folder)
scenarios(
    os.path.join(
        os.path.dirname(__file__), "../../../features/Apps/CMO/cmo-AMCC-XXXX.feature"
    )
)

# Test data + cross-step state
json_data = load_test_data_from_json("../../testData/cmo.json")
_ctx = {}

# ============================================================================
# Background Step
# ============================================================================
@given("login to the Cold Compass application")
def login_to_americold_compass(bdd_driver):
    try:
        login_to_application_sso(bdd_driver)
        allure.attach(
            "Logged in to Americold Compass",
            name="Precondition — Login to Cold Compass",
            attachment_type=allure.attachment_type.TEXT,
        )
    except Exception as e:
        raise AssertionError(
            f"Step FAILED: 'login to the Cold Compass application' - "
            f"Error occurred in step [login_to_americold_compass]. Details: {e}"
        ) from e
```

> ⚠️ **pytest-bdd resolves steps per test module.** A hyphenated sibling file
> (`test_cmo-AMCC-8490.py`) **cannot** be imported. So shared steps are **redefined verbatim**
> in each file that needs them, tagged with a `# REUSED FROM:` comment (see §6). This is the
> repo's deliberate convention — do not try to `import` steps from another `test_*.py`.

---

## 2. The non-negotiable step contract (every step uses this)

Every step body follows the **same five-part shape**:

```python
@when('Click on "Start New Audit" button')                       # 1. decorator == feature text
def click_start_new_audit_button(bdd_driver):                    # 2. takes bdd_driver
    try:
        web = CustomPyAutoWeb(bdd_driver)                        # 3. build the helper
        web.wait_element_visibility(                             # 4. wait → act via POM locators
            locatorList=MyAuditsLandingPage.locatorStartNewAuditButton, wait_time=60
        )
        web.click_wait_locator(
            locatorList=MyAuditsLandingPage.locatorStartNewAuditButton
        )
        allure.attach(                                          # 5a. Allure evidence
            "Clicked 'Start New Audit' button",
            name="Action — Click Start New Audit",
            attachment_type=allure.attachment_type.TEXT,
        )
    except Exception as e:                                       # 5b. fail loud, never silent
        raise AssertionError(
            f"Step FAILED: 'Click on \"Start New Audit\" button' - "
            f"Error occurred in step [click_start_new_audit_button]. Details: {e}"
        ) from e
```

Rules baked into that shape:

- **Decorator text is byte-identical** to the feature step (mind the quotes — escape inner
  `"` in the failure message: `\"Start New Audit\"`).
- The function **never** calls `driver.find_element` directly — it goes through
  `CustomPyAutoWeb` + a **page-object locator**.
- The `except` **re-raises** as `AssertionError` with `Step FAILED: '<exact step text>' - Error
occurred in step [<func_name>]. Details: {e}`. Never swallow an exception.
- **No `time.sleep`.** Use `web.wait_element_visibility(...)` / `web.wait(n)`.

---

## 3. Decorator forms

```python
@given("login to the Cold Compass application")          # plain Given (Background)
@when('Navigate to "Apps > CMO"')                         # plain When, literal quotes in text
@then("CMO landing page should be displayed")             # plain Then
@then(parsers.parse('Create a template from "{template_data}" and publish it'))  # parameterized
```

- **Plain string** when the Gherkin step has no captured parameter.
- **`parsers.parse('… "{name}" …')`** when it does. The placeholder name becomes the function
  argument, **after** `bdd_driver`:

```python
@when(parsers.parse('Enter keyword "{keyword}" and submit search in Recent Audits'))
def enter_keyword_and_search(bdd_driver, keyword):
    ...
```

- `Scenario Outline` `<columns>` arrive through the **same** `parsers.parse('… "{col}" …')`
  mechanism — the `Examples` column header must equal the placeholder name.

---

## 4. Given / When / Then templates

### @given — precondition / setup

```python
@given("login to the Cold Compass application")
def login_to_americold_compass(bdd_driver):
    try:
        login_to_application_sso(bdd_driver)
        allure.attach("Logged in to Americold Compass",
                      name="Precondition — Login to Cold Compass",
                      attachment_type=allure.attachment_type.TEXT)
    except Exception as e:
        raise AssertionError(
            f"Step FAILED: 'login to the Cold Compass application' - "
            f"Error occurred in step [login_to_americold_compass]. Details: {e}") from e
```

### @when — action (delegate reusable flows to the web_component)

```python
@when('Navigate to "Apps > CMO"')
def navigate_to_cmo(bdd_driver):
    try:
        CMO_Navigation(bdd_driver)            # ← web_component function, not inline Selenium
        allure.attach(f"Navigated to CMO\nURL: {bdd_driver.current_url}",
                      name="Action — Navigate to Apps > CMO",
                      attachment_type=allure.attachment_type.TEXT)
    except Exception as e:
        raise AssertionError(
            f"Step FAILED: 'Navigate to \"Apps > CMO\"' - "
            f"Error occurred in step [navigate_to_cmo]. Details: {e}") from e
```

### @when — parameterized action storing state for a later @then

```python
@when(parsers.parse('Search "{search_text}" in the search bar'))
def search_in_search_bar(bdd_driver, search_text):
    try:
        search_template_in_flyer(bdd_driver, search_text)   # web_component
        _ctx["last_search"] = search_text                   # share with @then via _ctx
        allure.attach(f"Searched for: '{search_text}'",
                      name="Action — Search Template",
                      attachment_type=allure.attachment_type.TEXT)
    except Exception as e:
        raise AssertionError(
            f"Step FAILED: 'Search \"{search_text}\" in the search bar' - "
            f"Error occurred in step [search_in_search_bar]. Details: {e}") from e
```

### @then — verification (assertion lives here; see Skill 05)

```python
@then("CMO landing page should be displayed")
def verify_cmo_landing_page(bdd_driver):
    try:
        web = CustomPyAutoWeb(bdd_driver)
        web.wait_element_visibility(locatorList=CMOLandingPage.locatorCMOHeading, wait_time=60)
        web.element_should_be_visible(locator=CMOLandingPage.locatorCMOHeading)   # raises on fail
        allure.attach("CMO landing page is displayed — CMO heading visible",
                      name="Verification — CMO Landing Page Displayed",
                      attachment_type=allure.attachment_type.TEXT)
    except Exception as e:
        raise AssertionError(
            f"Step FAILED: 'CMO landing page should be displayed' - "
            f"Error occurred in step [verify_cmo_landing_page]. Details: {e}") from e
```

---

## 5. Reading test data — STRICT RULES (ZERO TOLERANCE)

### The testCaseId Pattern (MANDATORY for all new tests)

**ALL test data MUST flow from JSON through the Examples table using the `testCaseId` pattern:**

```gherkin
# Feature file — tests/features/Apps/CMO/cmo-AMCC-8438.feature
Scenario Outline: Verify template creation with help text
  Given login to the Cold Compass application
  When Navigate to "Apps > CMO"
  And the user creates a template using test data "<testCaseId>"
  Then the template should be created successfully

  Examples:
    | testCaseId    |
    | AMCC-TC-11035 |
    | AMCC-TC-11036 |
```

```python
# Step definition — tests/step_defs/apps/CMO/test_cmo-AMCC-8438.py
json_data = load_test_data_from_json("../../testData/cmo.json")   # module top

@when(parsers.parse('the user creates a template using test data "{testCaseId}"'))
def create_template_from_test_data(bdd_driver, testCaseId):
    try:
        td = json_data.get(testCaseId, {})        # testCaseId == AMCC-TC-11035
        template_name = td.get("templateName", "")
        description = td.get("description", "")
        # Use td values, NEVER hardcode
        create_template(bdd_driver, template_name, description)
        _ctx["created_template"] = template_name
        allure.attach(f"Created template: {template_name}",
                      name="Action — Create Template",
                      attachment_type=allure.attachment_type.TEXT)
    except Exception as e:
        raise AssertionError(
            f"Step FAILED: 'the user creates a template using test data \"{testCaseId}\"' - "
            f"Error occurred in step [create_template_from_test_data]. Details: {e}") from e
```

```json
// testData/cmo.json
{
  "AMCC-TC-11035": {
    "templateName": "Test Template {timestamp}",
    "description": "Automated test template",
    "expectedMessage": "Template created successfully"
  },
  "AMCC-TC-11036": {
    "templateName": "Another Template {timestamp}",
    "description": "Second test template",
    "expectedMessage": "Template created successfully"
  }
}
```

### STRICT RULES (ZERO TOLERANCE VIOLATIONS)

| Rule      | ❌ FORBIDDEN                                              | ✅ REQUIRED                                         |
| --------- | --------------------------------------------------------- | --------------------------------------------------- |
| Locators  | `bdd_driver.find_element(By.XPATH, "//button")`           | `web.click_element(locatorList=PAGE.locatorButton)` |
| Locators  | `web.click_element(locatorList=[(By.XPATH, "//button")])` | `web.click_element(locatorList=PAGE.locatorButton)` |
| Test data | `template_name = "My Template"`                           | `template_name = td.get("templateName", "")`        |
| Test data | `expected = "Success message"`                            | `expected = td.get("expectedMessage", "")`          |
| Feature   | `When the user enters "test@email.com"`                   | `When the user enters data from "<testCaseId>"`     |

### Pre-submission validation

Before creating/modifying any step definition file, validate:

```python
# ❌ FAIL — hardcoded locator in step def
def some_step(bdd_driver):
    bdd_driver.find_element(By.XPATH, "//button[@id='submit']").click()

# ❌ FAIL — inline locator list in step def
def some_step(bdd_driver):
    web.click_element(locatorList=[(By.XPATH, "//button[@id='submit']")])

# ❌ FAIL — hardcoded test data
def some_step(bdd_driver):
    template_name = "Test Template 123"

# ✅ PASS — POM locator reference
def some_step(bdd_driver):
    web = CustomPyAutoWeb(bdd_driver)
    web.click_element(locatorList=MyPage.locatorSubmitButton)

# ✅ PASS — JSON test data
def some_step(bdd_driver, testCaseId):
    td = json_data.get(testCaseId, {})
    template_name = td.get("templateName", "")
```

### Self-check for data standards

- [ ] Feature file uses `Examples: | testCaseId | AMCC-TC-XXXXX |` pattern
- [ ] Step def loads data via `json_data.get(testCaseId, {})`
- [ ] testData/<module>.json has entries keyed by AMCC-TC-XXXXX
- [ ] Zero hardcoded strings for names, emails, descriptions, expected values
- [ ] All locators reference `PAGE.locatorXxx` from POM classes
- [ ] No `By.XPATH`, `By.CSS`, or `By.ID` appears in step def code

---

## 6. REUSE rule (per-module redefinition)

Because steps can't be imported across hyphenated modules, a shared step is **copied** with a
provenance comment — never invented twice with different text:

```python
# REUSED FROM: test_cmo-AMCC-8490.py — step "Navigate to \"Apps > CMO\"".
# Redefined here because pytest-bdd resolves steps per module and the hyphenated
# sibling file cannot be imported.
@when('Navigate to "Apps > CMO"')
def navigate_to_cmo(bdd_driver):
    ...
```

When the original step delegates to a web_component, the **copy delegates to the same
web_component function** — so the actual logic still lives in exactly one place.

---

## 7. Data-table steps

```python
@when("the user fills the form")
def fill_form(bdd_driver, datatable):     # pytest-bdd injects the table as `datatable`
    web = CustomPyAutoWeb(bdd_driver)
    for row in datatable[1:]:             # row[0] is the header row
        field, value = row[0], row[1]
        ...
```

Match the iteration style of the nearest existing table-handling step in the module.

---

## 8. Strict verification — NO soft-verify, ever

Every `@then` step **must perform a real check**: a `CustomPyAutoWeb` verify method that raises
(preferred) or a bare `assert` with a multi-line message (§4 and Skill 05). This repo has **no
"soft-verify"** — never write an Allure-only `@then` that attaches evidence without asserting,
and never fabricate a passing result for a check you cannot actually perform.

```python
# ❌ FORBIDDEN — an Allure-only "soft" check that asserts nothing
@then("Help section should be in read only format")
def help_read_only(bdd_driver):
    allure.attach("looks read-only", name="...",
                  attachment_type=allure.attachment_type.TEXT)

# ✅ REQUIRED — a real assertion via a POM locator + CustomPyAutoWeb verify
@then("Help section should be in read only format")
def help_read_only(bdd_driver):
    try:
        web = CustomPyAutoWeb(bdd_driver)
        web.wait_element_visibility(locatorList=PAGE.locatorHelpSectionReadOnly, wait_time=60)
        web.element_should_be_visible(locator=PAGE.locatorHelpSectionReadOnly)   # raises on fail
        allure.attach("Help section is read-only — read-only marker visible",
                      name="Verification — Help Section Read-Only",
                      attachment_type=allure.attachment_type.TEXT)
    except Exception as e:
        raise AssertionError(
            f"Step FAILED: 'Help section should be in read only format' - "
            f"Error occurred in step [help_read_only]. Details: {e}") from e
```

If the locator for a check genuinely does not exist in the POM/HTML, **STOP and ask** for it
(AGENTS.md §11) — do **not** downgrade the step to a soft-verify or skip the
assertion. Every step is verified strictly.

---

## 9. Step-def self-check (run before finishing)

- [ ] Decorator text equals the feature step **exactly** (quotes/spacing/params).
- [ ] Function takes `bdd_driver`; parameter names equal the `parsers.parse` placeholders.
- [ ] No `driver.find_element` / raw locator in the step — only `CustomPyAutoWeb` + POM locators.
- [ ] Reusable multi-step flow is delegated to a web_component function, not inlined.
- [ ] `try/except … raise AssertionError("Step FAILED: '<step>' - … [func]. Details: {e}") from e`.
- [ ] `@when` attaches an **Action —** evidence; `@then` attaches a **Verification —** evidence.
- [ ] Every `@then` performs a **real** check (CustomPyAutoWeb verify or bare assert) — no soft-verify / Allure-only steps.
- [ ] Cross-step values go through `_ctx` / web*component `get*`/`set\_`, never globals you invent.
- [ ] No `time.sleep`. No hardcoded test data. No duplicate step text across the module.
