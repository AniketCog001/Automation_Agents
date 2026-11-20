---
description: The rules for executing QMetry test cases through the Selenium MCP.
name: mcp_execution_protocol
---

# Skill — MCP Execution Protocol (Selenium MCP · Cold Compass)

> How to **execute test cases through the `selenium` MCP** in this repo: sequential,
> one-by-one, context-first, with a screenshot per step — and how that maps onto the
> framework's 4-layer pytest-bdd architecture and Allure reporting.
>
> Read this with the repo cookbook open: [00_framework-architecture](../framework_architecture/SKILL.md)
> and [../authoring_bdd_from_mcp/SKILL.md](../authoring_bdd_from_mcp/SKILL.md).

---

## 0. The MCP server (selenium-mcp-server)

Registered in `.vscode/mcp.json` (copy the `servers` block from `MCP/mcp.servers.json`). We launch
**`MCP/selenium_mcp_launch.py`** (not `python -m selenium_mcp_server`): the launcher adds the
missing `describe_tool` so the server starts, **and registers our 12 fast tools** on top of the
20 built-ins.

```jsonc
{
  "servers": {
    "selenium": {
      "command": "${workspaceFolder}/.venv/Scripts/python.exe",
      "args": ["${workspaceFolder}/MCP/selenium_mcp_launch.py"],
      "env": { "PYTHONUNBUFFERED": "1" },
    },
  },
}
```

| Server         | Mode             | Use for                                                  |
| -------------- | ---------------- | -------------------------------------------------------- |
| **`selenium`** | visible Chromium | discovery / locator resolution **and** watched execution |

Install once: `pip install -r MCP/requirements-mcp.txt`. Chrome + a matching chromedriver must
be available (the framework already manages chromedriver via `webdriver-manager`). The full tool
list — **20 base tools + 12 fast tools** (`login_sso`, `fill_form`, `click_sequence`,
`wait_and_click`, `select_radix_option`, `get_texts`, `snapshot_dom`, `assert_text`,
`screenshot_step`, `scroll_into_view_and_click`, `get_element_state`, `wait_for_idle`) — is in
[../selenium_mcp_tools/SKILL.md](../selenium_mcp_tools/SKILL.md).

---

## 1. The application under test

- **Cold Compass portal**, QA URL `https://qa.americoldcompass.com/` (`qa_url` / `APP_BASE_URL`
  in `.env`).
- A **React SPA** built with **shadcn/ui + Radix UI + Tailwind**.
- **Auth = Microsoft / Azure AD SSO.** Every walk starts authenticated.

### How to locate elements (anchor priority — text first)

`text()` / `normalize-space()` → `@aria-label`/`@role` → `@data-testid` → `@id`/`@name` →
semantic `@class` + tag (last resort) → positional (flag it). This app exposes excellent
`@data-testid` hooks (e.g. `cmo-tab-templates`) and Tailwind **state classes**
(`text-primary-white` = active, `text-teal-100` = inactive). **Never** anchor on hashed/utility
classes alone (`css-1x2y3z`, `flex`, `gap-2`, `mt-4`).

### Radix/shadcn patterns (how to drive them with the MCP)

| Pattern                 | How to operate it with the `selenium` MCP                                                                  |
| ----------------------- | ---------------------------------------------------------------------------------------------------------- |
| **Select dropdown**     | `click_element` the trigger (combobox) → a listbox opens → `click_element` the option by its text.         |
| **Date picker** (rdp)   | Navigate months via the next/prev buttons (`aria-label`), then `click_element` the day. Future = disabled. |
| **Cascading fields**    | A child is disabled until its parent is chosen; changing the parent resets downstream — assert each.       |
| **Confirmation prompt** | An `alertdialog` with Yes/No buttons.                                                                      |
| **Loading spinner**     | A `progressbar` — `wait_for_element` for it to disappear; never sleep.                                     |

### Locator-resolution ladder (when a selector fails during a walk)

Work the ladder in order — never jump straight to a positional/utility-class selector:

1. `kg_search` — has a previous session already verified a locator for this element? USE/VERIFY it.
2. `snapshot_dom` — re-find the element by **role / visible text / data-testid** in the compact
   snapshot; author the new XPath from what the snapshot actually shows.
3. `get_element_state` on the candidate — confirm it is displayed/enabled before acting.
4. Element genuinely absent from the DOM → it is **not a locator problem**: either a flow
   mistake (wrong screen / missing precondition) or a product change/bug — decide via the
   three-way fork in [../heal_tests/SKILL.md](../heal_tests/SKILL.md) §0.
5. On success: `kg_upsert(kind="locator", …)` immediately; on drift: `kg_record_failure` the
   old selector so the graph stops recommending it.

**Stale-element discipline:** never cache and reuse an element reference across a re-render —
Radix/React re-mounts nodes on state change. Re-locate after every navigation, dialog
open/close, or list refresh; a `StaleElementReferenceException` in a walk means "re-locate",
not "retry the same handle" and not "add a sleep".

**Recovery ladder (walk hit an unexpected state):** dismiss stray overlay/toast → `wait_for_idle`
→ re-`snapshot_dom` → if still wrong, re-navigate to the screen fresh → if the session is dead,
one `start_browser` + `login_sso` re-entry. **One recovery pass per step** — if the state is
still unexpected after a full pass, record it honestly (BLOCKED or defect candidate) and move
to the next case; never loop retries.

---

## 2. SSO login — the verified 9-step flow

**Fastest:** call the **`login_sso`** fast tool once — `{ "url": APP_BASE_URL, "username": …,
"password": … }` — it runs the whole flow below in a single round-trip. Use the manual steps only
if you need to debug a specific screen. Drive it with the `selenium` MCP after `start_browser` +
`navigate(APP_BASE_URL)`. Credentials come from `.env` (`QAAdminEmail` / `QAAdminPassword`).
**Never hardcode credentials.** The same flow lives in code at
`business_components/web_components/login.py → login_to_application_sso`.

1. Click **Sign in with SSO** (button on the app login page).
2. _(conditional)_ If shown, click **Use another account**.
3. Enter email → `input[name="loginfmt"]` (`send_keys`).
4. Submit → `input[type="submit"]`.
5. Enter password → `input[name="passwd"]`.
6. Submit → `input[type="submit"]`.
7. _(conditional, 0–2×)_ **Stay signed in?** → confirm (`input[type="submit"]`).
8. Wait for the app shell (Home page) to load.
9. Confirm you are authenticated before walking any case.

Steps 2 and 7 are **conditional** — check visibility first (`find_element` / `wait_for_element`).
Allow up to ~3 minutes for the whole flow. If you get redirected to the Microsoft sign-in page
**mid-walk**, the session expired → re-run the login flow; don't edit the test.

---

## 3. The execution rules

### Rule 0 — Knowledge-brain first & Live DOM Grounding (anti-hallucination)

Before anything else, **load the module's knowledge brain**: `.agents/context/<module>.context.md`
(derive `<module>` from the story path). It gives you the module's **purpose, navigation, every
feature, the domain vocabulary, the canonical flows, and the business rules** — so you walk the
app with **intent** instead of guessing. The brain is deliberately **locator-free**: it says
_what_ you're looking for and _why_, never _which selector_. If it's missing, create it from
`.agents/skills/_template_context/SKILL.md` (mine the module's real code read-only) before you walk.

**CRITICAL DOM GROUNDING CONTRACT:**
1. **Never Fabricate Selectors**: Every locator used during execution MUST be retrieved from `kg_context_pack` or observed live via `snapshot_dom(interactive_only=true)`.
2. **Inspect Unverified Screens**: If an element or screen state is not present in Knowledge Graph context, run `snapshot_dom(interactive_only=true, max_elements=50)` to inspect true DOM attributes before issuing any click or fill command.
3. **Knowledge first, live DOM second — never hallucinate a screen, selector, or flow.**

### Rule 1 — Sequential execution

Execute cases in strict **ID order** (`AMCC-TC-#####` order as listed in `testcases.json`).
**Never** parallelise within a single story's MCP walk.

### Rule 2 — Fresh context per case

Each case starts from a clean, authenticated state and ends fully before the next begins. In the
watched authoring flow this is "**one by one**": execute a single case live, author & save its
`Scenario` + steps, verify it, then advance. In the generated tests, isolation is provided by the
`bdd_driver` fixture (a fresh browser per scenario, via `pytest_bdd_before_scenario`).

### Rule 3 — Step-by-step granularity

For each ordered step in `testcases.json`:

```
→ Perform steps[N].action          (via a selenium MCP tool / a fast tool, the way a user would)
→ Read steps[N] result from the DOM (get_element_state / get_texts / assert_text / get_page_info)
→ Compare to steps[N].expectedResult
→ Record status + actual + any bug in TestcaseExecution/<module>/<STORY-ID>/execution-results.json
→ Log ✅ PASS / ❌ FAIL with the reason        (PASS → NO screenshot — keep walking)
→ On a genuine FAIL (own mistake ruled out): screenshot_step(…, defect=true) the defect state,
  write <CASE-ID>_defect.json, render the defect .docx, keep the step FAIL, advance
```

> **Screenshot ONLY defects — nothing else.** Passing steps produce **zero** PNGs (the walk stays
> fast and the evidence tree stays clean). The moment a step's live result contradicts its
> expected result — and you've ruled out your own navigation/data mistake — capture the defect
> state with `screenshot_step(module, story_id, case_id, step=<shot #>, defect=true)` →
> `TestcaseExecution/<module>/<STORY-ID>/<CASE-ID>/<CASE-ID>_defect_<N>.png`, then follow
> [../defect_documentation/SKILL.md](../defect_documentation/SKILL.md) (payload + `.docx`).
>
> **Shared defects:** if the same defect affects other cases, pass `also_cases` so the same PNG
> lands in every affected case's folder — each still gets its own `_defect.json`.

In the authored pytest-bdd test this becomes one `@when`/`@then` per step, each `@then` asserting
that step's `expectedResult` with a `CustomPyAutoWeb` verify inside the `try/except` wrapper, with
an Allure `Action —` / `Verification —` attachment. See [../authoring_bdd_from_mcp/SKILL.md](../authoring_bdd_from_mcp/SKILL.md).

### Rule 4 — Context-first execution

Before executing a case, load and internalise, in order:

```
0. .agents/context/<module>.context.md             (the knowledge brain — purpose + rules; FIRST)
1. MCP/stories/<module>/<STORY-ID>/story.md        (acceptance criteria + business context)
2. MCP/stories/<module>/<STORY-ID>/testcases.json  (the ordered steps to execute)
3. The module's existing test_<module>-*.py / POM / web_component  (the exact style to match)
```

### Rule 5 — Evidence + report

- **Defect evidence (the ONLY screenshots):** when a genuine bug is confirmed, capture the defect
  state → `TestcaseExecution/<module>/<STORY-ID>/<CASE-ID>/<CASE-ID>_defect_<N>.png`
  (`screenshot_step(…, defect=true)`), write `<CASE-ID>_defect.json`, and render the formal
  **`<CASE-ID>_defect.docx`** (Title / Description / Prerequisites / Steps to Reproduce /
  Expected Result / Actual Result + embedded shots) with
  `python MCP/build_defect_docx.py --story … --case <CASE-ID>` — see
  [../defect_documentation/SKILL.md](../defect_documentation/SKILL.md). Passing steps → **no PNGs**.
- **Execution ledger:** `TestcaseExecution/<module>/<STORY-ID>/execution-results.json` — each
  step's status / actual / bug, written as you walk (see `MCP/build_execution_docx.py --scaffold`).
- **Coverage gate:** `MCP/stories/<module>/<STORY-ID>/coverage-report.md` — every story
  AC/feature mapped to its covering case(s); every gap called out as **NOT COVERED** before the
  walk starts.
- **Word report (per story):** after all cases, build the rolled-up
  `TestcaseExecution/<module>/<STORY-ID>/<STORY-ID>_Execution.docx` with
  `python MCP/build_execution_docx.py --story …` — one section per case, expected-vs-actual +
  status per step, defect screenshots embedded where a bug was found, and a Defects roll-up.
- **Primary CI report:** Allure (run → scenario → step), built by the framework's existing
  `conftest.py` hooks (per-step screenshots, failure analysis) when the authored tests run.

### Rule 6 — Memory ledger (never lose your place)

PREPARE writes `MCP/stories/<module>/<STORY-ID>/execution-progress.md` — a checklist that mirrors
the plan **1:1** (every case → every step → every authoring artefact). **Tick each box the moment
it's true** as you execute and author. It is your durable, on-disk **memory**, so a long
one-by-one run never repeats a step, skips one, or forgets which case it's on. Never tick a box
you didn't actually do; mark a real Expected-vs-Actual contradiction `[!]` (a bug / stale story)
instead of faking a pass.

---

## 4. Go fast (fewer Antigravity round-trips)

Each base tool is one action per call — slow when a step needs 5–15 of them. Prefer the **12 fast
tools** that bundle the common patterns into a single call:

| Want to…                                  | Use (one call)                                                         |
| ----------------------------------------- | ---------------------------------------------------------------------- |
| Log in (whole Azure-AD SSO)               | `login_sso { url, username, password }`                                |
| See the screen / pick locators            | `snapshot_dom` (compact role/text/test-id list — not full page_source) |
| Fill a form (many fields)                 | `fill_form { fields: […] }`                                            |
| Multi-hop nav (Apps → CMO → tab)          | `click_sequence { clicks: […] }`                                       |
| Open a shadcn/Radix Select + pick option  | `select_radix_option { trigger_selector, option_text }`                |
| Wait then click (JS fallback)             | `wait_and_click` / `scroll_into_view_and_click`                        |
| Read many texts (options / rows)          | `get_texts { selector }`                                               |
| Validate active tab / disabled / selected | `get_element_state { selector, attributes }`                           |
| Check expected text (honest PASS/FAIL)    | `assert_text { expected, mode }`                                       |
| Wait for the SPA to settle (no spinner)   | `wait_for_idle { target_selector? }` (never `time.sleep`)              |
| Save DEFECT evidence into the tree        | `screenshot_step { module, story_id, case_id, step, defect: true }`    |

Full argument shapes: [../selenium_mcp_tools/SKILL.md](../selenium_mcp_tools/SKILL.md) §7.

---

## 5. Hard rules (recap)

- **Headed at 75%, always** — `start_browser` in full mode so you watch it (never headless for a
  walk). The launcher auto-adds `--force-device-scale-factor=0.75` (matches the pytest-bdd runtest)
  - `--window-size=1920,1080`, so the whole app fits in each screenshot.
- **One by one, fully** — finish a case (all steps validated live) and author its test before the next.
- **Knowledge first** — load `.agents/context/<module>.context.md` before the walk, and keep the
  `execution-progress.md` ledger updated as your memory; never hallucinate a screen/flow.
- **Coverage first** — `coverage-report.md` written and every **NOT COVERED** gap called out
  before the first live step.
- **Go fast** — prefer the 12 fast tools over many single-action calls (see §4).
- **Defect-only evidence** — **zero** screenshots for passing steps. A confirmed bug →
  `screenshot_step(…, defect=true)` → `<CASE-ID>_defect_<N>.png` + `<CASE-ID>_defect.json` +
  the rendered `<CASE-ID>_defect.docx` ([../defect_documentation/SKILL.md](../defect_documentation/SKILL.md)).
  Record every step's status/actual/bug in `execution-results.json`, then build the story
  `<STORY-ID>_Execution.docx` (`python MCP/build_execution_docx.py --story …`).
- **Live DOM wins** — resolve every genuinely-new locator from the live snapshot; reuse existing
  POM locators first; never guess from memory (if blocked, STOP and report "blocked on MCP").
- **Never fake a pass** — a live contradiction of `expectedResult` is a bug/stale story; record it.
- **No `time.sleep`** in authored code — `web.wait_element_visibility(...)` / `web.wait(n)` only.

### Walk charter (session auditability — one KG chunk per walk)

At the **start** of any exploratory/diagnostic walk (not a scripted case execution), state a
one-line charter: **goal + scope + timebox** (e.g. "Explore the redesigned Facility filter on
ActionPlan landing to re-resolve AMCC-4417 locators — 15 min"). At the **end**, capture it:

```text
kg_add_knowledge(topic="Walk — <module> <goal>", tag="exploration",
                 text="charter: <goal/scope/timebox> · findings: <what was proven/changed> · follow-ups: <open questions>")
```

Link it to the story node with `kg_link`. This turns each MCP session into an auditable,
reusable exploration record instead of an unlogged browse — the next session's `kg_recall`
starts where this one ended.
