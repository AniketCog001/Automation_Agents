---
description: 'Checkpoint the current session into the knowledge graph so the next
  chat can /resume from it. Trigger: /checkpoint.'
agent: TestAuthoringCoordinator
name: checkpoint_prompt
---

Checkpoint now: call `kg_session_save` with

- **summary** — 2–6 sentences of what this session accomplished (outcomes
  over process),
- **decisions** — choices made that future sessions must respect,
- **todos** — work left open,
- **next_steps** — where to pick up,
- **worked_on** — node ids / stories touched,
- **completed_todos** — previously-open TODOs that are now done.

Then run `kg_reindex` and confirm the saved session node id.
Extra notes to include: ${input:notes}
