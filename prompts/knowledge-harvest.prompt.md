---
description: Systematically harvest knowledge from ALL stories/test cases across CMO, Action Plan, Audit Trail, Manage Config using parallel MCP browser sessions with 5-minute checkpoints.
agent: Explorer
---

# Knowledge Harvest — Systematic Module Understanding via Parallel MCP Walks

> **Purpose**: Systematically walk through ALL stories and test cases of target modules using parallel MCP browser sessions, capturing comprehensive knowledge (locators, flows, behaviors, business rules) to the Knowledge Graph every 5 minutes. This creates a verified, hallucination-proof knowledge base for future automation.

---

## Target Modules

| Module            | Feature Folder                              | Priority |
| ----------------- | ------------------------------------------- | -------- |
| **Action Plans**  | `tests/features/Apps/actionPlans/`          | 1        |
| **CMO**           | `tests/features/Apps/CMO/`                  | 2        |
| **Audit Trail**   | `tests/features/Apps/auditTrail/`           | 3        |
| **Manage Config** | `tests/features/Apps/manageConfigurations/` | 4        |

---

## Phase 0: Preparation (MANDATORY — Run Once at Start)

### 0.1 Bootstrap Knowledge Graph

```
Call: kg_bootstrap(max_sessions=5, max_modules=4)
```

Review what is already known. Do NOT re-walk screens that have high-confidence (>0.8) locators.

### 0.2 Inventory All Stories and Test Cases

For each target module, build a complete inventory:

```sql
-- Create tracking table for harvest progress
CREATE TABLE IF NOT EXISTS harvest_inventory (
    id TEXT PRIMARY KEY,
    module TEXT NOT NULL,
    story_id TEXT NOT NULL,
    feature_file TEXT NOT NULL,
    scenario_name TEXT,
    test_case_id TEXT,
    status TEXT DEFAULT 'pending',
    session_name TEXT,
    last_checkpoint TEXT,
    knowledge_nodes_created INTEGER DEFAULT 0,
    created_at TEXT DEFAULT CURRENT_TIMESTAMP,
    updated_at TEXT
);

CREATE TABLE IF NOT EXISTS harvest_checkpoints (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    harvest_id TEXT,
    checkpoint_time TEXT DEFAULT CURRENT_TIMESTAMP,
    nodes_added INTEGER,
    locators_verified INTEGER,
    flows_captured INTEGER,
    facts_recorded INTEGER,
    notes TEXT,
    FOREIGN KEY (harvest_id) REFERENCES harvest_inventory(id)
);
```

### 0.3 Parse All Feature Files

Read every `.feature` file in target modules and extract:

- Story ID (from filename: `actionPlans-AMCC-4555.feature` → `AMCC-4555`)
- All Scenario/Scenario Outline names
- All Given/When/Then steps (these become the walk instructions)
- Any test case IDs from tags (`@AMCC-TC-XXXX`)

Insert each scenario into `harvest_inventory`.

---

## Phase 1: Parallel Session Setup

### 1.1 Launch Named Browser Sessions

Launch **up to 4 parallel sessions** (one per module) to maximize throughput:

```
Session 1: "ACTIONPLANS" — walks Action Plans module
Session 2: "CMO"         — walks CMO module
Session 3: "AUDITTRAIL"  — walks Audit Trail module
Session 4: "MANAGECONFIG"— walks Manage Config module
```

For each session:

```
Call: start_browser(browser="chrome", session_name="<MODULE>")
Call: login_sso(
    url="https://qa-portal.coldcompass.com",
    username=<from .env QAAdminEmail>,
    password=<from .env QAAdminPassword>,
    success_selector="[data-testid='app-shell']"
)
```

### 1.2 Navigate to Module Landing

Each session navigates to its module's landing page and captures initial state:

```
Call: navigate(url="<module_landing_url>")
Call: wait_for_idle(timeout_ms=15000)
Call: snapshot_dom(max_elements=300)
```

Immediately record the landing page knowledge:

```
Call: kg_upsert(kind="locator", page=<current_url>, element_name="<name>", selector="<xpath>", selector_type="xpath")
```

---

## Phase 2: Systematic Test Case Walk (Per Session)

### 2.1 Walk Algorithm

For each scenario in `harvest_inventory` WHERE `module = <session_module>` AND `status = 'pending'`:

```
1. UPDATE harvest_inventory SET status = 'in_progress', session_name = '<SESSION>' WHERE id = '<scenario_id>'

2. Parse the feature file to get the step sequence:
   - Given steps → navigate to precondition state
   - When steps  → perform the action being tested
   - Then steps  → verify the expected outcome (capture what to check)

3. For EACH step in the scenario:

   a. BEFORE acting, check KG for existing knowledge:
      Call: kg_context_pack(task="<step_text>", module="<MODULE>")

   b. If locator exists with confidence > 0.7 → USE it directly
      If locator exists with confidence 0.4-0.7 → VERIFY it first
      If locator unknown or confidence < 0.4 → PROBE the live DOM

   c. Execute the step action:
      - Navigation: navigate() + wait_for_idle()
      - Click: wait_and_click() or click_element()
      - Type: fill_form() or send_keys()
      - Select: select_radix_option()
      - Verify: get_element_state() or assert_text()

   d. AFTER each action, capture knowledge:
      Call: snapshot_dom(max_elements=200)

      For every NEW element discovered:
      Call: kg_upsert(
          kind="locator",
          page=<current_url>,
          element_name="<descriptive_name>",
          selector="<best_xpath>",
          selector_type="xpath",
          testid="<data-testid if present>"
      )

   e. If action changes page state or navigates:
      Call: kg_upsert(
          kind="navigation",
          page=<from_url>,
          to_page=<to_url>,
          selector="<trigger_element_xpath>"
      )

   f. Record any observed behaviors:
      Call: kg_add_knowledge(
          page=<current_url>,
          topic="<behavior_name>",
          text="<2-6 sentence description of what happens>",
          tag="behaviour"
      )

4. After completing all steps in scenario:
   UPDATE harvest_inventory SET status = 'done', knowledge_nodes_created = <count> WHERE id = '<scenario_id>'
```

### 2.2 Knowledge Categories to Capture

For EVERY screen visited, capture these knowledge types:

| Category           | What to Record                                                      | KG Tool                                 |
| ------------------ | ------------------------------------------------------------------- | --------------------------------------- |
| **Locators**       | Every interactive element (buttons, inputs, links, tabs, dropdowns) | `kg_upsert(kind="locator")`             |
| **Navigation**     | What clicking an element leads to                                   | `kg_upsert(kind="navigation")`          |
| **Actions**        | Multi-step interactions (open dropdown → select → confirm)          | `kg_upsert(kind="action")`              |
| **Behaviors**      | How the UI responds (toast messages, loading states, validation)    | `kg_add_knowledge(tag="behaviour")`     |
| **Business Rules** | Validation rules, required fields, constraints                      | `kg_add_knowledge(tag="business-rule")` |
| **UI Purpose**     | What each screen/section is for                                     | `kg_add_knowledge(tag="ui-purpose")`    |
| **Flows**          | Complete user journeys (create → edit → delete)                     | `kg_upsert(kind="flow")`                |
| **Facts**          | Static truths about the module                                      | `kg_upsert(kind="fact")`                |

---

## Phase 3: 5-Minute Checkpoint Protocol (CRITICAL)

Every **5 minutes of wall-clock time**, ALL sessions MUST pause and checkpoint:

### 3.1 Checkpoint Trigger

Set a timer. When 5 minutes elapse:

```
1. For EACH active session:

   a. Record current progress:
      INSERT INTO harvest_checkpoints (harvest_id, nodes_added, locators_verified, flows_captured, facts_recorded, notes)
      VALUES ('<current_scenario_id>', <counts>, '<summary_of_last_5min>');

   b. Save session state to KG:
      Call: kg_session_save(
          agent="KnowledgeHarvester",
          module="<MODULE>",
          summary="Checkpoint: walked <N> scenarios, captured <M> locators, <P> flows. Currently on <scenario_name>.",
          decisions=["<any locator choices made>", "<any ambiguous elements resolved>"],
          todos=["<remaining scenarios in this module>"],
          next_steps=["Continue from scenario <next_scenario>"],
          worked_on=["<list of page node IDs touched>"]
      )

   c. Reindex knowledge for fresh retrieval:
      Call: kg_reindex()

2. Query overall progress:
   SELECT module,
          COUNT(*) as total,
          SUM(CASE WHEN status='done' THEN 1 ELSE 0 END) as completed,
          SUM(knowledge_nodes_created) as knowledge_nodes
   FROM harvest_inventory
   GROUP BY module;

3. Log progress to console/user:
   "CHECKPOINT @ <time>: ActionPlans <X>/<Y>, CMO <A>/<B>, AuditTrail <C>/<D>, ManageConfig <E>/<F>"
```

### 3.2 Knowledge Quality Gate

At each checkpoint, verify knowledge quality:

```
Call: kg_stats()

Review per-module:
- If FAILED locators > 5 → pause that session, investigate drift
- If STALE locators > 20 → re-verify before continuing
- If confidence avg < 0.6 → slow down, capture more carefully
```

---

## Phase 4: Deep Knowledge Capture Patterns

### 4.1 For Form Screens (Create/Edit)

```
1. Snapshot the empty form state
2. Record ALL input fields:
   - Field label
   - Field type (text, dropdown, checkbox, date, etc.)
   - Required indicator
   - Placeholder text
   - Default value

3. For each field:
   kg_upsert(kind="locator", element_name="<FormName> - <FieldLabel> Input", ...)

4. Record validation behaviors:
   - Try submitting empty → capture error messages
   - Try invalid values → capture validation text
   kg_add_knowledge(topic="<FormName> Validation Rules", text="...", tag="business-rule")

5. Record submit/cancel button locations and behaviors
```

### 4.2 For List/Grid Screens

```
1. Capture table/grid structure:
   - Column headers
   - Row structure
   - Pagination controls
   - Sort controls
   - Filter controls

2. Record search/filter interactions:
   kg_upsert(kind="action", text="Search in <GridName>", effect="Filters grid to matching rows", ...)

3. Record row actions (edit, delete, view):
   kg_upsert(kind="navigation", page=<grid_url>, to_page=<detail_url>, selector=<row_action_xpath>)
```

### 4.3 For Navigation Elements

```
1. Capture all nav tabs/menu items:
   kg_upsert(kind="locator", element_name="<Module> Nav Tab - <TabName>", ...)

2. Record navigation flows:
   kg_upsert(kind="flow", name="<Module> Navigation", steps=[
       {page: "/landing", action: "click", selector: "<tab_xpath>"},
       {page: "/tab-destination", action: "verify", selector: "<content_xpath>"}
   ])
```

### 4.4 For Modal/Dialog Screens

```
1. Record trigger element that opens modal
2. Record modal title and purpose
3. Record all modal controls (inputs, buttons)
4. Record close/cancel/confirm behaviors
5. Record what happens after modal closes

kg_add_knowledge(
    topic="<ModalName> Dialog Behavior",
    text="Opened by clicking <trigger>. Contains <fields>. Confirm saves <action>. Cancel discards changes.",
    tag="behaviour"
)
```

---

## Phase 5: Cross-Module Knowledge Linking

After completing individual module walks, create cross-references:

```
1. Identify shared components:
   - Common header/footer elements
   - Shared navigation patterns
   - Reused form components

2. Link related nodes:
   Call: kg_link(from_id="<page_node>", to_id="<related_page_node>", rel="related")

3. Create dependency links:
   Call: kg_link(from_id="<feature_flow>", to_id="<auth_flow>", rel="depends_on")
```

---

## Phase 6: Final Knowledge Validation

### 6.1 Coverage Report

```sql
SELECT
    module,
    COUNT(DISTINCT story_id) as stories_covered,
    COUNT(*) as scenarios_walked,
    SUM(knowledge_nodes_created) as total_knowledge_nodes,
    ROUND(AVG(knowledge_nodes_created), 1) as avg_nodes_per_scenario
FROM harvest_inventory
WHERE status = 'done'
GROUP BY module;
```

### 6.2 Knowledge Graph Health Check

```
Call: kg_stats()

Verify:
- Each module has page nodes for all major screens
- Locator confidence avg > 0.7
- Zero FAILED locators (all fixed)
- Flows captured for key user journeys
```

### 6.3 Final Session Save

```
Call: kg_session_save(
    agent="KnowledgeHarvester",
    summary="HARVEST COMPLETE: Walked <total> scenarios across 4 modules. Created <N> locator nodes, <M> navigation edges, <P> behavior facts. Knowledge graph ready for test authoring.",
    decisions=["<major locator decisions>", "<ambiguity resolutions>"],
    todos=["<any screens that need re-walk>", "<any gaps identified>"],
    next_steps=["Run kg_context_pack for any new test authoring task"],
    worked_on=["<all module node IDs>"]
)
```

---

## Execution Commands

### Start Full Harvest (All 4 Modules in Parallel)

```
/knowledge-harvest --modules ActionPlans,CMO,AuditTrail,ManageConfig --parallel 4 --checkpoint-interval 5m
```

### Start Single Module Harvest

```
/knowledge-harvest --modules CMO --parallel 1 --checkpoint-interval 5m
```

### Resume Interrupted Harvest

```
/knowledge-harvest --resume --from-checkpoint
```

### View Harvest Progress

```sql
SELECT * FROM harvest_inventory WHERE status != 'done' ORDER BY module, story_id;
```

---

## Anti-Hallucination Guarantees

This prompt enforces these guarantees to prevent hallucination:

1. **Every locator comes from live DOM** — `snapshot_dom()` before any `kg_upsert(kind="locator")`
2. **Every navigation verified by actual click** — no assumed page transitions
3. **Every behavior observed in real browser** — no invented UI responses
4. **5-minute checkpoints** — frequent saves prevent knowledge loss
5. **Confidence scores decay** — old unverified knowledge becomes PROBE-worthy
6. **Failed locators tracked** — drift is recorded, not hidden
7. **Session memory persists** — `kg_session_save` enables `/resume` continuity

---

## Output Artifacts

After harvest completion:

| Artifact        | Location                        | Purpose                                  |
| --------------- | ------------------------------- | ---------------------------------------- |
| Knowledge Graph | `knowledge/*.md`                | Verified locators, flows, facts per page |
| Module Digests  | `knowledge/modules/<module>.md` | Per-module summaries                     |
| Session History | `knowledge/sessions/`           | Checkpoint trail for auditing            |
| Index           | `knowledge/index.json`          | Fast node lookup                         |
| Obsidian Hub    | `knowledge/_index.md`           | Visual graph navigation                  |

---

## Error Recovery

### If a session crashes:

```
1. Query last checkpoint: SELECT * FROM harvest_checkpoints WHERE harvest_id LIKE '<module>%' ORDER BY checkpoint_time DESC LIMIT 1;
2. Restart session from that scenario
3. Mark interrupted scenario as 'pending' to re-walk
```

### If locator fails during walk:

```
1. Call: kg_record_failure(selector="<failed_xpath>", reason="<what_changed>")
2. Call: snapshot_dom() to get fresh DOM
3. Find new locator and continue
4. This is VALUABLE — drift detection feeds future healing
```

### If login expires:

```
1. Call: login_sso(...) to re-authenticate
2. Navigate back to last page
3. Continue from current step
```

---

## Usage Example

To start harvesting all 4 modules:

```
User: @workspace /knowledge-harvest

Agent will:
1. Bootstrap KG and check existing knowledge
2. Parse all feature files in target modules
3. Launch 4 parallel browser sessions
4. Walk every scenario, capturing knowledge
5. Checkpoint every 5 minutes
6. Save comprehensive module understanding to KG
7. Report completion with coverage stats
```

After harvest, any future test authoring benefits from this verified knowledge:

```
Call: kg_context_pack(task="Create template in CMO", module="CMO")
→ Returns USE/VERIFY/PROBE locators, flows, and behaviors — zero hallucination
```
