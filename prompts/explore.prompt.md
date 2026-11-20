---
description: Explore/verify a module's screens with the live browser and grow the knowledge graph (no code authoring).
agent: Explorer
---

Module: ${input:module:CMO}
Screens / flows to cover: ${input:scope}

`kg_context_pack` first, then walk the scope graph-first: reuse USE locators,
gate VERIFY ones, probe PROBE/unknown screens live with `snapshot_dom`.
Capture every proven locator/flow/fact the moment it is verified
(`kg_upsert` / `kg_add_knowledge`), record drift with `kg_record_failure`,
finish with `kg_reindex` and a report of what was learned vs what remains
unknown.
