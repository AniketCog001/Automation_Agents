---
description: Resume where the last session left off — warm-start from the knowledge graph's session memory.
agent: TestAuthoringCoordinator
---

Call `kg_bootstrap` now. Then tell me, based ONLY on what it returns:

1. What the last session(s) did (summary + decisions).
2. All open TODOs and where each was left.
3. The recommended next step, with the exact node ids / files involved.

Do not ask me to re-supply context — expand pointers with `kg_get` /
`kg_get_module` / `kg_context_pack` if you need more detail. Wait for my
confirmation before delegating any work.
