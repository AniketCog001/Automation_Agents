---
name: core-protocol
version: 1.0.0
priority: P0
trigger: always_on
---

# Core Protocol - AG Kit

> The highest-priority workspace rules. How the AI loads agents/skills and what it must do before any implementation.

---

## CRITICAL: AGENT & SKILL PROTOCOL (START HERE)

> **MANDATORY:** You MUST read the appropriate agent file and its skills BEFORE performing any implementation. This is the highest priority rule.

### 1. Modular Skill Loading Protocol

Agent activated → Check frontmatter "skills:" → Read SKILL.md (INDEX) → Read specific sections.

- **Selective Reading:** DO NOT read ALL files in a skill folder. Read `SKILL.md` first, then only read sections matching the user's request.
- **Token Efficiency Protocol:** Use `kg_context_pack` to fetch concise, token-budgeted context. Use Selenium MCP fast tools (`fill_form`, `click_sequence`, `login_sso`, `select_radix_option`) to minimize tool-call roundtrips and cut token consumption by ~70%.
- **Rule Priority:** P0 (Workspace Rules in `.agents/rules/`) > P1 (Agent `.md`) > P2 (SKILL.md). All rules are binding.

### 1.1 Skill Announcement (MANDATORY)

**Every time you load and apply a skill, announce it BEFORE using it** — so the user can verify which knowledge is active.

```markdown
📚 **Using skill: `@[skill-name]`...**
```

- List multiple skills together: `📚 Using skills: @frontend-design + @design-spec...`
- Announce on-demand skills too (e.g. a companion skill pulled from a hub, or `app-builder` for a new app), not just frontmatter ones.
- ❌ Applying a skill without announcing it = **USER CANNOT VERIFY THE SKILL WAS USED**.

### 1.2 Anti-Hallucination & Live DOM Verification Protocol (MANDATORY)

- **Zero Guesswork / Fabrication**: Never invent element XPaths, dropdown values, labels, or application flows.
- **Inspect Before Authoring**: When generating Page Objects or step definitions, all locators MUST come from `kg_context_pack` or live MCP `snapshot_dom(interactive_only=true)`.
- **Empirical Execution**: When walking test steps via Selenium MCP tools, verify every UI state change with `assert_text`, `get_texts`, or `get_element_state` before declaring a step PASSED.
- **Self-Correction Guard**: If an MCP action fails or an element is missing, take a fresh `snapshot_dom` to inspect current UI state before attempting any recovery.

### 1.3 Developer UI Repo Source Code Analysis Protocol (MANDATORY)

- **UI Repo Path**: `C:\Users\Administrator\UIREPOAMCC\amc-cold-compass-portal-ui\apps\<module_name>`
- **Mandatory Source Inspection**: Before authoring or healing any Page Object, Web Component, Step Definition, or Feature File, the agent MUST inspect the target module's Next.js component files (`app/_components/`, `app/_hooks/`, `app/_common/`).
- **Ground-Truth Extraction**: Extract exact `data-testid` attributes, ARIA attributes (`aria-expanded`, `role`), initial React state hooks (`useState`), component nesting, and API field mappings directly from developer components. Never guess element states or selectors.

### 2. Enforcement Protocol

1. **When agent is activated:**
    - ✅ Activate: Read Rules → Check Frontmatter → Load SKILL.md → Apply All.
2. **Forbidden:** Never skip reading agent rules or skill instructions. "Read → Understand → Apply" is mandatory.

---

## 📁 File Dependency Awareness

**Before modifying ANY file:**

1. If `CODEBASE.md` exists, check its File Dependencies section.
2. Otherwise, discover dependencies with targeted search/import analysis; do not block waiting for a missing file.
3. Identify dependent files and update all affected files together.

---

## 🗺️ System Map & Memory Read

> 🔴 **MANDATORY:** At session start, you MUST read `.agents/memory/MEMORY.md` to load persistent project conventions, user preferences, and decisions.

> 📚 **Catalog lookup (on-demand, NOT every session):** Need the full list of Agents / Skills / Scripts? The `quick-reference` rule has the essentials. For the complete catalog, read `.agents/ARCHITECTURE.md` only when you actually need it (e.g. orchestration, or discovering a skill you're unsure exists) — do NOT load it on every request.

**Path Awareness (Note: the project directory name is `.agents` plural):**

- Agents: `.agents/agent/` (Project)
- Skills: `.agents/skills/` (Project)
- Memory: `.agents/memory/` (Project)
- Runtime Scripts: `.agents/skills/<skill>/scripts/`

---

## 🧠 Read → Understand → Apply

```
❌ WRONG: Read agent file → Start coding
✅ CORRECT: Read → Understand WHY → Apply PRINCIPLES → Code
```

**Before coding, answer:**

1. What is the GOAL of this agent/skill?
2. What PRINCIPLES must I apply?
3. How does this DIFFER from generic output?

---
