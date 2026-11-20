---
description: Prove how much of a user story the test cases actually cover.
name: coverage_traceability
---

# Skill — Coverage & Traceability (story ↔ test case ↔ code ↔ defect)

Read [../framework_architecture/SKILL.md](../framework_architecture/SKILL.md) first. This skill
governs **coverage claims**: a story feature is _covered_ only when a test case genuinely
verifies it — never assume, always map. It extends the coverage-check in
[../skills/mcp_execute_and_author_prompt/SKILL.md](../skills/mcp_execute_and_author_prompt/SKILL.md) and
the gap analysis in [../skills/02_feature_file/SKILL.md](../skills/02_feature_file/SKILL.md).

---

## 1. The coverage rule (never assume)

- Extract acceptance criteria **verbatim** from `MCP/stories/<module>/<STORY-ID>/story.md`.
- Map every criterion to the case(s) that verify it. A criterion with no covering case is a
  **COVERAGE GAP — reported, never papered over**.
- A case "covers" a criterion only if one of its `@then` steps **asserts** that criterion's
  outcome. Navigating past a screen is NOT coverage of that screen's behaviour.
- Verdict format: `N of M story features covered (X%)` + explicit `NOT COVERED` list.

## 2. Per-story artifact — `coverage-report.md`

Written to `MCP/stories/<module>/<STORY-ID>/coverage-report.md` (structure defined in the
mcp-execute-and-author prompt). It is the story-local proof.

## 3. Repo-level artifact — the persistent traceability matrix

Maintain **one** cumulative CSV: `MCP/stories/traceability-matrix.csv`

```csv
story_id,module,acceptance_criterion,case_id,feature_file,status,defect_ref
AMCC-8438,CMO,"Supervisor can start a new audit",AMCC-TC-15200,tests/features/Apps/CMO/cmo-AMCC-8438.feature,COVERED,
AMCC-8438,CMO,"Audit list paginates at 25 rows",,,NOT COVERED,
AMCC-4717,CMO,"Search by full name returns results",AMCC-TC-15693,tests/features/Apps/CMO/cmo-AMCC-4717.feature,COVERED-DEFECT,TestcaseExecution/CMO/AMCC-4717/AMCC-TC-15693/AMCC-TC-15693_defect.docx
```

Rules:

- **Append/update rows** whenever a coverage-check runs, a feature file is authored, or a defect
  doc is raised — one row per (story, criterion). Never delete rows; update `status`.
- `status` ∈ `COVERED` · `NOT COVERED` · `COVERED-DEFECT` (covered, currently failing on a
  confirmed product bug) · `PARTIAL` (only happy path — see §5).
- This CSV is the single answer to "how much do our tests cover?" — total % = COVERED rows /
  all rows, per story or repo-wide.

## 4. Test conditions BEFORE test cases (design-technique pass)

When generating feature files, first list **test conditions** (what to test) per acceptance
criterion, and tag each with the technique used:

| Technique             | Apply when…                                     |
| --------------------- | ----------------------------------------------- |
| Equivalence Partition | input classes (valid/invalid types of values)   |
| Boundary Value        | limits, lengths, counts, dates, pagination      |
| Negative / error path | invalid input, missing data, denied permissions |
| State transition      | draft → published → archived style flows        |

Include this conditions table in the gap report so missing negative/boundary scenarios are
visible **before** authoring, not after a production defect.

## 5. Happy-path-only anti-pattern check

For every feature under test, coverage is `PARTIAL` unless the cases include (where applicable):

- [ ] the happy path
- [ ] empty input
- [ ] invalid input
- [ ] error/denied response handling
- [ ] boundary conditions

If a story's cases are happy-path only, say so explicitly in the coverage verdict — never report
`COVERED` for a criterion whose negative side is untested when the criterion implies one.

## 6. Self-check

- [ ] Acceptance criteria taken verbatim — none invented, none dropped.
- [ ] Every criterion has a row in `traceability-matrix.csv` with an honest status.
- [ ] Coverage verdict states N of M + every NOT COVERED / PARTIAL item.
- [ ] Defect-linked rows carry the `defect_ref` path.
- [ ] No coverage claimed for a criterion without a real `@then` assertion behind it.
