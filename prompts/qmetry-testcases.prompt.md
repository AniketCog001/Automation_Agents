---
description: "Download QMetry test cases by FOLDER PATH + LABELS as testcases.json (deterministic script, no invention), then render testcases.md from that JSON. Saves the organized, story-number-wise way: MCP/stories/<module>/<STORY-ID>/. Module-agnostic — any module."
agent: agent
---

# Download QMetry test cases (JSON) + render testcases.md (MCP pipeline · Step 2)

Pull the **exact** set of QMetry test cases under a **folder path**, scoped by **labels**, into
`MCP/stories/<module>/<STORY-ID>/testcases.json` (the machine contract), then **render**
`MCP/stories/<module>/<STORY-ID>/testcases.md` from that JSON for human review.

The JSON download is **deterministic Python — never invented** (`MCP/fetch_qmetry.py`). This is
the framework's port of the reference Playwright `/qmetry-testcases` flow. Keep narration short
— run it, render, report.

> **🧠 Knowledge-graph protocol:** after rendering `testcases.md`, link the cases to the story
> node: `kg_add_knowledge(node_id="<Module>--story-<story-id>" or the 'Story — <STORY-ID>' fact
node, topic="Test cases", tag="story", text="<N> QMetry cases · ref:
MCP/stories/<module>/<STORY-ID>/testcases.md")` — so `/story-to-tests` and the coordinator
> can find them from the graph alone.

> ⚠️ **Folder consistency rule:** if the story already exists under
> `MCP/stories/<module>/<STORY-ID>/story.md`, reuse that same `<module>` folder for
> `testcases.json`/`testcases.md` (pass `--module <that-module>`). Do **not** create a second
> module folder from a QMetry naming variant.

## Inputs (the only things I provide)

- **QMetry path:** ${input:qmetryPath:QMetry test-case folder path, e.g. /Release 3/CMO/AMCC-5128}
- **labels:** ${input:labels:comma-separated labels, e.g. CMO - Regression, CMO_Sub - Regression}

## Steps — do ALL of them, in order

1. **Pre-flight `.env`.** Confirm `QMETRY_OPEN_API_KEY` + `QMETRY_BASE_URL` (+ `JIRA_PROJECT_KEY`)
   are set in `MCP/.env.mcp`. If either is missing, **stop and report** which one.

2. **Resolve the output module first.** Extract the `STORY-ID` (last `PROJECT-NNNN` segment) from
   the QMetry path. If `MCP/stories/*/<STORY-ID>/story.md` already exists, use that `<module>`.
   Otherwise fall back to the path segment before `<STORY-ID>`.

3. **Download the JSON + render the MD (deterministic).** Run:

   ```
   python MCP/fetch_qmetry.py --folder "${input:qmetryPath}" --labels "${input:labels}" --story <STORY-ID> --module <resolved-module>
   ```

   The script resolves the folder **path → numeric folderId** (walking the project's test-case
   folder tree), searches the cases under it scoped by the labels (**including sub-folders**),
   **hydrates each with its ordered steps**, writes `testcases.json` (the source of truth), and
   renders a faithful `testcases.md` (header with folder+labels, a coverage summary, then one
   block per case with `Step <n>` → `Expected` / `Data`).

   > If the tenant exposes no folder-tree endpoint, run with `--labels` only (no `--folder`), or
   > pass explicit keys: `--keys "AMCC-TC-15200,AMCC-TC-15201"`.

4. **Verify the render mirrors the JSON.** Open `testcases.md` and confirm it is **1:1** with
   `testcases.json` — same cases, same step order, every expected result the concrete observable
   from the JSON (never "looks correct"). Fix the render only if it drifted from the JSON; never
   add/drop a case or step.

5. **Report** the resolved **story id + module**, the **case count**, and the two written paths.
   Then state the **human-review gate** (below).

## Edge cases (handle, don't guess)

- **0 test cases:** the script writes a stub `testcases.md` with a warning. Re-check the folder
  **path** and **labels** (try removing labels), then re-run. Do **not** fabricate cases.
- **401 / AUTH FAILURE:** `QMETRY_OPEN_API_KEY` is wrong/expired — report it; don't retry.
- **FOLDER NOT FOUND:** the error names the missing path **segment** and lists what's available
  there — fix the spelling/slashes and re-run.

## Human-review gate

After the fetch + render, **pause** for a human to review `testcases.md` before any analysis or
code is generated. Do **not** bypass it. When approved, continue with `/mcp-execute-and-author`.

## Hard rules

- **JSON is deterministic, never authored.** The cases come straight from QMetry — never invent,
  drop, re-order, or "improve" a case or step. `testcases.json` is the source of truth and
  `testcases.md` must mirror it exactly.
- **Verbatim.** The folder path and labels in the output match exactly what I passed.
- **Organized layout, always:** `MCP/stories/<module>/<STORY-ID>/testcases.{json,md}` — and if
  `story.md` already exists, `<module>` must be that same folder.

## Example invocation

```
/qmetry-testcases
QMetry path → /Release 3/CMO/AMCC-5128
labels      → CMO - Regression, CMO_Sub - Regression
```

→ writes `MCP/stories/CMO/AMCC-5128/testcases.json` and `…/testcases.md`.
