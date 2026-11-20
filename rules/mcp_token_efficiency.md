---
name: mcp_token_efficiency
version: 1.0.0
priority: P0
trigger: always_on
---

# MCP Token Efficiency & Execution Optimization Rules

> **MANDATORY PROTOCOL**: Selenium MCP tools MUST be driven with minimal token overhead without compromising test fidelity or ground-truth verification.

---

## 1. Fast-Tool Collapsing (Mandatory ~70% Token Reduction)

When interacting with the live browser via Selenium MCP, agents MUST collapse individual tool call chains into **fast tool batch invocations**:

| UI Operation Sequence | FORBIDDEN (High Tokens) | MANDATORY (Token Optimized) |
| :--- | :--- | :--- |
| **SSO Login** | `navigate` $\rightarrow$ `find_element` $\rightarrow$ `send_keys` $\rightarrow$ `click` $\rightarrow$ ... (10+ calls) | **`login_sso`** (1 call) |
| **Filling Forms** | N $\times$ (`find_element` $\rightarrow$ `send_keys` $\rightarrow$ `clear`) | **`fill_form`** (1 call) |
| **Multi-Click Sequence** | N $\times$ (`wait_for_element` $\rightarrow$ `click_element`) | **`click_sequence`** (1 call) |
| **Combobox Selection** | `click` trigger $\rightarrow$ `wait` popup $\rightarrow$ `click` option | **`select_radix_option`** (1 call) |
| **Text Validations** | N $\times$ `get_element_text` + manual comparison | **`assert_text`** / **`get_texts`** (1 call) |
| **Spinner / Load Gate** | Polling `execute_script` or repeated `wait_for_element` | **`wait_for_idle`** (1 call) |

---

## 2. Developer UI Repo & Knowledge-Graph Precedence (Zero Redundant DOM Probes)

1. **Inspect Developer UI Code First**: Check `C:\Users\Administrator\UIREPOAMCC\amc-cold-compass-portal-ui\apps\<module_name>` to extract ground-truth `data-testid` values, ARIA attributes, and React state logic before probing live DOM.
2. **Check KG Second**: Call `kg_context_pack(module=..., task=...)` to retrieve token-budgeted locators and flows.
3. **Reuse Verified Locators**: If the target XPath is already verified in the UI Repo or Knowledge Graph, directly invoke `wait_and_click` or `send_keys` with the verified XPath.
4. **No Re-probing**: Never run `snapshot_dom` or `find_element` on elements whose XPaths are already verified unless a test execution failure indicates UI drift.

---

## 3. DOM Payload Trimming

1. **No Raw HTML Dumps**: Never dump raw 50KB+ HTML payloads into the conversation window.
2. **Compact Snapshots**: Use `snapshot_dom(interactive_only=true, max_elements=50)` to inspect new screens. It returns a lightweight JSON tree under 5KB containing only interactable elements.
3. **Targeted Element Queries**: Use `get_element_state` or `find_element` with specific XPaths when diagnosing a single target element.

---

## 4. Single-Session Lifecycle

1. **Reuse Active Sessions**: Maintain a single active browser session (`list_sessions` / `switch_session`) across all scenarios of a story walk. Do not restart Chrome between individual test cases.
2. **Cookie Warmup Integration**: Inject cached authentication cookies directly into the session via `login_sso` or `conftest.py` to bypass full Azure-AD SSO redirects.

---

## 5. Anti-Hallucination & Ground-Truth Verification Protocol (MANDATORY)

1. **DOM Verification Before Action**: If an element locator is not found in `kg_context_pack` or POM, call `snapshot_dom(interactive_only=true, max_elements=50)` before attempting any interaction. NEVER guess XPaths.
2. **Strict Verification Tools**: Validate step completion live using `assert_text` or `get_texts`. Never mark a step PASSED without verifying the expected text or element state on screen.
3. **Recovery Inspection**: If an MCP action fails or an element is missing, run `snapshot_dom(interactive_only=true)` to inspect current screen state before taking recovery steps.
