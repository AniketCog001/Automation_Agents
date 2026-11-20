---
description: 'How assertions and Allure evidence actually work in this repo: CustomPyAutoWeb
  verify methods.'
name: assertions_and_allure
---

# Skill 05 — Assertions & Allure Evidence

Read [../step_definitions/SKILL.md](../step_definitions/SKILL.md) and
[../custompyautoweb_api/SKILL.md](../custompyautoweb_api/SKILL.md) first. Assertions live
**only** in step-def `@then` steps. Web_components and page objects contain **zero** assertions.

---

## 1. The repo's assertion mechanism (two complementary parts)

**Part A — a `CustomPyAutoWeb` verify method that raises on failure** does the actual check:

```python
web.element_should_be_visible(locator=CMOLandingPage.locatorCMOHeading)
web.element_should_not_visible(locator=PAGE.locatorX, timeout=10)
web.verify_current_url(expected_url, timeout=10)
web.verify_toast_alert("Template published successfully")
```

**Part B — the step's `try/except` re-raises as a standard `AssertionError`** so every failure
reads the same way in the report:

```python
@then("CMO landing page should be displayed")
def verify_cmo_landing_page(bdd_driver):
    try:
        web = CustomPyAutoWeb(bdd_driver)
        web.wait_element_visibility(locatorList=CMOLandingPage.locatorCMOHeading, wait_time=60)
        web.element_should_be_visible(locator=CMOLandingPage.locatorCMOHeading)   # ← the check
        allure.attach(
            "CMO landing page is displayed — CMO heading visible",
            name="Verification — CMO Landing Page Displayed",
            attachment_type=allure.attachment_type.TEXT,
        )
    except Exception as e:                                                        # ← the wrapper
        raise AssertionError(
            f"Step FAILED: 'CMO landing page should be displayed' - "
            f"Error occurred in step [verify_cmo_landing_page]. Details: {e}"
        ) from e
```

> **Prefer a `CustomPyAutoWeb` verify method** for the check. Use a **bare `assert`** only when
> no verify method fits (e.g. comparing two values you computed) — and then give it a
> descriptive message, still inside the same `try/except`.

---

## 2. Bare-assert fallback (when no verify method fits)

```python
actual = web.get_element_attribute(locatorList=PAGE.locatorField, attribute_name="value")
expected = _ctx["expected_value"]
try:
    assert expected in actual, (
        f"Text validation failed.\n"
        f"Expected to contain : '{expected}'\n"
        f"Actual found        : '{actual}'"
    )
    allure.attach(
        f"Validation type : text contains\n"
        f"Element checked : Field value\n"
        f"Expected        : '{expected}'\n"
        f"Actual          : '{actual}'\n"
        f"Result          : PASSED",
        name="Verification — Field Value Contains Expected",
        attachment_type=allure.attachment_type.TEXT,
    )
except Exception as e:
    raise AssertionError(
        f"Step FAILED: '<exact then text>' - Error occurred in step [<func>]. Details: {e}"
    ) from e
```

- Text **contains** → `in`. **Exact** → `==`. **Count** → cast with `int(...)` first.
  **Negative** → `not in` / `!=`. Always a multi-line message; never a bare `assert x`.

---

## 3. Allure evidence convention

`allure.attach(...)` takes the **body as the first positional argument**, then `name=` and
`attachment_type=` (this is the real repo call style):

```python
allure.attach(
    "<body text>",
    name="<Category> — <specific what & where>",
    attachment_type=allure.attachment_type.TEXT,
)
```

Naming by step type:
| Step | `name=` prefix | Body should include |
|------|----------------|---------------------|
| `@given` | `Precondition — …` | what setup state was established |
| `@when` | `Action — …` | the action + the input value + (optionally) the URL |
| `@then` | `Verification — …` | validation type, element, expected, actual, result |

Real examples:

```python
# @when
allure.attach(f"Navigated to CMO\nURL: {bdd_driver.current_url}",
              name="Action — Navigate to Apps > CMO",
              attachment_type=allure.attachment_type.TEXT)

# @then
allure.attach("Supervisor My Audit landing page displayed",
              name="Verification — My Audit Landing Page Displayed",
              attachment_type=allure.attachment_type.TEXT)
```

Rules:

- Every `@when` and every `@then` attaches evidence. `@given` attaches the precondition.
- `name=` must say **what** and **where** — never `"Result"`, `"Check"`, `"Step"`, `"Verification"` alone.
- Put the `@then` attach **after** the check passes (so it only records on success).
- `attachment_type` matches the body: `TEXT` for strings, `JSON` for objects, `PNG` for screenshots.

---

## 4. Assertion/Allure self-check

- [ ] The check uses a `CustomPyAutoWeb` verify method where one fits; otherwise a bare `assert`
      with a multi-line message.
- [ ] The check sits inside `try/except … raise AssertionError("Step FAILED: '<step>' - … [<func>]. Details: {e}") from e`.
- [ ] No assertion exists in any web_component or page object.
- [ ] `@given` → `Precondition —`, `@when` → `Action —`, `@then` → `Verification —` attach present.
- [ ] No vague Allure `name=`; `@then` attach placed after the passing check.
