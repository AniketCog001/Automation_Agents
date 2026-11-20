---
description: 'Cookbook for page objects and locators: CustomPyAutoWeb base class,
  locator list shape, dynamic locators.'
name: page_objects
---

# Skill 03 — Page Objects & Locators

Read [../framework_architecture/SKILL.md](../framework_architecture/SKILL.md) first. Page
objects live in object_repository/pages/Apps/`<Module>`/ and are the **only** place locators
are allowed to exist.

> **🧠 ANTI-HALLUCINATION CONTRACT (STRICT):**
> Every locator declared in a Page Object MUST be grounded in ground-truth source code or DOM evidence — extracted from the developer UI repository (`C:\Users\Administrator\UIREPOAMCC\amc-cold-compass-portal-ui\apps\<module_name>`), retrieved from `kg_context_pack`, or observed live via `snapshot_dom(interactive_only=true)`. **NEVER guess XPaths or attribute names.**

---

## 1. Class skeleton (real shape)

```python
# pylint: disable=all
from selenium.webdriver.common.by import By
from utilities.genericUtils import CustomPyAutoWeb


class CMOLandingPage(CustomPyAutoWeb):

    # ----------------- header locators --------------------
    locatorCMOHeading = [(By.XPATH, '//*[contains(@class , "h6") and text() = "CMO"]')]
    locatorTemplatesHeading = [(By.XPATH, '//h2[text() = "Templates"]')]
```

- Base class is **`CustomPyAutoWeb`** (never `BasePage`/`NoPage`).
- File name is PascalCase ending in `Page.py` (e.g. `CmoLandingPage.py`, `NewAuditPage.py`).
- Class name is PascalCase ending in `Page` (e.g. `CMOLandingPage`).

---

## 2. The locator shape — a LIST of ONE tuple (critical)

```python
locatorCMOHeading = [(By.XPATH, '//*[contains(@class,"h6") and text()="CMO"]')]
#                   ^ list   ^ tuple (By, "selector")
```

- Always `[(By.XPATH, "...")]` — a **list** wrapping a **`(By, selector)` tuple**.
- This is what `CustomPyAutoWeb` expects via `locatorList=` / `locator=`.
- A bare string or a bare tuple will break the helper methods — keep the list.

---

## 3. Naming convention

| Kind                       | Pattern                                                        | Example                        |
| -------------------------- | -------------------------------------------------------------- | ------------------------------ |
| Locator variable           | `locator` + PascalCase description                             | `locatorCMOTabTemplates`       |
| State variant              | base name + `Active` / `Inactive` / `Selected` / `NotSelected` | `locatorCMOTabTemplatesActive` |
| Dynamic (by runtime value) | `locator…ByName` as a `lambda`                                 | `locatorCMONavTabByName`       |

Real state-variant group:

```python
# --- Templates Tab ---
locatorCMOTabTemplates = [(By.XPATH, "//button[@data-testid='cmo-tab-templates']")]
locatorCMOTabTemplatesActive = [
    (By.XPATH, "//button[@data-testid='cmo-tab-templates' and contains(@class,'text-primary-white')]")
]
locatorCMOTabTemplatesInactive = [
    (By.XPATH, "//button[@data-testid='cmo-tab-templates' and contains(@class,'text-teal-100')]")
]
```

---

## 4. Dynamic locators — `lambda` returning the list

When the selector depends on a runtime value, declare a **lambda** that returns the same
`[(By.XPATH, f"...")]` list:

```python
# --- Dynamic tab by label text (reused across all tab names) ---
locatorCMONavTabByName = lambda tab_name: [
    (By.XPATH, f"//nav[contains(@class,'flex')]//button[normalize-space(text())='{tab_name}']")
]
```

Call it from a step/web_component by invoking the lambda:

```python
web.click_wait_locator(locatorList=CMOLandingPage.locatorCMONavTabByName("Supervisors"))
```

> Use an f-string lambda for runtime values; never bake a runtime value into a static locator.

---

## 5. Section dividers (match the file you're editing)

Two real divider styles appear in the repo — reuse whichever the target file already uses:

```python
# ----------------- header locators --------------------
# ========================= CMO Top Navigation Tabs =========================#
# --- Templates Tab ---
```

Group locators by UI section; put a `# --- <Element> ---` label above each element/state group.

---

## 6. XPath quality (anchor priority — text first)

Follow the priority order in [.agents/skills/01_xpath_pom/SKILL.md](../skills/01_xpath_pom/SKILL.md):
`text()/normalize-space()` → `@aria-label/@role` → `@data-testid` → `@id/@name` →
semantic `@class` + tag (last resort) → positional (flag it).

This repo's app exposes excellent `@data-testid` hooks (e.g. `cmo-tab-templates`) and Tailwind
**state classes** (`text-primary-white` = active, `text-teal-100` = inactive). Combine a
testid/text anchor with the state class for state variants. **Never** anchor on hashed/utility
classes alone (`css-1x2y3z`, `flex`, `gap-2`, `mt-4`).

---

## 7. Page object self-check

- [ ] `class <Name>Page(CustomPyAutoWeb):` and imports `By` + `CustomPyAutoWeb`.
- [ ] Every locator is `[(By.XPATH, "...")]` (list of one tuple).
- [ ] State variants suffixed `Active/Inactive/Selected/NotSelected` and only for states that
      exist in the HTML.
- [ ] Runtime-variable selectors are `lambda …: [(By.XPATH, f"…")]`.
- [ ] No locator string appears anywhere outside a page object.
