---
description: How to turn a live Selenium-MCP walk into a detailed, framework-correct
  pytest-bdd test.
name: authoring_bdd_from_mcp
---

# Skill — Authoring pytest-bdd tests from an MCP walk

After you walk a test case live through the `selenium` MCP, you author it as a real,
framework-correct pytest-bdd test. This skill is the **bridge**: it shows exactly which file each
piece lands in and gives copy-pasteable templates that match the existing `test_cmo-*.py` style.

> The repo cookbook is the ground truth — read it alongside this skill:
> [00_framework-architecture](../framework_architecture/SKILL.md) ·
> [01_step-definitions](../step_definitions/SKILL.md) ·
> [02_custompyautoweb-api](../custompyautoweb_api/SKILL.md) ·
> [03_page-objects](../page_objects/SKILL.md) ·
> [04_web-components](../web_components/SKILL.md) ·
> [05_assertions-and-allure](../assertions_and_allure/SKILL.md).

---

## 0. Author by the repo's canonical prompts (for accuracy)

This repo already has battle-tested authoring prompts at the **framework root** — treat them as
the **rulebook** for each layer so the generated code is byte-accurate to the repo's conventions.
Apply them in this order (it mirrors the repo's own pipeline):

| Order | Prompt (framework root)   | Produces                                                                                                             |
| ----- | ------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| 1     | `feature_file_prompt.txt` | the `.feature` Scenario(s), matching the existing module's style                                                     |
| 2     | `xpath_prompt.txt`        | the POM locators (text→aria→testid priority; one per confirmed state)                                                |
| 3     | _(web_component design)_  | reusable, assertion-free interactions (this skill §4 + the multipurpose-function rules inside `step_def_prompt.txt`) |
| 4     | `step_def_prompt.txt`     | the step-def file: web_component-first, **all** assertions, Allure                                                   |

Each prompt opens with a **"analyse the existing files first, then match them exactly"** phase
and closes with a **mandatory self cross-check** — do not skip either. The templates in the
sections below are the _shape_; those root prompts are the _rules_. On any conflict: existing
workspace files win on **style**, and the **live DOM** wins on **locators**.

---

## 1. Where each piece of the walk lands (the 4 layers)

| What you did/observed in the MCP walk                   | Author it in…                                                                   |
| ------------------------------------------------------- | ------------------------------------------------------------------------------- |
| The business intent of each step (plain language)       | **Feature** `tests/features/Apps/<Module>/<module>-<STORY-ID>.feature`          |
| The XPath/anchor of a touched element                   | **Page Object** `object_repository/pages/Apps/<Module>/<Name>Page.py` (locator) |
| A click/type/select sequence reused by 2+ steps/cases   | **web_component** `business_components/web_components/<module>.py`              |
| The action orchestration + **every assertion** + Allure | **Step def** `tests/step_defs/apps/<Module>/test_<module>-<STORY-ID>.py`        |
| A value the step typed / expected                       | **Test data** `testData/<module>.json` (one key per case)                       |

**Golden rules:** locators ONLY in the POM; assertions ONLY in `@then` steps; reusable
interactions ONLY in the web_component (assertion-free, locators passed in); no `time.sleep`; no
hardcoded data; no raw `bdd_driver.find_element` in steps.

---

## 2. Feature — one Scenario per case (plain business language)

Append one `Scenario` (or `Scenario Outline`) per QMetry case, tagged the module's way, with the
shared `Background`. **No locators, data, or implementation detail in Gherkin.**

```gherkin
Feature: CMO AMCC-5128 — Supervisor Audit Associate Details Page

    Background:
        Given login to the Cold Compass application

    #====================== AMCC-TC-15200 ======================
    @sanity @story-AMCC-5128 @cmo @AMCC-TC-15200 @regression
    Scenario: AMCC-TC-15200 Verify the information displayed in associate details section is read only
        When Navigate to "Apps > CMO"
        Then Click on "Supervisors" tab
        Then Supervisor My Audit landing page should be displayed
        When Select facility "Allentown" from the dropdown
        When Click on "Start New Audit" button
        Then Start New Audit screen should be displayed
        When Click on a displayed audit template
        Then User should be landed on Associate details page
        When Verify template name, description, audit category and job code fields
        Then Fields should be displayed as read-only
```

- One `When`/`Then` per QMetry step (`steps[].description` → `When`, its `expectedResult` → a
  `Then`). Tag with `@story-<STORY-ID> @<module> @<CASE-ID> @regression` (match the module's
  existing tags exactly — e.g. CMO also uses `@sanity`).

---

## 3. Page Object — bake the live locators (locators ONLY)

> **🧠 ANTI-HALLUCINATION CONTRACT (STRICT):**
> Every locator added to a Page Object MUST be directly copied from `kg_context_pack` or from a live `snapshot_dom` inspection during this session. **NEVER guess an XPath.** If an element was not inspected live or found in Knowledge Graph, run `snapshot_dom(interactive_only=true)` to capture its true attributes before writing code.

For every genuinely-new element you touched, add a locator **list** to the module's Page Object,
using the anchor you validated live. **Reuse** an existing locator if one already matches.

```python
# pylint: disable=all
from selenium.webdriver.common.by import By
from utilities.genericUtils import CustomPyAutoWeb


class MyAuditsLandingPage(CustomPyAutoWeb):

    # --- Start New Audit ---
    locatorStartNewAuditButton = [
        (By.XPATH, "//button[@data-testid='start-new-audit']")
    ]
    # --- Facility dropdown ---
    locatorFacilityDropdown = [
        (By.XPATH, "//button[@role='combobox' and @data-testid='facility-select']")
    ]
    # Dynamic option by visible text (runtime value → lambda returning the list)
    locatorFacilityOptionByName = lambda name: [
        (By.XPATH, f"//div[@role='option' and normalize-space(text())='{name}']")
    ]
```

- Shape is always `locatorXxx = [(By.XPATH, "...")]` (a **list of one tuple**).
- Runtime-variable selectors are `lambda value: [(By.XPATH, f"...{value}...")]`.
- Anchor priority: text → `@aria-label`/`@role` → `@data-testid` → `@id`/`@name` → semantic
  `@class`+tag (last resort). **Never** hashed/utility Tailwind classes.
- State variants only when they exist in the DOM: `...Active` / `...Inactive` / `...Selected`.

---

## 4. web_component — reusable interactions (assertion-free)

If a click/type/select flow is reused by 2+ steps or cases, put it in the module web_component as
a `driver`-first, parameterized, **assertion-free** function that drives POM locators. Reuse or
extend an existing function — one function per interaction pattern.

```python
# business_components/web_components/cmo.py
def select_facility_from_dropdown(driver, facility_name: str, timeout: int = 30):
    """
    Open the facility dropdown and pick `facility_name`.

    Args:
        driver:        Selenium WebDriver instance.
        facility_name: The facility option to select (e.g. "Allentown").
        timeout:       Max seconds to wait for the dropdown trigger.
    """
    web = CustomPyAutoWeb(driver)
    web.wait_element_visibility(
        locatorList=MyAuditsLandingPage.locatorFacilityDropdown, wait_time=timeout
    )
    web.click_wait_locator(locatorList=MyAuditsLandingPage.locatorFacilityDropdown)
    web.click_wait_locator(
        locatorList=MyAuditsLandingPage.locatorFacilityOptionByName(facility_name)
    )
```

- `driver` first, then values/labels, then `timeout: int = 30`. Locators come from the POM.
- **Zero assertions.** Getters return a value; actions return `None`.
- Cross-step values use paired `get_`/`set_` module globals (see Skill 04) or the step file's `_ctx`.

---

## 5. Step defs — orchestration + ALL assertions + Allure (the contract)

One step function per Gherkin step, decorator text **byte-identical**. Build
`web = CustomPyAutoWeb(bdd_driver)`, drive the **same action you walked live** (toolbox + POM
locator, or a web_component call), assert every `@then` with a `CustomPyAutoWeb` verify, wrap in
the standard `try/except`, and attach Allure evidence.

```python
# tests/step_defs/apps/CMO/test_cmo-AMCC-5128.py  (excerpt)
@when('Select facility "{facility}" from the dropdown')
def select_facility(bdd_driver, facility):
    try:
        select_facility_from_dropdown(bdd_driver, facility)          # web_component
        allure.attach(
            f"Selected facility: '{facility}'",
            name="Action — Select Facility",
            attachment_type=allure.attachment_type.TEXT,
        )
    except Exception as e:
        raise AssertionError(
            f"Step FAILED: 'Select facility \"{facility}\" from the dropdown' - "
            f"Error occurred in step [select_facility]. Details: {e}"
        ) from e


@then("Fields should be displayed as read-only")
def verify_fields_read_only(bdd_driver):
    try:
        web = CustomPyAutoWeb(bdd_driver)
        web.wait_element_visibility(
            locatorList=CmoAssociateDetailsPage.locatorReadOnlyFields, wait_time=60
        )
        web.element_should_be_visible(
            locator=CmoAssociateDetailsPage.locatorReadOnlyFields
        )                                                            # raises on failure
        allure.attach(
            "Associate detail fields are read-only — read-only marker visible",
            name="Verification — Associate Detail Fields Read-Only",
            attachment_type=allure.attachment_type.TEXT,
        )
    except Exception as e:
        raise AssertionError(
            f"Step FAILED: 'Fields should be displayed as read-only' - "
            f"Error occurred in step [verify_fields_read_only]. Details: {e}"
        ) from e
```

The five-part shape every step uses:

1. Decorator text == the feature step (escape inner `"` in the failure message).
2. Takes `bdd_driver`; `parsers.parse('… "{x}" …')` params come **after** `bdd_driver`.
3. `web = CustomPyAutoWeb(bdd_driver)` — never `bdd_driver.find_element` directly.
4. `@when` acts (delegate reusable flows to the web_component); `@then` **asserts** with a
   `CustomPyAutoWeb` verify (e.g. `element_should_be_visible`, `verify_current_url`,
   `verify_toast_alert`).
5. `except Exception as e: raise AssertionError("Step FAILED: '<step>' - … [<func>]. Details: {e}") from e`
   - an Allure attach (`@when` → `Action — …`, `@then` → `Verification — …`).

**Module file header** (bind to the feature, load data, declare `_ctx`):

```python
# pylint: disable=all
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

scenarios(
    os.path.join(
        os.path.dirname(__file__), "../../../features/Apps/CMO/cmo-AMCC-5128.feature"
    )
)

json_data = load_test_data_from_json("../../testData/cmo.json")
_ctx = {}
```

**Reused steps** (pytest-bdd resolves per module → redefine verbatim with provenance, delegate to
the same web_component):

```python
# REUSED FROM: test_cmo-AMCC-5127.py — step "Navigate to \"Apps > CMO\"".
# Redefined here because pytest-bdd resolves steps per module and the hyphenated
# sibling file cannot be imported. Body delegates to the shared web_component.
@when('Navigate to "Apps > CMO"')
def navigate_to_cmo(bdd_driver):
    try:
        CMO_Navigation(bdd_driver)
    except Exception as e:
        raise AssertionError(
            f"Step FAILED: 'Navigate to \"Apps > CMO\"' - Error occurred in step [navigate_to_cmo]. Details: {e}"
        ) from e
```

---

## 6. Test data — one key per case (no inline literals)

```json
{
  "test": {
    "AMCC-5128": {
      "AMCC-TC-15200": { "facility": "Allentown" },
      "AMCC-TC-15201": { "facility": "Allentown", "expectedEmailVisible": true }
    }
  }
}
```

Read it in the step: `td = json_data.get("AMCC-5128", {}).get("AMCC-TC-15200", {})`. The feature
passes a **key**; the step looks it up — never hardcode the value.

---

## 7. Self-check before you call a case done

**Coverage / fidelity**

- [ ] One `Scenario` for the case; one `When`/`Then` per QMetry step; each `@then` asserts that
      step's `expectedResult`.
- [ ] Decorator text equals the feature step **exactly** (quotes/params).

**Architecture**

- [ ] Zero `bdd_driver.find_element` / raw locators in steps. Locators only in the POM (list shape).
- [ ] Assertions only in `@then` (CustomPyAutoWeb verify inside `try/except`); none in
      web_component/POM.
- [ ] Reusable interactions in the web_component (assertion-free, locators as params); one per pattern.
- [ ] No `time.sleep`; no hardcoded data; no invented names/imports.

**Allure**

- [ ] Every `@when` has an `Action —` attach; every `@then` has a `Verification —` attach (after
      the passing check). No vague names.

**Grounding**

- [ ] Every new locator was resolved from the **live DOM** via the `selenium` MCP (role/text/
      `data-testid` first). No per-step screenshots — any confirmed bug has its defect evidence
      (`<CASE-ID>_defect_<N>.png` + `_defect.json` + rendered `_defect.docx`) under
      `TestcaseExecution/<module>/<STORY-ID>/<CASE-ID>/` (see
      [../defect_documentation/SKILL.md](../defect_documentation/SKILL.md)).

**Gate**

- [ ] `python MCP/mcp_execute_and_author.py --story "/<module>/<STORY-ID>/story.md" --verify --case <CASE-ID>` passes.
