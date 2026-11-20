---
description: The cookbook for HEALING failing pytest-bdd tests via the Selenium MCP.
name: heal_tests
---

# Skill — Heal Tests (fix failing pytest-bdd tests via the Selenium MCP)

> How to **fix (heal) failing tests** in this repo: take an **Allure** run, triage every
> **FAILED + BROKEN** test, classify _why_ it failed, then re-walk the failing step **live** through
> the `selenium` MCP and **fix it at the source** — or, if the app truly misbehaves, **find the bug
> and leave it failing**. Healing fixes _how a test drives/reads the app_, **never** _what it verifies_.
>
> Read with the cookbook open: [00_framework-architecture](../framework_architecture/SKILL.md) ·
> [../mcp_execution_protocol/SKILL.md](../mcp_execution_protocol/SKILL.md) ·
> [../selenium_mcp_tools/SKILL.md](../selenium_mcp_tools/SKILL.md) ·
> [../page_objects/SKILL.md](../page_objects/SKILL.md) ·
> [../custompyautoweb_api/SKILL.md](../custompyautoweb_api/SKILL.md) ·
> [../assertions_and_allure/SKILL.md](../assertions_and_allure/SKILL.md).

Driven by the **`/heal-tests`** prompt ([../skills/heal_tests_prompt/SKILL.md](../skills/heal_tests_prompt/SKILL.md)).

---

## 0. The two golden rules of healing

1. **Heal the test, not the verification.** You may fix the **locator**, a **wait**, the
   **web_component flow**, or the **expected data** — never weaken, delete, or `xfail` an
   assertion. A genuine Expected-vs-Actual mismatch is a **product bug**: find it and **leave the
   test failing**.
2. **The Developer UI Source & Live DOM Decide.** The classifier gives a _hypothesis_; you **must inspect the developer UI source code** (`C:\Users\Administrator\UIREPOAMCC\amc-cold-compass-portal-ui\apps\<module_name>`) to understand component structure, state hooks (`useState`), ARIA attributes, and initial render states (e.g. accordions collapsed by default), and then **confirm the true root cause on the live app** (via the MCP) before changing one character. Never guess a locator or flow from memory.
3. **The three-way fork — decide it FIRST for every failure:**
   - the **app changed correctly** → update the expected value / testData (TEXT_CHANGED);
   - the **test mis-drives a conformant app** → fix locator / wait / flow (drift/timing/script);
   - the **app is broken** → report the bug, leave failing (APP_BUG).
     Naming which branch you are on — before editing — prevents the classic heal mistake of
     "fixing" the test when the product is at fault.
4. **One fix at a time.** When a test has multiple errors, fix the first, re-run, then tackle the
   next. Never shotgun several hypotheses into one edit — you won't know which one worked.
5. **Never widen a wait as the only fix, and never raise `wait_time` above 60.** If an element
   "needs more time", the real cause is usually a missing gate (spinner/overlay) or a wrong
   locator — find it. Inflated timeouts hide root causes and slow the whole suite.
6. **Timing fixes need a stability confirmation.** If the root cause was WAIT_TIMING (or the test
   had a flaky history), the confirmation re-run is **2 consecutive green runs**, not 1 — a
   single green after a timing tweak proves nothing. (Both runs count as the case's run budget.)

---

## 1. The flow (and the one CLI)

```
ANALYZE  →  STORY-FIRST  →  RE-WALK LIVE  →  HEAL or ESCALATE  →  RE-RUN  →  VERIFY
```

The deterministic spine is **`MCP/mcp_heal_tests.py`** (the heal counterpart of
`MCP/mcp_execute_and_author.py`):

```bash
# Triage + plan the whole last run (reads raw results OR a generated report):
python MCP/mcp_heal_tests.py --allure reports/allure-results

# Scope to one story (artefacts land in MCP/stories/<module>/<STORY-ID>/):
python MCP/mcp_heal_tests.py --allure reports/allure-results --story "/CMO/AMCC-8438/story.md"

# Just list the failing tests + classification:
python MCP/mcp_heal_tests.py --allure reports/allure-results --list

# After fixing + re-running, confirm nothing in scope is still red:
python MCP/mcp_heal_tests.py --allure reports/allure-results --verify
```

ANALYZE writes two artefacts (in the story folder when `--story` is given, else `MCP/healing/`):

| File               | What it is                                                                                                                                       |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `heal-plan.md`     | The **worklist** — a triage table + one block per failing test (root cause, source files, the live re-walk, the fix recipe, the re-run command). |
| `heal-progress.md` | The **live memory ledger** — tick each box as you heal so a long run never loses its place.                                                      |
| `healing-log.md`   | Append-only audit trail — one line per heal/escalation (created on first run).                                                                   |

---

## 2. How the Allure run is read

`mcp_heal_tests.py` (via `mcp_pipeline/allure_results.py`) accepts **either**:

- the **raw** results Allure-pytest writes (`pytest --alluredir=reports/allure-results` →
  `*-result.json`), **or**
- a **generated** Allure HTML report (`data/test-cases/*.json`).

It then:

- keeps only the **latest attempt** per test (retries collapsed — a test that finally passed is
  **not** reported as failing),
- isolates **FAILED + BROKEN** tests,
- recovers each test's **case id** (`AMCC-TC-#####`), **story id** (`AMCC-####`), and **module**
  (`CMO` …) from the Allure labels / scenario name / feature-file path,
- finds the **first failing Gherkin step**, and
- **classifies** the failure (§3).

> Always run the suite with `--alluredir=reports/allure-results` so there is a run to heal. The
> framework's `conftest.py` already attaches per-step screenshots + failure analysis to Allure.

---

## 3. The failure taxonomy (signals → root cause → fix)

The classifier reads the failure `message` + `trace` and assigns one category. **Selenium / timeout
/ Python causes are detected before the generic `AssertionError` wrapper**, so a wrapped
`TimeoutException` is read as a wait/timing issue, not a value mismatch. A pure value/state
assertion mismatch defaults to **APP_BUG (do not auto-fix)** — so a real bug is never silently healed.

| Category                    | Typical Allure signal                                                                                                                   | Confirm live                                                | Fix (at the source) — or escalate                                                                   |
| --------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| **LOCATOR_DRIFT** 🔧        | `NoSuchElementException`, "Unable to locate element", `InvalidSelectorException`                                                        | The element is there but the selector no longer matches.    | **Update the POM locator** (role/text/`data-testid` first). Don't touch the step text/assertion.    |
| **WAIT_TIMING** 🔧          | `TimeoutException`, `ElementClickInterceptedException`, `StaleElementReferenceException`, "not interactable", "to be visible/clickable" | The element appears, just later / behind a spinner/overlay. | **Gate with `web.wait_element_visibility(...)`** (or wait for the spinner). **Never** `time.sleep`. |
| **SCRIPT_FUNCTIONALITY** 🔧 | `KeyError`, `TypeError`, `IndexError`, `AttributeError`, `FileNotFoundError`, `JSONDecodeError`                                         | The step/web_component logic or a data key is wrong.        | **Fix the flow / data key** in the step or web_component. Keep layers clean.                        |
| **TEXT_CHANGED** 🔧         | "Text validation failed", "Expected to contain", "Negative validation failed"                                                           | The app's **copy** was reworded (and is correct).           | **Update the expected string / `testData/<module>.json`** — _only_ if the new copy is correct.      |
| **APP_BUG** 🐞              | "Count validation failed", "Exact match failed", bare `AssertionError`, `Expected:/Actual:`                                             | The app's value/state genuinely contradicts the expected.   | **Reproduce, screenshot, LEAVE FAILING, escalate.** Never loosen the assertion.                     |
| **UNKNOWN** 🔧              | driver/session/env noise (`SessionNotCreatedException`, `chromedriver`, `connection refused`) or anything unmatched                     | Re-walk to find the real cause.                             | Fix the runner/setup (env) or the source; escalate if it's a bug.                                   |

> **TEXT_CHANGED vs APP_BUG** is the one judgment call. A reworded label/title/toast that the app
> now shows correctly → TEXT*CHANGED (update the expected). A \_wrong* value/count/state → APP_BUG
> (leave failing). When unsure, treat it as a **bug** and escalate — never edit the expectation to
> make red go green.

---

## 4. Map a failed Allure test → the files to edit

`heal-plan.md` already prints these for you, but the rule is:

```
story id  AMCC-8438   →  module folder (from labels / .../Apps/<Module>/ path)  →  CMO
case  id  AMCC-TC-15200

Feature   tests/features/Apps/CMO/cmo-AMCC-8438.feature        (the Scenario; @AMCC-TC-15200 tag)
Step def  tests/step_defs/apps/CMO/test_cmo-AMCC-8438.py       (the @when/@then for the failing step)
Locators  object_repository/pages/Apps/CMO/*.py                (the POM — the ONLY place to edit a selector)
Reuse     business_components/web_components/cmo.py            (shared interaction flows)
Data      testData/cmo.json                                    (expected values / keys)
```

Open the **step-def** for the failing step first; from it you'll see which **POM locator** /
**web_component** function it uses — that's what you re-resolve and fix.

---

## 5. STORY-FIRST diagnosis, then re-walk the failing step LIVE

### 5.0 Story-first (before any browser)

1. Read `MCP/stories/<module>/<STORY-ID>/story.md` — fetch it if missing:
   `python MCP/fetch_userstory.py <STORY-ID> --module <Module>` (JIRA creds in `MCP/.env.mcp`;
   if auth is unavailable, continue and flag it — never fabricate acceptance criteria).
2. Read the case's steps in `MCP/stories/<module>/<STORY-ID>/testcases/*.md`.
3. Map: failing Gherkin step → test-case step → the acceptance criterion it verifies. Decide the
   _hypothesis_: app violates the story (→ APP_BUG candidate) vs test mis-drives a conformant app
   (→ drift/timing/flow candidate). Note it as `story-first: <verdict>` in `heal-plan.md`.
4. Only then walk live — the DOM confirms or refutes the story-first verdict.

### 5.1 Re-walk live (the proof before the fix)

For each finding (one at a time):

1. `start_browser(chrome)` → `navigate(APP_BASE_URL)` → **`login_sso`** (one call; see
   [../mcp_execution_protocol/SKILL.md](../mcp_execution_protocol/SKILL.md) §SSO).
2. Reach the case's screen the way the test does, **up to the failing step** (reuse the fast tools:
   `click_sequence` for nav, `select_radix_option` for shadcn Selects, `fill_form` for forms).
3. `snapshot_dom` (compact role/text/test-id list) and/or `get_element_state` on the target, and
   **compare the live DOM to what the step/POM expects**. Now you know the true root cause.

> Re-walking is non-negotiable: it's how you tell **locator drift** (element present, selector
> stale) from a **real bug** (element/value genuinely absent/wrong).

---

## 6. The fix recipes (copy the module's existing style exactly)

### 6.1 LOCATOR_DRIFT — update the POM locator only

Re-resolve live, then edit the **Page Object** (never the step):

```python
# object_repository/pages/Apps/CMO/CmoLandingPage.py
# BEFORE (stale — element renamed):
locatorStartNewAuditButton = [(By.XPATH, "//button[@data-testid='start-new-audit']")]
# AFTER (grounded in the live snapshot — prefer role/text/data-testid):
locatorStartNewAuditButton = [(By.XPATH, "//button[normalize-space()='Start New Audit']")]
```

The step is untouched — it still does `web.click_wait_locator(locatorList=PAGE.locatorStartNewAuditButton)`.

### 6.2 WAIT_TIMING — gate the action/read (never sleep)

Add the framework wait **before** the action/read in the step or web_component:

```python
web = CustomPyAutoWeb(bdd_driver)
web.wait_element_visibility(locatorList=PAGE.locatorAuditList, wait_time=60)   # ← the heal
web.click_wait_locator(locatorList=PAGE.locatorAuditList)
```

For a spinner: wait for the `progressbar` to disappear (`web.element_should_not_visible(...)`), or
use the MCP `wait_for_idle` while walking. **Never** `time.sleep`; `web.wait(n)` only for a short,
bounded settle (typeahead/animation).

### 6.3 SCRIPT_FUNCTIONALITY — fix the flow / data key

Wrong order, a missing interaction, or a bad data key (`KeyError`). Fix it in the **web_component**
(shared) or the step, keeping it assertion-free and locator-list driven:

```python
# business_components/web_components/cmo.py — was reading before the list populated
def open_first_audit(driver, timeout: int = 30):
    web = CustomPyAutoWeb(driver)
    web.wait_element_visibility(locatorList=CMOLandingPage.locatorAuditRows, wait_time=timeout)  # ← add the wait
    web.click_first_element(locatorList=CMOLandingPage.locatorAuditRows)
```

```python
# testData/cmo.json — the step looked up a key that was renamed
{ "CMO_template_SH_NoHelp": { ... } }   # ← align the key the step requests with the data
```

### 6.4 TEXT_CHANGED — update the expectation (only if the new copy is correct)

Confirm live that the app now shows the new copy, then update the **expected value** (or its
`testData` entry) — **not** the assertion logic:

```python
# testData/cmo.json
{ "expected_heading": "Create Master Audit" }   # ← was "Create Audit"; app reworded it correctly
```

If the value is genuinely _wrong_ (not just reworded) → it's a **bug** (§6.5), not a TEXT_CHANGED.

### 6.5 APP_BUG — find it, leave it failing, document it formally

Reproduce the exact step live. If the app truly contradicts the expected result:

- **Leave the test FAILING** (do **not** edit the assertion, the expected data, or add `xfail`).
- **Document the defect formally** (see [../defect_documentation/SKILL.md](../defect_documentation/SKILL.md)):
  1. Capture the defect state: `screenshot_step(module, story_id, case_id, step=1, defect=true)`
     → `TestcaseExecution/<Module>/<STORY-ID>/<CASE-ID>/<CASE-ID>_defect_1.png` (more shots as needed).
  2. Write `<CASE-ID>_defect.json` — title ("QA - <Area>: <summary>"), description, prerequisites,
     stepsToReproduce (exactly how you reproduced it live), expectedResult, actualResult,
     screenshots, evidence (the live-DOM/API proof), severity.
  3. Render the document: `python MCP/build_defect_docx.py --story "<story>" --case <CASE-ID>`
     → `<CASE-ID>_defect.docx`.
- **Escalate** — log it in `healing-log.md` **with the defect doc's path**; raise it per the
  team's process. A bug without its defect doc is an unreported bug.

```python
# DO NOT "heal" a real bug like this — this is forbidden:
# assert actual == expected            →  assert True            ❌ never
# assert count == 5                    →  assert count in (3,5)  ❌ never
```

---

## 7. Re-run + verify gate

After each fix, re-run **just that test** as a **BLOCKING foreground command** — issue it, wait
for the terminal to return, read the returned output. That output IS the result:

```bash
pytest "tests/step_defs/apps/CMO/test_cmo-AMCC-8438.py" -k "AMCC-TC-15200" --alluredir=reports/allure-results
```

### MANDATORY PARALLEL EXECUTION (for 2+ tests)

**ALL healing runs with multiple tests MUST use pytest-xdist parallel execution.**

```powershell
# Calculate workers: min(test_count, 8)
$testCount = <number of failing tests>
$workers = [Math]::Min($testCount, 8)

# Single test — no -n flag
pytest "tests/step_defs/apps/CMO/test_cmo-AMCC-8438.py" -k "AMCC-TC-15200" --alluredir=reports/allure-results

# 2-8 tests — use -n <test_count>
pytest -k "AMCC-TC-15200 or AMCC-TC-15201 or AMCC-TC-15202" -n 3 --alluredir=reports/allure-results

# 9+ tests — cap at -n 8
pytest tests/step_defs/apps/CMO/ -n 8 --alluredir=reports/allure-results

# Whole module verification — ALWAYS parallel
pytest tests/step_defs/apps/CMO/test_cmo-AMCC-8438.py -n auto --alluredir=reports/allure-results
```

**❌ FORBIDDEN — Sequential execution of multiple tests:**

```powershell
# NEVER DO THIS — runs tests one by one, wastes session time
pytest -k "A or B or C" --alluredir=reports/allure-results  # Missing -n!
```

> ⛔ **Never** redirect the run to a log file (`Out-File`/`Tee-Object`/`>`) and then
> `Start-Sleep`-poll it, tail it, check its `.Length`, or count chrome processes to "monitor"
> progress. Redirected logs flush only at process exit — mid-run polls show stale/empty data
> and produce hallucinated "run is stuck" conclusions about healthy runs. Even a 15–20 minute
> confirmation run is simply waited on. Background + artefact polling (max 3 polls, ≥10 min
> apart, via fresh `*-result.json` mtimes) is reserved for the ONE end-of-session whole-scope
> run only. See [../efficiency_and_hygiene/SKILL.md](../efficiency_and_hygiene/SKILL.md) §3.

When all findings are handled, re-run the scope and confirm only genuine bugs remain:

```bash
pytest "tests/step_defs/apps/CMO/test_cmo-AMCC-8438.py" -n auto --alluredir=reports/allure-results
python MCP/mcp_heal_tests.py --allure reports/allure-results --verify
```

`--verify` **PASSES** when nothing in scope is red. Confirmed product bugs are **expected** to stay
red — exclude them with a narrower `--story`/`--category`; never loosen them to pass.

---

## 7.5 MANDATORY LIVE MCP WALK (before ANY fix)

**CRITICAL: You MUST complete a live MCP walk for EVERY failing test BEFORE making any fix.**

This is the anti-hallucination contract. Without the live walk, you risk:

- Fixing the wrong locator (element name changed but you guessed wrong)
- Missing the real issue (timing when you thought it was drift)
- Breaking other tests (the locator you "fixed" was actually correct)

### Live MCP Walk Protocol

```text
1. start_browser(chrome)
2. login_sso(username=..., password=...)
3. navigate to the test's starting screen
4. Execute EACH step of the failing test:
   - For each action: wait_for_idle → find element → verify it matches POM
   - For each assertion: check the actual DOM state
5. At the failing step:
   - snapshot_dom to see the full page state
   - get_element_state on the target element
   - Compare live DOM to what the test expects
6. ONLY NOW do you have the true root cause
```

### Real-time KG Updates During Walk

**Update the knowledge graph IMMEDIATELY as you discover issues:**

```text
# The moment you find a drifted locator:
kg_record_failure(selector="//old/xpath", reason="Element renamed to 'New Name'")

# The moment you verify a working locator:
kg_upsert(kind="locator", element_name="Submit Button", selector="//new/xpath", ...)

# The moment you identify the root cause:
kg_add_knowledge(topic="Healing — AMCC-TC-15200", tag="healing",
                 text="LOCATOR_DRIFT: button[data-testid='submit'] renamed to button[text()='Submit']")
```

**Do NOT batch these updates for the end of the session — update DURING the walk.**

### Knowledge-graph capture (continuous, story-linked)

Write knowledge **the moment it is proven**, not at the end:

```text
kg_upsert(kind="locator", …)                       # the NEW working locator, right after live proof
kg_record_failure(selector=…, reason=…)            # the OLD drifted locator
kg_upsert(kind="fact", name="Story — <STORY-ID>", tag="story",
          text="<summary> · ref: MCP/stories/<module>/<STORY-ID>/story.md")
kg_link(from_id=<story node>, to_id=<flow/locator/healing node>)   # wire the graph
kg_add_knowledge(topic="Healing — <CASE-ID>", tag="healing", text="root cause → fix/escalation")
kg_reindex()                                        # once, at session end
```

The story node's `ref:` path is the canonical pointer back to the full `story.md`; the `kg_link`
edges (story ↔ feature ↔ flow ↔ locators ↔ healing facts) are what make the graph navigable for
the next session.

### `healing-log.md` line format

```
| Date | Case | Category | Root cause | Action (heal at file / escalated bug) | Result |
| 2026-06-30 | AMCC-TC-15200 | LOCATOR_DRIFT | locator renamed start-new-audit | heal: CmoLandingPage.locatorStartNewAuditButton | green |
| 2026-06-30 | AMCC-TC-15201 | APP_BUG | badge shows 3, expected 5 | escalate: product bug AMCC-XXXX + defect docx | left failing |
```

The **Category** column is mandatory (one of the §3 taxonomy buckets). It makes the log minable:
a case that keeps re-appearing as WAIT_TIMING across sessions is a **flaky test** — stop
patching waits and re-diagnose its flow (isolation, data collisions, missing idle gate).

---

## 8. Heal self-check (run before you call a test "healed")

- [ ] I read the **user story + test-case steps first** and wrote a `story-first:` verdict before the live walk.
- [ ] I **re-walked the failing step live** and confirmed the true root cause (not just the classifier guess).
- [ ] The fix is at the right layer: **locator → POM** · **wait → step/web_component** · **flow → web_component/step** · **expected text → expected data**.
- [ ] I did **not** weaken, delete, or `xfail` any assertion; I added **no** `time.sleep`.
- [ ] New locators are grounded in the live MCP snapshot (role / text / `data-testid` first) and live **only** in the Page Object.
- [ ] I matched the module's existing naming / imports / style exactly (opened a sibling file to confirm).
- [ ] The single-test re-run is **green** — or it's a **confirmed product bug**, left failing, screenshotted, and escalated.
- [ ] I appended the outcome to `healing-log.md`, and `--verify` shows only genuine bugs remaining.
- [ ] I created **zero scratch files** and **zero log-file redirects**; confirmation runs were blocking foreground calls (no `Start-Sleep` polls, no process counting), only the final whole-scope run (if >20 min) went to the background with ≤3 artefact-based polls, and I stayed inside the run budgets.
- [ ] Knowledge was captured **as it was proven** and linked to the story node (`kg_link`); `kg_reindex` ran at session end.
