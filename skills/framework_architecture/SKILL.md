---
description: Canonical map of this repo — the BDD stack, the real folder layout, the
  4 layers, the driver fixture, and where every kind of code lives.
name: framework_architecture
---

# Skill 00 — Framework Architecture (the canonical map)

This is the **single source of truth** for _where things live_ and _what tools are used_.
Every other skill and prompt assumes these facts. If a generic POM/Behave instruction ever
conflicts with this file, **this file wins** — because it is derived from the real repo.

---

## 1. The stack (verified, not assumed)

| Concern            | This repo actually uses                                         | Do **not** assume           |
| ------------------ | --------------------------------------------------------------- | --------------------------- |
| BDD runner         | **pytest-bdd** (`pytest_bdd`)                                   | ~~Behave~~                  |
| Step binding       | `scenarios("…/x.feature")` at module top                        | ~~`behave` auto-discovery~~ |
| Driver handoff     | **`bdd_driver`** pytest fixture                                 | ~~`context.driver`~~        |
| Page driver helper | `CustomPyAutoWeb(bdd_driver)` from `utilities.genericUtils`     | ~~bare Selenium in steps~~  |
| Cross-step state   | module-level `_ctx = {}` + web*component `get*`/`set\_` globals | ~~`context.<attr>`~~        |
| Test data          | `load_test_data_from_json("…/testData/<module>.json")`          | ~~hardcoded data~~          |
| Reporting          | Allure (`import allure`; `allure.attach(...)`)                  | —                           |
| Parsers            | `from pytest_bdd import parsers` → `parsers.parse('… "{x}" …')` | ~~regex by default~~        |

---

## 2. The 4 layers and their REAL folders

> The architecture is still "one layer per concern" — but the folders are repo-specific.

| Layer             | Real folder (this repo)                     | File pattern                       | Contains                                     |
| ----------------- | ------------------------------------------- | ---------------------------------- | -------------------------------------------- |
| **Feature**       | tests/features/Apps/`<Module>`/             | `<module>-<JIRA>.feature`          | Gherkin only                                 |
| **Step Defs**     | tests/step_defs/apps/`<Module>`/            | `test_<module>-<JIRA>.py`          | step orchestration **+ all assertions**      |
| **Page Objects**  | object_repository/pages/Apps/`<Module>`/    | `<Name>Page.py`                    | locators + page methods                      |
| **web_component** | business_components/web_components/         | `<module>.py` (e.g. cmo.py)        | reusable parameterized interactions          |
| Shared helper     | utilities/genericUtils.py                   | `CustomPyAutoWeb`                  | the Selenium toolbox (base class)            |
| Hooks / fixtures  | tests/step_defs/conftest.py                 | —                                  | `bdd_driver`, driver lifecycle, Allure hooks |
| Dynamic Auth Allocator | utilities/role_session_allocator.py     | `build_scenario_credential_map`    | AST reflection & dynamic test-to-cookie mapping |
| Login helper      | business_components/web_components/login.py | `login_to_application_sso(driver)` | SSO logins                                   |

**Concrete CMO example (use as your mental model):**

```
tests/features/Apps/CMO/cmo-AMCC-8438.feature              ← feature
tests/step_defs/apps/CMO/test_cmo-AMCC-8438.py             ← step defs (test_ prefix!)
object_repository/pages/Apps/CMO/CmoLandingPage.py         ← page object(s)
object_repository/pages/Apps/CMO/Supervisor/NewAuditPage.py
business_components/web_components/cmo.py                   ← shared interactions
testData/cmo.json                                          ← test data
```

---

## 3. The driver fixture (there is NO `context`)

Defined in tests/step_defs/conftest.py:

```python
@pytest.fixture
def bdd_driver():
    global d
    yield d            # `d` is the live WebDriver, created per-scenario by the bdd hooks
```

Every step function therefore takes `bdd_driver` and turns it into a helper:

```python
@when('Click on "Templates" tab')
def click_templates_tab(bdd_driver):
    web = CustomPyAutoWeb(bdd_driver)
    web.click_wait_locator(locatorList=CMOLandingPage.locatorCMOTabTemplates)
```

**Passing data between steps:** use the module-level `_ctx` dict (declared once near the top
of the step file) and/or `get_`/`set_` globals exposed by the web_component:

```python
_ctx = {}                                  # near the top of the step-def module
...
_ctx["template_data_key"] = template_data  # write in a @when
...
td = json_data.get(_ctx.get("template_data_key", ""), {})   # read in a @then
```

```python
# web_component globals (business_components/web_components/cmo.py)
set_template_name(name)   # write
get_template_name()       # read back the auto-generated template name in another step
```

---

## 4. Standard import block for a step-def file (real, copy this order)

```python
# pylint: disable=all
import pytest
from pytest_bdd import given, when, then, scenarios, parsers
from selenium.webdriver.common.by import By
import allure
import os
from datetime import date, time

from business_components.web_components.login import login_to_application_sso
from business_components.web_components.cmo import *          # the module's web_component
from object_repository.pages.HomePage import HomePage
from object_repository.pages.Apps.CMO.CmoLandingPage import (
    CMOLandingPage,
    CMOTemplateLandingPage,
)
from tests.step_defs.conftest import bdd_driver
from utilities.genericUtils import CustomPyAutoWeb
from utilities.helper import load_test_data_from_json

scenarios(
    os.path.join(
        os.path.dirname(__file__), "../../../features/Apps/CMO/cmo-AMCC-8438.feature"
    )
)

json_data = load_test_data_from_json("../../testData/cmo.json")
_ctx = {}
```

Import grouping order: **stdlib → pytest/pytest-bdd → selenium → allure → business_components
(web_component) → object_repository (POM) → conftest → utilities**. Match the closest sibling
`test_cmo-*.py` exactly when unsure.

---

## 5. Base class & locator shape (one fact that breaks everything if wrong)

- Page objects subclass **`CustomPyAutoWeb`** (NOT `BasePage`):
  `class CMOLandingPage(CustomPyAutoWeb):`
- Locators are **a list containing one `(By, "selector")` tuple**:
  ```python
  locatorCMOHeading = [(By.XPATH, '//*[contains(@class,"h6") and text()="CMO"]')]
  ```
- `CustomPyAutoWeb` methods receive that list via `locatorList=` (actions) or `locator=`
  (some verifies). See [../custompyautoweb_api/SKILL.md](../custompyautoweb_api/SKILL.md).

---

## 6. Waits — the only sanctioned timing

- ✅ `web.wait_element_visibility(locatorList=..., wait_time=60)` — preferred gate before acting.
- ✅ `web.wait(2)` — the framework's short, bounded settle wait (used sparingly for animations /
  typeahead population). This is the framework helper — use it instead of `time.sleep`.
- ❌ `time.sleep(...)` — **never** call it directly anywhere.

---

## 7. Quick "where do I put this?" decision table

| The code is…                                        | It goes in…                                                                     |
| --------------------------------------------------- | ------------------------------------------------------------------------------- |
| A locator (XPath/CSS)                               | a **page object** (`object_repository/pages/Apps/<Module>/`)                    |
| One page's click/read wrapped with a wait           | a **page object** method, called via `CustomPyAutoWeb`                          |
| A multi-step interaction reused by 2+ steps/stories | the module **web_component** (`business_components/web_components/<module>.py`) |
| A value shared between `@when` and `@then`          | `_ctx` dict or a web*component `get*/set\_` global                              |
| An assertion / verification                         | a **step def** `@then` (via `CustomPyAutoWeb` verify + `try/except`)            |
| Plain business language                             | the **`.feature`** file                                                         |
