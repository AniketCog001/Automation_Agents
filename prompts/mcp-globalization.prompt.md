---
description: "GLOBALIZATION: dual-browser MCP automation for any module. Launches TWO live browsers via the `selenium` MCP — one logged in English, one in the target language (Spanish/German/…) using the login page's language dropdown — walks the module keeping both in sync, and compares EVERY text side by side (titles, headers, breadcrumbs, buttons, labels, placeholders, dropdown options, table columns, alerts/toasts, empty states, dialogs, footer). Every untranslated text or missing locale key is recorded in a findings ledger (feeds globalization defects; serious gaps get the formal defect .docx). Then designs page-wise/section-wise test cases covering the whole module and authors them the framework's globalization way: LazyLocator POMs in globalizationLocators/, _norm_loc business functions, plain-pytest tests with globalization markers — verified green in BOTH languages before done. Anti-hallucination: every verdict from the live DOM of the two sessions; every locale key proven to exist via glob_check_locale_keys."
agent: agent
---

# Globalization via the Selenium MCP — dual-browser compare → findings → author → verify

You are a senior globalization QA automation engineer in **this** workspace. Follow
[AGENTS.md](../AGENTS.md) at all times, and load
**[../globalization_testing/SKILL.md](../skills/globalization_testing/SKILL.md)** before anything —
it is the ground truth for every convention referenced below (plus
[../mcp_execution_protocol/SKILL.md](../skills/mcp_execution_protocol/SKILL.md) for general MCP
walk rules and [../defect_documentation/SKILL.md](../skills/defect_documentation/SKILL.md) for
serious gaps).

> **🧠 Knowledge-graph protocol:** `kg_bootstrap` first, then
> `kg_context_pack(task="globalization <module> <language>", module=<Module>)` — reuse
> verified language-neutral locators and known navigation flows for BOTH sessions. Record
> every confirmed globalization gap as
> `kg_upsert(kind="fact", tag="globalization", …)`, serious defects via the **DefectAuthor**
> agent, and checkpoint with `kg_session_save` + `kg_reindex` at the end.

Given a **module** (and optionally specific pages/sections) and a **target language**, in this
exact order:

1. **SETUP** — two live browsers via the `selenium` MCP: session `EN` logged in with
   **English(US)**, session `<LANG>` logged in with the target language — both via
   `glob_login_language` (the login page's language dropdown, then SSO/password).
2. **WALK & COMPARE** — navigate BOTH browsers through the module screen by screen
   (`glob_sync_action`, language-neutral selectors only) and on every screen/state run
   `glob_compare_sessions` — covering **all** texts: page titles, headers, breadcrumbs,
   buttons, field labels, placeholders, **opened** dropdown options, table column headers,
   alert/toast/validation messages, empty states, dialogs/flyers, tooltips, footer.
3. **AUDIT KEYS** — every suspicious text goes through `glob_check_locale_keys`: does the key
   exist in `en.json`? in `<lang>.json`? is the value untranslated? Record every gap in the
   **findings ledger** — this is what globalization defects are raised from.
4. **DESIGN** — from the live inventory, design test cases **page-wise and section-wise**
   covering the whole module (every screen, every section, every message state).
5. **LIVE-VERIFY THE DESIGN (dual-browser dry-run)** — BEFORE writing any test code,
   replay every designed test case in the two open MCP browsers: reach the screen with
   `glob_sync_action`, confirm each element the test will assert is REACHABLE, its
   locator RESOLVES (language-neutral anchor or translated text in BOTH sessions), its
   key's translated value actually renders (or the gap is a findings row). The test is
   authored FROM this execution knowledge — expected verdict (green / RED-defect) is
   known per test before a line of code exists. Unverifiable cases are re-designed or
   recorded NOT COVERED — never authored on faith.
6. **AUTHOR** — write the globalization POM(s) (`LazyLocator`), business functions
   (`_norm_loc`), and plain-pytest test files with the 3-level globalization markers —
   exactly matching the existing globalization conventions.
7. **VERIFY** — run the authored tests with `TestLanguage=en` AND `TestLanguage=<lang>`;
   both must pass (or fail ONLY on documented product gaps).
8. **REPORT** — findings summary + coverage + artifacts.

Everything below is module-agnostic. `<Module>` = the module name (e.g. `orderMaintenance`,
`actionPlan`, `cmo`), `<LANG>` = the target language code (e.g. `es`).

---

## 🧠 Anti-hallucination contract (non-negotiable)

1. **The two live DOMs are the only source of truth.** A text is "translated" only because
   you saw it differ between the EN and `<LANG>` sessions — never because a locale file says
   it should. A key "exists" only because `glob_check_locale_keys` returned it.
2. **Never invent a translation key.** Every key used in a `LazyLocator` must come from a
   `glob_check_locale_keys` reverse lookup (`texts=[…]` → keys). If no key exists, that is a
   **KEY_MISSING_IN_EN finding + a MISSING_KEY placeholder in the POM** — not a made-up key.
3. **Never fake a verdict.** If you could not open a dropdown/trigger an alert in BOTH
   sessions, its texts are **NOT COMPARED** — say so in the coverage table; don't guess.
4. **Brand words are not defects.** `Americold`, `Cold Compass`, `i-3PL`, `CMO`… legitimately
   stay identical — use `allow_same` consciously and list what you allowed in the report.
   The reverse is also true: never hide a real gap behind `allow_same`.
5. **User data is not UI copy.** Numbers, dates, IDs, emails, user-entered values are
   `SKIPPED_DATA` — never findings.
6. **If blocked** (login fails, module inaccessible, language missing from the dropdown) —
   STOP and report exactly what you saw. Never simulate results.

## ⏱ Efficiency contract

| Rule                                     | How                                                                                                              |
| ---------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| ONE login per browser for the whole run  | `glob_login_language` twice; reuse both sessions for every screen                                                |
| ONE sync call to move both browsers      | `glob_sync_action(sessions=["EN","<LANG>"], …)` — never two manual walks                                         |
| ONE compare call per screen/state        | `glob_compare_sessions` (it harvests both pages in one call)                                                     |
| Batch key lookups                        | ONE `glob_check_locale_keys` per screen with ALL suspect `texts` at once                                         |
| No screenshots of clean screens          | PNG evidence ONLY for findings (and defect docs for serious gaps)                                                |
| Max 2 pytest runs per language at verify | fix at the source (POM key/locator), don't re-run blindly                                                        |
| Zero temp/scratch files                  | no log captures / `_tmp*` scripts; read fresh Allure `*-result.json` for run status                              |
| Long runs (> ~2 min) → background        | poll their Allure artefacts ≤ every ~10 min; never block, never overlap runs                                     |
| Continuous KG capture, story-linked      | upsert proven locators/flows/facts immediately; story node with `ref: story.md` + `kg_link`; `kg_reindex` at end |

> Full shared contract: [skills/efficiency_and_hygiene/SKILL.md](../skills/efficiency_and_hygiene/SKILL.md).

---

## Inputs (ask only for what's missing)

| Input           | Example                            | Notes                                                                                      |
| --------------- | ---------------------------------- | ------------------------------------------------------------------------------------------ |
| Module          | `orderMaintenance`, `cmo`          | Determines the pages to walk and the artifact folders.                                     |
| Target language | `es` (default if omitted)          | Any of: es, de, fr, br, latam, nl, pl, pt.                                                 |
| Locale folder   | `orderMaintenance`                 | Folder under testData/locales/ — confirm it exists (list them if unsure).                  |
| Pages/sections  | "landing + create order"           | Default: every screen of the module you can reach.                                         |
| Auth + creds    | sso / password                     | From .env: SSO `QAAdminEmail`/`QAAdminPassword`; password `OrderUsername`/`OrderPassword`. |
| App URL         | `https://qa.americoldcompass.com/` | Default QA portal.                                                                         |

**Artifact folders for this run:**

```
MCP/stories/<Module>/globalization/<lang>/
├── <screen>.compare.json     ← every glob_compare_sessions save_to
├── findings.md               ← THE ledger (one row per gap)
└── coverage.md               ← screens/sections visited + compared vs skipped
```

---

## Step 0 — Ground yourself (before any browser)

1. Read [../globalization_testing/SKILL.md](../skills/globalization_testing/SKILL.md).
2. Confirm the locale folder: list `testData/locales/` if needed; run a baseline audit —
   `glob_check_locale_keys(language_folder="<folder>", language="<lang>")` — and note the
   counts (missing/empty/identical keys). This is your prior, NOT your verdict.
3. Read the **manageConfiguration reference trio** to lock in the exact structure to mirror:
   `ManageConfigurationsGlobalizationLandingPage.py` (POM),
   `business_components/web_components/manageConfigurations.py` (business), and
   `tests/test_scripts/globalization/manageConfiguration/test_manageConfigurationsLandingPage.py`
   (test). Also read the target module's **functional step-def POM(s)** — globalization
   locators are DERIVED from them (same XPath, `text()` literal → `translate('<key>')`).
4. Check the module's `.agents/context/` brain (if present) for navigation intent — intent
   only; selectors come from the live DOM.
5. **GRAPH FIRST:** call `kg_brief` for the module — previous walks (functional OR
   globalization) already stored verified navigation selectors, page maps, quirks and
   locale keys there. Use `kg_get_page_map` on each screen before probing; screens whose
   globalization compare was already recorded (facts tagged `globalization`) can be
   spot-checked instead of fully re-walked. `glob_capture_texts` / `glob_compare_sessions` /
   `glob_check_locale_keys` auto-feed the graph (testid locators, compare summaries,
   locale keys); persist extra insights with `kg_record`.

## Step 1 — SETUP: two logged-in browsers

```jsonc
glob_login_language { "language": "en", "session_name": "EN",
                      "url": "<app url>", "auth": "sso|password",
                      "username": "<.env>", "password": "<.env>" }
glob_login_language { "language": "<lang>", "session_name": "<LANG>", … }
```

- Both sessions must land on the app shell. If the language dropdown or option is missing →
  STOP, report (that itself is a globalization defect candidate).
- Verify visually: the `<LANG>` browser's home page should already show translated chrome.

## Step 2 — WALK & COMPARE, screen by screen

For **each screen** of the module (and each meaningful state of that screen):

1. **Move both browsers together** — language-neutral selectors ONLY
   (`data-testid`, structural XPath, URL navigation):

   ```jsonc
   glob_sync_action { "sessions": ["EN","<LANG>"],
                      "action": {"type":"click","selector":"//button[@data-testid='…']"} }
   ```

   Never click by visible text — it differs per browser.

2. **Compare the whole screen**:

   ```jsonc
   glob_compare_sessions { "base_session":"EN", "target_session":"<LANG>",
                           "page_label":"<Module> > <Screen>",
                           "save_to":"MCP/stories/<Module>/globalization/<lang>/<screen>.compare.json" }
   ```

3. **Compare EVERY text category — the 100% inventory contract.** A screen is NOT done
   until every applicable category below has been compared (or explicitly recorded as
   NOT COMPARED with a reason). Work through this checklist per screen — most gaps hide
   in the hidden-state categories:

   | #   | Category                       | How to surface it in BOTH sessions                                                                                                                                                     |
   | --- | ------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
   | 1   | Page titles & headers (h1–h6)  | visible on load                                                                                                                                                                        |
   | 2   | Breadcrumbs                    | visible on load                                                                                                                                                                        |
   | 3   | Tab labels & nav items         | visible; also compare ACTIVE vs INACTIVE state text if it changes                                                                                                                      |
   | 4   | ALL buttons                    | primary/secondary/icon buttons, aria-labels AND title attributes, disabled-state text                                                                                                  |
   | 5   | Status labels / badges / chips | render rows with every reachable status (filter/search per status if needed)                                                                                                           |
   | 6   | Input field labels             | every form field label + REQUIRED marker text                                                                                                                                          |
   | 7   | Input placeholders & aria      | placeholder=, aria-label=, title= on inputs/searches                                                                                                                                   |
   | 8   | Dropdown/select options        | OPEN each dropdown (`glob_sync_action` click) → compare the listbox (`scope_selector`) → close — EVERY dropdown on the screen, including filter dds, page-size dds, kebab/action menus |
   | 9   | Table column headers           | render the table first (search/apply); include sort-icon aria-labels                                                                                                                   |
   | 10  | Table cell UI texts            | status cells, action-link cells, "N items" counters — NOT user data                                                                                                                    |
   | 11  | Alert / toast messages         | trigger success + error toasts (submit valid/invalid) and compare before they disappear                                                                                                |
   | 12  | Validation messages            | submit empty required form; type invalid values (email, number range, date) — EVERY field's message                                                                                    |
   | 13  | Empty / no-result states       | search gibberish in both → compare "no results" title + helper text                                                                                                                    |
   | 14  | Dialogs / flyers / modals      | open EVERY one (create/edit/confirm/delete) → compare title, body, field labels, footer buttons, X/close aria → close                                                                  |
   | 15  | Confirmation prompts           | trigger delete/cancel confirms → compare question + Yes/No/Cancel buttons                                                                                                              |
   | 16  | Tooltips, pagination, footer   | hover-reachable tooltips; "Show row:", "Page X of Y", footer links, loading texts ("Loading…")                                                                                         |

   Maintain a **per-screen category matrix** in `coverage.md` (rows = screens, columns =
   the 16 categories, cells = ✅ / — n/a / ❌ NOT COMPARED + reason). 100% coverage of the
   module = every cell is ✅ or n/a with an honest reason.

4. **Audit every suspect immediately** (one batched call per screen):

   ```jsonc
   glob_check_locale_keys { "language_folder":"<folder>", "language":"<lang>",
                            "texts": ["<every NOT_TRANSLATED / suspicious EN text on this screen>"] }
   ```

   Classify each finding:

   | Finding type          | Evidence                                                            |
   | --------------------- | ------------------------------------------------------------------- |
   | `NOT_TRANSLATED`      | live `<LANG>` text identical to EN (non-brand, non-data)            |
   | `KEY_MISSING_IN_LANG` | key exists in en.json, absent from `<lang>.json`                    |
   | `VALUE_IDENTICAL`     | key exists in both but `<lang>` value == English value              |
   | `KEY_MISSING_IN_EN`   | live UI text has NO key at all in en.json (reverse lookup empty)    |
   | `MISSING_ELEMENT`     | element present in EN page, absent in `<LANG>` page (or vice versa) |

5. **Record findings as you go** — append to `findings.md` (skill §7 row format: screen,
   element, EN text, `<LANG>` text, type, locale key, key-in-en?, key-in-lang?) and tick the
   screen in `coverage.md`. A screen is ✅ only when its visible texts AND its dropdowns/
   dialogs/messages were compared; anything you couldn't reach is listed as **NOT COMPARED**
   with the reason.

6. **Serious gap → formal defect document** (whole page/section untranslated, missing
   language, broken layout in `<LANG>`): follow
   [../defect_documentation/SKILL.md](../skills/defect_documentation/SKILL.md) — switch to the
   `<LANG>` session and capture the annotated defect screenshot with **`glob_defect_screenshot`**
   (NOT the plain `screenshot_step`) so the exact untranslated word/sentence is **outlined in
   RED** in the image that lands in the .docx:

   ```jsonc
   glob_defect_screenshot {
     "session": "<LANG>",                       // the target-language browser, on the defect screen
     "module": "<Module>", "story_id": "GLOB-<lang>", "case_id": "<CASE-ID>",
     "index": 1,
     "highlight_texts": ["Search Audits", "Reset"],   // the exact English strings that leaked into <lang>
     "scope_selector": "[role='dialog']",       // optional — limit to a flyer/dialog
     "defect": {                                // optional — writes/merges <case>_defect.json
       "title": "QA - <Area>: <concise summary>",
       "severity": "Major",
       "description": "…", "prerequisites": ["…"], "stepsToReproduce": ["…"],
       "expectedResult": "…", "actualResult": "…"
     },
     "render_docx": true                        // build <case>_defect.docx in the same call
   }
   ```

   `glob_defect_screenshot` outlines each `highlight_texts` entry in red with a numbered badge,
   saves `<case>_defect_<N>.png`, auto-fills `screenshots` + `highlightedTexts` in the payload,
   and (with `render_docx:true`) renders the .docx whose Evidence section captions the
   highlighted strings. It then removes the highlights so the walk continues on a clean page.
   If you prefer to render separately, omit `render_docx` and run

   `python MCP/build_defect_docx.py`.

## Step 3 — DESIGN the test cases (page-wise & section-wise)

From the walk (not from imagination), write the test-case table into
`MCP/stories/<Module>/globalization/<lang>/testcases.md`:

```markdown
| #   | Test (function name)                           | Page           | Section         | Verifies (elements & keys)                    |
| --- | ---------------------------------------------- | -------------- | --------------- | --------------------------------------------- |
| 1   | test*globalization*<page>\_landing_page        | Orders landing | Title + actions | pageHeading.orders, outboundOrder.createOrder |
| 2   | test*globalization*<page>\_breadcrumbs         | Orders landing | Breadcrumbs     | breadcrumbs.home/reports/…                    |
| 3   | test*globalization*<page>\_search_panel_labels | Orders landing | Search panel    | every field label + placeholder key           |
| 4   | test*globalization*<page>\_dropdown_options    | Orders landing | Date-range dd   | calendarRangeOptions.\*                       |
| 5   | test*globalization*<page>\_validation_messages | Create order   | Form validation | error/helper message keys                     |
```

**Coverage rule:** every screen and every section compared in Step 2 gets at least one test;
elements with findings STILL get tested (the test asserts the CORRECT translation via
`translate(key)` — it will fail red until the product/locale gap is fixed, which is the
deliverable). Sections you could not compare are listed as NOT COVERED with the reason.

**Per-category test minimums** — design one test function per category-per-page wherever the
category exists (mirror of the Step 2 matrix; DON'T collapse categories into one giant test):

| Category (from Step 2)         | Test function shape                                                  | Verifies                                                             |
| ------------------------------ | -------------------------------------------------------------------- | -------------------------------------------------------------------- |
| Titles/headers/breadcrumbs     | `test_globalization_<page>_headers_breadcrumbs`                      | `_verify_text` per h-tag + each breadcrumb via `translate(key)`      |
| Tabs & nav                     | `test_globalization_<page>_tabs`                                     | every tab label key                                                  |
| Buttons (all)                  | `test_globalization_<page>_buttons`                                  | every visible button + icon-button aria-label key                    |
| Status labels/badges           | `test_globalization_<page>_status_labels`                            | EVERY status value key (loop the status map — one assert per status) |
| Field labels + placeholders    | `test_globalization_<page>_form_labels_placeholders`                 | every label key + every placeholder/aria key                         |
| Dropdown options               | `test_globalization_<page>_<dd>_dropdown_options` (one per dropdown) | open dd live → `_verify_text` per option key (`translate` of each)   |
| Table column headers           | `test_globalization_<page>_table_columns`                            | every column-header key (render table first)                         |
| Alerts/toasts                  | `test_globalization_<page>_toast_messages`                           | trigger → assert toast text key before it fades                      |
| Validation messages            | `test_globalization_<page>_validation_messages`                      | submit invalid → assert EVERY field's error key                      |
| Empty/no-result states         | `test_globalization_<page>_empty_state`                              | gibberish search → assert empty-title + helper keys                  |
| Dialogs/flyers (one test each) | `test_globalization_<page>_<dialog>_dialog`                          | title, body, labels, footer buttons, close-aria keys                 |
| Confirmation prompts           | `test_globalization_<page>_confirm_prompts`                          | question + Yes/No/Cancel keys                                        |
| Pagination/footer/tooltips     | `test_globalization_<page>_pagination_footer`                        | "Show row", page indicator, footer link keys                         |

Keys with **no i18n key at all** (KEY_MISSING_IN_EN) still appear — as `_verify_missing_key_locator`
soft probes inside the owning test, cross-referenced to the findings row. The testcases.md table
must show, per page, WHICH categories are covered by which test — an auditor must be able to
confirm 100% of the Step 2 matrix maps to a test or a documented exclusion.

## Step 3.5 — LIVE-VERIFY the designed test cases (dual-browser dry-run — MANDATORY before authoring)

No test is written from imagination or from compare JSONs alone. With the SAME two
browsers still open, dry-run **every row of testcases.md** and only then author:

1. **Navigate both sessions to the test's screen/state** (`glob_sync_action`,
   language-neutral selectors) exactly the way the future test will navigate.
2. **Prove each assertion target**, per (element, key) pair the test will loop over:
   - the locator the POM will use RESOLVES in the EN session (element found, visible);
   - `translate(key)`'s **en value** is the live EN text (anchor is correct);
   - in the `<LANG>` session the translated value renders → test row verdict **green**,
     OR the English/raw-key text renders → verdict **RED-defect**, cross-referenced to
     its findings.md row (append the row NOW if missing);
   - placeholders/aria/React input values proven via the live attribute
     (`get_attribute("placeholder"|"aria-label"|"value")`), not assumed.
3. **Record the dry-run verdict into testcases.md** — add a `Verified live` column:
   `✅ green-both` / `🔴 red-in-<lang> (findings #N)` / `❌ NOT AUTHORABLE + reason`
   (unreachable state, no stable anchor, data-dependent). Only ✅/🔴 rows get authored;
   ❌ rows stay in the table as documented exclusions.
4. **Feed the graph** — `kg_record` every locator/flow/quirk proven during the dry-run
   so the authored POM uses ONLY live-proven selectors.
5. Efficiency: batch per screen (one sync-nav, then prove all that screen's test rows);
   reuse Step 2 evidence where the exact element+state was already compared — the
   dry-run then only re-proves the LOCATOR, not the translation verdict.

> The authored test file is a **transcript of a dry-run that already happened** —
> navigation path, locators, keys, and expected verdict all come from this step.

## Step 4 — AUTHOR (globalization conventions — skill §3–5; reference module = **manageConfiguration**)

Order: **POM → business functions (only if a flow is reused) → tests**.

> 🎯 **Reference module — copy its structure, not just its spirit.** Before authoring,
> read the manageConfiguration trio and mirror it file-for-file:
>
> - POM: `object_repository/pages/Apps/ManageConfigurations/ManageConfigurationsGlobalizationLandingPage.py`
> - business: `business_components/web_components/manageConfigurations.py`
> - test: `tests/test_scripts/globalization/manageConfiguration/test_manageConfigurationsLandingPage.py`

1. **POM** — `object_repository/pages/<Area>/globalizationLocators/<Name>GlobalizationPage.py`
   (or the module's existing globalization POM folder), structured EXACTLY like
   `ManageConfigurationsGlobalizationLandingPage`:
   - `class <Name>GlobalizationPage(CustomPyAutoWeb):` with a class-level
     `configure_i18n(language_folder="<folder>")` at the top, then locators built with
     f-string `translate('key')` (manage-config style) — or `LazyLocator` where the
     sibling globalization POMs of that area already use it. **Match the area's existing
     globalization POM; when creating a brand-new one, use the manage-config style.**
   - **DERIVE locators from the functional (BDD/step-def) POM — do NOT invent new XPaths.**
     Find the element's locator in the module's existing functional POM
     (`object_repository/pages/Apps/<Module>/…Page.py`) and reuse the SAME XPath structure,
     replacing ONLY the hardcoded visible-text literal (the `text()='…'` /
     `normalize-space()='…'` part) with `{translate('<key>')}`. Language-neutral parts
     (`data-testid`, `aria`, structure, axes, indexes) stay byte-identical to the
     functional POM. Only probe the live DOM for elements that have NO functional locator.
   - Section dividers per UI section (`# ----------------- breadcrumb locators ------------------#`),
     `locator<PascalCase>` names, and grouped `(locator, key, name)` orders matching the
     functional POM's grouping.
2. **Business functions** — `business_components/web_components/<module>Globalization.py`:
   `_norm_loc()` around every locator; `translate(key)` for dropdown values; driver-first;
   zero assertions. Only for flows used by 2+ tests (navigation, search, form fill).
3. **Tests** — `tests/test_scripts/globalization/<module>/test_<name>.py`, anatomy copied
   from `test_manageConfigurationsLandingPage.py`:
   - 3-level markers + `@allure.title` + a full `@allure.description("""…""")` block listing
     Preconditions and a numbered Validations list;
   - `configure_i18n(language_folder="<folder>")` first, then
     `login_to_application_sso_with_language(driver)` (or the module's sibling login);
   - one `with allure.step(…)` per section; grouped checks loop over
     `[(locator, key, name), …]` tuples exactly like the manage-config breadcrumb loop;
   - verification = `expected = translate(key)` → `actual = web.get_element_text(locator)` →
     `web.element_text_should_be(web_element=locator, expected_string=expected)`;
   - every check attaches `Language / Translation Key / Expected Text / Actual Text /
Result: PASS ✅|FAIL ❌` evidence like the manage-config file;
   - dropdown option selection via the shared `select_translated_dropdown_option` helper
     pattern. Test data goes in `testData/globalization<Module>.json` — never hardcoded.

### 🚫 Locale JSON files are READ-ONLY — let the test fail

If a text is untranslated because a key is **missing or empty in the locale JSON**
(`KEY_MISSING_IN_LANG`, `KEY_MISSING_IN_EN`, `VALUE_IDENTICAL`):

- **NEVER add, rename, or edit keys/values in any `testData/locales/**/\*.json` file.\*\*
  Those files mirror the product's i18n bundles — patching them fakes a green and hides
  the product gap.
- Author the test against the key that SHOULD exist (or the `MISSING_KEY` probe for
  KEY_MISSING_IN_EN) and **let it fail red** — the red test IS the deliverable, tied to its
  findings.md row. Never weaken the assertion, never skip, never soft-pass.

## Step 5 — VERIFY in BOTH languages

```powershell
$env:TestLanguage="en";   python -m pytest tests/test_scripts/globalization/<module>/ -m globalization_<module> --alluredir=reports/allure-results -q
$env:TestLanguage="<lang>"; python -m pytest tests/test_scripts/globalization/<module>/ -m globalization_<module> --alluredir=reports/allure-results -q
```

- **en must be green.** `<lang>` must be green EXCEPT tests that assert texts with open
  findings — each such red is cross-referenced to its findings.md row (that's a correct
  failure, not a broken test). Locator/timing failures are yours: fix at the source (POM
  key/locator, wait) — max 2 runs per language, then stop and report.
- Never weaken an assertion (casefold-contains instead of equals, etc.) just to go green.

## Step 6 — REPORT (chat summary, in this order)

1. **Coverage** — the screens × 16-categories matrix from coverage.md: every cell ✅ / n/a /
   NOT COMPARED (reason) — the module is only
   "done" at 100% compared or every gap explained.
2. **Findings table** — every gap with type, EN/`<LANG>` texts, locale key, key status —
   plus baseline audit counts (missing/identical keys) for the whole locale folder.
3. **Defect documents** created (paths) for serious gaps.
4. **Authored artifacts** — POM(s), business functions, test files (+ test count per page).
5. **Verify results** — per-language pass/fail with each red tied to its finding row.
6. **Artifacts** — compare JSONs, findings.md, coverage.md, testcases.md paths.

---

## Hard rules (recap)

- **Language-neutral selectors** for every sync action — never translated text.
- **Every test case dry-run live BEFORE authoring** (Step 3.5) — locator resolved, key
  value seen rendering (or its gap recorded), verdict known; a test authored without
  its dual-browser dry-run is invalid.
- **Every locale key proven** via `glob_check_locale_keys` — never invented.
- **Locale JSONs are read-only** — a missing/empty/identical key is a finding + a red test,
  NEVER a key you add to `testData/locales/`.
- **Globalization locators derive from the functional step-def POM** — same XPath, only the
  `text()` literal swapped for `translate('<key>')`; new XPaths only where no functional
  locator exists.
- **manageConfiguration is the reference structure** — new globalization POMs/tests mirror
  its file organization, verification style, and Allure evidence format.
- **Findings ledger is append-as-you-go** — a gap seen but not recorded is a lost defect.
- **Serious gaps get the formal defect .docx** (defect-documentation skill) — captured with
  **`glob_defect_screenshot`** so the offending text is **outlined in RED** in the embedded
  image; never ship a defect .docx whose screenshot doesn't point at the gap.
- **Globalization tests are plain pytest** (`driver` fixture, markers, `configure_i18n`) —
  NOT pytest-bdd; never mix with the BDD tree.
- **Both-language verification is mandatory** — authored-but-unverified tests are not done.
- No `time.sleep`, no hardcoded test data, no assertions outside tests — the constitution
  ([AGENTS.md](../AGENTS.md)) always applies.
