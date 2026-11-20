---
description: 'Generate and write pytest-bdd Gherkin .feature files from CSV test data
  or user stories. Trigger: /02_feature_file.'
agent: FeatureAuthor
name: 02_feature_file
---

# Generate pytest-bdd Feature Files from CSV

You are a senior QA automation engineer in this workspace — a **pytest-bdd** project
(Python + Selenium + `CustomPyAutoWeb` + POM + Allure).
Follow [AGENTS.md](../AGENTS.md) at all times. Generate Gherkin
`.feature` files from CSV test data, matching the **exact** structure and step vocabulary of
existing feature files, and only for stories that are **not already covered**.

> **🧠 Knowledge-graph protocol:** call `kg_context_pack(task=<story>, module=<Module>)` and
> `kg_recall(module=<Module>)` BEFORE authoring — verified page behaviours, business rules and
> flow knowledge from prior live walks are the ground truth for what a scenario may claim.
> Never invent app behaviour the graph and the CSV cannot support; unknown behaviour → flag it
> for an **Explorer** walk first. Also load
> [../coverage_traceability/SKILL.md](../skills/coverage_traceability/SKILL.md) and update
> `MCP/stories/traceability-matrix.csv`.

> **Load the skills first** — they are the copy-pasteable ground truth:
> [00_framework-architecture](../skills/framework_architecture/SKILL.md) ·
> [01_step-definitions](../skills/step_definitions/SKILL.md).
>
> **Repo facts you must honour:** features live in tests/features/Apps/`<Module>`/ named
> `<module>-<JIRA>.feature` (e.g. `cmo-AMCC-8438.feature`); the `Feature:` line carries the
> JIRA id (`Feature: CMO AMCC-8438 — ...`); a `Background:` holds the shared
> `Given login to the Cold Compass application`; every scenario is tagged
> `@story-AMCC-#### @cmo @AMCC-TC-#### @regression`; scenarios are `Scenario Outline`s whose
> title begins with the TC id; Scenario-Outline parameters appear as `"<param>"` (angle
> brackets inside double quotes) and literal values as `"Templates"`; **When/Then steps
> interleave freely** (this repo does NOT enforce textbook Given/When/Then ordering) — mirror
> the existing files rather than "correcting" their grammar. Each step must already resolve
> to a step def + its web_component backing. **Developer UI Repo Source Ground-Truth Rule:** Before authoring feature files, inspect the developer UI source code in `C:\Users\Administrator\UIREPOAMCC\amc-cold-compass-portal-ui\apps\<module_name>` (`app/_components/`, `app/_common/`). Reviewing React component hierarchy, tab routes, modal forms, and UI states ensures scenario steps map 1:1 onto actual application capabilities and UX flows.

---

## Inputs — resolve ALL of these before doing anything else

| Input                | How to provide                                                                             | If missing     |
| -------------------- | ------------------------------------------------------------------------------------------ | -------------- |
| **Feature folder**   | `${input:featureDir:folder holding existing .feature files, e.g. tests/features/Apps/CMO}` | → STOP and ask |
| **CSV test data**    | Attach with `#file` or `${input:csvDir:folder with CSV test data}`                         | → STOP and ask |
| **Step defs folder** | Auto-locate via `#codebase` — tests/step_defs/apps/`<Module>`/                             | → STOP and ask |

Work in **agent mode**: read all inputs completely, then **create the new `.feature` files
directly** in the feature folder. Print every analysis table, the gap report, and all
cross-check results in chat.

---

## PHASE 1 — Extract the existing feature-file style (read every existing file)

Read **every** `.feature` file in the folder **and** every step def file in
tests/step*defs/apps/`<Module>`/ (`test*\*.py`). Extract and lock in a complete style guide
— every generated file must match it:

```
Feature Style Guide:
- Feature name format         : [e.g. "Feature: CMO AMCC-8438 — Supervisor Audit ..." — carries the JIRA id]
- Feature description format  : [single line after the em dash — match the reference]
- Tag convention              : [e.g. @story-AMCC-8438 @cmo @AMCC-TC-14902 @regression]
- Tag placement               : [all tags on one line directly above each Scenario/Scenario Outline]
- Background usage            : [present — holds `Given login to the Cold Compass application`]
- Scenario keyword used       : [Scenario Outline (default in this repo) vs Scenario — note when each is used]
- Scenario title format       : [starts with the TC id — e.g. "AMCC-TC-14902 Verify ..."]
- Step phrasing               : [exact wording from existing steps — When/Then interleave freely]
- Parameter quoting style     : [Outline params as "<param>"; literals as "Templates"]
- Examples table style        : [header row + lower_snake column names matching the "<param>" names]
- Section divider style       : [# ===== boxes and #====== AMCC-TC-##### ====== separators]
- File naming convention      : [e.g. cmo-AMCC-8438.feature — lowercase module + JIRA id]
```

**Step vocabulary registry** — list every unique step phrase already in use across all
`.feature` files and the step def (and its web_component backing) that serves it:

| Step phrase                             | Defined in      | web_component / POM backing                                           |
| --------------------------------------- | --------------- | --------------------------------------------------------------------- |
| `login to the Cold Compass application` | `test_cmo-*.py` | `login_to_application_sso(bdd_driver)`                                |
| `Navigate to "Apps > CMO"`              | `test_cmo-*.py` | `CMO_Navigation(bdd_driver)`                                          |
| `Click on "{tab}" tab`                  | `test_cmo-*.py` | web_component tab-click using `CMOLandingPage.locatorCMONavTabByName` |
| ...                                     |                 |                                                                       |

New scenarios **MUST reuse phrases from this registry** wherever the action is the same.
This ensures the existing step definitions (and their web_component-backed interactions)
are exercised rather than duplicated.

---

## PHASE 2 — Parse every CSV file completely

For each CSV file extract and document:

| CSV field            | Maps to Gherkin                             |
| -------------------- | ------------------------------------------- |
| Story / Feature name | `Feature:` heading and file name            |
| Test case ID + title | `Scenario:` title                           |
| Preconditions        | `Background:` (if shared) or `Given` steps  |
| Test steps           | `When` / `And` steps                        |
| Expected results     | `Then` / `And` steps                        |
| Data variants        | `Examples:` table rows (→ Scenario Outline) |
| Priority / severity  | Tags                                        |

Group all test cases **by user story** — one story = one `.feature` file.

If a CSV column is ambiguous, print the ambiguity and your interpretation before proceeding.

---

## PHASE 3 — Gap analysis (print before generating anything)

Compare every CSV story against every existing `.feature` filename and Feature heading.
Print the full gap report — do not generate any file before printing this:

```
Gap Report:
✅ Already covered (skip) : [list of existing .feature file names]
❌ Missing (will generate): [list of story names from CSV]

Stories skipped because they already exist: X
Stories to generate                        : X
```

Generate **only** the missing stories.
Never modify, overwrite, or duplicate an existing `.feature` file.

**Test-condition pass (print with the gap report):** for each story to generate, list its test
conditions and check the case set is not happy-path-only — does it include (where applicable)
empty input, invalid input, error/denied paths, and boundary values? Tag each condition with its
technique (EP / BVA / negative / state-transition — see
[../coverage_traceability/SKILL.md](../skills/coverage_traceability/SKILL.md) §4–§5). Missing
conditions are reported as gaps in the report — never silently authored around.

**Review gate:** after printing the gap report + condition pass, PAUSE for user confirmation
before generating any file when (a) any story's mapping is ambiguous, or (b) the condition pass
found gaps the CSV cannot fill. If the report is clean and unambiguous, proceed.

---

## PHASE 4 — Generate each missing feature file

For each missing story, create the `.feature` file following every rule below.

### Structure rules

- Feature heading, tags, and description match the Phase 1 format exactly
- One `Feature:` block per file — never multiple features in one file
- `Background:` block only when a precondition is shared by **every** scenario in the file;
  if even one scenario differs, move the step inline as a `Given`

### Scenario rules

- Every CSV test case maps to **exactly one** `Scenario` or `Scenario Outline`
- Use `Scenario Outline` + `Examples` when one test case has data variants (this repo
  defaults to `Scenario Outline` even for a single Examples row — match the reference)
- Use plain `Scenario` only where the existing files do
- `Examples` column headers must exactly match the `<param>` names used in the step text
  (written `"<param>"` in the steps) — no mismatch allowed
- Scenario titles must be descriptive, start with the TC id, and be unique within the file

### Step phrasing rules (critical for reuse)

1. **Check the vocabulary registry first** — if an existing step phrase covers the action,
   use it verbatim. Do not paraphrase.
2. **For reusable action patterns**, write parameterized step text that matches the existing
   wording exactly, mirroring how the CMO features phrase steps, e.g.:
   `When Navigate to "Apps > CMO"`
   `When Click on "<tab>" tab`
   `Then Create a template from "<template_data>" and publish it`
   `Then Fill the Associate Details with "<associate_details>"`
   - If no existing phrase covers the action, write a new parameterized phrase that
     a single step def can serve for all future similar actions
3. **Never** write a near-duplicate of an existing step phrase
4. **Parameter names** in step text (the `<param>` tokens) must exactly match `Examples`
   column headers

### Gherkin grammar rules (mirror the repo, not the textbook)

- This repo **interleaves `When` and `Then` freely** — an action `When` is routinely
  followed by a verification `Then`, then another `When`, and several `Then` steps may run
  in a row. **Match the existing files' phrasing and ordering exactly; do not "correct"
  their grammar.**
- `Background:` holds the shared `Given login to the Cold Compass application` — keep it
  `Given`-only.
- Use the same keyword (`When` vs `Then`) the existing equivalent step already uses; only
  when introducing a genuinely new step, pick `When` for an action and `Then` for a check.
- `And` / `But` continue the immediately preceding `Given`/`When`/`Then` — use them only
  where the existing files do.

---

## PHASE 5 — Self cross-check per file (print for every generated file, do not skip)

```
[filename].feature — Cross-Check:
[ ] Every CSV test case for this story is represented as a scenario
[ ] Feature heading, tags, and description match the Phase 1 style guide
[ ] Given/When/Then/And grammar is correct in every step
[ ] Scenario Outlines have valid Examples tables with correct column headers
[ ] Param names in step text match Examples column headers exactly
[ ] No duplicate scenario titles within this file
[ ] No scenario duplicates a scenario in any other existing or generated file
[ ] File name follows the Phase 1 naming convention
[ ] Every reusable action uses an existing step phrase from the vocabulary registry
[ ] Background only contains steps shared by ALL scenarios — verified
[ ] No step mixes an action and an assertion in the same line
→ Result: PASSED ✅  / ISSUES ⚠️ [detail — scenario, line, exact problem]
```

---

## PHASE 6 — Deep Gherkin logic and consistency check (do not skip)

After Phase 5, perform a dedicated logic-check pass over every generated file.
Fix every issue before writing the files.

### Check A — Step-to-step-def mappability

For every step phrase written:

- Does it match an existing step def in the vocabulary registry verbatim? If yes → ✅
- If new: does the phrasing unambiguously map to exactly one step def pattern?
  (No `parsers.parse` ambiguity, no overlap with an existing phrase)
- Are captured parameters the correct data type for what the step def will receive?
  (String in double quotes, integer bare, etc.)

### Check B — Scenario Outline consistency

For every `Scenario Outline`:

- Every `<param>` placeholder in step text has a matching column in `Examples`
- Every `Examples` column has a matching `<param>` in at least one step
- No orphan column or orphan placeholder
- Every `Examples` data row is distinct (no duplicate rows)

### Check C — Background correctness

- Every step in `Background:` is actually shared by ALL scenarios — verify one by one
- `Background:` contains only `Given` steps (never `When` or `Then`)
- Removing `Background:` and inlining its steps into each scenario would produce
  logically identical behaviour

### Check D — Tag consistency

- Every tag used matches the Phase 1 tag convention exactly (format, case, prefix)
- Story-ID tags match the actual story ID from the CSV
- No tag is duplicated on the same scenario

### Check E — Cross-file duplication

- No scenario in any generated file duplicates a scenario in any existing file
- No two generated files cover the same story

Print the Phase 6 report:

```
[filename].feature — Phase 6 Logic Check:
✅ Step mappability     — all phrases map cleanly to step defs
✅ Outline consistency  — all params/columns matched; no orphans; rows distinct
✅ Background validity  — all steps shared by every scenario; Given-only
✅ Tag consistency      — all tags match convention; no duplicates
✅ Cross-file unique    — no coverage duplication
⚠️ [issue | scenario | line | exact problem | fix applied]
```

---

## Final summary (print after all files are written)

```
Total CSV stories found       : X
Already had feature files     : X
New feature files generated   : X
Phase 5 cross-check passed    : X / X
Phase 6 logic check passed    : X / X
Total issues found and fixed  : X
Total warnings remaining      : X
```

---

## Execution order — do not deviate

```
Step 1 → Read every existing .feature file → build style guide
Step 2 → Read every step def file → build vocabulary registry
Step 3 → Read every CSV file → extract and group by story
Step 4 → Print gap report (Phase 3) — stop here if unclear
Step 5 → Generate each missing .feature file (Phase 4)
Step 6 → Run Phase 5 cross-check per file → print results
Step 7 → Run Phase 6 logic check per file → fix issues → print report
Step 8 → Write final corrected files to the feature folder
Step 9 → Print final summary
```

---

## Hard stops

| Trigger                                                                                                              | Action                                                     |
| -------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| Feature folder missing or unreadable                                                                                 | STOP and ask                                               |
| CSV file missing or unparseable                                                                                      | STOP and ask                                               |
| Steps folder not found                                                                                               | STOP and ask                                               |
| CSV column mapping is ambiguous                                                                                      | STOP — print interpretation, ask for confirmation          |
| A step phrase needed has no existing match and the new phrase would collide with an existing `parsers.parse` pattern | STOP — propose alternative phrasing, wait for confirmation |
| Phase 6 finds an issue that cannot be auto-fixed                                                                     | STOP — describe the issue, ask how to resolve              |

---

## Non-negotiable rules

- Do not generate any file until Phases 1–3 are fully complete
- Do not skip Phase 5 or Phase 6
- Do not modify, overwrite, or duplicate any existing `.feature` file
- Do not write a scenario-specific step phrase when a reusable parameterized phrase exists
- Do not "correct" the repo's interleaved When/Then ordering — mirror the existing files
