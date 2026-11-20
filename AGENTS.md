# Antigravity Instructions — Python + Selenium + pytest-bdd (BDD / POM) Automation

> [!IMPORTANT]
> **Use Selenium MCP & Developer UI Source Code Ground Truth**: Antigravity Chat MUST utilize the `selenium` MCP server tools to drive the browser and **MUST analyze the Developer UI Source Code** at `C:\Users\Administrator\UIREPOAMCC\amc-cold-compass-portal-ui\apps\<module>` when authoring or healing Page Objects, Web Components, Step Definitions, Feature Files, or test logic for any module.


These are **always-on, non-negotiable rules** for this workspace.
Every prompt file in `.agents/prompts/` builds on the architecture defined here.
Every skill file in `.agents/skills/` is the repo-specific cookbook those prompts load.
Do NOT restate these rules back to the user — follow them silently and precisely.

> **Developer UI Repo Source Ground Truth Rule**: Before authoring or healing ANY test file (POM, Web Component, Step Def, Feature file), agents MUST inspect the developer Next.js UI source code located at `C:\Users\Administrator\UIREPOAMCC\amc-cold-compass-portal-ui\apps\<module_name>` or `packages/`. Analyzing the React components, `data-testid` attributes, state hooks (`useState`), ARIA attributes (`aria-expanded`, `role`), conditional rendering logic, and API maps provides 100% accurate ground-truth for locators, initial render states, and UI interactions.

> **This is a `pytest-bdd` project, not Behave.** Steps bind to features via `scenarios(...)`,
> receive the driver through the **`bdd_driver`** fixture (there is **no `context` object**),
> drive pages through **`CustomPyAutoWeb`** + page-object locator **lists** `[(By.XPATH, "...")]`,
> and share state via a module-level **`_ctx = {}`** dict and web_component `get_`/`set_` globals.
When any generic BDD/POM instinct conflicts with these facts, the facts win — they are derived
from the real code and captured in`.agents/skills/`.

---

## STEP 0 — Knowledge-graph bootstrap & Token Optimization (always on)

Before answering ANY request in this workspace, call the **`kg_bootstrap`** MCP tool
(if the Selenium/KG MCP server is available) and treat what it returns — previous session
summaries, decisions, open TODOs, module digests — as **verified prior context**. Never ask
the user to re-supply project context the graph already holds.

### Token Optimization & Knowledge-Graph RAG Contract (STRICT)

1. **Selective Context Packing**: Call **`kg_context_pack(task=…, module=…)`** before working on any screen/story/test. It returns the minimum-correct, token-budgeted context (USE/VERIFY/PROBE locators, flows, facts). **Never load whole bloated module nodes or raw files when a context pack will do.**
2. **MCP Fast Tool Collapsing (Consume ~70% Less Tokens)**:
   - When driving the live browser via Selenium MCP, **ALWAYS prefer fast tools**: `login_sso`, `fill_form`, `click_sequence`, `select_radix_option`, `get_texts`, `assert_text`, `wait_for_idle`.
   - Collapsing multi-step browser calls into a single fast tool call dramatically reduces tool-call roundtrips and cuts token consumption.
3. **DOM Payload Trimming**:
   - Never dump raw 50KB+ HTML trees into context window. Use `snapshot_dom` with selective interactive element filters or targeted `get_element_state` / `find_element` calls.
4. **Anti-Hallucination & DOM-Verification Policy (STRICT)**:
   - **Zero Selector Fabrication**: Code-authoring agents (`PomAuthor`, `FeatureAuthor`, `StepDefAuthor`, `WebComponentAuthor`) MUST fetch locators from `kg_context_pack` or existing POMs. They MUST NOT fabricate or guess XPaths under any circumstance.
   - **DOM Snapshot Mandatory for Unverified Elements**: Before writing any new POM locator or performing browser interactions on unverified UI components, run `snapshot_dom(interactive_only=true)` or query `kg_context_pack` to verify actual tags, classes, and text.
   - **Strict Ground-Truth Execution**: Only DOM-verified agents (`Explorer`, `Healer`) write live-validated selectors to the Knowledge Graph (`kg_upsert`). Step pass/fail verdicts MUST be backed by live `assert_text`, `get_texts`, or `get_element_state` evidence.
5. **Exact Framework Code Standards**:
   - Generated test code MUST strictly match existing workspace code:
     - 4-layer separation (Feature -> Step Def -> POM -> Web Component).
     - Pytest-BDD `scenarios(...)` binding + `bdd_driver` fixture.
     - POM locators declared strictly as `[(By.XPATH, "...")]` lists or `lambda` getters.
     - Cross-step data in module-level `_ctx = {}` dict or web_component getters/setters.
     - Assertions strictly in `@then` steps with Allure attachments (`allure.attach`).
- At every natural end point (story done, heal concluded, session ending), checkpoint with
  **`kg_session_save`** (summary, decisions, todos, next_steps, worked_on, completed_todos)
  so the next chat can `/resume`.
- If the MCP server is not running, say so once and continue — never fabricate graph content.

## Multi-agent team — who does what (`.agents/agents/`)

This workspace runs a coordinator/worker agent team. Enter it via thin prompt launchers:
`/story-to-tests` (full pipeline), `/heal`, `/explore`, `/resume`, `/checkpoint`.

| Agent                      | Single job                                                           | KG access                                |
| -------------------------- | -------------------------------------------------------------------- | ---------------------------------------- |
| `TestAuthoringCoordinator` | Plan + delegate; never writes code or drives the browser             | bootstrap/pack/session (read+checkpoint) |
| `Explorer`                 | Live graph-first browser walk; verify locators/flows on the real DOM | **write** (upsert/knowledge/failure)     |
| `PomAuthor`                | Page objects / locators from KG-verified selectors only              | read-only                                |
| `WebComponentAuthor`       | Shared driver-first interaction functions, zero assertions           | read-only                                |
| `FeatureAuthor`            | Gherkin features after gap + test-condition analysis                 | read-only                                |
| `StepDefAuthor`            | Step defs — the ONLY home of assertions                              | read-only                                |
| `Healer`                   | Classify DRIFT/TIMING/FLOW/BUG; fix at source; budget-capped         | **write** (drift/healing)                |
| `DefectAuthor`             | Product-bug `_defect.docx` + defect log + matrix                     | read (+ defect knowledge)                |

Anti-hallucination contract: **only DOM-verified agents (Explorer, Healer) write locator
knowledge to the graph**; code-authoring agents consume it and report gaps instead of
inventing selectors. All agents inherit every rule in this file.

## Daily-activity ledger — log every agent action (always on)

This workspace has a personal work tracker (`ToDoSystem/`). **Every agent session must log
each completed unit of work** (test run, test fix, heal, locator fix, authoring, analysis)
into the ledger immediately after it completes, via:

```powershell
.\.venv\Scripts\python.exe -m ToDoSystem.todo copilot <test_run|test_fix|heal|locator_fix|authoring|analysis|other> "<one-line description>" --module <Module> --story <AMCC-XXXX> --test-id <AMCC-TC-XXXX> --status <PASSED|FAILED|FIXED|DONE> --details "<short detail>"
```

- Log once per **completed unit of work**, not per tool call. Omit unknown flags.
- Individual pytest results are auto-captured by a conftest hook — the `copilot test_run`
  entry summarizes the run intent/outcome only.
- Logging failures must never block the main task.
- The full agent behaviour (daily reports, task commands) lives in
  `.agents/chatmodes/todo-agent.chatmode.md`.

## TODO-Based Work Decomposition (MANDATORY for ALL tasks)

For ANY task given to you by the user, **you MUST first read the task, analyze it, decompose it into trackable TODOs, and complete the TODOs systematically one by one**. This keeps work organized, prevents skipping steps, and ensures completeness.

### When to create TODOs

| Task type                                   | Create TODOs?            |
| ------------------------------------------- | ------------------------ |
| Any user-assigned task / query              | **YES — mandatory**      |
| Single-file edit                            | **YES — mandatory**      |
| Multi-file change (2-3 files)               | **YES — mandatory**      |
| Story automation (feature + POM + step def) | **YES — mandatory**      |
| Healing multiple test cases                 | **YES — mandatory**      |
| Any task with dependencies                  | **YES — mandatory**      |

### TODO decomposition pattern

```sql
-- Break work into discrete, executable units
INSERT INTO todos (id, title, description, status) VALUES
  ('<story>-explore', 'Exploring <story> screens',
   'Explorer walks <screens>. Capture locators for: <elements>.', 'pending'),
  ('<story>-pom', 'Creating POM for <story>',
   'PomAuthor creates/extends <Module>Page.py with verified locators.', 'pending'),
  ('<story>-feature', 'Writing feature file for <story>',
   'FeatureAuthor writes <module>-<story>.feature with all AC scenarios.', 'pending'),
  ('<story>-webcomp', 'Creating web_components for <story>',
   'WebComponentAuthor adds shared interactions.', 'pending'),
  ('<story>-stepdef', 'Wiring step definitions for <story>',
   'StepDefAuthor creates test_<module>-<story>.py with all assertions.', 'pending'),
  ('<story>-verify', 'Verifying <story> tests',
   'Run pytest -n auto, heal failures if any.', 'pending');

-- Define dependencies (what must complete before what)
INSERT INTO todo_deps (todo_id, depends_on) VALUES
  ('<story>-pom', '<story>-explore'),
  ('<story>-stepdef', '<story>-pom'),
  ('<story>-stepdef', '<story>-feature'),
  ('<story>-verify', '<story>-stepdef');
```

### Execution workflow

1. **Query ready TODOs** — find TODOs with no pending dependencies:

   ```sql
   SELECT t.* FROM todos t
   WHERE t.status = 'pending'
   AND NOT EXISTS (
       SELECT 1 FROM todo_deps td
       JOIN todos dep ON td.depends_on = dep.id
       WHERE td.todo_id = t.id AND dep.status != 'done'
   );
   ```

2. **Mark in_progress** before starting:

   ```sql
   UPDATE todos SET status = 'in_progress' WHERE id = '<todo-id>'
   ```

3. **Delegate to the right agent** (or do it yourself if appropriate)

4. **Mark done** when complete:

   ```sql
   UPDATE todos SET status = 'done' WHERE id = '<todo-id>'
   ```

5. **Repeat** until all TODOs are done

### TODO rules

- **Descriptive IDs**: Use `<story>-<phase>` pattern, not `t1`, `t2`
- **Gerund titles**: "Creating POM for AMCC-8438", not "POM"
- **Complete descriptions**: Include enough detail to execute without referring back
- **Status discipline**: Always update status before/after work
- **One TODO at a time**: Complete one before starting the next (unless parallel-safe)

---

## Skills — load the repo cookbook before generating code

The `.agents/skills/` folder is the **ground-truth, copy-pasteable** reference for how this exact
repository works. Read the relevant skill **before** writing code; the prompts will tell you which.

| Skill                                                                                  | Read it before…                                                                                                                                                  |
| -------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [skills/framework_architecture/SKILL.md](skills/framework_architecture/SKILL.md) | doing anything — the canonical stack + folder map (**always start here**)                                                                                        |
| [skills/step_definitions/SKILL.md](skills/step_definitions/SKILL.md)             | writing/editing a **step definition** (test def)                                                                                                                 |
| [skills/custompyautoweb_api/SKILL.md](skills/custompyautoweb_api/SKILL.md)       | calling a Selenium helper from a step or web_component                                                                                                           |
| [skills/page_objects/SKILL.md](skills/page_objects/SKILL.md)                     | writing/editing a **page object / locator**                                                                                                                      |
| [skills/web_components/SKILL.md](skills/web_components/SKILL.md)                 | writing/editing a **web_component** (shared interaction)                                                                                                         |
| [skills/assertions_and_allure/SKILL.md](skills/assertions_and_allure/SKILL.md)   | adding an **assertion or Allure evidence**                                                                                                                       |
| [skills/efficiency_and_hygiene/SKILL.md](skills/efficiency_and_hygiene/SKILL.md)       | **any run/execution session** — zero temp files, blocking foreground pytest runs (never sleep-poll logs), anti-loop budgets, continuous story-linked KG capture  |
| [skills/coverage_traceability/SKILL.md](skills/coverage_traceability/SKILL.md)         | any **coverage-check / gap analysis / "how much is covered?"** question — coverage-report.md, the persistent traceability-matrix.csv, test-condition design pass |

---

## 0. Prime Directives — Read First, Apply Always

If you read nothing else, obey these. Every rule below in this file expands on one of them.

1. **One layer per concern.** Code lives in exactly one of the 4 layers — Feature, Step Def, Page Object, web_component. Never blur the boundaries.
2. **Assertions live ONLY in step def `@then` steps.** Never inside page objects or web_component functions.
3. **Locators live ONLY in page objects.** They are declared as `locatorXxx = [(By.XPATH, "...")]` lists. Step defs and web_component reference those page-object locators — never a raw inline selector.
4. **No `time.sleep()` — ever.** Use the framework waits: `web.wait_element_visibility(...)` to gate, `web.wait(n)` for a short bounded settle.
5. **No raw `bdd_driver.find_element(...)` in step defs.** Go through `CustomPyAutoWeb(bdd_driver)` + a page-object locator, or a web_component function.
6. **No hardcoded test data.** Values come from `parsers.parse` step parameters, `Examples` tables, `testData/*.json`, or the `_ctx` dict / web*component `get*`/`set\_` globals.
7. **Reuse rule: any UI interaction used in 2+ places goes to web_component** — parameterized, single responsibility, zero assertions.
8. **Match existing conventions exactly.** Never invent a name, style, import order, or pattern that is not already present in the workspace. When unsure, read the existing files first (§11).
9. **Evidence on every action and check.** Every `@when` and `@then` step attaches Allure evidence with a specific, descriptive name.
10. **Stop and ask — never guess or fabricate** — when a referenced file, page method, locator, or element state cannot be determined from the available files / HTML (§12).
11. **Verification over memory.** For anything version-, DOM-, or state-dependent (a locator, a label, a file's current content, a run result), check the artifact or live source first (read the file, `kg_recall`, `snapshot_dom`, Allure results) — never answer from general knowledge, and never present an unverified claim as fact.
12. **Surgical edits only.** Modify exactly what the task targets — no unsolicited refactoring, renaming, cleanup, or scope expansion on untouched code. One hypothesis per edit; verify before the next.
13. **Always analyze tasks first and decompose them into trackable TODOs.** For ANY task or request, first read and analyze the requirements, establish a clear implementation plan, create a `task.md` or database TODO list, and systematically complete the TODOs one by one.

**Precedence:** when this file conflicts with a generated suggestion, _this file wins_. When it conflicts with an existing workspace convention on **style only** (naming, formatting, docstring shape), the **existing workspace convention wins** — see §1.

---

## 1. Prompt Library & Routing

This workspace runs a **layered prompt system**. This file (`AGENTS.md`) holds the shared, always-applied rules — the constitution. Each task prompt in `.agents/prompts/` builds on these rules and owns one stage of the automation pipeline. Pick the specialist(s) that match the request.

| Prompt file                                                  | Owns                                                                                                        | Primary inputs                                                | Produces                                                                            |
| ------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| `skills/01_xpath_pom/SKILL.md` (POM / XPath generation)            | Writing accurate, stable locators and the POM class for a page.                                             | Reference POM file, target page HTML + screenshot             | A POM class — one best XPath per **confirmed** state, dynamic locators as `lambda`s |
| `skills/02_feature_file/SKILL.md` (Feature file creation)          | Generating pytest-bdd Gherkin `.feature` files from CSV test data, for **missing** stories only.            | Existing `.feature` files, CSV test data                      | New `.feature` files matching the existing style guide (after a gap report)         |
| `skills/03_step_definitions/SKILL.md` (Step definition generation) | Wiring a feature file to code — step defs + the web_component functions they call.                          | Target feature file, existing step defs / POM / web_component | web_component functions first, then the step def file                               |
| `skills/web_component_prompt_prompt/SKILL.md` (web_component generation)         | Designing **optimized, deduplicated, reusable** shared UI-interaction functions in the web_component layer. | Existing web_component file, the interactions to support      | New / extended web_component functions matching the existing file's style           |

### Pipeline order for a brand-new story (end to end)

1. **`skills/02_feature_file/SKILL.md`** → build the `.feature` file from CSV (gap analysis first).
2. **`skills/01_xpath_pom/SKILL.md`** → build the POM locators from HTML + screenshot.
3. **`skills/web_component_prompt_prompt/SKILL.md`** → build / extend the reusable interaction functions.
4. **`skills/03_step_definitions/SKILL.md`** → wire it together (calls web_component, holds the assertions, attaches Allure).

### Request-type routing table (orchestrator view — pick the sequence, not just one prompt)

| The user asks for…                                | Route (prompts + skills, in order)                                                                                          |
| ------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| Fix failing / red tests                           | `skills/heal_tests_prompt/SKILL.md` (+ heal-tests, efficiency-and-hygiene skills)                                                        |
| New story end-to-end (fetch → execute → author)   | `skills/fetch_userstory_prompt/SKILL.md` → `skills/qmetry_testcases_prompt/SKILL.md` → `skills/mcp_execute_and_author_prompt/SKILL.md` (+ mcp-execution-protocol, authoring-bdd-from-mcp skills) |
| "How much of story X is covered?"                 | coverage-check in `skills/mcp_execute_and_author_prompt/SKILL.md` (+ coverage-traceability skill) — no browser needed                              |
| Feature files from CSV                            | `02_feature_file` (gap report + condition pass + review gate) → then the 4-step pipeline above                              |
| Locator work only                                 | `01_xpath_pom` (+ 03_page-objects skill)                                                                                    |
| Repeatedly-flaky test (same case healed 2+ times) | heal-tests §flaky re-diagnosis — stop patching waits; re-walk the flow, check data isolation/xdist collisions               |
| Globalization / translation audit                 | `skills/mcp_globalization_prompt/SKILL.md` (+ globalization-testing skill)                                                                         |
| Document a product bug                            | defect-documentation skill + `build_defect_docx.py` (append `TestcaseExecution/defect-log.csv`)                             |

### Handoff contract between pipeline stages

When one prompt's output feeds the next stage (feature → POM → web_component → step defs), the
handoff always states: **story/module**, **previous output path(s)** (the exact files just
written), **what the next stage must consume from them**, and **open questions carried forward**.
Each stage ends with a short status line: files created/modified + issues found. Never start a
stage without the previous stage's real output paths — no stage regenerates or guesses a
previous stage's artifacts.

### How the two interaction-writing prompts relate

`skills/03_step_definitions/SKILL.md` creates web*component functions for the \_immediate* need of the feature it is wiring. `skills/web_component_prompt_prompt/SKILL.md` is the **specialist** for the shared layer itself — invoke it when web*component work is non-trivial, when refactoring/consolidating the file, or when the step def workflow flags meaningful shared-interaction work. When both could apply, **`skills/web_component_prompt_prompt/SKILL.md` is authoritative for the design of web_component functions**, and `skills/03_step_definitions/SKILL.md` simply \_calls* the functions it defines.

### Precedence / conflict resolution (apply in order)

1. **Safety and architecture rules in this file are absolute** — never overridden by any prompt, suggestion, or request.
2. **For style** (naming, formatting, import order, docstring format, section dividers), the **existing workspace files are the source of truth** — match them, even over generic examples shown inside a prompt file.
3. If two existing files disagree on style, **prefer the most recent or most common pattern** and add a one-line comment noting the choice.
4. If a prompt file disagrees with this file, **this file wins**.
5. **Never invent a convention.** If no precedent exists and it cannot be inferred unambiguously, stop and ask (§12).

---

## 2. Stack

| Tool              | Role                                                                                         |
| ----------------- | -------------------------------------------------------------------------------------------- |
| Python            | Primary language                                                                             |
| Selenium          | UI browser driver                                                                            |
| pytest-bdd        | BDD runner (Gherkin `.feature` + `tests/step_defs/`, bound with `scenarios(...)`)            |
| `bdd_driver`      | pytest fixture that hands each step the live WebDriver (no `context` object)                 |
| `CustomPyAutoWeb` | Selenium toolbox in `utilities/genericUtils.py`; the gateway to every browser action         |
| Page Object Model | All page interactions abstracted into page classes; locators are `[(By.XPATH, "...")]` lists |
| Allure            | Test reporting (attach on every @when and @then)                                             |

---

## 3. The 4-Layer Architecture — CARDINAL RULE

Code lives in exactly one layer. Never blur the boundaries.

| Layer         | Folder                                                    | Contains                                             | NEVER contains                                          |
| ------------- | --------------------------------------------------------- | ---------------------------------------------------- | ------------------------------------------------------- |
| Feature       | `tests/features/Apps/<Module>/`                           | Gherkin steps in plain business language             | Implementation detail, locators, data                   |
| Step Defs     | `tests/step_defs/apps/<Module>/` (files prefixed `test_`) | Step orchestration + ALL assertions                  | Raw Selenium, locators, `time.sleep()`, hardcoded data  |
| Page Objects  | `object_repository/pages/Apps/<Module>/`                  | Locators (`[(By.XPATH,"...")]` lists) + page methods | Assertions, cross-page logic, raw driver calls in steps |
| web_component | `business_components/web_components/<module>.py`          | Reusable, parameterized UI interaction helpers       | Assertions, hardcoded locators, hardcoded test data     |

If you are unsure which layer code belongs in, re-read the table above.
The answer is always in the table.

### Layer Decision Flow — quick reference

Ask these in order for any line of code you are about to write:

1. **Is it an assertion / verification?** → Step def `@then` only. (Never anywhere else.)
2. **Is it a locator (XPath/CSS)?** → Page object only, as a `[(By.XPATH, "...")]` list. (Reference it via `PAGE.locatorX`; never write a raw selector in a step def or web_component.)
3. **Is it a Selenium call (`click`, type, wait, read)?** Route it through `CustomPyAutoWeb` + a page-object locator.
   - Used by **one** page's flow → a **page object** method or directly in the step via `web = CustomPyAutoWeb(bdd_driver)`.
   - Reused across 2+ steps/stories → put it in **web_component** (`driver` first; locators come from the POM).
4. **Is it business-readable orchestration** (call a web*component/POM method, store/read `_ctx` or a `get*`/`set\_` global, then assert)? → **Step def**.
5. **Is it plain product language describing intent?** → **Feature** file.

When two answers seem to apply, the **most restrictive** layer wins: assertion → step def; locator → page object.

---

## 4. web_component Contract — Read This Twice

> The full authoring workflow for this layer lives in **`skills/web_component_prompt_prompt/SKILL.md`** (`.agents/skills/web_component_prompt_prompt/SKILL.md`) and the cookbook in **`skills/web_components/SKILL.md`**. This section is the contract every function must satisfy regardless of how it is written.

The web_component file is the **single home for any UI interaction pattern
reused across more than one step, page, or story** within a module.

### A function MUST go into web_component if ANY of these is true:

- The same interaction pattern is used in 2 or more steps, pages, or stories
  (even if the parameter values differ each time)
- It wraps a multi-step Selenium sequence into one callable unit
  (e.g. type → wait for dropdown → click suggestion)
- It is a generic getter or setter of UI state that multiple steps read or write
- It isolates fragile or wait-heavy logic so step defs and page objects stay clean

### Categories that always go in web_component:

| Category               | Examples                                                                 |
| ---------------------- | ------------------------------------------------------------------------ |
| Search form fill       | Type into search bar, apply filters, clear search, trigger submit        |
| General form fill      | Fill create/edit/config forms, set field values, handle required fields  |
| Button group selection | Select one button from a toggle/filter group, verify only one is active  |
| Typeahead/Autocomplete | Type partial value → wait for list → click matching suggestion           |
| Getters (read state)   | Get element text, count items, get selected option, get active tab label |
| Setters (write state)  | Set dropdown value, set toggle state, set checkbox, set input field      |
| Generic Selenium wraps | Waits, scrolls, alert handling, drag-and-drop, hover, iframe switch      |

### Every web_component function MUST:

- Accept `driver` (or `self`) as the first parameter — match the existing file's style
- Accept **locators as parameters** — never hardcode a locator inside the function
- Accept **values/labels as parameters** — never hardcode test data inside the function
- Expose a `timeout` parameter with a sensible default matching the existing file's style
- Have ONE responsibility: one action OR one read — never both in the same function
- Use explicit waits — `web.wait_element_visibility(...)` / `web.wait(n)` (or a `WebDriverWait` if the existing file uses one) — NEVER `time.sleep()`
- Contain ZERO assertions — assertions always live in step defs only
- Match the docstring format, naming convention, return type, and exception handling
  style already present in the existing web_component file

### A function STAYS in the step def (does NOT go to web_component) only if:

- It is single-use and entirely specific to one scenario
- It contains assertion logic (assertions must never leave step defs)
- It combines action + assertion (split them: action → web_component, assert → step def)

---

## 5. Assertion Rules — Step Defs Only

All assertions live exclusively in step def `@then` steps. The repo asserts in two complementary
parts (see `skills/assertions_and_allure/SKILL.md`):

1. **A `CustomPyAutoWeb` verify method that raises on failure** does the actual check — e.g.
   `web.element_should_be_visible(locator=PAGE.locatorX)`, `web.verify_current_url(...)`,
   `web.verify_toast_alert(...)`. **Prefer this** whenever a verify method fits.
2. **The step's `try/except` re-raises as a standard `AssertionError`** so every failure reads
   the same way:
   ```python
   except Exception as e:
       raise AssertionError(
           f"Step FAILED: '<exact step text>' - "
           f"Error occurred in step [<func_name>]. Details: {e}"
       ) from e
   ```

Use a **bare `assert`** only when no verify method fits (e.g. comparing two values you
computed). It still sits inside the same `try/except` and always carries a multi-line message.

### Visibility / state check (preferred — verify method):

```python
web = CustomPyAutoWeb(bdd_driver)
web.wait_element_visibility(locatorList=PAGE.locatorX, wait_time=60)
web.element_should_be_visible(locator=PAGE.locatorX)   # raises on failure
```

### Text contains check (bare-assert fallback):

```python
actual = web.get_element_attribute(locatorList=PAGE.locatorField, attribute_name="value")
assert expected in actual, (
    f"Text validation failed.\n"
    f"Expected to contain : '{expected}'\n"
    f"Actual found        : '{actual}'"
)
```

### Exact text match:

```python
assert actual == expected, (
    f"Exact match failed.\n"
    f"Expected : '{expected}'\n"
    f"Actual   : '{actual}'"
)
```

### Number / count check:

```python
assert actual_count == int(expected_count), (
    f"Count validation failed.\n"
    f"Expected : {expected_count}\n"
    f"Actual   : {actual_count}"
)
```

### Visibility / state check:

```python
assert is_visible, (
    f"Visibility check failed.\n"
    f"Expected '{element_name}' to be visible but it was not found."
)
```

### Negative check:

```python
assert expected not in actual, (
    f"Negative validation failed.\n"
    f"Expected '{expected}' NOT to be present.\n"
    f"Actual text found: '{actual}'"
)
```

### Active button / tab state check:

```python
actual_label = get_active_button_label(bdd_driver, group_locator)
assert actual_label == expected_label, (
    f"Active button validation failed.\n"
    f"Expected active : '{expected_label}'\n"
    f"Actual active   : '{actual_label}'"
)
```

Every assertion MUST sit inside the step's `try/except` wrapper and carry a multi-line failure
message. Never use a bare `assert` with no message.

---

## 6. Allure Reporting Rules — Mandatory on Every Step

### @given steps — attach initial state:

```python
allure.attach(
    f"Precondition established.\nCurrent URL: {bdd_driver.current_url}",
    name="Precondition — [Short description of setup state]",
    attachment_type=allure.attachment_type.TEXT,
)
```

### @when steps — attach action performed:

```python
allure.attach(
    (
        f"Action      : [describe what was done]\n"
        f"Input value : '{parameter_value}'\n"
        f"On page     : {bdd_driver.current_url}"
    ),
    name="Action — [Short description of what was done]",
    attachment_type=allure.attachment_type.TEXT,
)
```

### @then steps — attach after every assertion:

```python
allure.attach(
    (
        f"Validation type : [text contains / exact match / count / visible]\n"
        f"Element checked : [element name or description]\n"
        f"Expected        : '{expected_value}'\n"
        f"Actual          : '{actual_value}'\n"
        f"Result          : PASSED"
    ),
    name="Verification — [Short description of what was verified]",
    attachment_type=allure.attachment_type.TEXT,
)
```

### Allure rules:

- Every `@when` step has an `allure.attach` describing the action and input value
- Every `@then` step has an `allure.attach` after the assertion with all 5 fields above
- `name=` must describe WHAT was validated and WHERE — never generic names like
  "Result", "Check", "Step", or "Verification"
- `attachment_type` must match the content: TEXT for strings, JSON for objects,
  PNG for screenshots

---

## 7. XPath / Locator Rules — Page Objects Only

> The full locator-authoring workflow lives in **`skills/01_xpath_pom/SKILL.md`** (`.agents/skills/01_xpath_pom/SKILL.md`) and the cookbook in **`skills/page_objects/SKILL.md`**. This section is the contract every locator must satisfy.

Locators live exclusively in page object files. They never appear in step defs
or web_component functions (locators are passed as parameters to web_component).

### XPath priority order — follow strictly:

| Level | Anchor                         | Use when                                              |
| ----- | ------------------------------ | ----------------------------------------------------- |
| 1     | `text()` / `normalize-space()` | Visible text exists on the element — always try first |
| 2     | `@aria-label`, `@role` + text  | Element has no visible text (icon buttons, inputs)    |
| 3     | `@data-testid`, `@data-cy`     | Levels 1–2 are not unique                             |
| 4     | `@id`, `@name`                 | Levels 1–3 not available                              |
| 5     | Semantic `@class` + text       | Last resort — only unique, non-generated class names  |
| 6     | Positional / axis              | Absolute last resort — flag it with a comment         |

### Class-based XPaths — strict rules:

- NEVER use auto-generated or hashed classes: `css-1x2y3z`, `sc-abc123`
- NEVER use utility/layout classes: `flex`, `mt-4`, `d-flex`, `col-md-6`, `text-sm`
- NEVER use a class that appears 5 or more times in the DOM
- ONLY use semantically named, component-unique class tokens
- ALWAYS combine with tag + text: `//button[contains(@class,'submit-btn') and normalize-space()='Submit']`

### Dynamic XPaths — always parameterized for runtime variables:

```python
# Correct — runtime-variable text is a parameter; lambda returns the locator LIST
locatorCMONavTabByName = lambda tab_name: [
    (By.XPATH, f"//nav//button[normalize-space(text())='{tab_name}']")
]
```

### Forbidden XPath patterns:

- Absolute paths: `/html/body/div/...`
- Floating index-only: `(//div)[3]`
- State XPaths without HTML evidence (never assume a state)
- More than one XPath per state per element

### Allowed states — write only what exists in the HTML:

- `DEFAULT` — element with no special state attribute
- `SELECTED` — confirmed by `aria-selected="true"`, `class="active"`, etc.
- `NOT_SELECTED` — confirmed by `aria-selected="false"`, non-active class, etc.
- `DISABLED` — confirmed by `disabled` attribute or `aria-disabled="true"`

---

## 8. Import Order — Match the Existing Files Exactly

```python
# 1. Standard library
import os
from datetime import date, time

# 2. pytest / pytest-bdd
import pytest
from pytest_bdd import given, when, then, scenarios, parsers

# 3. Selenium
from selenium.webdriver.common.by import By

# 4. Allure
import allure

# 5. web_component layer (business_components)
from business_components.web_components.login import login_to_application_sso
from business_components.web_components.cmo import *

# 6. Page Object classes (object_repository)
from object_repository.pages.Apps.CMO.CmoLandingPage import CMOLandingPage

# 7. conftest fixture + utilities
from tests.step_defs.conftest import bdd_driver
from utilities.genericUtils import CustomPyAutoWeb
from utilities.helper import load_test_data_from_json
```

Deviation from the import order found in existing files is not allowed.
If two existing files disagree on order, match the most recent file and note it.

---

## 9. Forbidden Everywhere — Zero Exceptions

| Forbidden                               | Reason                                                                                |
| --------------------------------------- | ------------------------------------------------------------------------------------- |
| `time.sleep()`                          | Causes flaky tests — use `web.wait_element_visibility(...)` / `web.wait(n)`           |
| `bdd_driver.find_element(...)` in steps | Breaks POM architecture — use `CustomPyAutoWeb` + a page-object locator               |
| Raw XPaths/CSS in step defs             | Locators belong in page objects only                                                  |
| Hardcoded test data                     | Values come from step params, `Examples`, `testData/*.json`, `_ctx`, or `get_`/`set_` |
| Silent `try/except`                     | Swallowed failures hide real bugs — log + re-raise                                    |
| Duplicate step definitions              | One definition per step text across all files                                         |
| Duplicate web_component functions       | One function per interaction pattern                                                  |
| Unused imports                          | Clutters files — import only what is used                                             |
| Commented-out dead code                 | Noise — remove it entirely                                                            |
| Assertions in web_component             | Assertions live in step defs exclusively                                              |
| Locators hardcoded in web_component     | Locators are passed as parameters                                                     |
| Invented conventions                    | Match only what already exists in the workspace                                       |
| Vague `allure.attach` names             | Every name must say what was validated and where                                      |

---

## 10. Pylint Compliance — CI Gate (Mandatory)

All generated Python code **must pass the repository's Pylint check** (`.pylintrc`). The CI
workflow (`.agents/workflows/ci.yml`) runs `pylint` on every changed `.py` file — a failure
blocks the PR.

### Rules enforced by Pylint (match these exactly)

| Rule                        | Limit / Requirement                                                                                         |
| --------------------------- | ----------------------------------------------------------------------------------------------------------- |
| **Line length**             | Max **280** characters per line                                                                             |
| **Indentation**             | **4 spaces** — never tabs                                                                                   |
| **Function arguments**      | Max **10** arguments total, max **8** positional                                                            |
| **Class attributes**        | Max **15** instance attributes                                                                              |
| **Boolean expressions**     | Max **10** boolean expressions in a single `if` statement                                                   |
| **Exception handling**      | **Never catch bare `Exception` or `BaseException`** — always catch a specific exception type (see below)   |
| **Unused imports**          | Remove any import that is not used in the file                                                              |
| **Undefined variables**     | Every variable must be defined before use                                                                   |

### Exception handling — the #1 CI failure cause

The `.pylintrc` sets `overgeneral-exceptions=builtins.Exception, builtins.BaseException`. This
means **`except Exception`** triggers a Pylint error and fails CI.

**WRONG — fails Pylint:**
```python
try:
    web.click_element(locator)
except Exception as e:          # ❌ overgeneral-exceptions (E0012)
    raise AssertionError(...) from e
```

**CORRECT — passes Pylint:**
```python
from selenium.common.exceptions import (
    TimeoutException,
    NoSuchElementException,
    ElementClickInterceptedException,
    StaleElementReferenceException,
)

try:
    web.click_element(locator)
except (TimeoutException, NoSuchElementException, ElementClickInterceptedException,
        StaleElementReferenceException) as e:   # ✅ specific exceptions
    raise AssertionError(
        f"Step FAILED: '<step text>' - Error occurred in step [<func_name>]. Details: {e}"
    ) from e
```

### Common Selenium exceptions to catch (use these instead of `Exception`)

| Exception                         | When it occurs                                           |
| --------------------------------- | -------------------------------------------------------- |
| `TimeoutException`                | Element not found within wait time                       |
| `NoSuchElementException`          | Element does not exist in DOM                            |
| `ElementClickInterceptedException`| Another element covers the target                        |
| `StaleElementReferenceException`  | Element reference is stale (DOM changed)                 |
| `ElementNotInteractableException` | Element exists but cannot be interacted with             |
| `InvalidSelectorException`        | XPath/CSS syntax error                                   |
| `WebDriverException`              | Generic Selenium error (use as last resort in tuple)     |

**Import pattern for step defs:**
```python
from selenium.common.exceptions import (
    TimeoutException,
    NoSuchElementException,
    ElementClickInterceptedException,
    StaleElementReferenceException,
    ElementNotInteractableException,
)
```

### What Pylint does NOT enforce (disabled in `.pylintrc`)

These are **allowed** and will not fail CI:
- Missing docstrings (module, class, function) — C0114, C0115, C0116 disabled
- Wildcard imports (`from module import *`) — W0401 disabled
- Global statements — W0602, W0603 disabled
- Non-snake_case names — C0103 disabled
- TODO comments — W0511 disabled

### Pre-commit self-check

Before finalizing any Python file, mentally verify:
- [ ] No `except Exception:` or `except BaseException:` — use specific exception types
- [ ] No unused imports
- [ ] No lines over 280 characters
- [ ] No undefined variables
- [ ] Functions have ≤10 args (≤8 positional)

---

## 11. How to Discover Conventions — Do This, Never Guess

Before writing any code for a module, read these files in full:

1. One representative existing step def file for this module (`tests/step_defs/apps/<Module>/test_*.py`)
2. The module's page object class(es) (`object_repository/pages/Apps/<Module>/*.py`)
3. The module's web_component file (`business_components/web_components/<module>.py`)
4. `tests/step_defs/conftest.py` (the `bdd_driver` fixture + hooks)

From those files, extract and match exactly:

- Decorator import source (`from pytest_bdd import given, when, then, scenarios, parsers`)
- Feature binding (`scenarios(os.path.join(os.path.dirname(__file__), "skills/skills/skills/features/..."))`)
- Driver handoff (`bdd_driver` fixture) and the `web = CustomPyAutoWeb(bdd_driver)` pattern
- Cross-step state keys (`_ctx["..."]`, web*component `get*`/`set\_` globals)
- Assertion style (`CustomPyAutoWeb` verify method + `try/except` → `raise AssertionError("Step FAILED: ...")`)
- Wait strategy (`web.wait_element_visibility(...)`, `web.wait(n)` — never `WebDriverWait` inline, never `time.sleep`)
- Allure attach pattern (body as first positional arg, then `name=`, `attachment_type=`)
- web_component function signature style (`driver` first, `timeout` default, locators as parameters)
- Naming convention for step def files (`test_<module>-<JIRA>.py`), functions, and local variables
- Section comment divider style

If two existing files disagree, prefer the most recent or most common pattern
and add a one-line comment noting the choice.

---

## 12. When to Stop and Ask — Never Guess or Fabricate

Pause and ask in chat when ANY of these is true:

- A referenced file (feature, page object, web_component) is missing or unreadable
- A step needs a page method that does not exist in the POM and cannot be
  unambiguously defined from the available HTML or existing patterns
- A web_component function is needed but cannot be clearly defined from context
- An element has no unique anchor attribute or text in the HTML
- A state XPath is requested but there is no HTML evidence the state exists
- The only available locator is an auto-generated or hashed class name
  → Flag it: recommend adding `@data-testid` to the HTML instead
- An edit would change what a test actually verifies
- A multipurpose function's correct locator parameter is ambiguous
  → List the candidate locators and ask which one to use

Guessing or fabricating in any of these situations produces code that
silently fails at runtime. Stop and ask every time.

---

## 13. Step Definition File — Required Output Structure

Every generated step def file must follow this structure in order:

```
1. Module docstring          (if present in existing files)
2. Imports                   (in the exact order defined above)
3. Section divider comment   (if present in existing files)
4. Step functions            (ordered by scenario appearance in the feature file)
```

Reused steps follow the repo's per-module convention: because **pytest-bdd resolves steps per
test module** and the hyphenated sibling file (`test_cmo-AMCC-8490.py`) cannot be imported, a
shared step is **redefined verbatim** in the file that needs it, tagged with a provenance
comment, and its body delegates to the **same web_component function** so the real logic still
lives in exactly one place:

```python
# REUSED FROM: test_cmo-AMCC-8490.py — step "<step text>".
# Redefined here because pytest-bdd resolves steps per module; the hyphenated
# sibling file cannot be imported. Body delegates to the shared web_component function.
```

---

## 14. Self Cross-Check — Run Before Every Final Output

Before writing any file to disk, verify:

**Coverage**

- [ ] Every Given/When/Then/And step in the feature has a matching definition
- [ ] Every Scenario Outline parameter matches its Examples column header exactly
- [ ] Background steps handled and not duplicated per scenario

**Code correctness**

- [ ] Every POM method called exists in the page object class
- [ ] Every web_component function called exists (existing or newly written)
- [ ] Every `_ctx` key / `get_`/`set_` global written in one step is read with the identical name elsewhere
- [ ] All imports resolve to real files in the project
- [ ] No step decorator text has a typo vs the feature file step text

**Architecture**

- [ ] Zero `bdd_driver.find_element` calls in step defs
- [ ] Zero raw locators or XPaths in step defs
- [ ] Zero hardcoded test data anywhere
- [ ] Zero `time.sleep()` calls
- [ ] Zero assertions inside web_component functions
- [ ] All reusable interaction patterns are in web_component, not inline

**Allure**

- [ ] Every `@when` step has `allure.attach` describing action + input
- [ ] Every `@then` step has `allure.attach` after the check with all 5 fields
- [ ] No `allure.attach` name is vague or generic

**Conventions**

- [ ] File name matches naming convention (`test_<module>-<JIRA>.py`)
- [ ] All function names match naming convention
- [ ] Import order matches existing files exactly
- [ ] Assertion style matches existing step def files (`CustomPyAutoWeb` verify + `try/except`)
- [ ] web_component function style matches existing functions

**Duplication**

- [ ] No step definition duplicates an existing step in any other file
- [ ] No web_component function duplicates an existing function
