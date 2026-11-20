---
description: 'Fetch JIRA user stories and details for test automation. Trigger: /fetch-userstory.'
agent: agent
name: fetch_userstory_prompt
---

# Fetch a JIRA user story (MCP pipeline · Step 1)

Pull a single JIRA user story by its id and write it to the MCP pipeline's organized
layout — `MCP/stories/<module>/<STORY-ID>/story.md`. This is the **first** pipeline
stage; everything else (`/qmetry-testcases`, `/mcp-execute-and-author`) reads from this
folder.

This is the Python/Selenium framework's port of the reference Playwright `/fetch-userstory`
flow. The fetch is **deterministic** (a real script, not invention). Keep narration short —
fetch it, then report.

> **🧠 Knowledge-graph protocol:** after saving `story.md`, register the story in the graph so
> every later stage/agent can link to it:
> `kg_upsert(kind="fact", name="Story — <STORY-ID>", tag="story", module=<Module>,
text="<one-line story summary> · ref: MCP/stories/<module>/<STORY-ID>/story.md")`.
> If this begins a work session, `kg_bootstrap` first — the story may already be known.

> **Repo facts:** this is a **pytest-bdd** project. The MCP tooling lives under `MCP/`; the
> fetched story lands under `MCP/stories/<module>/<STORY-ID>/`. Read
> [skills/mcp_execution_protocol/SKILL.md](../skills/mcp_execution_protocol/SKILL.md) first.

## Inputs (the only thing I provide)

- **Story id:** ${input:storyId:JIRA story id, e.g. AMCC-5128}
- **Module (optional):** ${input:module:CMO | ActionPlan | Claims | … (derived from JIRA if omitted)}

## Steps — do ALL of them, in order

1. **Pre-flight `.env`.** Confirm `JIRA_EMAIL` + `JIRA_API_TOKEN` (+ `JIRA_BASE_URL`,
   `JIRA_PROJECT_KEY`) are set in `MCP/.env.mcp` (copy from `MCP/.env.mcp.example`). If a
   value is missing, **stop and report** which one.

2. **Fetch the story (deterministic).** Run the fetch script:

   ```
   python MCP/fetch_userstory.py ${input:storyId} ${input:module:+--module }${input:module}
   ```

   It calls the **JIRA Cloud REST API v3** (`GET /rest/api/3/issue/<id>`), converts the ADF
   description/comments to Markdown, derives the module from JIRA `components`/`labels`
   (unless `--module` is given), and writes `MCP/stories/<module>/<STORY-ID>/story.md`.

   > **Alternative (atlassian MCP):** if the `atlassian` MCP server is registered in
   > `.vscode/mcp.json` and you prefer it, fetch the issue with that tool and write the same
   > `story.md` shape yourself — but the script above is the sanctioned, deterministic path.

3. **Module resolution.** If the script exits with "could not determine the module" (code 2),
   it found no `component`/`label` match. **Do NOT guess.** Ask me:
   _"I cannot determine the module for ${input:storyId} from JIRA. What module should it be
   under? (e.g., CMO, ActionPlan, Claims, …)"_ — then re-run with `--module <name>`.

4. **Confirm + report** the resolved **module**, the **written path**, the **AC count**, and a
   one-line summary of the story. Then state the next step:
   `/qmetry-testcases` (fetch its test cases).

## Hard rules

- **Never invent content.** The script copies JIRA fields verbatim; a missing field becomes
  "Not provided". Acceptance criteria are extracted verbatim.
- **Organized layout, always:** `MCP/stories/<module>/<STORY-ID>/story.md` — the exact path the
  rest of the pipeline consumes.
- **Module-agnostic.** Nothing here is specific to one module; the module comes from the JIRA
  issue (or `--module`).
- JIRA **401/403** → stop and report the auth error; don't retry. **404** → check the id /
  project-key prefix.

## Example invocation

```
/fetch-userstory
story id → AMCC-5128
module   → CMO
```

→ writes `MCP/stories/CMO/AMCC-5128/story.md`.
