---
description: 'Shared efficiency & hygiene contract: zero temp/scratch files, blocking
  runs, budgets.'
name: efficiency_and_hygiene
---

# Skill — Efficiency & Hygiene (the shared execution contract)

These rules exist because wasted runs and scratch files are the two biggest time sinks in an
agent session. **Every prompt in `.agents/prompts/` inherits this skill.** Violating any rule
invalidates the work.

---

## 1. ZERO temp / scratch files

- **Never create scratch files** — no `_tmp*.py`, `*.tmp`, ad-hoc `Tee-Object` logs, throwaway
  `.ipynb`, debug scripts, or "notes" markdown. If a diagnostic needs code, run it **in memory**
  (a Python snippet via the workspace interpreter / `execute_script` in the browser) — never
  write it to disk first.
- **The repo's artefacts ARE the memory.** Allure `*-result.json`, `heal-plan.md`,
  `heal-progress.md`, `healing-log.md`, `story.md`, `testcases/*.md`, the knowledge graph —
  read those; do not duplicate their content into new files.
- **A file may be created only if** it is (a) a real framework file (POM / step def / feature /
  web_component / testData), (b) a pipeline artefact with a defined home
  (`MCP/stories/…`, `MCP/healing/…`, `TestcaseExecution/…`), or (c) explicitly requested by the user.
- If a temp file is truly unavoidable, it goes in `temp/`, and it is **deleted in the same
  session** — before the final report.

## 2. Read artifacts — do NOT re-run to "see" anything

- The failure message, failing step, stack trace, and screenshots are **already** in the Allure
  results and `heal-plan.md`. **Never re-run a test to look at its failure again.**
- Before running anything, ask: _"Is the answer already on disk?"_ An Excel report, an old Allure
  run, a `story.md`, or the knowledge graph usually answers the question in seconds.
- Query the knowledge graph FIRST (`kg_recall` / `kg_search`) — a previous session may have
  already diagnosed the same failure.

## 3. Running pytest — foreground by default; background ONLY for the whole-scope run

**Default: run pytest as a normal BLOCKING foreground command and simply wait for it to
return.** The terminal tool waits and hands back the full output in one shot — that IS the
result. A per-case or batched confirmation run (a handful of scenarios, even 10–20 minutes) is
still a foreground run: issue it, wait, read the returned output. Done.

**FORBIDDEN polling anti-patterns (these waste more time than the run itself and cause
hallucinated conclusions):**

- ❌ `Start-Sleep -Seconds N; Get-Content <log> -Tail X` loops — blind sleeps teach you nothing;
  a redirected log (`Out-File`/`>`) often only flushes at process exit, so tailing it mid-run
  shows an empty/stale file and leads to WRONG conclusions ("run is stuck") about a healthy run.
- ❌ Redirecting pytest output to a log file (`Out-File`, `Tee-Object`, `> file`) just to poll
  it. Never capture output to a file — read the terminal's own returned output.
- ❌ Counting `chrome`/`chromedriver` processes, checking file sizes/`.Length`, or `Test-Path`
  loops to "detect" whether a run is alive.
- ❌ Killing and restarting a run because a poll looked quiet. A quiet poll is NOT evidence of
  a hung run.
- ❌ Issuing any terminal command into the SAME terminal where a run is executing — it queues
  behind (or interrupts) the run and corrupts both outputs.

**The ONLY background run** is the single end-of-session whole-scope verification when it is
expected to take **> ~20 minutes**:

- Start it with `isBackground=true`, note the start time, then **do other useful work**
  (defect docs, KG capture, report drafting) — do not sit in a sleep loop.
- Poll **only** via fresh Allure `*-result.json` mtimes (the one artefact written per-scenario
  as it finishes), **at most every ~10 minutes**, using the background terminal's output tool —
  never a new blocking `Start-Sleep` command:

  ```powershell
  Get-ChildItem <alluredir> -Filter "*-result.json" |
    Where-Object {$_.LastWriteTime -gt $runStart} |
    ForEach-Object { $d = Get-Content $_.FullName -Raw | ConvertFrom-Json; "$($d.status)  $($d.name)" }
  ```

- **Max 3 polls per background run.** If after 3 polls it is still running, state that honestly
  in the report and read the results when they exist — never invent a status.
- **Never run two overlapping suite runs** against the same environment/session.

## 4. Anti-garbage-loop budgets (hard)

| Activity                              | Budget                                   | On breach                            |
| ------------------------------------- | ---------------------------------------- | ------------------------------------ |
| pytest runs per test case             | **2** (fix → run №1; refine → run №2)    | STOP: re-classify / escalate / ask   |
| Whole-scope runs per session          | **1** (at the very end)                  | —                                    |
| Edit rounds on one file for one case  | 2                                        | STOP and re-diagnose live            |
| Diagnostic MCP walks per failing case | 1 (reuse the browser + SSO across cases) | —                                    |
| Retries of the same failed command    | 1 retry with a **changed** hypothesis    | Never re-issue the identical command |
| Polls of a background run             | **3** (≥10 min apart, artefact-based)    | Report honestly; read results later  |
| `Start-Sleep` blocking poll commands  | **0** — forbidden entirely               | —                                    |

- **One hypothesis at a time.** Change one thing, verify once. Shotgun edits are forbidden.
- Confirm several fixed cases in **one** pytest invocation (`-k "A or B or C"`), never one
  process per test.

### MANDATORY PARALLEL EXECUTION (STRICT RULE)

**ALL test runs with 2+ tests MUST use pytest-xdist parallel execution.**

```powershell
# ✅ REQUIRED — Parallel execution formula
$testCount = <number of tests selected>
$workers = [Math]::Min($testCount, 8)

# Single test — no -n flag
pytest tests/step_defs/apps/CMO/test_cmo-AMCC-1234.py --alluredir=allure-results

# 2-8 tests — use -n <test_count>
pytest -k "AMCC-TC-1234 or AMCC-TC-1235" -n 2 --alluredir=allure-results

# 9+ tests — cap at -n 8
pytest tests/step_defs/apps/CMO/ -n 8 --alluredir=allure-results

# Healing multiple cases — ALWAYS parallel
pytest -k "test_case_A or test_case_B or test_case_C" -n 3 --alluredir=allure-results
```

**❌ FORBIDDEN — Sequential execution of multiple tests:**

```powershell
# NEVER DO THIS — runs tests one by one
pytest tests/step_defs/apps/CMO/test_cmo-AMCC-1234.py
pytest tests/step_defs/apps/CMO/test_cmo-AMCC-1235.py
pytest tests/step_defs/apps/CMO/test_cmo-AMCC-1236.py

# NEVER DO THIS — multiple tests without -n flag
pytest -k "A or B or C" --alluredir=allure-results  # Missing -n!
```

Each worker opens its own browser/SSO session, so N tests take ~1 test's wall time.
Sequential execution of multiple UI tests in the terminal is FORBIDDEN — it wastes the
session's time budget. A single test runs without `-n` (worker startup isn't worth it).

- Tests that already passed in this session are **never** re-run.

## 5. Continuous knowledge-graph capture (build the brain in the background)

Knowledge capture is not a final chore — it happens **as you learn**, so a crash never loses it:

- **The moment a locator is proven live** → `kg_upsert(kind="locator", …)`.
- **The moment a flow/navigation works end-to-end** → `kg_upsert(kind="flow"/"navigation", …)`.
- **The moment a quirk/business rule is confirmed** → `kg_upsert(kind="fact", …)`.
- **The moment a heal/escalation concludes** → `kg_add_knowledge(topic="Healing — <CASE-ID>",
tag="healing", …)` with root cause → fix.
- **Link everything to its story**: create/refresh a story node
  (`kg_upsert(kind="fact", name="Story — <STORY-ID>", tag="story",
text="<one-line summary> · ref: MCP/stories/<module>/<STORY-ID>/story.md")`) and wire edges
  with `kg_link` — story ↔ feature ↔ flow ↔ locators ↔ healing facts. The `story.md` path in the
  node text is the canonical reference the next session follows.
- End of session: `kg_reindex` once.

## 5.5 Surgical edits & anti-hallucination discipline

- **Minimal necessary changes.** Only modify code explicitly targeted by the task. **No
  unsolicited refactoring, cleanup, renaming, or style changes** on untouched parts — a heal
  session edits the failing locator/wait/flow and NOTHING else. Integrate, don't replace.
- **No scope expansion.** Implement exactly what was asked; do not add extra features, extra
  edge-case handling, or "while I'm here" improvements that were not requested.
- **Verification over memory.** For anything version-, DOM-, or state-dependent (a locator, a
  label, an API shape, a file's current content), **check the artifact/live source first**
  (read the file, `kg_recall`, `snapshot_dom`) instead of answering from general knowledge.
  What cannot be verified is stated as unverified — never presented as fact.
- **Declare intent before acting.** One short line stating what you are about to run/edit and
  why, immediately before the tool call — it keeps the session auditable and the todo ledger
  accurate.
- **Parallel-safe test data.** Every entity a test creates (template names, audit names, plan
  titles…) must carry a **run-unique suffix** (timestamp / short UUID via the existing
  web_component name generators) so `-n 8` xdist workers never collide. Never author a test
  that depends on another test's data or on fixed entity names.

## 6. Session-end hygiene checklist

- [ ] No new files outside framework folders / defined artefact homes.
- [ ] Anything in `temp/` that this session created is deleted.
- [ ] pytest caches removed (`.pytest_cache`) — never the real Allure `*-result.json` data.
- [ ] Knowledge captured incrementally (not batched at the end) + `kg_reindex` run.
- [ ] Budgets respected — report any budget breach and its re-classification honestly.
