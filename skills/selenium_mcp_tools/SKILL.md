---
description: Reference for the selenium-mcp-server tools and how each maps onto CustomPyAutoWeb.
name: selenium_mcp_tools
---

# Skill — Selenium MCP Tools (selenium-mcp-server)

The `selenium` MCP server (`selenium-mcp-server`, PyPI 1.2.0) is the toolbox the AI assistant
drives during a live walk. This skill is the catalog of its tools, their argument shapes, and —
crucially — **which `CustomPyAutoWeb` method each one maps to** when you turn the walk into an
authored pytest-bdd test. (Argument names follow the published package; if the live tool picker
shows a slightly different name, use what the picker shows — the capability is the same.)

The server is launched via **`MCP/selenium_mcp_launch.py`**, which (a) fixes the 1.2.0 startup
crash and (b) registers **12 extra fast tools** on top of the 20 built-ins — see §7.

> Install: `pip install -r MCP/requirements-mcp.txt`. Register: copy the `servers` block from
> `MCP/mcp.servers.json` into `.vscode/mcp.json` (it runs `selenium_mcp_launch.py` with the venv
> python), reload VS Code.

---

## 1. Selector types

Most element tools take a `selector` + `selector_type`. Supported `selector_type` values:
`css` (default), `xpath`, `id`, `name`, `tag`, `class`, `link_text`, `partial_link_text`.

> **Author with XPath.** The repo's POM locators are `[(By.XPATH, "...")]`. When you resolve a
> locator live, prefer `selector_type: "xpath"` so the exact string you validated is the one you
> bake into the Page Object.

---

## 2. Browser & session management

| MCP tool         | Example args                                                | Purpose                                                                                                                                                                                                                                                                                      |
| ---------------- | ----------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `start_browser`  | `{ "browser": "chrome", "options": { "headless": false } }` | Launch a session (use full mode to watch). The launcher auto-adds **`--force-device-scale-factor=0.75`** (75% zoom, same as the pytest-bdd runtest) + `--window-size=1920,1080`, so the walk + every screenshot render the viewport the scripted suite uses — you don't pass these yourself. |
| `list_sessions`  | `{}`                                                        | List active sessions (sanity check).                                                                                                                                                                                                                                                         |
| `switch_session` | `{ "session_id": "<id>" }`                                  | Switch the active session.                                                                                                                                                                                                                                                                   |
| `close_session`  | `{ "session_id": "<id>" }`                                  | Close a session when the walk is done.                                                                                                                                                                                                                                                       |

> Keep **one** session for a single-story walk (sequential, one by one). Start it once, reuse it.

---

## 3. Navigation & page info

| MCP tool        | Example args                                                           | Maps to (`CustomPyAutoWeb`)                 |
| --------------- | ---------------------------------------------------------------------- | ------------------------------------------- |
| `navigate`      | `{ "url": "https://qa.americoldcompass.com/", "wait_for_load": true }` | `driver.get(url)` / nav web_component       |
| `get_page_info` | `{ "include_title": true, "include_url": true }`                       | `driver.current_url` / `verify_current_url` |

---

## 4. Element interaction (the workhorses)

| MCP tool           | Example args                                                                                                 | Author it as (`web = CustomPyAutoWeb(bdd_driver)`)                                            |
| ------------------ | ------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------- |
| `find_element`     | `{ "selector": "//button[@data-testid='cmo-tab-templates']", "selector_type": "xpath" }`                     | `web.is_element_present(locatorList=PAGE.locatorX)` / `wait_element_visibility`               |
| `click_element`    | `{ "selector": "...", "selector_type": "xpath" }`                                                            | `web.click_wait_locator(locatorList=PAGE.locatorX)`                                           |
| `send_keys`        | `{ "selector": "...", "selector_type": "xpath", "text": "Allentown", "clear_first": true }`                  | `web.enter_text(locatorList=PAGE.locatorX, text=...)` (clear → `web.clear_element_with_keys`) |
| `get_element_text` | `{ "selector": "...", "selector_type": "xpath" }`                                                            | `web.get_element_text(web_element)` / `get_element_attribute`                                 |
| `hover`            | `{ "selector": "...", "selector_type": "xpath" }`                                                            | `web.hover_over_menu_header(...)` / ActionChains helper                                       |
| `double_click`     | `{ "selector": "...", "selector_type": "xpath" }`                                                            | a web_component using ActionChains                                                            |
| `right_click`      | `{ "selector": "...", "selector_type": "xpath" }`                                                            | a web_component using ActionChains                                                            |
| `drag_and_drop`    | `{ "selector": "...", "target_selector": "...", "selector_type": "xpath", "target_selector_type": "xpath" }` | a drag web_component                                                                          |
| `press_key`        | `{ "key": "Enter" }`                                                                                         | `web.enter_text_and_enter(...)` / send a Keys chord                                           |
| `upload_file`      | `{ "selector": "input[type='file']", "selector_type": "css", "filePath": "C:/.../POD.pdf" }`                 | the module's file-upload web_component (real file from `MCP/fixtures/<STORY-ID>/`)            |
| `wait_for_element` | `{ "selector": "...", "selector_type": "xpath", "wait_for_visible": true }`                                  | `web.wait_element_visibility(locatorList=PAGE.locatorX, wait_time=60)`                        |

---

## 5. Advanced actions

| MCP tool          | Example args                             | Use for                                                                                                                                                                                                                                   |
| ----------------- | ---------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `take_screenshot` | `{ "full_page": true }`                  | A raw shot (base64 / `outputPath`). For **defect evidence** prefer the **`screenshot_step`** fast tool with `defect: true` (§7) — it writes `TestcaseExecution/<module>/<STORY-ID>/<CASE-ID>/<CASE-ID>_defect_<N>.png` + creates folders. |
| `execute_script`  | `{ "script": "return document.title;" }` | Read computed state / scroll / inspect attributes when role/text isn't enough.                                                                                                                                                            |

---

## 6. The walk → author contract

Every MCP action you take **must** have a 1:1 counterpart in the authored pytest-bdd test, so a
case that passed live passes as the script (same locator, same wait):

```
MCP walk                                   Authored pytest-bdd (the @when/@then body)
─────────────────────────────────────────  ──────────────────────────────────────────────────
click_element(xpath = X)              →     web.click_wait_locator(locatorList=PAGE.locatorX)
send_keys(xpath = X, text = "Allentown") →  web.enter_text(locatorList=PAGE.locatorX, text=val)
wait_for_element(xpath = X, visible)  →     web.wait_element_visibility(locatorList=PAGE.locatorX, wait_time=60)
get_element_text(xpath = X)           →     actual = web.get_element_text(web.find_element(PAGE.locatorX))
                                            web.element_should_be_visible(locator=PAGE.locatorX)   # the @then assert
take_screenshot                       →     (only for defect evidence — the framework's conftest handles Allure screenshots)
```

When you resolve a new locator live (e.g. `//button[@data-testid='start-new-audit']`), add it to
the Page Object as `locatorStartNewAuditButton = [(By.XPATH, "//button[@data-testid='start-new-audit']")]`
and reference it from the step — **never** put the raw selector in the step.

> Full toolbox of `CustomPyAutoWeb` methods: [../custompyautoweb_api/SKILL.md](../custompyautoweb_api/SKILL.md).

---

## 7. Fast tools (fewer round-trips, registered by the launcher)

`MCP/selenium_mcp_launch.py` appends **12 extra tools** to the server's live `tool_registry`.
Each one collapses a multi-call sequence (login, fill a form, walk a Radix combobox, prove a
step) into **one** tool call — so Antigravity does far fewer round-trips and the walk runs fast.
Prefer these whenever one fits; fall back to the 20 built-ins for anything bespoke.

| Fast tool                    | What it does in one call                                                                                                                                                                                                                                              | Replaces (built-ins)                                   |
| ---------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------ |
| `login_sso`                  | The **whole** Azure-AD SSO: navigate → Sign in with SSO → email → Next → password → Sign in → Stay-signed-in ×2 → land on the app shell.                                                                                                                              | ~10 × (`navigate`/`send_keys`/`click_element`)         |
| `fill_form`                  | Fill **many** fields in order (each `{selector, selector_type, text, clear_first?, press_enter?}`).                                                                                                                                                                   | N × `send_keys`                                        |
| `click_sequence`             | Click an **ordered list** of targets (wait-clickable + JS fallback), `continue_on_error?`, `pause_ms?`.                                                                                                                                                               | N × (`wait_for_element` + `click_element`)             |
| `wait_and_click`             | Wait-clickable → click (JS fallback) one target.                                                                                                                                                                                                                      | `wait_for_element` + `click_element`                   |
| `select_radix_option`        | Open a shadcn/Radix combobox/menu (click trigger) → wait for the `role=option`/`menuitem` by text → click it.                                                                                                                                                         | `click_element` + `wait_for_element` + `click_element` |
| `scroll_into_view_and_click` | `scrollIntoView(center)` → click (JS fallback).                                                                                                                                                                                                                       | `execute_script` + `click_element`                     |
| `get_texts`                  | Read text of **all** matches of one selector, or of a list of targets, in one call.                                                                                                                                                                                   | N × `get_element_text`                                 |
| `get_element_state`          | One snapshot of an element: text/value/displayed/enabled/selected/aria-\*/disabled/class + all `attributes`.                                                                                                                                                          | several `execute_script` / `get_element_text`          |
| `snapshot_dom`               | Compact JSON of the visible **interactive** elements (`tag/role/testid/name/type/text`, capped) — fast grounding without a full a11y dump.                                                                                                                            | a big `execute_script`                                 |
| `assert_text`                | Verify text **contains / exact / absent**; a mismatch is returned as an **error** (never silently "passed").                                                                                                                                                          | `get_element_text` + manual compare                    |
| `wait_for_idle`              | Wait until `document.readyState=complete` **and** no visible spinner (`[role=progressbar]`, `.animate-spin`, `[aria-busy=true]`, `[data-loading=true]`), optional target visible.                                                                                     | polling `execute_script`                               |
| `screenshot_step`            | Save **defect evidence** (`defect: true`) to `TestcaseExecution/<module>/<STORY-ID>/<CASE-ID>/<CASE-ID>_defect_<N>.png` (creates folders) — used ONLY when a genuine bug is observed, never per step; `also_cases` copies the **same** PNG into other affected cases. | `take_screenshot` + manual file I/O                    |

### Argument shapes (the ones with structure)

```jsonc
// login_sso — credentials come from the environment (SSO_USERNAME / SSO_PASSWORD); never inline.
{ "username": "<optional override>", "password": "<optional override>", "timeout_ms": 60000 }

// fill_form — each field is filled in array order
{ "fields": [
    { "selector": "input[name='loginfmt']", "selector_type": "css", "text": "a@b.com", "clear_first": true },
    { "selector": "//input[@name='q']",      "selector_type": "xpath", "text": "Allentown", "press_enter": true }
] }

// click_sequence — ordered, resilient
{ "targets": [
    { "selector": "//button[normalize-space()='Apps']", "selector_type": "xpath" },
    { "selector": "//button[normalize-space()='CMO']",  "selector_type": "xpath" }
  ], "continue_on_error": false, "pause_ms": 200 }

// select_radix_option — combobox / dropdown / menu by visible option text
{ "trigger_selector": "//button[@role='combobox']", "trigger_selector_type": "xpath",
  "option_text": "Allentown, PA", "match": "contains" }   // match: "exact" | "contains"

// assert_text — a FAIL returns isError:true (do NOT fake a pass)
{ "selector": "...", "selector_type": "css", "expected": "Audit created", "mode": "contains" } // contains|exact|absent

// screenshot_step — DEFECT evidence writer (1-based defect-shot index; only on a confirmed bug)
{ "module": "CMO", "story_id": "AMCC-5128", "case_id": "AMCC-TC-15200", "step": 1,
  "defect": true,
  "also_cases": ["AMCC-TC-15201"] }   // same defect visible in another affected case
```

### How the fast tools map back to the authored test

The walk → author contract (§6) still holds — a fast tool just stands in for the sequence it
replaces, and you author the **same** `CustomPyAutoWeb` calls behind a web_component:

```
login_sso                 →  business_components/web_components/login.py :: login_to_application_sso(driver)
fill_form([...])          →  a web_component that does N × web.enter_text(locatorList=PAGE.locatorX, text=...)
click_sequence([...])     →  a web_component of N × web.click_wait_locator(locatorList=PAGE.locatorX)
select_radix_option(...)  →  web.type_and_click_first_dropdown_option(...) / a typeahead web_component
assert_text(expected)     →  the @then: web.element_should_contain(...) / a bare assert with a message
screenshot_step(…, defect=true) →  (defect evidence for the live MCP run + the defect .docx — never authored code;
                             authored tests rely on conftest's Allure screenshots)
```

> Evidence + report: **screenshots ONLY for defects** — a confirmed bug gets
> `<CASE-ID>_defect_<N>.png` + `<CASE-ID>_defect.json` + the rendered `<CASE-ID>_defect.docx`
> (`python MCP/build_defect_docx.py --story … --case <ID>`; see
> [../defect_documentation/SKILL.md](../defect_documentation/SKILL.md)). Passing steps produce **zero**
> PNGs. Statuses/actuals/bugs go into `execution-results.json`, and
> `MCP/build_execution_docx.py` rolls the story into `<STORY-ID>_Execution.docx`. **Never fake a
> pass** — a real mismatch stays a FAIL with the bug documented.
> See [../mcp_execution_protocol/SKILL.md](../mcp_execution_protocol/SKILL.md) §4–§5.
