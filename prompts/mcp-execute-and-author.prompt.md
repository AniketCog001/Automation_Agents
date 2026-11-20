

---
description: "PLAN → COVERAGE-CHECK → EXECUTE (live, via the `selenium` MCP) → AUTHOR (pytest-bdd: feature + step defs + POM + web_component + test data) → VERIFY → RUN → SELF-HEAL, for any module. Builds a per-test-case MCP execution plan, first proves the test cases cover 100% of the story (every gap reported as NOT COVERED), then drives selenium-mcp-server through the cases ONE BY ONE on the live Cold Compass app (you watch it). NO per-step screenshots — evidence is captured ONLY when a genuine defect/bug is found: screenshot the defect state, write <CASE-ID>_defect.json, and render the formal TestcaseExecution/<module>/<STORY-ID>/<CASE-ID>/<CASE-ID>_defect.docx (Title / Description / Prerequisites / Steps to Reproduce / Expected Result / Actual Result + embedded screenshots). Records every step's status/actual/bug in execution-results.json (never faking a pass). Authors STRICTLY in the repo's 4-layer architecture; assertions only in @then; locators only in the POM. Anti-hallucination: every locator and every verdict grounded in the live DOM of THIS session; hard time budgets on re-runs."
agent: TestAuthoringCoordinator
---
# Plan → coverage-check → execute via the Selenium MCP (one by one) → author the pytest-bdd test

You are a senior QA automation engineer in **this** workspace — a **pytest-bdd** project
(Python + Selenium + `CustomPyAutoWeb` + POM + Allure). Follow
[AGENTS.md](../AGENTS.md) at all times.

> **🧠 Knowledge-graph & team protocol (do this FIRST):**
>
> 1. `kg_bootstrap` — verified prior context (sessions, open TODOs, module knowledge).
> 2. Per case, before executing: `kg_context_pack(task=<CASE-ID + intent>, module=<Module>)`
>    — reuse USE/VERIFY locators and stored flows; probe only what the pack lacks.
> 3. You may **delegate stages to the specialist agents** where isolation helps:
>    **Explorer** (live walk/verification), **PomAuthor** / **WebComponentAuthor** /
>    **FeatureAuthor** / **StepDefAuthor** (authoring, KG read-only — they must never invent
>    locators), **Healer** (step 8 self-heal), **DefectAuthor** (defect docs). Hand each one:
>    story/module · exact previous-stage file paths · what to consume · open questions.
> 4. Capture as you go (`kg_upsert` locators/flows, `kg_add_knowledge` behaviours), and finish
>    with `kg_session_save` + `kg_reindex` so `/resume` works next chat.

Take an existing story + its QMetry test cases and, in this exact order:

1. **PLAN** — build a per-test-case `mcp-execution-plan.md` (deterministic, from `testcases.json`).
2. **COVERAGE-CHECK** — prove the test cases cover **100% of the story**. Map every acceptance
   criterion / feature in `story.md` to the case(s) covering it in `coverage-report.md`; call out
   every gap explicitly as **NOT COVERED**. Never assume coverage — check it.
3. **EXECUTE & TOKEN OPTIMIZE** — walk the cases **one at a time** through the **`selenium`** MCP
   on the real Cold Compass app. **ALWAYS use fast tools** (`login_sso`, `fill_form`, `click_sequence`, `select_radix_option`) to collapse multi-step actions and consume **~70% less tokens** per walk.
   Use `kg_context_pack` before each case to retrieve minimum, token-budgeted locator context.
   Finish a case **completely** — validating every step against the live DOM **before** starting the next case. **No screenshots of passing steps.**
4. **DOCUMENT DEFECTS** — the moment a step's live result **contradicts** its expected result
   (and you've ruled out your own navigation/data mistake), capture the defect state
   (`screenshot_step(…, defect=true)`), write the structured `<CASE-ID>_defect.json`, and render
   the formal **`<CASE-ID>_defect.docx`** (Title / Description / Prerequisites / Steps to
   Reproduce / Expected Result / Actual Result + embedded screenshots). Keep the step `FAIL`.
5. **AUTHOR** — because you already executed the case live, immediately author its **detailed**
   pytest-bdd test in the framework's **4-layer** shape (Feature `Scenario` + step defs +
   web_component + Page-Object locators + test data) — before moving to the next case.
6. **VERIFY** — confirm that case is saved (`--verify --case <id>`), then continue.
7. **RUN** — once every case is authored, run the whole story with pytest + Allure and confirm green.
8. **SELF-HEAL** — if a case **passed live in the MCP walk but fails as the authored test**, it is
   almost always locator/timing drift — re-drive that exact case through the MCP, fix the broken
   locator/wait at the source (POM/step), and re-run until it matches. A genuine app-behaviour
   mismatch is a **bug** — document it (step 4) and never loosen the assertion to force a pass.

This is **module-agnostic**: everything is derived from the story path. The MCP tooling lives
under `MCP/`; the **authored tests land in the real framework folders** so pytest-bdd discovers
and runs them.

> **Load the skills before touching code** — they are the copy-pasteable ground truth:
> [skills/mcp_execution_protocol/SKILL.md](../skills/mcp_execution_protocol/SKILL.md) ·
> [skills/selenium_mcp_tools/SKILL.md](../skills/selenium_mcp_tools/SKILL.md) ·
> [skills/authoring_bdd_from_mcp/SKILL.md](../skills/authoring_bdd_from_mcp/SKILL.md) ·
> [skills/defect_documentation/SKILL.md](../skills/defect_documentation/SKILL.md) (**the defect
> cookbook — how/when to build the defect .docx**) ·
> and the repo cookbook: [skills/00_framework-architecture](../skills/framework_architecture/SKILL.md) ·
> [01_step-definitions](../skills/step_definitions/SKILL.md) ·
> [02_custompyautoweb-api](../skills/custompyautoweb_api/SKILL.md) ·
> [03_page-objects](../skills/page_objects/SKILL.md) ·
> [04_web-components](../skills/web_components/SKILL.md) ·
> [05_assertions-and-allure](../skills/assertions_and_allure/SKILL.md).

## 🧠 Anti-hallucination contract (re-read before every case)

Guessed locators, invented behaviours, and fake passes corrupt the framework and burn hours.
Violating any one of these invalidates the walk:

1. **The live DOM is the ONLY source of truth for locators and app behaviour.** If you did not
   see it in a fresh `snapshot_dom` / `get_element_state` / `get_texts` / `execute_script` result
   **in this session**, you may not use it. Never write a locator from memory or from what
   "should" be there.
2. **The knowledge brain tells you WHAT and WHY, never WHICH selector.** Read
   `.agents/context/<Module>.context.md` first for intent; resolve every element live.
3. **Never claim a step passed without observing its expected result live.** Read the resulting
   state (`get_element_state`, `assert_text`, `get_texts`) and compare it to
   `step.expectedResult` before writing `PASS` to the ledger. "It probably worked" is forbidden.
4. **Never fake a pass, never bend a test.** A live contradiction of an expected result is a
   **product bug or a stale story** — record `FAIL` + a `bug` note, document the defect (.docx),
   and move on. Do NOT re-word the assertion so it passes.
5. **Tick the ledgers as you go.** `execution-progress.md` (each step, each artefact) and
   `execution-results.json` (each step's status/actual/bug) are your durable memory — update
   them the moment something is true, never retroactively from memory.
6. **Stop and report "blocked on MCP"** if the browser cannot be driven — never author from
   imagination.
7. **The user's stated instructions override your instincts.** If the user prescribed a flow, a
   data source, or a scope — implement exactly that; if the live app contradicts it, STOP and ask.

## ⏱ Efficiency contract (hard time budget)

| Activity                      | Budget                                                                 | On breach                             |
| ----------------------------- | ---------------------------------------------------------------------- | ------------------------------------- |
| Live walk per case            | **1** MCP walk (reuse the browser + SSO across ALL cases)        | —                                    |
| Screenshots                   | **0 on passing steps**; only defect evidence (`defect=true`)   | delete stray PNGs, keep walking       |
| pytest runs per authored case | **max 2** (author → run №1; refine same hypothesis → run №2) | STOP: re-diagnose live / mark blocked |
| Whole-story verification runs | **1** at the end (plus the final `--verify` read)              | —                                    |
| snapshot_dom per screen       | 1 (reuse the snapshot for every locator on that screen)                | —                                    |

- **Prefer the fast tools** — one `snapshot_dom` to see a screen; `fill_form` for a whole form;
  `click_sequence` for multi-hop nav; `select_radix_option` for shadcn Selects; `wait_for_idle`
  instead of polling; `get_element_state` for active/disabled/selected in one read.
- **Batch confirmations — in parallel:** several authored cases in one file are confirmed in ONE
  pytest invocation — `-k "TC-11111 or TC-22222"` **with `-n <min(tests, 8)>`** (pytest-xdist) —
  never one process per test, and never 2+ tests sequentially in one process. Only a single
  test runs without `-n`.
- **Never re-run a case that already passed** in this session.
- **Reuse the ONE visible browser + SSO session** for the whole story. Never restart per case.
- **Zero temp/scratch files** (no `Tee-Object`/`Out-File` logs, `_tmp*` scripts, debug notebooks) —
  diagnostics run in memory; a run's result is the output the blocking terminal call returns.
- **pytest runs are BLOCKING foreground calls** — issue the command, wait for it to return, read
  the returned output (even 10–20 min runs). **Never** redirect the run to a log file and
  `Start-Sleep`-poll it, tail it, check `.Length`, or count chrome processes — redirected logs
  flush at exit, so mid-run polls show stale data and cause hallucinated "stuck run" conclusions.
  Background (`isBackground=true`) is reserved for the ONE whole-scope verify expected to exceed
  ~20 minutes, polled ≤3 times (≥10 min apart) via fresh Allure `*-result.json` mtimes — never
  kill a healthy run, never overlap runs.
- **Build the knowledge graph continuously:** upsert locators/flows/facts the moment they are
  proven live, upsert a story node (`kind="fact", name="Story — <STORY-ID>", tag="story", text="… ref: MCP/stories/<module>/<STORY-ID>/story.md"`) and `kg_link` it to everything this
  story touched; `kg_reindex` once at session end. Full contract:
  [skills/efficiency_and_hygiene/SKILL.md](../skills/efficiency_and_hygiene/SKILL.md).

## Inputs (the only things I provide)

- **Story path:** ${input:storyPath:path to the story, e.g. /CMO/AMCC-5128/story.md}
- **Test cases to execute:** ${input:testCaseRange:"one by one all" (default), or "one by one TC01-TC06", or a count}

> `testcases.json` (under the same story folder) is the source of truth for execution;
> `testcases.md` is the human-readable mirror. "one by one" requests the incremental
> execute → validate → author cadence below.

## Step 0 — Read the knowledge brain + story + cases + existing code

Before planning or touching the browser, read — in this order:

0a. **KNOWLEDGE GRAPH (GRAPH FIRST — recall before probing).** Call `kg_stats` once; then
`kg_recall` (module = e.g. `CMO`) to load the **descriptive memory chunks** previous
sessions wrote (module purposes, page behaviours, gotchas) and `kg_get_module` for the
locator digest. Then, on EVERY screen you arrive at, call `kg_get` (or `kg_search` for
one element) **before** `snapshot_dom` or any live probing:

- A **USE recommendation** (effective confidence ≥ 0.6, no `last_failed`) → use that
  locator directly. No re-probing, no page_source pulls.
- A **miss / VERIFY / PROBE** → probe live as usual; every successful fast-tool call is
  auto-captured back into the graph.
- When you PROVE a non-obvious behaviour live (needs native JS click, dialog stays open,
  typeahead timing, i18n-payload false positives …) persist it with `kg_upsert`
  (kind=`fact`/`action`/`flow`) so the next walk never rediscovers it.
- **Write descriptive memory as you learn (mandatory, once per screen/feature):** call
  `kg_add_knowledge` with a stable `topic` and a 2–10 sentence prose chunk describing
  what the module/page DOES, what an action is for, or a business rule you confirmed.
  Same topic on the same node = update in place, so knowledge converges session by
  session. Link cross-node insight with the `related` ids. This prose memory + the
  locators are what keep future sessions from hallucinating.
  The graph lives as markdown nodes under `MCP/knowledge_graph/knowledge/` — it is
  VERIFIED knowledge, not documentation; it wins over guesses but the **live DOM still
  wins over a stale graph entry** (then record the drift with `kg_record_failure`).

0. **`.agents/context/<module>.context.md` — the module knowledge brain (READ FIRST).** It gives you
   the module's purpose, navigation, every feature, the domain vocabulary, the canonical flows,
   and the business rules, so you walk the app with **intent** and never hallucinate. It is
   **locator-free** by design — it tells you _what_ you're looking for and _why_, never _which
   selector_. If it's missing, create it from `.agents/skills/_template_context/SKILL.md` by mining the
   module's real code (read-only) first.
1. `MCP/stories/<module>/<STORY-ID>/story.md` (acceptance criteria + business context).
2. `MCP/stories/<module>/<STORY-ID>/testcases.json` (the ordered steps to execute) and
   `testcases.md` (the readable mirror).
3. The module's **existing** automation, to extract the exact coding style and reuse what
   exists (NEVER invent a name/pattern): the closest sibling step-def file
   `tests/step_defs/apps/<Module>/test_<module>-*.py`, the Page Object(s) in
   `object_repository/pages/Apps/<Module>/`, the web_component
   `business_components/web_components/<module>.py`, the feature file(s) in
   `tests/features/Apps/<Module>/`, and `testData/<module>.json`.
4. The repo's **canonical authoring prompts** at the framework root — your rulebook for the
   AUTHOR stage: `feature_file_prompt.txt`, `xpath_prompt.txt`, `step_def_prompt.txt`.

Derive **`<module>`** from the story path (first segment, e.g. `/CMO/AMCC-5128/story.md` → `CMO`).
The **live DOM always wins** on any locator conflict.

## Step 1 — PLAN (build the per-test-case execution plan first)

Run the PREPARE phase. It reads `testcases.json`, scans for upload-file tokens, and writes a
**1:1 per-test-case** `mcp-execution-plan.md`:

```
python MCP/mcp_execute_and_author.py --story "${input:storyPath}" --cases "${input:testCaseRange}"
```

Then **open** `MCP/stories/<module>/<STORY-ID>/mcp-execution-plan.md`. It is your script: a
detailed, step-by-step MCP plan **per test case**, in order. Do **not** start executing until
this plan exists and you've read it.

PREPARE also writes `MCP/stories/<module>/<STORY-ID>/execution-progress.md` — your **memory
ledger**. Open it side-by-side and **tick each box the moment it's true** as you execute and
author, so a long one-by-one run never loses its place, repeats a step, or skips one.

## Step 1.5 — COVERAGE-CHECK (prove 100% story coverage — or report every gap)

**Before executing anything**, write
`MCP/stories/<module>/<STORY-ID>/coverage-report.md` with this exact structure:

```markdown
# Story Coverage — <STORY-ID> (<Module>)

> Every acceptance criterion / feature of story.md mapped to the test case(s) that
> verify it. A story feature with no covering case is a COVERAGE GAP — reported,
> never silently ignored.

| #   | Story feature / acceptance criterion (verbatim from story.md) | Covered by                   | Status         |
| --- | ------------------------------------------------------------- | ---------------------------- | -------------- |
| 1   | <AC text>                                                     | AMCC-TC-XXXXX, AMCC-TC-YYYYY | ✅ COVERED     |
| 2   | <AC text>                                                     | —                            | ❌ NOT COVERED |

## Coverage verdict

- **Covered:** N of M story features (NN%)
- **NOT COVERED (gaps):**
  - <feature> — no test case exercises this; suggest a new case: "<1-line proposed case>"

## Orphan test cases (cases that map to no story feature — possible stale/extra scope)

- <CASE-ID> — <why it doesn't map>
```

Rules for the analysis:

- Extract the acceptance criteria / feature list **verbatim** from `story.md` — do not invent,
  merge, or paraphrase away criteria.
- Map by **behaviour**, not by keyword overlap: a case covers a criterion only if one of its
  steps' expected results actually verifies that behaviour.
- **Every** gap must appear in the final report to the user as
  `"Feature X of story <STORY-ID> is NOT covered by any test case"` — with a 1-line suggested
  case. Do not proceed silently past a gap; note it, then continue with the cases that exist.
- If `story.md` has no explicit AC section, derive the feature list from its Description +
  Business Context and say so at the top of the report.
- **Update the persistent traceability matrix** `MCP/stories/traceability-matrix.csv`
  (one row per story×criterion — append/update, never delete; schema + status values in
  [../coverage_traceability/SKILL.md](../skills/coverage_traceability/SKILL.md)). A criterion whose
  cases test only the happy path is `PARTIAL`, not COVERED.
- **Design-technique pass:** for every NOT COVERED / PARTIAL criterion, the suggested case names
  the technique it needs (negative / boundary / equivalence / state-transition) — see the
  conditions table in ../coverage_traceability/SKILL.md §4.

## Step 2 — Launch the visible browser via the Selenium MCP + authenticate

> Use the **`selenium`** MCP server (`selenium-mcp-server`, launched via
> `MCP/selenium_mcp_launch.py`). If it isn't in the tool picker, install it
> (`pip install -r MCP/requirements-mcp.txt`), copy the `servers` block from
> `MCP/mcp.servers.json` into `.vscode/mcp.json`, and reload VS Code. The launcher exposes
> **20 base tools + 12 fast tools** (`login_sso`, `fill_form`, `click_sequence`, `wait_and_click`,
> `select_radix_option`, `get_texts`, `snapshot_dom`, `assert_text`, `screenshot_step`,
> `scroll_into_view_and_click`, `get_element_state`, `wait_for_idle`).

1. `start_browser` → `{ "browser": "chrome" }` (full mode, NOT headless — you watch it). The
   launcher auto-launches Chrome at **75% zoom** + `--window-size=1920,1080` (matching the
   pytest-bdd runtime in `tests/step_defs/conftest.py`).
2. `navigate` → `APP_BASE_URL` (e.g. `https://qa.americoldcompass.com/`).
3. Complete the **Microsoft / Azure AD SSO** login in **one call** with the **`login_sso`** fast
   tool (`{ "url": APP_BASE_URL, "username": …, "password": … }`). Credentials come from `.env`
   (`QAAdminEmail` / `QAAdminPassword`) — **never hardcode**. Confirm you land on the app shell.

## Step 3 — The ONE-BY-ONE loop (execute → validate → document defects → author → verify)

For **each** test case in `mcp-execution-plan.md`, **in order — do NOT batch**. Finish the whole
case (A–E) before you touch the next case. **Update `execution-progress.md` as you go.**

**A. Execute it live (visible), one step at a time.** Walk the case's steps **in order**. For
each step, drive the visible browser with the `selenium` MCP tools (`navigate`, `click_element`,
`send_keys`, `press_key`, `hover`, `upload_file`, `execute_script`, `wait_for_element` — prefer
the fast tools). Reach each screen the way a user would. Do **not** jump ahead — complete and
verify a step before the next. **Take NO screenshots of steps that behave as expected.**

**B. Validate against the live DOM — and record it honestly.** Read the resulting state with the
MCP (`get_element_text`, `get_element_state`, `get_texts`, `get_page_info`, `assert_text`,
`execute_script`). For Radix Selects open the listbox and read each option's text; for dialogs read
the title/buttons; for uploads read the attached-file row. **Compare what you observe to
`step.expectedResult`** and write the result (status + actual + any bug) into
`TestcaseExecution/<module>/<STORY-ID>/execution-results.json`.

**C. On a genuine defect — document it formally (this is the ONLY time you screenshot).**
When the live app **contradicts** the expected result and you have ruled out your own mistake
(wrong navigation, missing precondition/data you were supposed to set up, wrong test data):

1. **Capture the defect state** — one or more screenshots of the exact screens that prove it:
   `screenshot_step(module="<module>", story_id="<STORY-ID>", case_id="<CASE-ID>", step=1, defect=true)`
   → `TestcaseExecution/<module>/<STORY-ID>/<CASE-ID>/<CASE-ID>_defect_1.png` (increment `step`
   for each additional defect shot).
2. **Write the structured payload** `TestcaseExecution/<module>/<STORY-ID>/<CASE-ID>/<CASE-ID>_defect.json`
   following [skills/defect_documentation/SKILL.md](../skills/defect_documentation/SKILL.md):
   `title` ("QA - : <concise defect summary></concise>"), `description`, `prerequisites[]`,
   `stepsToReproduce[]` (numbered, human-replayable), `expectedResult`, `actualResult`,
   `screenshots[]`, `evidence` (the live-DOM/API proof), `severity`.
3. **Render the formal defect document:**
   ```
   python MCP/build_defect_docx.py --story "${input:storyPath}" --case <CASE-ID>
   ```

   → `TestcaseExecution/<module>/<STORY-ID>/<CASE-ID>/<CASE-ID>_defect.docx`
4. Keep the step **`FAIL`** with a `bug` note in `execution-results.json` — do **NOT** bend the
   test to pass and **never fake a pass**. Then continue with the case's remaining steps if
   they are still reachable, or mark the rest `BLOCKED (by defect)` and move to the next case.

> A "serious defect" is one where the app's actual behaviour breaks the story's acceptance
> criteria, loses data, blocks a user flow, or shows wrong data. Cosmetic drift you are unsure
> about → record it as a `bug` note in the ledger and flag it in the report, but ask before
> raising a formal defect doc.

**D. Author & SAVE this case immediately (before the next case).** Author it in the repo's
**4-layer architecture**, matching the module's existing style exactly, and following the repo's
canonical authoring prompts in order — `feature_file_prompt.txt` (feature) → `xpath_prompt.txt`
(POM locators) → web_component design → `step_def_prompt.txt` (step defs + assertions + Allure) —
plus [skills/authoring_bdd_from_mcp/SKILL.md](../skills/authoring_bdd_from_mcp/SKILL.md):

- **Feature** → `tests/features/Apps/<Module>/<module>-<STORY-ID>.feature`: append **one
  `Scenario` (or `Scenario Outline`)** titled `<CASE-ID> <summary>`, tagged
  `@story-<STORY-ID> @<module> @<CASE-ID> @regression` (match the module's existing tags), with a
  `Background: Given login to the Cold Compass application`. Each QMetry step becomes a plain
  business-language `When`/`Then`. **No locators, no data, no implementation detail in Gherkin.**
- **Page Object** → `object_repository/pages/Apps/<Module>/<Name>Page.py` (`extends CustomPyAutoWeb`): add only **genuinely-new** locators as `locatorXxx = [(By.XPATH, "...")]`
  lists (reuse existing ones first), baking in the anchors you observed live — role / text /
  `data-testid` first, never hashed/utility Tailwind classes. Locators ONLY; **no assertions**.
- **web_component** → `business_components/web_components/<module>.py`: when an interaction is
  reused by 2+ steps/cases, add a `driver`-first, parameterized, **assertion-free** function that
  drives POM locators via `CustomPyAutoWeb` (reuse/extend existing functions; one function per pattern).
- **Step defs** → `tests/step_defs/apps/<Module>/test_<module>-<STORY-ID>.py`: one step function
  per Gherkin step, byte-identical decorator text. Build `web = CustomPyAutoWeb(bdd_driver)`,
  drive the **same actions you just walked live** via the toolbox + POM locators (or a
  web_component call), and **assert every `@then`** with a `CustomPyAutoWeb` verify (e.g.
  `web.element_should_be_visible(locator=PAGE.locatorX)`) — assertions live ONLY here. Wrap every
  step in `try/except … raise AssertionError("Step FAILED: '<step>' - … [<func>]. Details: {e}") from e`.
  Attach Allure evidence: `@when` → `Action — …`, `@then` → `Verification — …`. Reuse shared steps
  with the `# REUSED FROM:` provenance comment + delegate to the same web_component.
- **Test data** → `testData/<module>.json`: add this case's data key under the story id; **no
  inline literals** in steps.
- **A case with a confirmed defect is still authored** — the test asserts the STORY's expected
  behaviour, so it fails against the buggy app. That failing test + the defect .docx ARE the
  deliverable for that case. Never write the test to match the buggy behaviour.

**E. VERIFY this one case, then continue.**

```
python MCP/mcp_execute_and_author.py --story "${input:storyPath}" --verify --case <CASE-ID>
pytest "tests/step_defs/apps/<Module>/test_<module>-<STORY-ID>.py" -k "<CASE-ID>" --alluredir=reports/allure-results
```

The `--verify --case` gate proves that case's `Scenario` + tag + step-def binding are saved. If a
path is **DATA-BLOCKED** (e.g. a facility has no CMO config so the button never appears), follow
the module's existing self-heal/skip convention — never a guessed locator. Then move to the next case.

## Step 4 — RUN the full story + report

When every case is saved, run the whole story and confirm green:

```
python MCP/mcp_execute_and_author.py --story "${input:storyPath}" --verify
pytest "tests/step_defs/apps/<Module>/test_<module>-<STORY-ID>.py" -n 8 --alluredir=reports/allure-results
```

Then build the **Word reports**:

```
python MCP/build_defect_docx.py --story "${input:storyPath}" --all       # every documented defect → <CASE-ID>_defect.docx
python MCP/build_execution_docx.py --story "${input:storyPath}"          # the rolled-up <STORY-ID>_Execution.docx
```

→ defects: `TestcaseExecution/<module>/<STORY-ID>/<CASE-ID>/<CASE-ID>_defect.docx` (one per
documented bug — Title / Description / Prerequisites / Steps to Reproduce / Expected Result /
Actual Result + embedded screenshots); story: the rolled-up execution report (statuses + bug
notes; defect screenshots embedded where a bug was found).

**Clean up the run's scratch files.** After the story run is green and the reports are built
(and after any Step 5 self-heal re-runs), remove **only** caches + temp scratch — never the real
Allure `*-result.json` data, the defect evidence, or the `.docx`:

```powershell
Remove-Item -Recurse -Force .pytest_cache, reports\report_dates\*\.pytest_cache -ErrorAction SilentlyContinue
Get-ChildItem reports -Recurse -File -Include *.tmp,*.temp,~*,_tmp* -ErrorAction SilentlyContinue | Remove-Item -Force -ErrorAction SilentlyContinue
```

> `MCP/stories/…`, `TestcaseExecution/…`, and `MCP/.env.mcp` are git-ignored, so the fetched
> stories, defect evidence, `.docx` reports, and MCP secrets stay local and are never pushed.

## Step 5 — SELF-HEAL any case that passed live but fails as a test

For each scenario that **failed** in the Step 4 run even though it **passed during the live MCP
walk**, it is almost always **test-infrastructure drift** (a locator/timing/state difference) —
not a product bug. Fix it via the MCP:

> **Healing a whole Allure run** (not just a freshly-authored case) has its own dedicated
> pipeline — the **`/heal-tests`** prompt + `MCP/mcp_heal_tests.py` + [skills/heal_tests/SKILL.md](../skills/heal_tests/SKILL.md).

1. **Re-drive that exact case** through the `selenium` MCP, comparing the live DOM to what the
   step/POM does at the failing step.
2. **Find the drift** — a wrong/stale locator, a missing `web.wait_element_visibility(...)`, or a
   read at the wrong moment. Re-resolve the correct role/text/`data-testid` locator live.
3. **Fix at the source** — patch **only** the broken locator/wait in the Page Object (or the
   step). Never touch the assertion/intent; never add `time.sleep`.
4. **Re-run only that case** (budget: max 2 runs) and confirm it matches the live behaviour.
5. If instead the **app itself** behaves differently from `expectedResult` (a real assertion
   mismatch), that is a **genuine product bug** — document it with the full defect flow (Step 3.C:
   defect screenshots + `<CASE-ID>_defect.json` + `<CASE-ID>_defect.docx`), leave the test
   failing, and do **NOT** loosen the assertion. Log every heal in
   `MCP/stories/<module>/<STORY-ID>/healing-log.md`.

## Step 6 — Report

Report, in this order:

1. **Coverage verdict** — N of M story features covered; every **NOT COVERED** feature listed
   verbatim ("Feature X of story <STORY-ID></story> is NOT covered by any test case") + suggested case.
2. **Per-case pass/fail** table.
3. **Defects documented** — each with its title, severity, and the
   `TestcaseExecution/<module>/<STORY-ID>/<CASE-ID>/<CASE-ID>_defect.docx` path.
4. Which cases needed MCP self-healing and what drifted.
5. The upload fixtures used, the `<STORY-ID>_Execution.docx` path, and the Allure report path.

## Hard rules

- **Repo facts (NOT generic Behave):** pytest-bdd; the **`bdd_driver`** fixture (no `context`);
  `web = CustomPyAutoWeb(bdd_driver)`; POM locator **lists** `[(By.XPATH, "...")]`; cross-step
  state via module-level `_ctx = {}` + web*component `get*`/`set\_`globals; every step wrapped in`try/except … raise AssertionError("Step FAILED: …")`.
- **One layer per concern.** Locators ONLY in the POM. Assertions ONLY in `@then` step defs.
  Reusable interactions ONLY in the web_component (assertion-free, locators passed in). Gherkin is
  plain business language.
- **Ground every locator live.** Resolve genuinely-new locators from the live DOM through the
  `selenium` MCP and bake them into the POM — role / text / `data-testid` first, hashed/utility
  Tailwind classes never. Reuse existing POM locators/methods before resolving new ones. If the
  MCP can't be driven, **STOP and report "blocked on MCP"** — never guess a locator from memory.
- **Coverage first.** `coverage-report.md` exists and every gap is called out BEFORE the first
  live step. 100% coverage is proven or its absence reported — never assumed.
- **Plan first.** Build & read `mcp-execution-plan.md` before any live execution.
- **One by one, fully.** Finish a case completely — every step validated live — then **author its
  test**, before the next case.
- **NO per-step screenshots.** Passing steps produce zero PNGs. Evidence is captured ONLY for
  genuine defects: `screenshot_step(…, defect=true)` → `<CASE-ID>_defect_<N>.png` + the
  `<CASE-ID>_defect.json` payload + the rendered `<CASE-ID>_defect.docx`.
- **Every serious defect gets its formal .docx** — Title / Description / Prerequisites / Steps to
  Reproduce / Expected Result / Actual Result + embedded screenshots — at
  `TestcaseExecution/<module>/<STORY-ID>/<CASE-ID>/<CASE-ID>_defect.docx`, built with
  `python MCP/build_defect_docx.py`. A bug without its defect doc is an unreported bug.
- **Record honestly + report.** Write each step's status/actual/bug to
  `TestcaseExecution/<module>/<STORY-ID>/execution-results.json`, then build the
  `<STORY-ID>_Execution.docx` at the end.
- **Go fast.** Prefer the fast tools (`login_sso`, `fill_form`, `click_sequence`,
  `select_radix_option`, `snapshot_dom`, `wait_for_idle`, `get_element_state`, …) over many
  single-action calls; max 2 pytest runs per case; one whole-story run at the end.
- **No `time.sleep`** — use `web.wait_element_visibility(...)` / `web.wait(n)`. **No hardcoded
  data** — `testData/<module>.json` / `parsers.parse` params. **No raw `bdd_driver.find_element`
  in steps** — go through `CustomPyAutoWeb` + a POM locator.
- **Never fake a pass.** A live contradiction is a bug/stale story (document it); a DATA-BLOCKED
  path is skipped per the module convention.
- **1:1 fidelity to `testcases.json`.** One `Scenario` per case, one `When`/`Then` per step, each
  `@then` asserts that step's `expectedResult`.
- **Match existing conventions exactly.** Never invent a name, import order, or pattern not
  already present in the module.
- **Clean up after the final run** (caches + `*.tmp` scratch only — never results, evidence, or docx).

## Example invocation

```
/mcp-execute-and-author
Story path → /CMO/AMCC-5128/story.md
Test cases to execute → one by one all
```
