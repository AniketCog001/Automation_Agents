---
description: Catalog of the CustomPyAutoWeb helper methods (the Selenium toolbox)
  with real call signatures.
name: custompyautoweb_api
---

# Skill 02 — CustomPyAutoWeb API (the Selenium toolbox)

`CustomPyAutoWeb` (in utilities/genericUtils.py, `class CustomPyAutoWeb(commonUtilities)`) is the
gateway to every browser action. Build it once per step/function:

```python
web = CustomPyAutoWeb(bdd_driver)   # in a step      (driver from the bdd_driver fixture)
web = CustomPyAutoWeb(driver)       # in a web_component (driver is the first parameter)
```

Page objects also **subclass** it, so a page instance has these methods too.

> **Call signatures below are taken from real usage in the repo.** Locators are passed as the
> page-object **list** `[(By.XPATH, "...")]`. Action methods take `locatorList=`; some verify
> methods take `locator=` — match the examples exactly.

---

## 1. Waits & presence

| Method                       | Real call                                                                        | Use for                                                                                           |
| ---------------------------- | -------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| `wait_element_visibility`    | `web.wait_element_visibility(locatorList=PAGE.locatorX, wait_time=60)`           | **Gate before acting/reading.** Most common wait.                                                 |
| `wait`                       | `web.wait(2)` / `web.wait(0.3)`                                                  | Short bounded settle (animation, typeahead populate). The sanctioned alternative to `time.sleep`. |
| `is_element_present`         | `web.is_element_present(locatorList=PAGE.locatorX, timeout=15)` → `bool`         | Branch on presence without failing.                                                               |
| `is_element_present_with_js` | `web.is_element_present_with_js(locatorList=PAGE.locatorX, timeout=15)` → `bool` | Presence via JS (DOM-only).                                                                       |

## 2. Clicks

| Method                        | Real call                                                    | Use for                                               |
| ----------------------------- | ------------------------------------------------------------ | ----------------------------------------------------- |
| `click_wait_locator`          | `web.click_wait_locator(locatorList=PAGE.locatorX)`          | **Default click** (waits then clicks).                |
| `element_click_using_JS`      | `web.element_click_using_JS(locatorList=PAGE.locatorX)`      | Fallback when a normal click is intercepted/overlaid. |
| `click_element_by_text`       | `web.click_element_by_text(locator, text, timeout=10)`       | Click an item in a collection by its visible text.    |
| `click_element_by_exact_text` | `web.click_element_by_exact_text(locator, text, timeout=10)` | Exact-text variant.                                   |
| `click_first_element`         | `web.click_first_element(locatorList=PAGE.locatorX)`         | Click the first match of a collection.                |

Common resilient pattern (from `CMO_Navigation`):

```python
try:
    web.click_wait_locator(locatorList=HomePage.locatorAppsBtn)
except Exception:
    web.element_click_using_JS(locatorList=HomePage.locatorAppsBtn)
```

## 3. Typing & input

| Method                    | Real call                                                             | Use for                                |
| ------------------------- | --------------------------------------------------------------------- | -------------------------------------- |
| `enter_text`              | `web.enter_text(locatorList=PAGE.locatorInput, text=value)`           | **Default type.**                      |
| `enter_text_and_enter`    | `web.enter_text_and_enter(locatorList=PAGE.locatorInput, text=value)` | Type then press Enter (submit/search). |
| `clear_element_with_keys` | `web.clear_element_with_keys(locatorList=PAGE.locatorInput)`          | Clear field before retyping.           |

Clear-then-type idiom (from `search_template_in_flyer`):

```python
current_val = web.get_element_attribute(locatorList=PAGE.locatorInput, attribute_name="value")
if current_val:
    web.clear_element_with_keys(locatorList=PAGE.locatorInput)
    web.wait(0.3)
web.enter_text(locatorList=PAGE.locatorInput, text=value)
```

## 4. Reading state (getters — return values, never assert)

| Method                                               | Real call                                                                      | Returns                                |
| ---------------------------------------------------- | ------------------------------------------------------------------------------ | -------------------------------------- |
| `get_element_attribute`                              | `web.get_element_attribute(locatorList=PAGE.locatorX, attribute_name="value")` | `str`                                  |
| `get_element_text`                                   | `web.get_element_text(web_element)`                                            | `str` (pass an element, not a locator) |
| `find_element`                                       | `web.find_element(locator)`                                                    | a single `WebElement`                  |
| `find_elements_from_list`                            | `web.find_elements_from_list(locatorList=PAGE.locatorX)`                       | `list[WebElement]`                     |
| `get_dropdown_options_text`                          | `web.get_dropdown_options_text(locator, wait_time=10)`                         | `list[str]`                            |
| `get_element_color` / `get_element_background_color` | `web.get_element_background_color(locator)`                                    | color string                           |
| `get_element_size`                                   | `web.get_element_size(locatorList=PAGE.locatorX)`                              | size                                   |

## 5. Assertions / verifies (raise on failure — call these in @then)

| Method                                                                        | Real call                                                                   | Checks                |
| ----------------------------------------------------------------------------- | --------------------------------------------------------------------------- | --------------------- |
| `element_should_be_visible`                                                   | `web.element_should_be_visible(locator=PAGE.locatorX)`                      | element visible       |
| `element_should_not_visible`                                                  | `web.element_should_not_visible(locator=PAGE.locatorX, timeout=10)`         | element hidden/absent |
| `element_should_exist`                                                        | `web.element_should_exist(locator=PAGE.locatorX, timeout=10)`               | element in DOM        |
| `element_should_not_exist`                                                    | `web.element_should_not_exist(locator=PAGE.locatorX, timeout=10)`           | element not in DOM    |
| `element_should_contain`                                                      | `web.element_should_contain(...)`                                           | element text contains |
| `element_should_be_empty`                                                     | `web.element_should_be_empty(locatorList=PAGE.locatorX)`                    | field empty           |
| `verify_current_url`                                                          | `web.verify_current_url(expected_url, timeout=10)`                          | URL match             |
| `verify_page_title`                                                           | `web.verify_page_title(expected_title, wait_time=10)`                       | title match           |
| `verify_toast_alert`                                                          | `web.verify_toast_alert(expected_message, timeout=5)`                       | toast text            |
| `verify_element_not_clickable`                                                | `web.verify_element_not_clickable(locatorList=PAGE.locatorX, wait_time=10)` | disabled/blocked      |
| `verify_dropdown_options` / `verify_dropdown_values` / `verify_dropdown_text` | `web.verify_dropdown_options(locator, expected_options)`                    | dropdown contents     |
| `verify_row_count_in_table`                                                   | `web.verify_row_count_in_table(table_locator, expected_row_count)`          | table rows            |

> These verify methods **raise** when the check fails. In a `@then` they sit **inside** the
> step's `try/except`, so the failure becomes the standard `AssertionError("Step FAILED: …")`.
> See [../assertions_and_allure/SKILL.md](../assertions_and_allure/SKILL.md).

## 6. Scrolling, hover, dropdowns, misc

| Method                                                   | Real call                                                                  |
| -------------------------------------------------------- | -------------------------------------------------------------------------- |
| `scroll_element_into_view`                               | `web.scroll_element_into_view(web_element)`                                |
| `scroll_to_bottom` / `scroll_to_top` / `scroll_to_right` | `web.scroll_to_bottom()`                                                   |
| `scroll_element_to_right`                                | `web.scroll_element_to_right(element_locator)`                             |
| `select_value_from_dropdown`                             | `web.select_value_from_dropdown(web_element, element_value, wait_time=20)` |
| `type_drop_down_click`                                   | `web.type_drop_down_click(locator, text)`                                  |
| `type_and_click_first_dropdown_option`                   | typeahead → pick first                                                     |
| `type_char_by_char_and_select_dropdown_option`           | slow typeahead select                                                      |
| `hover_over_menu_header`                                 | `web.hover_over_menu_header(menu_name)`                                    |
| `open_url_in_new_tab`                                    | `web.open_url_in_new_tab(url)`                                             |
| `logout_from_application`                                | `web.logout_from_application(profile_button, logOut_button)`               |

---

## 7. Rules when using this API

1. **Never** drop to raw `bdd_driver.find_element(...)` in a step — use these methods + a POM locator.
2. Pass the page-object **locator list** (`PAGE.locatorX`), not a bare string.
3. Use `locatorList=` for actions and `locator=` for the `element_should_*` verifies (as shown).
4. If a needed capability is missing here, check the method list in
   utilities/genericUtils.py before inventing logic — it is a large toolbox (toasts, tables,
   colors, dropdowns, calendars, dates are already covered).
5. Short waits use `web.wait(n)` — **never** `time.sleep`.
