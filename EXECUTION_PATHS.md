# OpenJarvis — Tool Execution Paths and Authorization Traces

**Target System:** OpenJarvis Autonomous Personal AI Assistant  
**Repository Working Tree:** `D:/Jarvis/OpenJarvis`  
**Inspection Date:** 2026-11-23  
**Mode:** Source-of-Truth Inspection (Local Disk Only)

---

## Executive Summary

This document traces every execution path that originates, authorizes, approves, and executes tools within OpenJarvis. It analyzes all seven operational entry points:
1. Command Line Interface (CLI)
2. HTTP / REST API
3. Agent Orchestration (`ToolUsingAgent`)
4. Background Task Scheduler (`TaskScheduler`)
5. Model Context Protocol (`MCPServer` & `MCPToolAdapter`)
6. Browser Automation (`BrowserTool` suite)
7. Managed Background Agents and Channel Bridges

Each path is mapped to its exact entry point, intermediate dispatch mechanics, policy enforcement points, approval gates, audit mechanisms, and potential bypass risks.

---

## Execution Path Matrix

| Entry Route | Initiating Component | Enforcement Gate | Approval Gate | Execution Engine | Audit Logging | Bypass Potential |
|---|---|---|---|---|---|---|
| **1. CLI** | `src/openjarvis/cli/tool_cmd.py` | `PolicyEngine.evaluate()` | Interactive CLI / `ApprovalStore` | `_BoundedToolRunner` | `AuditLogger._on_event` | **None** (Enforced) |
| **2. HTTP API** | `src/openjarvis/server/approval_routes.py` | `AuthMiddleware` + `PolicyEngine` | `ApprovalStore` REST API | `ToolExecutor` / Subsystems | `AuditLogger._on_event` | **None** (Enforced) |
| **3. Agent Loop** | `src/openjarvis/system/orchestrator.py` | `CapabilityPolicy` + `PolicyEngine` | `ApprovalStore` Queue & Wait | `ToolExecutor.execute()` | `AuditLogger._on_event` | **None** (Enforced) |
| **4. Scheduler** | `src/openjarvis/scheduler/scheduler.py` | `CapabilityPolicy("scheduler")` | Consequential -> `ApprovalStore` | `ToolExecutor.execute()` | `AuditLogger._on_event` | **None** (Enforced) |
| **5. MCP Outbound**| `src/openjarvis/tools/mcp_adapter.py` | Schema Hash + Mutating Check | Unapproved -> `ApprovalStore` | JSON-RPC Client | `AuditLogger._on_event` | **None** (Enforced) |
| **6. Browser** | `src/openjarvis/tools/browser.py` | SSRF Proxy + Network Guard | DOM Revalidation + `ApprovalStore`| Playwright Engine | `AuditLogger._on_event` | **None** (Enforced) |
| **7. Managed Agt**| `src/openjarvis/agents/manager.py` | `wire_agent_security()` | Profile-based Approval | `ToolExecutor.execute()` | `AuditLogger._on_event` | **None** (Enforced) |

---

## Detailed Trace by Entry Route

### Route 1: Command Line Interface (CLI)

- **Entry Point:** `src/openjarvis/cli/tool_cmd.py:run(name, ...)`
- **Detailed Call Sequence:**
  ```text
  CLI Command: jarvis tool run <name> [args]
    --> cli/tool_cmd.py:run()
      --> ToolRegistry.get(name) -> tool instance
      --> ToolExecutor([tool], confirm_callback=interactive_confirm, user_id=cli_user)
        --> ToolExecutor.execute(tool_call)
          --> PolicyEngine.evaluate(tool_name, params, context)
            --> [Decision.DENY] -> Raise SecurityError & Exit
            --> [Decision.REQUIRE_APPROVAL] -> Call interactive_confirm()
                --> Compute sha256(tool:target:canonical_params)
                --> On user confirm: mark_executed() with matching hash
            --> [Decision.ALLOW] -> Proceed directly
          --> _BoundedToolRunner.run(tool.execute, **params)
          --> AuditLogger.log(EventType.TOOL_EXECUTION)
  ```
- **Security & Authorization Verification:**
  - Evaluates `PolicyEngine` rules before invoking tool logic.
  - Interactive confirmation checks action payload hash against command arguments.
  - Direct shell or file write operations cannot bypass workspace or approval policies.
- **Bypass Vulnerability Analysis:** Resolved. In legacy versions, `tool.execute()` was called directly from Typer CLI handlers without `ToolExecutor`. Current source binds CLI calls through `ToolExecutor`.

---

### Route 2: HTTP / REST API Direct and Approved Execution

- **Entry Points:**
  - `src/openjarvis/server/approval_routes.py`:
    - `GET /api/v1/approvals/pending`
    - `POST /api/v1/approvals/{action_id}/decide`
  - `src/openjarvis/server/agent_manager_routes.py`: Agent execution endpoints.
- **Detailed Call Sequence:**
  ```text
  Client Request: POST /api/v1/approvals/{action_id}/decide
    --> auth_middleware.py:AuthMiddleware.dispatch()
      --> Verify Authorization: Bearer <token> against OPENJARVIS_API_KEY
      --> Check X-User-Id header: If present and != authenticated principal -> HTTP 403
      --> Set request.state.authenticated_user = "admin"
    --> approval_routes.py:decide_action()
      --> Load action from ApprovalStore.get_action(action_id)
      --> Ownership Gate: If action.user_id != principal and principal != "admin" -> HTTP 403
      --> Update SQLite status to "approved" or "denied"
      --> If approved, ToolExecutor.execute() calls ApprovalStore.mark_executed():
          --> Atomic SQL: UPDATE pending_actions SET status='executed'
                          WHERE id=? AND status='approved' AND user_id=? AND task_id=?
          --> If rows_affected == 0 -> ABORT (Prevents concurrency race & replay)
      --> Execute tool via _BoundedToolRunner
      --> Persist event in AuditLogger with linear hash chain
  ```
- **Security & Authorization Verification:**
  - Caller authentication mandatory when API key is set.
  - Action ownership strictly enforced: callers cannot approve actions owned by other users.
  - Concurrency lock: SQL update requires `status='approved'`, `user_id`, and `task_id` match.
- **Bypass Vulnerability Analysis:** Resolved. Caller spoofing via `X-User-Id` is rejected; empty owner bypass is eliminated.

---

### Route 3: Agent Orchestration (`QueryOrchestrator` -> `ToolUsingAgent`)

- **Entry Points:**
  - `src/openjarvis/system/orchestrator.py`: `QueryOrchestrator.dispatch()`, `_run_agent()`
  - `src/openjarvis/agents/_stubs.py`: `ToolUsingAgent.step()`, `run()`
- **Detailed Call Sequence:**
  ```text
  User Prompt / Task
    --> QueryOrchestrator.dispatch(query, user_id=...)
      --> Propagate authenticated user_id and task_id into AgentContext
      --> ToolUsingAgent.step(context)
        --> Engine (Ollama): Local LLM inference generates ToolCall(name, args)
        --> ToolExecutor.execute(tool_call, context)
          --> CapabilityPolicy.check(agent_role, tool_name)
          --> PolicyEngine.evaluate(tool_name, args, context)
            --> Boundary check: SSRF / Path Traversal / Sensitive credentials
            --> Taint check: Restricted sink verification (taint.py)
            --> If REQUIRE_APPROVAL:
                --> ApprovalStore.queue_action(..., action_payload_hash)
                --> Notify via SSE / Tray / Console and block until decided
                --> Validate executed hash matches queued action payload hash
          --> Execute tool in _BoundedToolRunner
          --> Publish Event to EventBus -> AuditLogger
  ```
- **Security & Authorization Verification:**
  - Agent role mapped to profile (`primary`, `subagent`, `scheduler`).
  - `PolicyEngine` evaluates every tool call; absolute `DENY` cannot be overridden.
  - Tainted secrets blocked from leaving local context.
- **Bypass Vulnerability Analysis:** Verified solid. `ToolUsingAgent` does not possess private references to raw tool objects; all tool dispatch is mediated by `ToolExecutor`.

---

### Route 4: Background Task Scheduler (`TaskScheduler`)

- **Entry Points:**
  - `src/openjarvis/scheduler/scheduler.py`: `TaskScheduler._execute_task()`, `ScheduledTask`
- **Detailed Call Sequence:**
  ```text
  Cron / Interval Trigger
    --> TaskScheduler._execute_task(task)
      --> Construct TaskContext(agent_id="scheduler", user_id=task.user_id, task_id=task.id)
      --> SystemBuilder._resolve_tools() creates ToolExecutor with agent_id="scheduler"
      --> ToolExecutor.execute()
        --> CapabilityPolicy.check("scheduler", tool_name)
            --> Grants: file_read, safe notifications
            --> Denies: shell_exec, code_execute, file_write
        --> PolicyEngine.evaluate()
            --> Consequential mutating tools evaluate to REQUIRE_APPROVAL
            --> Queues in ApprovalStore; scheduler task pauses or logs pending approval
      --> Action audited with scheduler context in AuditLogger
  ```
- **Security & Authorization Verification:**
  - Least privilege: Scheduler profile denies code execution and arbitrary writes.
  - Autonomous privilege escalation prevented: Mutating actions cannot self-approve.
- **Bypass Vulnerability Analysis:** Verified. `TaskScheduler` explicitly sets `agent_id="scheduler"` on its context and executor.

---

### Route 5: Model Context Protocol (MCP)

- **Entry Points:**
  - `src/openjarvis/tools/mcp_adapter.py`: `MCPToolAdapter.execute()` (Outbound)
  - `src/openjarvis/mcp/server.py`: `MCPServer._handle_call_tool()` (Inbound)
- **Detailed Call Sequence (Outbound to External MCP):**
  ```text
  Agent Call: MCPToolAdapter.execute(**params)
    --> Schema Trust Verification:
        --> Compute current schema hash: sha256(name:description:params:category)
        --> Compare with persisted hash in mcp_trust.json
        --> If mutated or unrecorded -> Requires human re-confirmation
    --> Autonomous Execution Qualification:
        --> Server in trusted_mcp_servers allowlist?
        --> Tool in trusted_mcp_tools allowlist?
        --> Mutating tokens (delete, write, patch, remove) absent?
        --> Sensitive parameter names absent?
        --> If ANY check fails -> Deny autonomous execution -> Require approval
    --> Taint Sink Guard:
        --> Check incoming arguments for TaintLabel.SECRET
        --> If tainted -> Block dispatch (Taint policy violation)
    --> Dispatch JSON-RPC to external MCP endpoint
    --> Audit execution in AuditLogger
  ```
- **Detailed Call Sequence (Inbound to OpenJarvis MCP Server):**
  ```text
  External JSON-RPC Client -> MCPServer: tools/call
    --> MCPServer._executor.execute(tool_call)
    --> PolicyEngine and workspace allowed_dirs enforced
  ```
- **Bypass Vulnerability Analysis:** Verified. Passing `requires_confirmation=False` on the public adapter constructor cannot bypass server allowlist, mutating token filters, or schema mutation verification.

---

### Route 6: Browser Automation (`BrowserTool` Suite)

- **Entry Points:**
  - `src/openjarvis/tools/browser.py`: `BrowserNavigateTool`, `BrowserClickTool`, `BrowserTypeTool`
- **Detailed Call Sequence:**
  ```text
  Browser Tool Invocation
    --> SSRF Defense Layer:
        --> SSRFSafeBrowserProxy on 127.0.0.1 (bounded thread pool max 8)
        --> Playwright launched with proxy routing
        --> _setup_page_network_guard() attaches page.route("**/*")
        --> Destination IP pre-resolved and checked against private/metadata IPs
    --> Action Authorization Layer:
        --> BrowserNavigateTool:
            --> Validate HTTP/HTTPS protocol and public IP -> Autonomous
        --> BrowserTypeTool:
            --> Inspect target field: If search query on Google/Bing -> Autonomous
            --> If password/card/sensitive -> Require approval
            --> Redact typed text in action summary and approval payload ([REDACTED INPUT])
        --> BrowserClickTool:
            --> Inspect DOM element (tag, role, href, onclick, insideForm)
            --> If safe navigation link / disclosure -> Autonomous
            --> If form submit / mutating / JS handler -> Require approval
            --> TOCTOU Target Revalidation: Immediately before click(), re-inspect element
                --> If attributes/href mutated during approval -> ABORT execution
    --> Audit log action with scrubbed input text
  ```
- **Bypass Vulnerability Analysis:** Resolved. TOCTOU DOM substitution is blocked by pre-execution revalidation; loopback SSRF proxy and page route guards fail closed.

---

### Route 7: Managed Background Agents and Channel Bridges

- **Entry Points:**
  - `src/openjarvis/agents/manager.py`: `AgentManager.run_agent()`
  - `src/openjarvis/server/channel_bridge.py`: Messaging webhooks
- **Detailed Call Sequence:**
  ```text
  Incoming Webhook / Channel Event (e.g. Twilio / SendBlue)
    --> channel_bridge.py:ChannelBridge.handle_incoming()
    --> Authenticate webhook signature
    --> Spawn / dispatch to managed ToolUsingAgent
    --> wire_agent_security() initializes CapabilityPolicy and ToolExecutor
    --> Tool execution routes through ToolExecutor.execute()
    --> Normal PolicyEngine and ApprovalStore rules apply
  ```
- **Bypass Vulnerability Analysis:** Verified. Managed agents run with security primitives wired by `wire_agent_security()`.

---

## Conclusion: Verification of Central Enforcement

Across all seven execution routes:
1. **No direct tool execution bypasses `ToolExecutor`:** Every tool call is routed through `ToolExecutor.execute()`.
2. **`PolicyEngine` is the authoritative PEP:** No component can execute without evaluating capability checks, security boundaries, and approval requirements.
3. **Approvals are tamper-resistant:** Bound cryptographically to `sha256(tool:target:canonical_params)` and locked conditionally in SQLite.
4. **Audit logging is comprehensive:** Every execution, rejection, and taint declassification is written to the cryptographic hash chain.
