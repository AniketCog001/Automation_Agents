---
description: How to document a genuine product defect found during an MCP walk or
  heal session.
name: defect_documentation
---

# Skill — Defect Documentation (defect-only evidence)

Read [../mcp_execution_protocol/SKILL.md](../mcp_execution_protocol/SKILL.md) first. This skill is the
cookbook for the **only** moment screenshots are taken in this pipeline: when the live app
**genuinely contradicts** a test case's expected result. Everything that behaves as expected is
recorded in `execution-results.json` as `PASS` — with **no PNG**.

---

## 1. When to raise a defect document (the trigger)

Raise the formal defect doc when **ALL** of these are true:

1. A step's **live actual result contradicts its expected result** (from `testcases.json` /
   the story's acceptance criteria).
2. You **ruled out your own mistake** — wrong navigation, a precondition you skipped, missing
   test data you were supposed to create, or reading the screen too early (re-check with
   `wait_for_idle` + a fresh `get_element_state` before concluding).
3. You **reproduced it** (or can cite the exact deterministic evidence — e.g. an XPath count of
   0 with the data present, an API response, an error toast).

A defect is **serious** when it breaks a story acceptance criterion, blocks a user flow, loses
or corrupts data, or shows wrong data. Cosmetic drift you are unsure about → record a `bug` note
in the ledger + flag it in the report, and **ask** before raising a formal doc.

**Never** raise a defect doc for: missing seeded data (that's DATA-BLOCKED), an expired session,
an environment outage, or your own locator/timing drift (that's a heal, not a bug).

### 1.1 Required quality bar (no docx ships without ALL of these)

- **Title** in the form `QA - [<Area/Component>]: <symptom> when <action>` — symptom first, not
  a vague "issue with X".
- **Severity** (Critical/High/Major/Minor) **with a one-line business-impact rationale** —
  "Major: plan IS created but the user gets no confirmation, so users re-submit duplicates".
- **Environment/build**: URL, date/time, browser, and the logged-in role used.
- **Minimal repro**: the shortest steps-to-reproduce that still reproduce — strip incidental steps.
- **Expected vs Actual** quoted against the story's acceptance criterion (cite the criterion).
- **Evidence**: screenshot(s) of the broken state + the deterministic proof (XPath count, toast
  text, API response) — observed only, nothing inferred.

### 1.2 Aggregate defect ledger — `TestcaseExecution/defect-log.csv`

Every raised defect also appends one row to the repo-level CSV (create it on first use):

```csv
date,module,story_id,case_id,severity,title,status,docx_path
2026-07-17,ActionPlan,AMCC-4417,AMCC-TC-FU001,Major,"QA - [Create Plan]: success dialog missing when attachment added",OPEN,TestcaseExecution/ActionPlan/fileUpload/AMCC-TC-FU001/AMCC-TC-FU001_defect.docx
```

This ledger feeds trend analysis (found vs fixed, by module/severity) and **risk-based test
selection** — modules with recent escaped defects get their regression scope widened first.
Update `status` (OPEN → FIXED/REJECTED) when a re-test confirms it; never delete rows.

---

## 2. The evidence tree (defect-only)

A case folder exists **only** when that case surfaced a defect:

```
TestcaseExecution/
  └── <Module>/                          e.g. CMO
      └── <STORY-ID>/                    e.g. AMCC-4717
          ├── execution-results.json     (every case/step's status — ALWAYS written)
          └── <CASE-ID>/                 e.g. AMCC-TC-15693   ← ONLY if it found a defect
              ├── <CASE-ID>_defect_1.png    (screenshot OF THE DEFECT state)
              ├── <CASE-ID>_defect_2.png    (…more shots if more screens prove it)
              ├── <CASE-ID>_defect.json     (the structured payload — §4)
              └── <CASE-ID>_defect.docx     (the formal document — §5)
```

## 3. Capture the defect screenshot(s)

The moment the defect is confirmed, shoot the exact screen(s) that prove it — **before**
navigating away — with the `screenshot_step` fast tool in **defect mode**:

```jsonc
// 1st proof shot → TestcaseExecution/CMO/AMCC-4717/AMCC-TC-15693/AMCC-TC-15693_defect_1.png
{ "module": "CMO", "story_id": "AMCC-4717", "case_id": "AMCC-TC-15693",
  "step": 1, "defect": true }

// 2nd shot (e.g. the contrasting working state) → …_defect_2.png
{ "module": "CMO", "story_id": "AMCC-4717", "case_id": "AMCC-TC-15693",
  "step": 2, "defect": true }
```

- `step` is the 1-based **defect-shot index** (`_defect_1`, `_defect_2`, …) — not the test step.
- If the same defect affects other cases, pass `also_cases=["AMCC-TC-YYYY"]` to drop the same
  PNG into their folders too (each affected case still gets its own `_defect.json`).
- Shoot what a triaging developer needs: the broken state, and where it helps, the working
  contrast state (e.g. "search by first name works" vs "full name returns nothing").

## 4. Write the payload — `<CASE-ID>_defect.json`

Write it into the case folder. **Required:** `module`, `storyId`, `caseId`, `title`. Everything
observed, nothing invented:

```json
{
  "module": "CMO",
  "storyId": "AMCC-4717",
  "caseId": "AMCC-TC-15693",
  "title": "QA - Operations Manager: Supervisor search fails to return results when querying both first name and surname",
  "severity": "Major",
  "environment": "https://qa.americoldcompass.com/ (QA)",
  "foundDuring": "MCP live execution — Add Supervisors flyer, step 3",
  "description": "In the OM My Team > Add Supervisors flyer, searching a supervisor by their FULL name (first name + surname) returns no results, even though the supervisor exists and is returned when searching by first name only.",
  "prerequisites": [
    "User is logged in to Cold Compass as an Operations Manager",
    "A facility with at least one assignable supervisor exists (e.g. Atlanta - Tradewater)",
    "The supervisor 'Cypress Supervisor' exists in the facility's supervisor pool"
  ],
  "stepsToReproduce": [
    "Navigate to Apps > CMO > My Team",
    "Select the facility from the facility dropdown",
    "Click the 'Add Supervisors' button to open the Add Supervisors flyer",
    "Type the supervisor's full name 'Cypress Supervisor' into the search field",
    "Observe the results list"
  ],
  "expectedResult": "The supervisor matching the full name is listed and can be selected for addition to the team.",
  "actualResult": "The results list is empty ('No supervisors found'). Searching 'Cypress' (first name only) returns the same supervisor — the API does not match multi-token full-name queries.",
  "screenshots": ["AMCC-TC-15693_defect_1.png", "AMCC-TC-15693_defect_2.png"],
  "evidence": "Live MCP walk: search input filled with full name → results list rendered 0 items; same session, first-name-only query → 1 item ('Cypress Supervisor').",
  "reportedBy": "MCP execute-and-author walk"
}
```

Field rules:

| Field              | Rule                                                                                                            |
| ------------------ | --------------------------------------------------------------------------------------------------------------- |
| `title`            | `"QA - <Area>: <concise, specific defect summary>"` — names the broken behaviour, not the test case.            |
| `description`      | Plain product language: what is wrong and its user impact. No test-framework jargon.                            |
| `prerequisites`    | The minimal state a human needs before the steps work (role, data, navigation starting point).                  |
| `stepsToReproduce` | Numbered, **human-replayable** — exactly how you reproduced it live. A dev must be able to follow blindly.      |
| `expectedResult`   | What the story/AC says should happen (cite it, don't paraphrase into something new).                            |
| `actualResult`     | What the app actually did — the observed facts, including the contrasting working behaviour if relevant.        |
| `screenshots`      | Bare filenames (resolved against the case folder) or full paths. Omit → the builder auto-scans `_defect_*.png`. |
| `evidence`         | The technical proof of root cause: exact XPath counts, API responses, console errors — from THIS session.       |
| `severity`         | `Critical` / `Major` / `Minor` / `Trivial` (defaults to Major).                                                 |

> **Anti-hallucination:** every sentence in `actualResult` and `evidence` must be something you
> observed live in this session. If you did not see it, it does not go in the defect doc.

## 5. Render the formal document — `build_defect_docx.py`

```powershell
# One case (reads <CASE-ID>_defect.json from its folder):
python MCP/build_defect_docx.py --story "AMCC-4717" --module CMO --case AMCC-TC-15693

# Every documented defect under a story (end-of-run roll-up):
python MCP/build_defect_docx.py --story "/CMO/AMCC-4717/story.md" --all

# From an explicit payload path:
python MCP/build_defect_docx.py --defect "TestcaseExecution/CMO/AMCC-4717/AMCC-TC-15693/AMCC-TC-15693_defect.json"
```

Output: `TestcaseExecution/<Module>/<STORY-ID>/<CASE-ID>/<CASE-ID>_defect.docx` with the exact
QA template — **Title** heading, meta table (case / story / module / severity / environment /
found-during / date / status), then **Description → Prerequisites → Steps to Reproduce →
Expected Result → Actual Result → Evidence** (embedded screenshots) → optional **Technical
evidence (root cause)**.

## 6. After the doc — keep the record honest

- The failing step stays **`FAIL`** (with a `bug` note) in `execution-results.json` — never
  bend the test to match the buggy behaviour, never fake a pass.
- The authored/healed test keeps asserting the **story's** expected behaviour, so it fails red
  against the buggy app — that failing test + the defect `.docx` are the deliverable.
- Heal sessions: add the defect doc's path to the `healing-log.md` entry — a bug entry without
  its defect doc is an unreported bug.
- The final report lists every defect: title · severity · `<CASE-ID>_defect.docx` path.

## 7. Defect-doc self-check (run before moving on)

- [ ] Trigger confirmed: live contradiction + own-mistake ruled out + reproducible/evidenced.
- [ ] Screenshot(s) captured with `defect=true` **at the defect state** (not after navigating away).
- [ ] `<CASE-ID>_defect.json` written — all template fields, everything observed, nothing invented.
- [ ] `stepsToReproduce` replayable by a human with only the prerequisites listed.
- [ ] `python MCP/build_defect_docx.py … --case <CASE-ID>` ran and printed the `.docx` path.
- [ ] Ledger keeps the step `FAIL` + bug note; report/healing-log references the doc.
- [ ] Zero screenshots exist for passing steps.
