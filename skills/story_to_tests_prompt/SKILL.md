---
description: 'Turn a Jira story (+ QMetry test cases) into runnable 4-layer BDD tests.
  Trigger: /story-to-tests.'
agent: TestAuthoringCoordinator
name: story_to_tests_prompt
---

Story: ${input:story:AMCC-XXXX}
Module: ${input:module:CMO}
Scope / flow notes: ${input:notes}

STEP 0: call `kg_bootstrap`. Then run the full pipeline for this story:

1. Delegate **Explorer** to walk the story's screens graph-first and capture
   verified locators/flows/facts.
2. Delegate **PomAuthor** and **FeatureAuthor** in parallel.
3. Delegate **WebComponentAuthor**, then **StepDefAuthor**.
4. Confirm with ONE batched foreground pytest run (`-n` if 2+ tests); on
   failure delegate **Healer**; confirmed product bugs → **DefectAuthor**.
5. Update the traceability matrix and finish with `kg_session_save`.

Author full 4-layer coverage for every acceptance criterion — report any
NOT COVERED criterion explicitly, never paper over gaps.
