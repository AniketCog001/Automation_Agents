---
name: cmo_context
description: Business and functional knowledge brain for the CMO module.
---

# CMO — Module Knowledge Brain

> **Purpose of this file.** This is the **business + functional knowledge brain** for the
> **CMO** module of the Cold Compass portal. The MCP pipeline (and Antigravity in Agent
> mode) **loads this file first** — before walking the live DOM — so it understands _what the
> module is for, how a real user moves through it, what every feature does, and the business
> rules that govern it_. Grounding the agent in this knowledge is what stops it from
> hallucinating screens, inventing flows, or misreading the app.
>
> **What this file deliberately contains:** plain-business-language knowledge — the module's
> purpose, its roles, every feature and sub-feature, the domain vocabulary, the canonical
> end-to-end flows, and the business rules/gotchas.
>
> **What this file deliberately OMITS (by design):** locators, XPaths, CSS selectors, and
> story/test-case-specific steps. Those are **not** knowledge — they live where they belong and
> always win on conflict: the **live DOM** (resolved through the `selenium` MCP), the **Page
> Objects** (`object_repository/pages/Apps/CMO/`), the **specs** (`tests/.../CMO/`), and the
> **test data** (`testData/cmo.json`). This file never goes stale because it never hard-codes a
> selector.
>
> Derived **read-only** from the real CMO automation (features, step defs, page objects, the
> `cmo.py` web_component, and `cmo.json`). The code was analysed, never edited.

---

## 1. Snapshot

| Attribute          | Value                                                                                                                                                              |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Module**         | CMO                                                                                                                                                                |
| **Where it lives** | Cold Compass portal → **Apps** menu → **CMO**                                                                                                                      |
| **Primary routes** | `/cmo` (landing), `/cmo/myaudit` (Supervisor dashboard), `/manage-configurations` (CMO Goal Setup)                                                                 |
| **App style**      | React SPA — shadcn/ui + Radix UI + Tailwind (drive Radix Selects/dialogs/date-pickers as a user would)                                                             |
| **Auth**           | Microsoft / Azure AD **SSO** — every session starts authenticated                                                                                                  |
| **Roles**          | Template Author/Admin · **Supervisor** · **Operations Manager**                                                                                                    |
| **Core entities**  | Template · Audit · Facility · Associate · Job Code · Audit Type · Global Region · CMO Goal Setup · Section · Question Component                                    |
| **Reference code** | `tests/features/Apps/CMO/` · `tests/step_defs/apps/CMO/` · `object_repository/pages/Apps/CMO/` · `business_components/web_components/cmo.py` · `testData/cmo.json` |

---

## 2. What CMO is (business purpose)

**CMO** is Cold Compass's structured **audit / observation program**. It lets the organisation
build reusable **audit templates**, have **Supervisors** run those audits against individual
**associates** at a **facility**, and have **Operations Managers** oversee their team's audit
activity — all backed by configurable **CMO Goal Setup** targets per facility.

In plain terms, three things happen in CMO:

1. **Author** — an admin/template author builds an **audit template**: a form made of ordered
   **sections** and **question components** (single/multi select, short/long answer, performance
   score, date & time, file upload, signature, section headers). The template is scoped by
   **audit type**, **job code(s)**, and **global region**, then **published** so it becomes
   available to supervisors.
2. **Execute** — a **Supervisor** opens their **My Audits** dashboard, picks a **facility**,
   clicks **Start New Audit**, chooses a published template from the **Select Template** sidebar,
   fills in the **Associate Details**, then answers each question component **section by section**
   and submits — producing a completed audit with a **summary**.
3. **Oversee & configure** — **Operations Managers** review their team's audits on the **My
   Team** page; the **All CMO Audits** list view shows every audit across the program; and
   **Manage Configurations → CMO Goal Setup** defines the per-facility goal targets (number of
   supervisors, work days, direct employees, etc.) that drive the dashboard's goal progress and
   gate whether **Start New Audit** is even available for a facility.

The acronym **"CMO"** is used verbatim in the UI (the "CMO" landing heading, "All CMO Audits",
"CMO Goal Setup") — treat it as the program's name, not something to expand or relabel.

---

## 3. How a user reaches CMO (navigation)

- The global **Apps** menu (top of the app shell) lists the portal's apps; **CMO** is one of
  them. **Apps → CMO** lands on the **CMO landing page** (a "CMO" heading + a top tab bar).
- The CMO landing page has a **top navigation tab bar**. The tabs map to the role-based areas:

  | Tab                    | Lands on…                          | For whom                    |
  | ---------------------- | ---------------------------------- | --------------------------- |
  | **Supervisors**        | Supervisor **My Audits** dashboard | Supervisors running audits  |
  | **Templates**          | **Templates** landing list         | Template authors / admins   |
  | **All CMO**            | **All CMO Audits** list view       | Anyone reviewing all audits |
  | **Operations Manager** | **My Team** page                   | Operations Managers         |

- **Manage Configurations** (CMO Goal Setup, Export, Scheduled Report) is reached from the
  **Apps → Manage Configurations** area, then choosing **CMO** as the category.
- Tabs show an **active vs inactive** visual state; only one tab is active at a time.

> The exact tab/button selectors are **not** here on purpose — resolve them live and reuse the
> CMO Page Objects. This section is about _where things are and why_, not _how to click them_.

---

## 4. Roles & personas

| Persona                     | What they do in CMO                                                                                                                                                                                                                            |
| --------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Template Author / Admin** | Creates, edits, duplicates, deletes, and **publishes** audit templates via the **Templates** tab and **Template Builder**. Configures **CMO Goal Setup** in Manage Configurations.                                                             |
| **Supervisor**              | Runs audits: selects a facility, starts a new audit, picks a published template, fills associate details, answers each section's components, submits, and reviews the audit summary. Sees their facility's **goal** progress on the dashboard. |
| **Operations Manager**      | Oversees a team's audit activity on **My Team**; reviews status/coverage across supervisors.                                                                                                                                                   |

The test login (a `QAAdmin` account from `.env`) is **multi-facility** and broadly permissioned,
so a single session can author templates _and_ execute audits — but the **business intent** of a
flow still belongs to one of the personas above. Keep that persona in mind when reasoning about
expected behaviour (e.g. a supervisor should see execution fields **read-only** that an author
can edit on the build side).

---

## 5. Domain vocabulary (glossary)

| Term                   | Meaning                                                                                                                                                                                                         |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Template**           | A reusable audit form: ordered **sections** of **question components**, scoped by audit type / job code / region. Must be **published** to be usable by supervisors.                                            |
| **Audit**              | One execution of a template by a supervisor against one associate at one facility. Ends in a **summary**.                                                                                                       |
| **Facility**           | A physical site (e.g. "Allentown PA (80502)"). Selecting a facility filters which templates are available and is required before **Start New Audit**.                                                           |
| **Associate**          | The employee being audited; chosen on the **Associate Details** page via a typeahead, with employment status / reason / region details.                                                                         |
| **Audit Type**         | The template's category — **CMO**, **Facility**, **Safety**, **Inventory** (mutually exclusive; exactly one is selected).                                                                                       |
| **Job Code**           | The work activity a template targets — e.g. Case Pick, Pallet Pick, Loading, Receiving, Putaway, E-commerce, Rewarehousing (one or more).                                                                       |
| **Global Region**      | **Americas**, **Europe**, **Asia-Pacific** (mutually exclusive button group on the template).                                                                                                                   |
| **Section**            | A named group of question components inside a template; audits are executed **section by section** with **Next** navigation.                                                                                    |
| **Question Component** | A single form element in a section — see §7 for the full catalogue (single/multi select, answers, performance score, date & time, file upload, signature, section header).                                      |
| **Help text**          | Optional guidance attached to a component/section; **collapsed by default**, expandable; **read-only** during execution; can hold large text (the suite verifies up to 2000 words).                             |
| **Required toggle**    | Marks a component as mandatory; when ON the supervisor must answer before proceeding.                                                                                                                           |
| **Performance Score**  | A scored component (a rating/score the supervisor assigns during execution); also tied to the LMS performance section.                                                                                          |
| **CMO Goal Setup**     | Per-facility configuration (ops supervisors, work days, direct employees, …) under Manage Configurations. Drives dashboard **goal progress** and gates whether **Start New Audit** is available for a facility. |
| **Dimension**          | A configurable measure used by goals/performance (stored/read as a structured value by the audit flow).                                                                                                         |

---

## 6. Feature areas (what each one is & does)

Each sub-feature below is a real, code-backed capability. The JIRA stories that exercise it are
listed so the agent can connect a story to the right area instantly. (Stories are pointers — the
behaviour description is the knowledge.)

### 6.1 Templates Landing · _AMCC-4718_

The list of existing templates (the **Templates** tab). Supports searching templates by name and
opening a template's row actions (the **ellipsis** menu → edit / duplicate / delete). Entry point
to **Create Template**.

### 6.2 Template Creation — new-template flow & facility selection · _AMCC-7146_

Starting a brand-new template: choosing the **audit type** (CMO/Facility/Safety/Inventory),
**job code(s)**, **global region**, naming it, and associating it with **facilities**. Facility
association matters downstream: **a template only appears for the facilities it's associated
with** when a supervisor later picks a facility.

### 6.3 Template Builder — question components, publish, save, navigate, preview · _AMCC-7149, AMCC-8118, AMCC-8122, AMCC-8125_

The form-building canvas. The author adds **question components** to sections, sets each
component's **question text**, optional **help text**, **required** state, and component-specific
options/toggles, then **saves**, **previews**, and **publishes**. Components covered across the
builder stories:

- **AMCC-8118** — Section Header, **Single Select**, **Multi Select**.
- **AMCC-8122** — **Long Answer**, **Short Answer**, **Performance Score**.
- **AMCC-8125** — **Date and Time**, **File Upload**, **Signature**.
- **AMCC-7149** — the umbrella: adding components, publishing, saving, navigation, and preview.

### 6.4 Template Edit / Duplicate / Delete · _AMCC-6614_

Row-level lifecycle actions from the Templates landing ellipsis menu: **edit** an existing
template, **duplicate** it (creates a copy), and **delete** it (with confirmation).

### 6.5 Supervisor — My Audits Landing · _AMCC-4716_

The supervisor's home for audits. Key behaviours:

- **Facility dropdown** — selecting a facility **enables Start New Audit** and filters available
  templates to that facility.
- **Start New Audit** button — only available when the chosen facility has **CMO Goal Setup**
  configured (otherwise the button is absent/disabled — see business rules §9).
- Surfaces facility **goal** progress (daily/weekly targets) and recent audits.

### 6.6 Select Template Sidebar · _AMCC-5127_

The flyer that opens after **Start New Audit**. The supervisor **searches** templates and
filters by **audit type** (Facility/Safety/Inventory/CMO) to find the right published template,
then selects it to begin. Only templates associated with the selected facility appear.

### 6.7 Associate Details Page · _AMCC-5128_

The first screen of an audit. The supervisor identifies the **associate** (typeahead search) and
fills employment details (employment status, reason, region, facility — Radix dropdowns). Certain
fields (template name, description, audit category, job code) are shown **read-only** to the
supervisor — they describe the template, not editable audit input.

### 6.8 Audit Execution — per-component behaviour · _AMCC-8435, AMCC-8437, AMCC-8438_

Answering each section's components as a supervisor. The execution side mirrors the build side
but is **constrained**: the supervisor answers questions but **cannot edit** the question text,
help text, field toggles, or delete/duplicate components (those are author-only). Stories:

- **AMCC-8435** — Section Header, **Single Select**, **Multi Select** execution.
- **AMCC-8437** — **Long Answer**, **Short Answer**, **Performance Score** execution.
- **AMCC-8438** — **Date and Time**, **File Upload**, **Signature** execution.
  Each component verifies things like: question text (incl. alphanumeric & large/2000-word text)
  renders correctly, **Help** is collapsed by default & expandable & read-only, **Required** state
  behaves, and component-specific rules (e.g. Date-only vs Time-only vs both toggles for Date &
  Time; valid/invalid file formats for File Upload).

### 6.9 Audit Execution — multi-section navigation & component behaviour · _AMCC-8490_

The cross-section behaviour: moving **Next**/back through sections, that answers persist across
navigation, required-gating between sections, and overall component behaviour during a full
multi-section audit. This is the "common engine" story other execution stories build on.

### 6.10 Follow-Up Audit Sidebar · _AMCC-4800_

A flyer for follow-up audits — initiating or continuing a follow-up on a prior audit.

### 6.11 Audit Summary Page · _AMCC-5136_

The end-of-audit summary: a read-back of what was answered, the completed-audit summary flyer,
and confirmation the audit was recorded.

### 6.12 LMS Performance Section during audit · _AMCC-5132_

The **performance** section shown during an audit, tied to LMS (Learning Management System) data
— performance scoring presented to the supervisor as part of execution.

### 6.13 Operations Manager — My Team · _AMCC-4717_

The Operations Manager view of their **team's** audit activity — supervisors, coverage, and
status across the team.

### 6.14 All CMO Audits — list view · _AMCC-9459_

A program-wide grid of every CMO audit. Behaviours: a defined **column set** (Template, Audit
Type, Supervisor, Facility, Created On, Global Region, Associate, Job Code), **search** by
keyword, **hide/show columns**, **date-preset** filters (e.g. "Last 30 Days"), draft vs published
filtering, and an audit **count** badge. Large data set (hundreds of rows).

### 6.15 Supervisor Dashboard — goals & facility states · _AMCC-9893_

The supervisor dashboard's facility-aware messaging and goals:

- **Configured facility** → shows goal progress (e.g. daily goal %, weekly remaining) and enables
  Start New Audit.
- **No-config facility** → Start New Audit disabled; a "facility is without CMO data
  configuration…" message is expected (note known DOM caveats in §10).
- **No-template facility** → a "No templates available yet…" empty state.
- **Inactive/closed facility** → a notification it can't be used.
- Multi-facility users get a facility selector; single-facility users auto-resolve.

### 6.16 Manage Configurations — CMO Goal Setup · _AMCC-8699, AMCC-8700, AMCC-8701, AMCC-9341_

The configuration surface that powers everything above:

- **AMCC-8699** — CMO Goal Setup **landing & data table** (the per-facility goal rows).
- **AMCC-8700** — **Add New** CMO Goal Setup (create a facility's goal config: ops supervisors,
  work days, direct employees, …).
- **AMCC-8701** — **Inline edit & delete** of a CMO Goal Setup row.
- **AMCC-9341** — **Export** the CMO report (downloads `CMO_Report_<timestamp>.xlsx`, toast
  "Report Downloaded Successfully") and **Schedule Report** (a flyer: report name/type/format,
  schedule type, time, time zone, email — submitting schedules a recurring report).

---

## 7. Question / Form components (the heart of CMO)

Templates are built from these components, and audits execute them. Each exists in **two
contexts** — _build side_ (the author configures it, classes named `FormBuilder*` in the POM) and
_execution side_ (the supervisor answers it, classes named `FormExecution*` / question modules).
The **knowledge** below is context-independent; how each is driven lives in the POM + live DOM.

| Component             | What it is / does                                                                                           | Common attributes (build side)                           | Execution behaviour (supervisor)                                      |
| --------------------- | ----------------------------------------------------------------------------------------------------------- | -------------------------------------------------------- | --------------------------------------------------------------------- |
| **Section Header**    | A non-answer label that titles/divides a section.                                                           | Header text.                                             | Displayed; not answerable.                                            |
| **Single Select**     | Choose exactly **one** option from a list.                                                                  | Question text, options, help, required.                  | Pick one option.                                                      |
| **Multi Select**      | Choose **one or more** options.                                                                             | Question text, options, help, required.                  | Pick several options.                                                 |
| **Short Answer**      | A single-line free-text answer.                                                                             | Question text, help, required.                           | Type a short text answer.                                             |
| **Long Answer**       | A multi-line free-text answer (supports large text incl. 2000-word checks).                                 | Question text, help, required.                           | Type a long text answer.                                              |
| **Performance Score** | A score/rating the supervisor assigns; ties into LMS performance.                                           | Question text, score scale, help, required.              | Assign the performance score.                                         |
| **Date and Time**     | Captures a date and/or time, governed by **date toggle** / **time toggle** (date-only, time-only, or both). | Question text, date toggle, time toggle, help, required. | Pick date and/or time per the enabled toggles; future dates disabled. |
| **File Upload**       | Attach a file as the answer; validates file format/size.                                                    | Question text, accepted formats, help, required.         | Upload a valid file (invalid formats rejected).                       |
| **Signature**         | Capture a signature.                                                                                        | Question text, help, required.                           | Provide the signature.                                                |

**Cross-component rules that hold everywhere:**

- **Question text** must render exactly as authored — including alphanumeric and very large
  (e.g. 2000-word) text.
- **Help text** is **collapsed by default**, **expandable**, and **read-only** during execution;
  it can carry large text (2000-word verification exists).
- **Required** components must be answered before the section/audit can proceed.
- On the **execution side**, the supervisor **cannot** edit question text/help, toggle field
  settings, or delete/duplicate components — those are author-only. (Some of these are known to
  lack execution-side locators today — see §10.)

---

## 8. Canonical end-to-end flows

These are the happy-path journeys the agent should expect. Each step is business intent — resolve
the actual elements live.

**Flow A — Author & publish a template**

1. Apps → CMO → **Templates** tab.
2. **Create Template** → choose **audit type**, **job code(s)**, **global region**, name it,
   associate **facilities**.
3. In the **Template Builder**, add **sections** and **question components**; set each
   component's question text, help text, required, and options/toggles.
4. **Save**, optionally **Preview**, then **Publish**. The template is now available to
   supervisors for its facilities.

**Flow B — Supervisor runs an audit**

1. Apps → CMO → **Supervisors** tab → **My Audits** dashboard.
2. **Select a facility** (must have CMO Goal Setup) → **Start New Audit** becomes available.
3. **Start New Audit** → the **Select Template** sidebar opens → **search/filter** → pick the
   published template (only facility-associated templates appear).
4. **Associate Details** → choose the associate (typeahead) + employment details; read-only
   template fields are shown for reference.
5. **Next** through each **section**, answering every **component**; respect **required** gating.
6. Submit → **Audit Summary** confirms the completed audit.

**Flow C — Configure a facility's goals**

1. Apps → **Manage Configurations** → category **CMO**.
2. **CMO Goal Setup** data table → **Add New** (ops supervisors, work days, direct employees, …)
   or **inline edit/delete** an existing row.
3. Optionally **Export** the report or **Schedule** a recurring report.

**Flow D — Oversight**

- Operations Manager → CMO → **Operations Manager** tab → **My Team** (team audit activity).
- Anyone → CMO → **All CMO** tab → **All CMO Audits** grid (search, filter by date, hide columns,
  read counts).

---

## 9. Business rules & gotchas (read before reasoning about expected results)

1. **Facility ↔ template association.** A template appears for a supervisor **only** for the
   facilities it was associated with at creation. Picking a different facility shows different
   templates. (The framework even remembers the chosen facility so a self-healed re-navigation
   reselects the **same** one — a template won't appear under the wrong facility.)
2. **CMO Goal Setup gates Start New Audit.** If the selected facility has **no** CMO config, the
   **Start New Audit** button is **absent/disabled**. The framework's self-heal for this is to
   create the CMO Goal Setup config (Manage Configurations) and retry — not to force the button.
3. **Default fallback facility** in the audit flow is **"Allentown"** when none is specified.
4. **Mutually-exclusive button groups.** Audit Type (CMO/Facility/Safety/Inventory) and Global
   Region (Americas/Europe/Asia-Pacific) are single-select — exactly one active at a time. Job
   Code can be multiple.
5. **Future dates are disabled** in the Date & Time / date-picker components.
6. **Help text** is collapsed by default and **read-only** in execution; large (2000-word) help
   and question text must render fully.
7. **Execution is read-only for authoring controls.** A supervisor cannot edit question/help
   text, field toggles, or delete/duplicate components during an audit.
8. **Radix/shadcn UI.** Selects open a listbox you click by option text; dialogs are
   `alertdialog`s with Yes/No; spinners are `progressbar`s to wait out (never sleep). Drive them
   the way a user would.
9. **Never fake a pass.** If the live app contradicts an expected result, that is a **product
   bug or a stale story** — record it; do not loosen the check. (Several real bugs are already
   catalogued in `cmo.json` — see §10.)

---

## 10. Known environment / data caveats (from `cmo.json` notes)

The CMO test data records real, observed caveats. Respect them — they are why some checks are
data-blocked, skipped, or flagged as bugs, **not** reasons to weaken a test:

- **No-config / inactive-facility messages may not render in the DOM** (recorded as BUG-D2 /
  BUG-D4): the message a story expects might be absent even though the button is correctly
  disabled. Record the real behaviour.
- **QAAdmin is multi-facility** (e.g. 34 facility options); single-facility-only assertions need
  a single-facility supervisor credential (recorded as XFAIL-D5).
- **Leesport** (a default QAAdmin facility) is **not** in CMO Goal Setup in QA, so some dashboard
  checks are skipped until it's configured (SKIP-D3).
- **Configured facilities** seen in QA include **Allentown PA (80502)** and **Ontario CA
  (75141)**; the Select Template flyer's Facility/Safety/Inventory tabs can legitimately be
  empty, and the flyer's close control may read **"Close"** (not "Cancel").
- **Export/Schedule** (AMCC-9341): Export downloads `CMO_Report_<timestamp>.xlsx` with toast
  "Report Downloaded Successfully" (backend 200); Schedule fires a 201 and the report appears
  under scheduled reports. **Email delivery itself is out of UI scope.**

> These specifics live in `testData/cmo.json` (the source of truth for data) — this list is a
> heads-up, not a substitute. Always re-check live.

---

## 11. Where the CMO code lives (reference map — no locators here)

Use this to find the **existing style to match** when authoring. The live DOM + these files win
over any memory.

| Layer                  | Path                                                                                                                                                                                                                                                                                                                                                               |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Features (Gherkin)** | `tests/features/Apps/CMO/cmo-<STORY-ID>.feature`                                                                                                                                                                                                                                                                                                                   |
| **Step defs**          | `tests/step_defs/apps/CMO/test_cmo-<STORY-ID>.py`                                                                                                                                                                                                                                                                                                                  |
| **Page Objects**       | `object_repository/pages/Apps/CMO/` — incl. `CmoLandingPage.py`, `CmoCreateTemplatePage.py`, `CmoAssociateDetailsPage.py`, `CmoSupervisorPage.py`, `AllCMOAuditsPage.py`, `Supervisor/MyAuditsLandingPage.py`, `Supervisor/SelectTemplateFlyerPage.py`, `Supervisor/NewAuditPage.py`, `Supervisor/FollowUpFlyerPage.py`, `OperationalManager/MyTeamLandingPage.py` |
| **web_component**      | `business_components/web_components/cmo.py` (reusable, assertion-free interactions; `CMO_Navigation`, facility selection, template create/publish, component helpers, `get_`/`set_` cross-step globals)                                                                                                                                                            |
| **Test data**          | `testData/cmo.json`                                                                                                                                                                                                                                                                                                                                                |
| **SSO login helper**   | `business_components/web_components/login.py → login_to_application_sso`                                                                                                                                                                                                                                                                                           |

**Reusable building blocks already in `cmo.py`** (reuse before inventing): `CMO_Navigation`,
`select_facility_for_audit` / `get_selected_facility_name`, All-CMO list helpers
(`navigate_to_all_cmo`, `apply_audit_filters`, `reset_audit_filters`, `search_audit_grid`),
template search/create/publish helpers, the audit-type / job-code / global-region **button maps**,
and `get_`/`set_` globals (`template_name`, `goal_num`, `dimension`, `last_audit_facility`, …).

---

## 12. How the MCP agent should use this file

1. **Load this file first** for any CMO story — before opening the browser — to understand the
   feature's intent and the rules that govern it.
2. **Map the story → the feature area** in §6 so you know which screens/flows you'll touch.
3. **Walk the live DOM** through the `selenium` MCP to resolve the _actual_ elements; this file
   tells you _what_ you're looking for and _why_, never _which selector_.
4. **Author** the pytest-bdd test in the repo's 4-layer architecture, matching the existing CMO
   style (§11) and following the repo's authoring prompts (`feature_file_prompt.txt`,
   `xpath_prompt.txt`, `step_def_prompt.txt`).
5. **On any conflict, the live DOM / Page Objects / specs win** — update your understanding, and
   if the behaviour genuinely differs from the story, flag a possible bug (don't fake a pass).
