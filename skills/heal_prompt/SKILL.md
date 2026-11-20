---
description: 'Heal a failing test case by diagnostic live MCP re-walk. Trigger: /heal.'
agent: Healer
name: heal_prompt
---

Failing scope: ${input:scope:test file, -k expression, or case ids}
Known context: ${input:notes}

STEP 0: `kg_bootstrap`, then `kg_recall` for prior healing knowledge on these
cases. Diagnose from Allure artifacts FIRST (never re-run to see a failure),
classify each failure (DRIFT / TIMING / FLOW / PRODUCT BUG), fix at the
source layer with surgical edits, confirm with batched foreground pytest
(max 2 runs per case), record healing knowledge to the graph, and finish with
`kg_session_save`. Product bugs: report back for defect documentation —
never loosen an assertion to force green.
