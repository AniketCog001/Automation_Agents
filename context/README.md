# `MCP/context/` — the module knowledge brain

This folder holds the **knowledge brain** for the MCP pipeline: one
`<Module>.context.md` per Cold Compass module (e.g. [../cmo_context/SKILL.md](../cmo_context/SKILL.md)).

The MCP execution prompt (`/mcp-execute-and-author`) and Antigravity in Agent mode **load the
matching `<Module>.context.md` first** — _before_ walking the live DOM — so the agent understands
the feature's **purpose, flows, vocabulary, and business rules** up front. This grounding is what
stops the agent from **hallucinating** screens, inventing flows, or misreading the app.

---

## What a `*.context.md` is — and is NOT

| ✅ It DOES contain (knowledge)                                             | ❌ It does NOT contain (volatile detail)              |
| -------------------------------------------------------------------------- | ----------------------------------------------------- |
| What the module is for, in plain business language                         | Locators / XPaths / CSS selectors                     |
| Roles & personas; navigation; the top-level areas                          | Story-specific or test-case-specific step lists       |
| Every feature & sub-feature, what each does, and the stories that cover it | Test data values                                      |
| Domain vocabulary (glossary) and the core entities                         | Page-Object method names / code signatures            |
| Canonical end-to-end flows (business intent)                               | Anything that changes when the DOM or a story changes |
| Business rules, gotchas, and known environment caveats                     | —                                                     |

**Why the split?** Locators, steps, and data live where they belong and **always win on
conflict**: the **live DOM** (resolved through the `selenium` MCP), the **Page Objects**
(`object_repository/pages/Apps/<Module>/`), the **specs** (`tests/.../<Module>/`), and the
**test data** (`testData/<module>.json`). Because the context file never hard-codes a selector,
it never goes stale — it stays a stable, trustworthy brief.

---

## How it's loaded

- **By the agent / prompt:** `MCP/skills/mcp_execute_and_author_prompt/SKILL.md` **Step 0** reads
  `MCP/context/<module>.context.md` first (the `<module>` is derived from the story path, e.g.
  `/CMO/AMCC-5128/story.md` → `CMO`).
- **By the deterministic tooling:** `MCP/mcp_execute_and_author.py` (PREPARE) records the
  resolved context path in the generated `mcp-execution-plan.md` and the
  `execution-progress.md` ledger, so the live walk and the authored test both stay grounded.

The relevant skill — [skills/mcp_execution_protocol/SKILL.md](../skills/mcp_execution_protocol/SKILL.md)
— makes "load the module context first" **Rule 0** of every walk.

---

## Files here

| File                                          | What it is                                                                 |
| --------------------------------------------- | -------------------------------------------------------------------------- |
| [../cmo_context/SKILL.md](../cmo_context/SKILL.md)              | The CMO module knowledge brain (worked, complete example).                 |
| [\../_template_context/SKILL.md](../_template_context/SKILL.md) | Copy this to start a new module's brain. Fill every section from the code. |

---

## Add a new module's brain

1. **Copy the template:** `../_template_context/SKILL.md` → `<Module>.context.md` (match the module's
   folder name exactly, e.g. `ActionPlan.context.md`, `Claims.context.md`).
2. **Mine the real code (read-only):** read that module's `tests/features/Apps/<Module>/*.feature`
   (the `Feature:` lines map the whole surface), `business_components/web_components/<module>.py`
   (the reusable flows + `get_`/`set_` globals), `object_repository/pages/Apps/<Module>/` (the
   screens), and `testData/<module>.json` (entities + known caveats). **Never edit that code.**
3. **Write knowledge, not selectors:** describe purpose, roles, navigation, every feature, the
   glossary, the canonical flows, and the business rules — **no locators, no steps, no data**.
4. **Keep the live DOM as the tie-breaker:** end with a "the live DOM / Page Objects / specs win"
   reminder, as the CMO brain does.

That's it — the next `/mcp-execute-and-author` run for that module will load it automatically.
