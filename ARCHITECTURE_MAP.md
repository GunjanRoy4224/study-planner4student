# OpenJarvis — Comprehensive Architecture and Security Map

**Target System:** OpenJarvis Autonomous Personal AI Assistant  
**Repository Working Tree:** `D:/Jarvis/OpenJarvis`  
**Inspection Date:** 2026-11-23  
**Mode:** Source-of-Truth Inspection (Local Disk Only)

---

## Overview

This architecture map represents the verified implementation of OpenJarvis based strictly on the local working tree at `D:/Jarvis/OpenJarvis`. Every component, call sequence, security check, and failure mode documented below was verified directly against source files on disk.

---

## 1. Application Startup and CLI

- **Primary Entry Points:**
  - `src/openjarvis/cli/main.py`: `app = typer.Typer(...)`, `cli()` entry point registering subcommands (`ask`, `run`, `serve`, `tool`, `agent`, `schedule`, `doctor`).
  - `src/openjarvis/cli/ask.py`: `ask(prompt: str, ...)` creates interactive CLI agent loop.
  - `src/openjarvis/cli/tool_cmd.py`: `run(name: str, ...)` dispatches direct tool executions from CLI.
  - `src/openjarvis/server/app.py`: `create_app(...)` factory building the FastAPI application.
  - `src/openjarvis/server/main.py`: Uvicorn server launcher for HTTP/WebSocket service.
- **Call Sequence:**
  1. CLI parses arguments via Typer.
  2. `load_config()` (`src/openjarvis/core/config.py`) loads `config.toml`, honoring `OPENJARVIS_CONFIG_PATH` or `~/.openjarvis/config.toml`.
  3. `setup_security(config, engine, bus)` (`src/openjarvis/security/__init__.py`) initializes `CapabilityPolicy`, `RateLimiter`, and `AuditLogger`.
  4. System components wired through `SystemBuilder` (`src/openjarvis/system/builder.py`).
- **Security Checks:**
  - Config directory path traversal rejection via `core/paths.py:get_config_dir()`.
  - API key check if server launched or remote requests received.
- **Failure Behavior:** Missing or malformed config falls back to safe defaults; security subsystem initialization errors halt startup (fail closed).
- **Bypass Risk:** CLI direct tool commands historically bypassed agent policies; now routed through `ToolExecutor` and `PolicyEngine`.

---

## 2. Agent Orchestration

- **Primary Entry Points:**
  - `src/openjarvis/system/orchestrator.py`: `QueryOrchestrator.dispatch()`, `_run_agent()`.
  - `src/openjarvis/agents/_stubs.py`: `ToolUsingAgent.step()`, `ToolUsingAgent.run()`.
- **Call Sequence:**
  1. User prompt delivered to `QueryOrchestrator`.
  2. Orchestrator constructs `AgentContext` and injects `user_id` (derived from auth principal or `OPENJARVIS_USER_ID`).
  3. `ToolUsingAgent` invokes LLM inference engine with prompt, system instructions, and tool definitions.
  4. Engine returns structured tool calls (`ToolCall(name, arguments)`).
  5. Agent dispatches calls to its bound `ToolExecutor.execute()`.
- **Security Checks:**
  - Agent role mapped to capability profile (`primary`, `scheduler`, `subagent`).
  - Context propagation guarantees `user_id` and `task_id` follow every dispatch.
- **Approval Requirements:** Dependent on tool classification by `PolicyEngine`.
- **Failure Behavior:** Engine connection errors retry with backoff; tool errors returned to agent as error tool results for reflection.
- **Bypass Risk:** If agent bypassed `ToolExecutor` to invoke tools directly, policy checks would not run. In implementation, agents only possess reference to `ToolExecutor`.

---

## 3. Tool Registry and ToolExecutor

- **Primary Entry Points:**
  - `src/openjarvis/tools/_stubs.py`: `ToolRegistry`, `ToolExecutor`, `_BoundedToolRunner`, `BaseTool`.
- **Call Sequence:**
  1. `ToolRegistry.register(name)` decorates tool classes and populates registry dictionary.
  2. `ToolExecutor.__init__(tools, confirm_callback, user_id, task_id)` binds available tools.
  3. `ToolExecutor.execute(tool_call, context)`:
     - Checks tool existence in local mapping.
     - Enforces capability check via `CapabilityPolicy.check(agent_id, tool_name)`.
     - Calls `PolicyEngine.evaluate(tool_name, arguments, context)`.
     - If `REQUIRE_APPROVAL`, queues action in `ApprovalStore` and waits or prompts.
     - Dispatches execution to `_BoundedToolRunner` (capped daemon thread pool).
     - Records result in `AuditLogger` and emits `EventType.TOOL_EXECUTION`.
- **Security Checks:**
  - Capability check (`CapabilityPolicy`).
  - PolicyEngine boundary checks (SSRF, path traversal, sensitive files).
  - Taint propagation and restricted sink validation (`check_taint`).
- **Failure Behavior:** Returns `ToolResult(success=False, content="...")`. Never crashes host process.
- **Bypass Risk:** None for registered tools routed through `execute()`.

---

## 4. Central Authorization and Security Enforcement

- **Primary Entry Points:**
  - `src/openjarvis/security/policy_engine.py`: `PolicyEngine.evaluate()`, `Decision`, `EvaluationResult`.
  - `src/openjarvis/security/capabilities.py`: `CapabilityPolicy`, `CapabilityProfile`.
- **Call Sequence:**
  1. `ToolExecutor` passes tool name, parameters, and calling context to `PolicyEngine.evaluate()`.
  2. `PolicyEngine` determines target resource (URL, filepath, command string).
  3. Static rules and security checks evaluate the target:
     - `http_request`: Resolved IP pre-check. If private, loopback, or metadata -> `Decision.DENY`.
     - `file_write`, `apply_patch`: If target outside `allowed_dirs` or is sensitive credential file -> `Decision.DENY`.
     - `shell_exec`, `docker_shell_exec`: Consequential -> `Decision.REQUIRE_APPROVAL`.
     - Safe search / read-only actions -> `Decision.ALLOW`.
  4. Returns `EvaluationResult(decision, reason, action_payload_hash)`.
- **Enforcement Rule:** `Decision.DENY` is ABSOLUTE. No human approval token or database injection can override `DENY`.
- **Failure Behavior:** Denied actions abort execution and return `SecurityError`.

---

## 5. Approval Creation, Ownership, Persistence, and API Routes

- **Primary Entry Points:**
  - `src/openjarvis/tools/approval_store.py`: `ApprovalStore`, `PendingAction`, `queue_action()`, `decide_action()`, `mark_executed()`.
  - `src/openjarvis/server/approval_routes.py`: FastAPI router mounting `/api/v1/approvals/*`.
- **Call Sequence:**
  1. Consequential action evaluated as `REQUIRE_APPROVAL`.
  2. `ApprovalStore.queue_action()` computes `action_payload_hash = sha256(tool:target:canonical_params)`.
  3. Action persisted to SQLite table `pending_actions` with `status="pending"`, `user_id`, `task_id`, `created_at`, `ttl`.
  4. `ApprovalNotifier` notifies CLI interactive prompt, SSE event stream, or desktop tray.
  5. User posts decision to `/api/v1/approvals/{action_id}/decide`.
  6. API verifies caller authentication and action ownership (`action.user_id == caller or caller == "admin"`).
  7. On approval, status updated to `"approved"`.
  8. `ToolExecutor` calls `mark_executed(action_id, expected_user_id, expected_task_id)`:
     - Atomic SQL: `UPDATE pending_actions SET status = 'executed' WHERE id = ? AND status = 'approved' AND user_id = ? AND task_id = ?`.
     - If rows updated == 0, execution aborts (prevents double execution and cross-user replay).
- **Failure Behavior:** Expired actions or rejected actions fail closed. Rejections are marked `"denied"` permanently and never convert to `"expired"`.

---

## 6. Browser Automation, Browser Sessions, and Network Proxy

- **Primary Entry Points:**
  - `src/openjarvis/tools/browser.py`: `BrowserTool`, `BrowserNavigateTool`, `BrowserClickTool`, `BrowserTypeTool`, `SSRFSafeBrowserProxy`, `_setup_page_network_guard`.
- **Call Sequence:**
  1. Navigation: `BrowserNavigateTool.execute(url=...)`. URL validated against SSRF blocklist and pinned IP.
  2. Proxy Startup: `SSRFSafeBrowserProxy` runs on `127.0.0.1:port` with bounded `ThreadPoolExecutor(max_workers=8)`.
  3. Playwright browser launched with `--proxy-server=http://127.0.0.1:port`.
  4. Page Guard: `_setup_page_network_guard(page)` hooks `page.route("**/*")`. If registration fails, page closes and navigation aborts.
  5. Interaction (Click/Type):
     - Element inspected in DOM for safety (`tag`, `role`, `href`, `hasJsHandler`).
     - Autonomous typing permitted only on search inputs (`q`, `query`, Google, Bing).
     - Sensitive fields (passwords, cards) require explicit approval.
     - Target Revalidation (TOCTOU Defense): Immediately prior to click, target element is re-queried to ensure attributes have not mutated.
  6. Session Isolation: `_current_browser_session_id` context variable partitions cookies and storage per task.
- **Failure Behavior:** Blocked requests aborted (`route.abort("blockedbyclient")`); proxy connection terminates on private IP connection attempts.

---

## 7. MCP Server Connections, Tool Schemas, and Trust Persistence

- **Primary Entry Points:**
  - `src/openjarvis/mcp/server.py`: `MCPServer` hosting local tools over JSON-RPC.
  - `src/openjarvis/tools/mcp_adapter.py`: `MCPToolAdapter`, `compute_mcp_tool_schema_hash()`, `record_approved_mcp_schema_hash()`.
- **Call Sequence:**
  1. External MCP tools discovered via `client.list_tools()`.
  2. Adapter computes SHA-256 fingerprint of `name`, `description`, `parameters`, `category`.
  3. Fingerprint verified against persistent local storage (`mcp_trust.json`).
  4. Autonomous Execution Gate:
     - Tool name and server must be explicitly in user allowlist.
     - Must not contain mutating keywords (`delete`, `write`, `patch`, etc.).
     - Schema fingerprint must match stored approval.
     - If schema mutated or unknown, autonomous execution rejected -> requires approval.
  5. Taint Sink Defense: Adapter inspects incoming arguments for `TaintLabel.SECRET`. If tainted, dispatch blocked.
- **Failure Behavior:** Schema mismatch or taint policy violation returns `ToolResult(success=False)`.

---

## 8. Shell Execution

- **Primary Entry Points:**
  - `src/openjarvis/tools/shell_exec.py`: `ShellExecTool.execute(command=...)`.
- **Call Sequence:**
  1. Evaluated by `PolicyEngine` -> classified as `REQUIRE_APPROVAL`.
  2. Once approved, verifies command arguments against payload hash.
  3. Checks capability policy for `shell_exec`.
  4. Invokes `run_sandboxed(command, timeout=30)` (`src/openjarvis/security/subprocess_sandbox.py`).
  5. Captures stdout/stderr within byte limits and returns `ToolResult`.
- **Failure Behavior:** Command timeouts kill the entire process tree via Job Object; output truncated if excessive.

---

## 9. Windows Job Objects and Process Supervision

- **Primary Entry Points:**
  - `src/openjarvis/security/subprocess_sandbox.py`: `_run_windows_job_object()`, Win32 API bindings via `ctypes`.
  - `src/openjarvis/tools/code_interpreter.py`: Supervised Python execution fallback.
- **Call Sequence:**
  1. Creates Win32 Job Object: `kernel32.CreateJobObjectW(None, None)`.
  2. Sets Job Limits:
     - `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE`: Ensures all child processes terminate when job handle closes.
     - `JOB_OBJECT_LIMIT_PROCESS_MEMORY` & `JOB_OBJECT_LIMIT_JOB_MEMORY`: 512 MB hard ceiling.
  3. Spawns process suspended: `subprocess.Popen(..., creationflags=0x00000004)`.
  4. Assigns process to Job Object: `kernel32.AssignProcessToJobObject(job, proc._handle)`.
  5. Verifies containment: `kernel32.IsProcessInJob(proc._handle, job, ctypes.byref(is_in_job))`.
  6. If verified, resumes process: `ctypes.windll.ntdll.NtResumeProcess(proc._handle)`.
  7. If assignment fails, process tree is terminated immediately, and `isolation_type="failed"` is set.
- **Honest Demarcation:** Labeled honestly as `isolation_type="job_object"` (process supervision, not a container sandbox).

---

## 10. Docker Sandbox and Container Configuration

- **Primary Entry Points:**
  - `src/openjarvis/tools/docker_shell_exec.py`: `DockerShellExecTool`, `verify_container_security()`.
- **Call Sequence:**
  1. Inspects active container via Docker socket API.
  2. Validates security profile against strict criteria:
     - `Privileged == False`
     - `User != "root"` and `User != "0"`
     - `ReadonlyRootfs == True`
     - Dropped capabilities include `"ALL"`
     - Mount allowlist: host root `/`, `C:\`, Windows system dirs, and user dotfiles strictly rejected.
  3. Verifies container ID has not changed between preflight inspection and execution.
  4. If container fails validation, execution halts with `SecurityError`.
- **Failure Behavior:** Docker daemon absence falls back to host supervision with honest reporting.

---

## 11. Filesystem Access, Path Validation, and Workspace Permissions

- **Primary Entry Points:**
  - `src/openjarvis/tools/_paths.py`: `resolve_safe_path()`, `validate_path_within_allowed_dirs()`.
  - `src/openjarvis/tools/file_read.py`: `FileReadTool`.
  - `src/openjarvis/tools/file_write.py`: `FileWriteTool`.
  - `src/openjarvis/tools/apply_patch.py`: `ApplyPatchTool`.
- **Call Sequence:**
  1. Target path resolved to absolute canonical path (`path.resolve()`).
  2. Containment Check: Must reside inside one of configured `allowed_dirs`.
  3. Empty Allowlist Rule: If `allowed_dirs` is empty, ALL operations fail closed.
  4. Symlink Escape Check: Symlinks inspected to ensure target destination remains inside workspace.
  5. Sensitive File Filter: `.env`, SSH keys (`id_rsa`), AWS credentials rejected.
  6. Atomic Backup: On modifying write (`file_write`, `apply_patch`), creates `.bak` file and verifies size. If backup creation fails, modification halts.
- **Failure Behavior:** Raises `SecurityError` or returns `ToolResult(success=False, content="Access denied")`.

---

## 12. HTTP Requests, SSRF Protection, and Network Egress

- **Primary Entry Points:**
  - `src/openjarvis/tools/http_request.py`: `HttpRequestTool`, `SSRFSafeTransport`, `SSRFSafeSyncStream`.
  - `src/openjarvis/security/ssrf.py`: `resolve_and_validate_destination()`, `is_ssrf_safe_ip()`.
  - `rust/crates/openjarvis-security/src/ssrf.rs`: Rust SSRF validator.
- **Call Sequence:**
  1. Destination URL parsed. Scheme must be `http` or `https`; port restricted to 80/443 (or configured allowlist).
  2. Hostname resolved via DNS. Every resolved IP checked against private, loopback, link-local, carrier-grade NAT, and cloud metadata (169.254.169.254) blocks.
  3. Pinned Connection: `SSRFSafeTransport` connects directly to validated IP to prevent DNS rebinding.
  4. Redirect Defense: HTTP redirects intercept next hop and re-validate IP before connection. Cross-domain redirects strip `Authorization` and `Cookie` headers.
  5. Fail-Closed Transport: If custom safe transport cannot initialize, request is rejected. Ordinary unpinned HTTP transport fallback is prohibited.
- **Failure Behavior:** SSRF attempts raise `SSRFSecurityError` and halt.

---

## 13. Prompt Injection and Untrusted Content Handling

- **Primary Entry Points:**
  - `src/openjarvis/security/injection_scanner.py`: `PromptInjectionScanner`, `scan_text_for_injection()`.
  - `src/openjarvis/security/taint.py`: `TaintSet`, `TaintLabel`, `check_taint()`, `declassify()`.
- **Call Sequence:**
  1. External inputs (web pages, file contents, MCP outputs) scanned for heuristic injection patterns (e.g., system prompt overrides, instruction delimiters).
  2. External data tagged with `TaintLabel.UNTRUSTED` or `TaintLabel.SECRET`.
  3. Taint propagates through agent reasoning and parameter transformations.
  4. Restricted sinks (external HTTP requests, external MCP adapters) check taint before execution.
  5. Secret-tainted data cannot leave local boundary without user declassification.
- **Failure Behavior:** High-confidence prompt injection aborts step; taint violations block tool execution.

---

## 14. Audit Logging, Redaction, and Persistence

- **Primary Entry Points:**
  - `src/openjarvis/security/audit.py`: `AuditLogger`, `verify_chain()`.
  - `src/openjarvis/core/events.py`: `EventBus`, `EventType`.
- **Call Sequence:**
  1. Security events (tool dispatches, approvals, denials, declassifications) published to `EventBus`.
  2. `AuditLogger._on_event()` handles events.
  3. Sensitive Data Scrubbing: Passwords, tokens (`sk-...`, Bearer), and browser typing inputs scrubbed via regex and replaced with `[REDACTED]`.
  4. Cryptographic Hash Chain: Each row records `prev_hash` and computes `sha256(id:timestamp:action:params:prev_hash)`.
  5. Row written to SQLite `audit_log` table.
- **Tamper Evidence:** `verify_chain()` scans ledger from genesis to head; any modified row or hash breaks verification.
- **Failure Behavior:** Audit logging failure on privileged actions halts execution (fail closed).

---

## 15. Scheduler and Background Execution

- **Primary Entry Points:**
  - `src/openjarvis/scheduler/scheduler.py`: `TaskScheduler`, `_execute_task()`, `ScheduledTask`.
- **Call Sequence:**
  1. Tasks registered with cron expressions or intervals.
  2. Scheduler daemon wakes on schedule.
  3. Execution Context: Constructs `TaskContext` with `agent_id="scheduler"`, authenticated `user_id`, and isolated session.
  4. Capability Profile: Enforces restricted `"scheduler"` profile (read-only and safe notifications).
  5. Consequential tools require user approval; cannot escalate privileges autonomously.
- **Failure Behavior:** Task failures logged to audit table; scheduler continues next cycle.

---

## 16. API Authentication and User Identity

- **Primary Entry Points:**
  - `src/openjarvis/server/auth_middleware.py`: `AuthMiddleware`.
  - `src/openjarvis/server/approval_routes.py`: `_verify_token_or_raise()`, `_get_caller_identity()`.
- **Call Sequence:**
  1. Incoming HTTP requests intercepted by `AuthMiddleware`.
  2. If `OPENJARVIS_API_KEY` configured, validates `Authorization: Bearer <token>`.
  3. On valid key, assigns `request.state.authenticated_user = "admin"`.
  4. Impersonation Defense: If request includes `X-User-Id` differing from authenticated principal, returns HTTP 403 Forbidden.
  5. Unauthenticated requests to protected endpoints return HTTP 401 Unauthorized.
- **Failure Behavior:** Request rejected immediately before reaching application routes.

---

## 17. Ollama and Local Model Integration

- **Primary Entry Points:**
  - `src/openjarvis/engine/ollama.py`: `OllamaInferenceEngine`.
  - `src/openjarvis/core/config.py`: `LLMConfig`, `ModelProvider`, `OllamaConfig`.
- **Call Sequence:**
  1. Agent queries `OllamaInferenceEngine.generate()` or `generate_stream()`.
  2. HTTP POST dispatched to local Ollama API (`http://127.0.0.1:11434/api/chat`).
  3. Model specified: `qwen3.5:4b` running locally on NVIDIA RTX 3050.
  4. Qwen Control Token Filtering: `_is_control_token_only_args()` filters degenerative `/no_think` or `/think` artifacts.
  5. Parses structured tool calls and returns to agent.
- **Privacy Guarantee:** All requests remain loopback (`127.0.0.1`). Zero external network egress for LLM inference.

---

## 18. Voice and Desktop Components

- **Primary Entry Points:**
  - `desktop/src-tauri/`: Tauri 2 desktop shell providing native window, system tray, and notifications.
  - `src/openjarvis/speech/`: Voice I/O subsystem (`voice_io.py`, `tts.py`, `faster_whisper.py`, `kokoro_tts.py`).
- **Call Sequence:**
  1. Desktop shell loads web frontend, communicates with backend over local HTTP/WS (`127.0.0.1:8000`).
  2. System tray receives approval notifications via SSE bridge.
  3. Voice input captured locally via Whisper; synthesized locally via TTS.
- **Security Checks:** Tauri origin restrictions enforced in CORS configuration (`tauri://localhost`, `http://tauri.localhost`).

---

## 19. Career-Assistant Workflows and Tool Access

- **Verified Implementation State:**
  - **No Dedicated Career Module:** Inspection of the entire repository confirms there is NO specialized career assistant module, CV generator, or job tracker codebase (e.g. `career.py` or `resume_manager.py`).
  - **Handling via General Capabilities:** Career workflows (job discovery, resume reading, application tailoring) are handled through the general assistant recipes (`src/openjarvis/recipes/`), `ToolUsingAgent`, and existing tools (`BrowserNavigateTool`, `FileReadTool`, `FileWriteTool`).
  - **Security Controls Applied:** When executing career workflows, standard workspace boundaries apply to resume files, SSRF guards apply to job portals, and explicit approval is required before submitting external forms or sending emails.
