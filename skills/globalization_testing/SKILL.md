---
description: 'Ground-truth cookbook for GLOBALIZATION testing: translate(), locales
  JSON, login-with-language.'
name: globalization_testing
---

# Skill — Globalization Testing (dual-browser MCP + i18n framework)

Read [../framework_architecture/SKILL.md](../framework_architecture/SKILL.md) first.
This skill is the canonical map of **how globalization actually works in this repo** — every
fact below is derived from the real code. The driving prompt is
[../mcp_globalization_prompt/SKILL.md](../skills/mcp_globalization_prompt/SKILL.md).

> ⚠️ Globalization tests are **plain pytest** (the `driver` fixture from
> tests/test_scripts/conftest.py) — **NOT pytest-bdd**, **not** `bdd_driver`.
> They live in their own tree and follow their own conventions, all documented here.

---

## 1. The translation core (utilities/globalizationUtilities/)

### `i18nextUtils.py` — `configure_i18n` + `translate`

```python
configure_i18n(language_folder="orderMaintenance")   # loads testData/locales/<folder>/<TestLanguage>.json
translate("outboundOrder.createOrder")               # → "Create Order" (en) / "Crear Pedido" (es)
```

- `configure_i18n(language_folder)` reads **`testData/locales/<language_folder>/<lang>.json`**
  where `<lang>` = the **`TestLanguage`** env var (`en`, `es`, `de`, `fr`, `br`, `latam`,
  `nl`, `pl`, `pt`). Raises `AssertionError` if the file can't load.
- `translate(key)` resolves the **dot-separated key** against the loaded cache:
  - during test **execution** a missing key raises
    `AssertionError: Translation key '<key>' not found for language '<lang>'`
  - during pytest **collection** it returns the sentinel `__MISSING_TRANSLATION__:<key>`
    so imports don't explode.

### `lazy_locator.py` — `LazyLocator` (the critical piece)

Class-level locators that call `translate()` directly are frozen at **import time** — wrong
language in a multi-language run. `LazyLocator` is a **descriptor** that re-evaluates the
builder on **every attribute access**, using the _current_ `TestLanguage`:

```python
class MyPage:
    locatorTitle = LazyLocator(
        lambda t: [(By.XPATH, f"//h2[text()=\"{t('myActions.pageTitle')}\"]")]
    )
```

- The builder receives `translate` as `t` and returns the normal locator **list**.
- Accessing `MyPage.locatorTitle` returns a fresh `[(By.XPATH, "...")]` for the active language.

---

## 2. Folder map (globalization-specific)

| Layer                       | Real folder                                                                                 | Example                                                                      |
| --------------------------- | ------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| Locales JSON                | testData/locales/`<languageFolder>`/`<lang>`.json                                           | `testData/locales/orderMaintenance/es.json`                                  |
| Globalization POM           | object_repository/pages/`<Area>`/**globalizationLocators**/`<Name>GlobalizationPage.py`     | `Orders/globalizationLocators/OutboundOrdersLandingPageGlobalizationPage.py` |
| Globalization web component | business_components/web_components/`<module>Globalization.py` or `globalization<Module>.py` | `actionPlanGlobalization.py`, `globalizationOrderMaintenance.py`             |
| Globalization tests         | tests/test*scripts/**globalization**/`<module>`/test*`<name>`Globalization.py               | `globalization/orderMaintenance/test_outboundCreateOrderGlobalization.py`    |
| Test data                   | testData/globalization`<Module>`.json                                                       | `testData/globalizationOrderMaintenance.json`                                |
| Orchestrator                | tests/test_scripts/RunTest.py (language loop) + conftest.py (fixture)                       | —                                                                            |

**Locale folders that exist** (each holds the 9 language files):
`accessManagement, actionPlan, auditTrail, holdManagement, manageConfiguration, messageCenter,
olas, orderMaintenance, profileManagement, reports, roleManagement, rulesEngine, shell, tms,
totalInventory, userManagement` — `shell` covers the app frame (login, header, footer, nav).

**Language codes ↔ login-dropdown display values** (from login.py):

| Code    | Login dropdown value   | Code | Login dropdown value          |
| ------- | ---------------------- | ---- | ----------------------------- |
| `en`    | English(US)            | `br` | Português(Brazil)             |
| `es`    | Español(Spain)         | `nl` | Netherlandic(The Netherlands) |
| `de`    | Deutsch(Germany)       | `pl` | Polski(Poland)                |
| `fr`    | Français(Canada)       | `pt` | Português(Portugal)           |
| `latam` | Español(Latin America) |      |                               |

---

## 3. Globalization POM conventions (different from the BDD POMs!)

> **Reference file (copy this structure):**
> `object_repository/pages/Apps/ManageConfigurations/ManageConfigurationsGlobalizationLandingPage.py`

```python
# pylint: disable=all
from selenium.webdriver.common.by import By

from utilities.genericUtils import CustomPyAutoWeb
from utilities.globalizationUtilities.i18nextUtils import configure_i18n, translate


class ManageConfigurationsGlobalizationLandingPage(CustomPyAutoWeb):

    configure_i18n(language_folder="manageConfiguration")

    # ----------------- breadcrumb locators ------------------#
    locatorHomeBreadCrump = [
        (By.XPATH, f"//nav[@aria-label='breadcrumb']//*[text()='{translate('common.home')}']/..")
    ]
    locatorManageConfigurationsHeading = [
        (By.XPATH, f"(//*[text()='{translate('header.manageConfigurations')}'])[2]")
    ]
    # language-neutral anchors stay plain (no translate needed):
    locatorSearchButton = [(By.XPATH, "//*[@data-testid='search-button']")]
```

Rules (all observed in the real files):

1. **Manage-config style (default for NEW POMs):** class subclasses `CustomPyAutoWeb`,
   calls `configure_i18n(language_folder="<folder>")` at class level, and builds locators
   with f-string `translate('key')` directly. Some areas (e.g. Orders) use `LazyLocator`
   plain classes instead — **match the sibling globalization POMs of the area you're in**;
   when creating the area's FIRST globalization POM, use the manage-config style.
2. **DERIVE, don't invent — the locator-derivation rule.** Every globalization locator
   starts from the module's **functional step-def POM** locator
   (`object_repository/pages/Apps/<Module>/…Page.py`): reuse the SAME XPath and replace
   ONLY the hardcoded visible-text literal with the translate call —
   `//button[text()='Reset']` → `f"//button[text()='{translate('buttons.reset')}']"`.
   Every other part (testids, aria, axes, indexes, structure) stays byte-identical, so the
   functional and globalization suites always point at the same element. Only write a new
   XPath when the element has no functional locator at all (then follow §7 anchor priority).
3. Locators anchored on `data-testid` / `aria` / structure (no text) stay **plain lists** —
   they are language-neutral by construction. **Prefer these anchors whenever they exist.**
4. Attribute names follow `locator<PascalCase>` exactly like the BDD POMs.
5. Section dividers `# ----------------- <UI section> ------------------#` group locators
   **page-section-wise**, in the same order as the functional POM.
6. **Every key used must be proven** (glob_check_locale_keys reverse lookup) — never invented.
7. **MISSING_KEY protocol** — when the UI text has **no key** in `en.json`:
   - keep a placeholder locator + trailing comment `# MISSING_KEY — <what's missing>`,
   - list it under a `⚠️ Search for 'MISSING_KEY.'` banner in the docstring,
   - the finding goes in the globalization findings ledger (§7) — it is a product/locale gap.
8. **🚫 Locale JSONs are READ-ONLY.** Never add/rename/edit keys or values in
   `testData/locales/**/*.json` to make a locator resolve or a test pass. A missing/empty/
   identical key = a findings row + a test that **fails red** until the product fixes it.

**Quoting rule:** translated text can contain apostrophes (`fr`), so f-string XPaths use
**double quotes** around the text: `f'//h2[text()="{translate("key")}"]'`.

---

## 4. Globalization web components (business functions)

File: business_components/web_components/`<module>Globalization.py` — module-level,
`driver`-first functions (same contract as Skill 04) with two globalization twists:

```python
def _norm_loc(loc):
    """Normalize locators — handles LazyLocator output, tuple, list, and None."""
    if loc is None:        return []
    if isinstance(loc, tuple): return [loc]
    if isinstance(loc, list):  return loc
    return [loc]


def Search_Orders_Globalization(driver, filter_data):
    auto_web = CustomPyAutoWeb(driver)
    auto_web.wait_element_visibility(
        locatorList=_norm_loc(LandingPage.locatorSavedSearchesInput), wait_time=10
    )
    # dropdown OPTION VALUES are translated too — translate the key first:
    translated_value = translate(filter_data["dateRangeField"])       # key → current-language text
    auto_web.select_value_from_dropdown_with_javascript_click(
        web_element=_norm_loc(LandingPage.locatorDateRangeFieldDropdown),
        element_value=translated_value,
    )
```

- **Always wrap POM locators in `_norm_loc(...)`** before passing to `CustomPyAutoWeb`.
- Test data carries **translation keys** (e.g. `"dateRangeOptions.OrderReceivedDate"`), the
  function calls `translate(key)` at runtime to get the language-correct visible value.
- Zero assertions, zero hardcoded locators — same contract as every web_component.

---

## 5. Globalization test files — page-wise & section-wise

One test **file per page**, one test **function per page section**.

> **Reference file (copy this anatomy):**
> `tests/test_scripts/globalization/manageConfiguration/test_manageConfigurationsLandingPage.py`

```python
# pylint: disable=all
import os, sys, pytest, allure
from selenium.webdriver.common.by import By

sys.path.append("../../")

from business_components.web_components.manageConfigurations import *
from business_components.web_components.login import login_to_application_sso_with_language
from object_repository.pages.Apps.ManageConfigurations.ManageConfigurationsGlobalizationLandingPage import (
    ManageConfigurationsGlobalizationLandingPage,
)
from object_repository.pages.HomePage import HomePage
from utilities.genericUtils import CustomPyAutoWeb
from utilities.globalizationUtilities.i18nextUtils import configure_i18n, translate


@pytest.mark.globalization                                        # ← always
@pytest.mark.globalization_manage_configurations                 # ← module marker
@pytest.mark.globalization_manage_configurations_basic           # ← section marker
@allure.title("Globalization - Manage Configurations Landing Page Basic Texts")
@allure.description("""
    This test case validates the globalization/internationalization of the …

    Preconditions:
    - User has access to … / Language is configured via TestLanguage.

    Validations:
    1. Breadcrumb texts …
    2. Page heading text …
    """)
def test_globalization_manage_configurations_basic(driver):      # ← `driver`, NOT bdd_driver
    configure_i18n(language_folder="manageConfiguration")         # 1. load locale JSON
    test_language = os.environ.get("TestLanguage", "en")
    web = CustomPyAutoWeb(driver)
    login_to_application_sso_with_language(driver)                # 2. login in TestLanguage

    with allure.step("Navigate to Manage Configurations Landing Page"):
        web.click_wait_locator(locatorList=HomePage.locatorAppsBtn)
        web.click_wait_locator(locatorList=HomePage.locatorManageConfigurationsBtn)

    with allure.step("Verify BreadCrumb Texts"):                  # 3. grouped (locator,key,name) loop
        for locator, key, name in [
            (Page.locatorHomeBreadCrump, "common.home", "Home BreadCrumb"),
            (Page.locatorAppsBreadCrump, "common.apps", "Apps BreadCrumb"),
        ]:
            expected = translate(key)
            actual = web.get_element_text(locator)
            web.element_text_should_be(web_element=locator, expected_string=expected)
            allure.attach(f"Expected: {expected}\nActual: {actual}", name=name,
                          attachment_type=allure.attachment_type.TEXT)

    with allure.step("Verify 'Help' button text"):                # 4. single-element pattern
        expected_text = translate("header.help")
        actual_text = web.get_element_text(Page.locatorHelpButton)
        web.element_text_should_be(web_element=Page.locatorHelpButton,
                                   expected_string=expected_text)
        allure.attach(
            f"Language: {test_language}\n"
            f"Translation Key: header.help\n"
            f"Expected Text: {expected_text}\n"
            f"Actual Text: {actual_text}\n"
            f"Result: {'PASS ✅' if expected_text == actual_text else 'FAIL ❌'}",
            name="Help Button Text Verification",
            attachment_type=allure.attachment_type.TEXT,
        )
```

Anatomy rules (all from the reference file):

- `@allure.title` + a full `@allure.description("""…""")` with Preconditions and a numbered
  Validations list on EVERY test.
- One `with allure.step` per UI section; grouped checks loop `(locator, key, name)` tuples.
- Verification = `translate(key)` → `web.get_element_text(…)` →
  `web.element_text_should_be(web_element=…, expected_string=…)` (raises on mismatch).
- Evidence attach carries `Language / Translation Key / Expected Text / Actual Text /
Result: PASS ✅|FAIL ❌`.
- Dropdown options: the shared `select_translated_dropdown_option(web, locator_options, key)`
  helper — iterate rendered options, click the one whose text == `translate(key)`, raise a
  descriptive `AssertionError` listing available options when absent.
- **Missing/untranslated key = red test.** Never patch `testData/locales/*.json`, never
  soft-pass — the failing assertion cross-referenced to findings.md IS the deliverable.

### The standard verification helpers (when the file uses them — copy per file)

| Helper                              | Check                                                                                                                                                         |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `_verify_text`                      | `actual.strip().casefold() == translate(key).strip().casefold()`                                                                                              |
| `_verify_text_ignore_case`          | same, explicit case-insensitive naming                                                                                                                        |
| `_verify_text_contains`             | `translate(key) in actual` (casefolded)                                                                                                                       |
| `_verify_text_contains_ignore_case` | contains, case-insensitive                                                                                                                                    |
| `_verify_element_visible`           | `web.is_element_present(...)` + assert with message                                                                                                           |
| `_verify_missing_key_locator`       | KEY_MISSING_IN_EN probe (no key exists AT ALL) → Allure note; in non-en it asserts the English text is NOT visible (fails red when hardcoded English renders) |

Every helper: `expected = translate(key)` → read actual via `CustomPyAutoWeb` → `assert` with
a descriptive message → `allure.attach(f"Expected: …\nActual: …", name=label, ...)`.

> ⚠️ **KEY_MISSING_IN_LANG is NOT a probe case.** When the key exists in en.json but is
> missing in the target language, `translate(key)` raises (or the strict verify mismatches) —
> the test goes RED. That is correct: do NOT add the key to the lang JSON, do NOT downgrade
> to a probe/soft check. Only truly keyless UI texts (no key in en.json either) use the
> `_verify_missing_key_locator` probe.

### Fixture & orchestration facts

- The **`driver` fixture** (tests/test_scripts/conftest.py) sets
  `os.environ["TestLanguage"]` from the parametrized language and builds a fresh Chrome.
- With `RunMultipleLanguage=True` (config/TestConfig.py), `pytest_generate_tests`
  parametrizes every `@pytest.mark.globalization` test over `GLOBALIZATION_LANGUAGES`
  (from the `Languages` env JSON list) as `[lang-en]`, `[lang-es]`, …
- `tests/test_scripts/RunTest.py` is the sequential multi-language orchestrator
  (loop → retry → merged Allure → Excel via allureExcelConversion.py).
- **Single-language dev run** (what the MCP verify phase uses):

```powershell
$env:TestLanguage="es"; python -m pytest tests/test_scripts/globalization/<module>/test_<name>.py -m globalization_<module> --alluredir=reports/allure-results -q
```

### Login-with-language flavours (business_components/web_components/login.py)

| Function                                           | Auth              | Creds (.env)                      |
| -------------------------------------------------- | ----------------- | --------------------------------- |
| `login_to_application_username_password_with_lang` | username/password | `OrderUsername` / `OrderPassword` |
| `login_to_application_sso_with_language`           | Azure-AD SSO      | `Email` / `Password` (QA admin)   |
| `login_to_application_sso_with_language_amc_user`  | SSO (AMC user)    | `AmcUserEmailForHolds` / …        |

All first select the language on the login page's dropdown
(`//*[@class='text-grey-600 w-32 text-sm space-y-0']//button` → option
`//*[@role='option']//span[contains(text(),'<display>')]`), then authenticate.

---

## 6. The 5 dual-browser MCP tools (MCP/mcp_globalization_tools.py)

Registered by selenium_mcp_launch.py on top of the base + fast tools. The base server
keeps **named sessions** (`start_browser(session_name=…)` / `switch_session`) — these tools
drive TWO of them side by side.

| Tool                     | One call does…                                                                                                                                                                                                                                                      |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `glob_login_language`    | start named session (`'EN'`/`'ES'`) → open portal → pick language on the login dropdown → SSO or password login. Call once per language.                                                                                                                            |
| `glob_sync_action`       | run the SAME `navigate/click/type/press_enter/wait_idle` on every listed session — keeps both browsers on the same screen. **Selectors must be language-neutral** (`data-testid`/structure), never translated text.                                                 |
| `glob_capture_texts`     | harvest EVERY visible text of one session's page (own text + placeholder + aria-label + title + alt + button value), each with tag/testid/structural path.                                                                                                          |
| `glob_compare_sessions`  | **the core**: harvest base (EN) + target (ES), align by testid/path, classify every text → `TRANSLATED` / `NOT_TRANSLATED` / `MISSING_IN_TARGET` / `MISSING_IN_BASE` / `SKIPPED_DATA`. `save_to=` persists the full JSON.                                           |
| `glob_check_locale_keys` | audit `testData/locales/<folder>/`: keys missing/empty/identical-to-English in `<lang>.json`, plus **reverse lookup** `texts=[…]` → translation key + lang value (the key you need for the LazyLocator, and the proof for a "key missing in locales JSON" finding). |

### The comparison loop for ONE screen (the pattern to repeat)

```jsonc
// 1. Both browsers on the same screen (language-neutral selector!)
glob_sync_action { "sessions": ["EN","ES"],
                   "action": {"type":"click","selector":"//button[@data-testid='cmo-tab-templates']"} }
// 2. Compare everything visible
glob_compare_sessions { "base_session":"EN", "target_session":"ES",
                        "page_label":"CMO > Templates tab",
                        "save_to":"MCP/stories/CMO/globalization/es/templates-tab.compare.json" }
// 3. For every NOT_TRANSLATED / suspicious text → map to locale keys
glob_check_locale_keys { "language_folder":"<folder>", "language":"es",
                         "texts":["Create Template","Audit Type", …] }
// 4. Repeat with opened dropdowns / flyers / dialogs / triggered validation errors
```

**Classification honesty rules:**

- `NOT_TRANSLATED` = the target text is byte-identical to English **and** it is not
  data/brand. Brand words (`Americold`, `CMO`, `i-3PL`…) are auto-skipped; pass extra
  legitimate ones via `allow_same=[…]` — never let a real gap hide behind it.
- `SKIPPED_DATA` = numbers, dates, emails, IDs, URLs — user data is **never** a finding.
- An element found in EN but absent in the target page is `MISSING_IN_TARGET` — investigate
  before calling it a defect (it may be data-dependent rendering).

### 6-B. The 16-category UI-text taxonomy — 100% module coverage

A screen (and later, its test file) is complete only when every APPLICABLE category has been
compared AND has at least one test function. This is the master checklist; `coverage.md`
keeps a screens × categories matrix (✅ / n/a / ❌ + reason).

| #   | Category                       | Where the texts hide                                              | Surfacing recipe (both sessions)                                                                           |
| --- | ------------------------------ | ----------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| 1   | Page titles & headers          | h1–h6, panel h5s                                                  | visible on load                                                                                            |
| 2   | Breadcrumbs                    | nav[aria-label=breadcrumb] li spans                               | visible on load                                                                                            |
| 3   | Tabs & nav items               | tab buttons, side-nav links; active/inactive text variants        | visible; click through states if labels change                                                             |
| 4   | Buttons — ALL                  | text buttons + icon buttons' `aria-label`/`title`, disabled text  | visible; hover for title attrs                                                                             |
| 5   | Status labels / badges / chips | table status cells, card badges, toggle-group buttons             | filter/search to render EVERY status value at least once                                                   |
| 6   | Input field labels             | label/span above each field + required `*` helper text            | open the form/dialog                                                                                       |
| 7   | Placeholders & input aria      | `placeholder=`, `aria-label=`, `title=` on inputs                 | glob_capture harvests them automatically                                                                   |
| 8   | Dropdown / select options      | Radix listboxes, filter dds, page-size dds, kebab/action menus    | `glob_sync_action` click to OPEN in both → compare with `scope_selector` → close. One compare PER dropdown |
| 9   | Table column headers           | thead cells + sort-icon aria                                      | render the table first (search/apply same query in both)                                                   |
| 10  | Table cell UI texts            | status texts, action links, "N items" counters (NOT user data)    | rendered with the table                                                                                    |
| 11  | Alert / toast messages         | success + error toasts                                            | trigger the same submit in both; compare before the toast fades                                            |
| 12  | Validation messages            | per-field errors, form-level errors                               | submit empty required form; enter invalid email/number/date                                                |
| 13  | Empty / no-result states       | "No X found" title + helper line                                  | search identical gibberish in both                                                                         |
| 14  | Dialogs / flyers / modals      | title, body copy, labels, footer buttons, X/close aria            | open EVERY create/edit/confirm/delete dialog in both → compare → close                                     |
| 15  | Confirmation prompts           | "Are you sure…?" + Yes/No/Cancel                                  | trigger delete/cancel/discard flows                                                                        |
| 16  | Pagination / footer / tooltips | "Show row:", "Page X of Y", footer links, loading texts, tooltips | visible / hover                                                                                            |

**Test-design mirror:** one test function per category-per-page (naming
`test_globalization_<page>_<category>`; dialogs and dropdowns get one test EACH). Status
maps and option lists are looped inside the test — one `_verify_text` per value, never a
single sampled value. Categories with `KEY_MISSING_IN_EN` findings keep a
`_verify_missing_key_locator` soft probe cross-referenced to findings.md.

---

## 7. The findings ledger (feeds globalization defects)

All comparison output for a module+language lives under:

```
MCP/stories/<Module>/globalization/<lang>/
├── <screen>.compare.json      ← raw glob_compare_sessions output (save_to)
├── findings.md                ← curated table: every real gap, one row each
└── coverage.md                ← screens/sections visited vs the module's full map
```

`findings.md` row format (one per gap — this is what a defect gets raised from):

```markdown
| #   | Screen / section              | Element (testid/path)        | EN text   | <LANG> text | Type           | Locale key                     | Key in en.json? | Key in <lang>.json? |
| --- | ----------------------------- | ---------------------------- | --------- | ----------- | -------------- | ------------------------------ | --------------- | ------------------- |
| 1   | Orders landing — search panel | th[data-testid=col-orderQty] | Order Qty | Order Qty   | NOT_TRANSLATED | outboundOrder.columns.orderQty | ✅              | ❌ MISSING          |
```

Finding **types**: `NOT_TRANSLATED` (UI shows English in the target language),
`KEY_MISSING_IN_LANG` (en.json has it, `<lang>.json` doesn't), `KEY_MISSING_IN_EN`
(UI text has no key at all → also a POM MISSING_KEY), `VALUE_IDENTICAL` (key exists but
value untranslated), `MISSING_ELEMENT` (element absent in target).

A **serious** globalization gap (a whole page/section untranslated, a broken layout, a
missing language file) additionally gets the formal defect document — follow
[../defect_documentation/SKILL.md](../defect_documentation/SKILL.md) (`screenshot_step(…, defect=true)`
on the TARGET-language session + `_defect.json` + build_defect_docx).

---

## 8. Globalization self-check (before finishing any glob work)

- [ ] **Category matrix complete**: every screen × every applicable category of §6-B compared
      (✅) or excluded with a written reason in coverage.md — no silent skips.
- [ ] **Per-category tests exist**: each ✅ cell maps to a named test function in testcases.md
      (headers, tabs, buttons, status labels, field labels+placeholders, EVERY dropdown's
      options, table columns, toasts, validation messages, empty state, EVERY dialog,
      confirm prompts, pagination/footer).
- [ ] **Loops, not samples**: status maps / dropdown option lists / column sets are asserted
      value-by-value inside the test — never just the first value.
- [ ] POM: plain class, `LazyLocator(lambda t: [...])` for every translated text, plain lists
      for testid/structural anchors, keys documented in the docstring, MISSING_KEY flagged.
- [ ] Every key used in a LazyLocator **exists in `en.json`** of the module's locale folder
      (verified via `glob_check_locale_keys` reverse lookup — never invented).
- [ ] Web component: `_norm_loc()` around every POM locator; dropdown values via `translate(key)`.
- [ ] Tests: `driver` fixture; 3-level markers; `configure_i18n(language_folder=…)` first;
      login-with-language; section-wise `allure.step` blocks; the standard `_verify_*` helpers;
      final `_take_screenshot`.
- [ ] Tests run green in **en** AND the target language (`TestLanguage` env swap).
- [ ] Findings ledger complete: every NOT_TRANSLATED / missing-key gap recorded with its
      locale key + file evidence; serious gaps have their formal defect docx.
- [ ] No `time.sleep` (use `web.wait…`), no assertions outside tests, no hardcoded test data.
