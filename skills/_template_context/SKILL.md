---
name: _template_context
description: Instructions and guidelines for context template.
---

<!--
  TEMPLATE — copy me to MCP/context/<Module>.context.md and fill every section
  from the REAL module code (read-only). Write business KNOWLEDGE only:
    • NO locators / XPaths / CSS selectors
    • NO story- or test-case-specific step lists
    • NO test-data values or Page-Object method signatures
  Those live on the live DOM, the Page Objects, the specs, and testData/<module>.json,
  and always win on conflict. Delete this comment when done.
-->

# <Module> — Module Knowledge Brain

> Business + functional knowledge brain for the **<Module>** module of the Cold Compass portal.
> The MCP pipeline loads this **first** (before walking the live DOM) so the agent understands
> the feature's intent and rules and does not hallucinate. Locators, steps, and data are
> deliberately **omitted** — they live on the live DOM / Page Objects / specs / `testData` and
> win on any conflict. Derived **read-only** from the real `<Module>` automation.

---

## 1. Snapshot

| Attribute          | Value                                                                                                                                                                                       |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Module**         | <Module>                                                                                                                                                                                    |
| **Where it lives** | Cold Compass → Apps → <Module>                                                                                                                                                              |
| **Primary routes** | `/<route>` …                                                                                                                                                                                |
| **App style**      | React SPA — shadcn/ui + Radix UI + Tailwind                                                                                                                                                 |
| **Auth**           | Microsoft / Azure AD **SSO** — every session starts authenticated                                                                                                                           |
| **Roles**          | <persona-1> · <persona-2> …                                                                                                                                                                 |
| **Core entities**  | <entity-1> · <entity-2> …                                                                                                                                                                   |
| **Reference code** | `tests/features/Apps/<Module>/` · `tests/step_defs/apps/<Module>/` · `object_repository/pages/Apps/<Module>/` · `business_components/web_components/<module>.py` · `testData/<module>.json` |

---

## 2. What <Module> is (business purpose)

_Two or three plain-language paragraphs: what problem this module solves, who uses it, and the
2–3 big things that happen here. No UI mechanics — just intent._

---

## 3. How a user reaches <Module> (navigation)

_Apps → <Module>; the landing page; the top-level tabs/areas and what each lands on; any
sub-areas (e.g. a configuration surface). Describe **where things are and why**, not how to click
them._

---

## 4. Roles & personas

| Persona       | What they do in <Module> |
| ------------- | ------------------------ |
| **<persona>** | …                        |

---

## 5. Domain vocabulary (glossary)

| Term       | Meaning |
| ---------- | ------- |
| **<term>** | …       |

---

## 6. Feature areas (what each one is & does)

_One subsection per real, code-backed capability. List the covering stories as pointers; the
behaviour description is the knowledge._

### 6.1 <Feature area> · _<STORY-IDs>_

_What it is and the key behaviours._

### 6.2 <Feature area> · _<STORY-IDs>_

…

---

## 7. Key components / concepts (if the module has a repeated building block)

_e.g. a component catalogue, a wizard's steps, a record's lifecycle. Knowledge only._

---

## 8. Canonical end-to-end flows

**Flow A — <name>**

1. …
2. …

**Flow B — <name>**

1. …

---

## 9. Business rules & gotchas

1. _Rule that governs expected behaviour (gating, mutual-exclusion, defaults, disabled states…)._
2. _Never fake a pass — a live contradiction is a bug or a stale story; record it._

---

## 10. Known environment / data caveats (from `testData/<module>.json` notes)

- _Real, observed caveats (data-blocked paths, skips, known bugs) — respect them; they are not
  reasons to weaken a test. The JSON remains the source of truth._

---

## 11. Where the <Module> code lives (reference map — no locators here)

| Layer                  | Path                                                                     |
| ---------------------- | ------------------------------------------------------------------------ |
| **Features (Gherkin)** | `tests/features/Apps/<Module>/<module>-<STORY-ID>.feature`               |
| **Step defs**          | `tests/step_defs/apps/<Module>/test_<module>-<STORY-ID>.py`              |
| **Page Objects**       | `object_repository/pages/Apps/<Module>/…`                                |
| **web_component**      | `business_components/web_components/<module>.py`                         |
| **Test data**          | `testData/<module>.json`                                                 |
| **SSO login helper**   | `business_components/web_components/login.py → login_to_application_sso` |

_List the already-reusable building blocks in `<module>.py` so the agent reuses before inventing._

---

## 12. How the MCP agent should use this file

1. **Load this file first** for any <Module> story — before opening the browser.
2. **Map the story → the feature area** in §6.
3. **Walk the live DOM** to resolve the actual elements; this file says _what_ and _why_, never _which selector_.
4. **Author** the pytest-bdd test in the 4-layer architecture, matching the existing `<Module>`
   style and the repo's authoring prompts (`feature_file_prompt.txt`, `xpath_prompt.txt`,
   `step_def_prompt.txt`).
5. **On any conflict, the live DOM / Page Objects / specs win** — and flag a genuine behaviour
   mismatch as a possible bug (never fake a pass).
