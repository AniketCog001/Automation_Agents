---
description: 'Cookbook for shared web_component interactions: driver-first functions,
  CustomPyAutoWeb usage.'
name: web_components
---

# Skill 04 — web_components (shared interactions)

Read [../framework_architecture/SKILL.md](../framework_architecture/SKILL.md) first. The module
web_component (e.g. business_components/web_components/cmo.py) is the **single home** for any UI
interaction reused across more than one step or story. Step defs **call** these functions; they
never re-implement the logic.

> **Developer UI Repo Source Ground-Truth Rule:** Before writing or editing web_component interaction functions, agents MUST inspect the developer Next.js UI source code in `C:\Users\Administrator\UIREPOAMCC\amc-cold-compass-portal-ui\apps\<module_name>` (`app/_components/`, `app/_hooks/`, `app/_common/`). Analyzing the React components, `data-testid` attributes, state hooks (`useState`), ARIA attributes (`aria-expanded`, `role`), conditional rendering logic, and API maps provides 100% accurate ground-truth for interaction flows, wait conditions, and locators.

---

## 1. File conventions

- Header: `# pylint: disable=all`.
- Imports: stdlib + selenium + `from utilities.genericUtils import CustomPyAutoWeb` + the POM
  classes whose locators the functions drive + `import allure`.
- Functions are **module-level** (not a class), `driver`-first.

---

## 2. Function shape (real)

```python
def search_template_in_flyer(driver, search_text: str, timeout: int = 30):
    """
    Type a search term into the Select Template sidebar search field.

    Clears any existing text, types the search_text, and waits briefly for
    the template list to update.

    Args:
        driver:      Selenium WebDriver instance.
        search_text: The text to type into the search input.
        timeout:     Max seconds to wait for the search input to be visible.
    """
    web = CustomPyAutoWeb(driver)
    web.wait_element_visibility(
        locatorList=SelectTemplateFlyerPage.locatorSelectTemplateSearchInput,
        wait_time=timeout,
    )
    current_val = web.get_element_attribute(
        locatorList=SelectTemplateFlyerPage.locatorSelectTemplateSearchInput,
        attribute_name="value",
    )
    if current_val:
        web.clear_element_with_keys(
            locatorList=SelectTemplateFlyerPage.locatorSelectTemplateSearchInput
        )
        web.wait(0.3)
    web.enter_text(
        locatorList=SelectTemplateFlyerPage.locatorSelectTemplateSearchInput,
        text=search_text,
    )
```

The contract (every function):

- **`driver` first**, then values/labels, then `timeout: int = <default>` (commonly `30` or `10`).
- Builds `web = CustomPyAutoWeb(driver)` and drives **POM locators** — never a hardcoded selector.
- Multi-line docstring with an **Args** section (match the file).
- **Zero assertions.** Returns a value (getter) or `None` (action).
- Short waits use `web.wait(n)`; gates use `web.wait_element_visibility(...)`. No `time.sleep`.

---

## 3. Cross-step state — `get_`/`set_` module globals

When a value produced in one step (e.g. an auto-generated template name) must be read by a
later step, expose it through paired globals in the web_component:

```python
TEMPLATE_NAME: str = ""

def set_template_name(name) -> None:
    """Set the global TEMPLATE_NAME. Pass None to clear."""
    global TEMPLATE_NAME
    TEMPLATE_NAME = "" if name is None else str(name)

def get_template_name() -> str:
    """Return the current global TEMPLATE_NAME (empty string if not set)."""
    return TEMPLATE_NAME
```

Steps then do `set_template_name(name)` in one step and `get_template_name()` in another. This
is the repo's pattern for shared state that outlives a single function call (the step-local
alternative is the `_ctx` dict — see Skill 01).

---

## 4. Button / option maps (parameterize by logical label)

For a group of mutually exclusive buttons, map a **logical label → POM locator variants** and
let one function act on any of them:

```python
audit_button_map = {
    "cmo": {
        "general":      CMOCreateTemplatePage.locatorAuditTypeCMOButton,
        "selected":     CMOCreateTemplatePage.locatorAuditTypeCMOSelected,
        "not_selected": CMOCreateTemplatePage.locatorAuditTypeCMONotSelected,
    },
    "facility": { ... },
}
```

A selection function takes the label (`"cmo"`), looks up `["general"]` to click and
`["selected"]` to confirm the active state — no locator is hardcoded inside the function.

---

## 5. Resilient navigation (retry + JS-click fallback) — real pattern

```python
def CMO_Navigation(driver):
    """Click Apps menu, then CMO button; resilient to slow loads / transient timeouts."""
    web = CustomPyAutoWeb(driver)
    for attempt in range(1, 4):
        try:
            web.wait_element_visibility(locatorList=HomePage.locatorAppsBtn, wait_time=60)
            try:
                web.click_wait_locator(locatorList=HomePage.locatorAppsBtn)
            except Exception:
                web.element_click_using_JS(locatorList=HomePage.locatorAppsBtn)
            # … click CMO, confirm landing …
            break
        except (TimeoutException, WebDriverException) as e:
            if attempt < 3:
                driver.refresh()
                web.wait(2)
```

---

## 6. What goes here vs the step def

**Goes to web_component** if ANY is true:

- Same interaction used by 2+ steps/stories (even with different parameter values).
- Wraps a multi-step Selenium sequence (type → wait → click suggestion) into one call.
- A generic getter/setter of UI state read/written by multiple steps.

**Stays in the step def** if:

- Single-use & scenario-specific, **or** it is an assertion (assertions never leave step defs),
  **or** it mixes action + assertion (split: action → here, assert → step def).

---

## 7. web_component self-check

- [ ] `driver` is the first parameter; `timeout` has a default; locators & values are parameters.
- [ ] No hardcoded locator or test data inside the function.
- [ ] Exactly one responsibility (one action OR one read).
- [ ] Zero assertions; getters return a value, actions return `None`.
- [ ] Explicit waits / `web.wait(n)` only — no `time.sleep`.
- [ ] No near-duplicate of an existing function — reuse/extend instead (one function per pattern).
- [ ] Docstring matches the file's Args/Returns style.
