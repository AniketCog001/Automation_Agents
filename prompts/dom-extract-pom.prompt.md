---
description: >
  Token-minimal POM generator: reads story + feature file, drives the live browser
  via MCP snapshot_dom to extract compact interactive-element JSON per page, then
  converts that JSON into a production-ready XPath POM file — saving ~90 % of the
  tokens that a raw-HTML approach would consume. Works in VS Code agent mode.
agent: DomExtractorPom
---

# DOM-Extract → POM (Token-Minimal XPath Generation)

> **Trigger:** `/dom-extract-pom`
> **Token saving:** ~90 % vs feeding raw HTML to the XPath prompt.

---

## Inputs

| Input                  | How to provide                                                                  | If missing                    |
| ---------------------- | ------------------------------------------------------------------------------- | ----------------------------- |
| **Module**             | `${input:module:CMO}`                                                           | → STOP and ask                |
| **Story ID**           | `${input:storyId:AMCC-XXXX}`                                                    | → STOP and ask                |
| **Target POM file**    | `${input:targetPom:object_repository/pages/Apps/CMO/CmoXxxPage.py}`             | → derive from module + story  |
| **Extra pages/states** | `${input:extraPages:optional comma-separated notes on dialogs or tabs to open}` | → derive from story + feature |

**Module:** ${input:module:CMO}
**Story ID:** ${input:storyId:AMCC-XXXX}
**Target POM file:** ${input:targetPom:}
**Extra pages/states to capture:** ${input:extraPages:}

---

## Load the skill first

Read `.agents/skills/dom_extract_pom/SKILL.md` completely before doing anything else.
That skill is the ground-truth cookbook for every step below.

Also read:

- `.agents/skills/page_objects/SKILL.md` — POM conventions
- `.agents/skills/selenium_mcp_tools/SKILL.md` — MCP tool reference

---

## Execution protocol

Work in **agent mode** — execute every step, write all files, never just describe.

### STEP 0 — Bootstrap

```
kg_bootstrap()
kg_context_pack(task="${input:storyId}", module="${input:module}")
```

Report what is already known (USE locators = skip in DOM extraction).

### STEP 1 — Context gathering (file reads only)

1. Read `MCP/stories/${input:module}/${input:storyId}/story.md`
   - If missing: run `python MCP/fetch_userstory.py ${input:storyId}` first, then read.
2. Read the feature file: `tests/features/Apps/${input:module}/` — find the `.feature`
   file for `${input:storyId}`.
   - If no feature file yet: note the gap and derive element needs from the story AC only.
3. Read the reference/existing POM for this module:
   `object_repository/pages/Apps/${input:module}/` — pick the most relevant existing file.
4. From steps 1–3, output:
   - **POM Style Ruleset** (captured from reference POM)
   - **Elements needed list** (one row per element, source = feature step text / AC)
   - **Page walk plan** (pages to visit + how to navigate to each)

### STEP 2 — DOM extraction via MCP (ZERO generation tokens)

Follow the DOM extraction protocol from the skill (`§2`):

1. `start_browser` (if not already running)
2. `login_sso` with credentials from `MCP/.env.mcp`
3. For each page in the walk plan:
   a. Navigate to the page via `click_sequence` / `navigate`
   b. `wait_for_idle`
   c. `snapshot_dom({ "max_elements": 300 })`
   d. Save the snapshot JSON to `MCP/fixtures/${input:storyId}/dom/<page-slug>_snapshot.json`
   e. For each needed sub-state (tabs, modals, dialogs): open → `wait_for_idle` →
   `snapshot_dom` → save to `<page-slug>_<substate>_snapshot.json`
   f. `kg_upsert` every element with a testid or unique text IMMEDIATELY

4. Print the **coverage table** (element needed → found / GAP).

### STEP 3 — POM generation (compact JSON → XPaths)

Follow the generation protocol from the skill (`§3`):

1. For each saved snapshot file, derive XPaths using the priority order:
   `testid → unique text → role+name → type+placeholder → positional (flagged)`
2. Apply the POM Style Ruleset from STEP 1.
3. Create state-variant locators only where the snapshot showed the state attribute.
4. Create lambda locators for any data-driven elements.
5. **Write the target POM file** at the path from `${input:targetPom}` (or a derived path).

### STEP 4 — Validate & report

1. Run the self cross-check from the skill (`§5`).
2. `kg_reindex()`
3. `kg_session_save(...)`
4. Print the final DOM-Extract → POM Report (as defined in the skill `§5`).

---

## Output

- **POM file written**: `${input:targetPom}` (or the derived path)
- **Snapshots saved**: `MCP/fixtures/${input:storyId}/dom/*.json`
- **KG updated**: verified locators, gaps noted
- **Console report**: elements covered, gaps, token savings

The POM file is immediately ready for use by `/03_step_definitions` and `/web_component_prompt`.
