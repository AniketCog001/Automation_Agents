---
description: Turn a Jira story (+ QMetry test cases) into runnable 4-layer BDD tests via the multi-agent pipeline (Explorer → POM ∥ Feature → WebComponent → StepDefs → Heal).
agent: TestAuthoringCoordinator
---

Story: ${input:story:AMCC-XXXX}
Module: ${input:module:CMO}
Scope / flow notes: ${input:notes}

STEP 0: call `kg_bootstrap`. Then run the full pipeline for this story:

1. **Locator mode decision** — choose ONE:
   - **Fast (locators-only)**: delegate **DomExtractorPom** (`/dom-extract-pom`) to
     extract compact DOM snapshots and generate the POM directly (~90 % fewer tokens).
     Use this when no flow/business-rule verification is needed.
   - **Full verification**: delegate **Explorer** to walk screens graph-first and
     capture verified locators/flows/facts. Use when the story needs flow entries in
     the KG or has complex unknown navigation paths.
2. Delegate **PomAuthor** (if Explorer was used) and **FeatureAuthor** in parallel.
3. Delegate **WebComponentAuthor**, then **StepDefAuthor**.
4. Confirm with ONE batched foreground pytest run (`-n` if 2+ tests); on
   failure delegate **Healer**; confirmed product bugs → **DefectAuthor**.
5. Update the traceability matrix and finish with `kg_session_save`.

Author full 4-layer coverage for every acceptance criterion — report any
NOT COVERED criterion explicitly, never paper over gaps.
