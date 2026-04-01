# Claude Code Hooks System -- Architecture Overview

## 1. What Are Hooks?

Hooks are user-defined actions (shell commands, LLM prompts, HTTP requests, or agentic verifiers) that Claude Code executes at specific lifecycle points. They allow users to customize, guard, and automate the behavior of the CLI without modifying its source. Hooks are the primary extension mechanism for injecting external logic into the tool-use pipeline.

The system supports **six hook types** that can be triggered:
- **command** -- Shell commands (bash or PowerShell)
- **prompt** -- LLM-evaluated conditions using a small fast model
- **agent** -- Multi-turn agentic verifiers that can use tools
- **http** -- HTTP POST requests to external endpoints
- **callback** -- Internal TypeScript callbacks (SDK/plugin use only)
- **function** -- In-memory session-scoped TypeScript callbacks (internal only)

---

## 2. Hook Configuration (settings.json Format)

Hooks are configured in `settings.json` under a top-level `hooks` key. The schema is a partial record mapping hook event names to arrays of matcher configurations.

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "my-lint-check.sh",
            "if": "Bash(git *)",
            "timeout": 30,
            "shell": "bash",
            "statusMessage": "Running lint check...",
            "once": false,
            "async": false
          }
        ]
      }
    ],
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "prompt",
            "prompt": "Does this prompt contain PII? $ARGUMENTS",
            "model": "claude-sonnet-4-6",
            "timeout": 10
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "agent",
            "prompt": "Verify that all tests pass. $ARGUMENTS",
            "model": "claude-sonnet-4-6",
            "timeout": 60
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "http",
            "url": "https://my-server.example.com/hook",
            "headers": {"Authorization": "Bearer $MY_TOKEN"},
            "allowedEnvVars": ["MY_TOKEN"],
            "timeout": 30
          }
        ]
      }
    ]
  }
}
```

### Configuration Sources (Priority Order)

Settings are loaded from multiple sources, merged with this precedence:
1. **policySettings** (managed/admin) -- Highest priority. Can set `allowManagedHooksOnly` or `disableAllHooks`.
2. **userSettings** -- `~/.claude/settings.json`
3. **projectSettings** -- `.claude/settings.json` in the project root
4. **localSettings** -- `.claude/settings.local.json` in the project root

Additional hook sources:
- **Plugin hooks** -- Registered from installed plugins' `hooks.json`
- **Session hooks** -- In-memory, temporary hooks from skill frontmatter or agents
- **Registered callback hooks** -- Internal SDK/system callbacks

### Matcher Configuration

Each hook matcher has:
- `matcher` (optional string) -- Pattern to filter which tool/event values trigger the hook. Supports exact match, pipe-separated values (`Write|Edit`), regex patterns (`^Bash.*`), or wildcard (`*`).
- `hooks` (array) -- One or more hook commands to execute when matched.

### Hook Command Properties

All hook types share:
- `if` (optional) -- Permission-rule-syntax condition for fine-grained filtering (e.g., `Bash(git *)` to only trigger for git commands)
- `timeout` (optional) -- Seconds before the hook is cancelled
- `statusMessage` (optional) -- Custom spinner text while the hook runs
- `once` (optional) -- If true, hook auto-removes after first successful execution

Command-specific:
- `shell` -- `"bash"` (default) or `"powershell"`
- `async` -- Run in background without blocking
- `asyncRewake` -- Run in background; on exit code 2, wake the model with a blocking error

Prompt-specific:
- `model` -- Override the LLM model (default: small fast model)

HTTP-specific:
- `headers` -- Key-value headers supporting `$VAR_NAME` env var interpolation
- `allowedEnvVars` -- Explicit allowlist of env vars that may be interpolated

---

## 3. Hook Lifecycle Events (All 27 Events)

### Tool Lifecycle
| Event | Fires When | Matcher Field | Can Block |
|---|---|---|---|
| **PreToolUse** | Before a tool executes | `tool_name` | Yes (exit 2 or JSON deny) |
| **PostToolUse** | After a tool executes successfully | `tool_name` | Yes (exit 2) |
| **PostToolUseFailure** | After a tool execution fails | `tool_name` | Yes (exit 2) |
| **PermissionRequest** | When a permission dialog would display | `tool_name` | Yes (allow/deny decision) |
| **PermissionDenied** | When auto-mode classifier denies a tool | `tool_name` | No (can signal retry) |

### Session Lifecycle
| Event | Fires When | Matcher Field | Can Block |
|---|---|---|---|
| **SessionStart** | New session starts (startup/resume/clear/compact) | `source` | No |
| **SessionEnd** | Session ending (clear/logout/exit) | `reason` | No |
| **Setup** | Repository initialization or maintenance | `trigger` | No |

### Model Response Lifecycle
| Event | Fires When | Matcher Field | Can Block |
|---|---|---|---|
| **Stop** | Before Claude concludes its response | (none) | Yes (exit 2 continues conversation) |
| **StopFailure** | When turn ends due to API error | `error` | No (fire-and-forget) |
| **UserPromptSubmit** | User submits a prompt | (none) | Yes (exit 2 blocks processing) |

### Subagent Lifecycle
| Event | Fires When | Matcher Field | Can Block |
|---|---|---|---|
| **SubagentStart** | Subagent (Agent tool call) starts | `agent_type` | No |
| **SubagentStop** | Subagent concludes its response | `agent_type` | Yes (exit 2) |

### Compaction
| Event | Fires When | Matcher Field | Can Block |
|---|---|---|---|
| **PreCompact** | Before conversation compaction | `trigger` | Yes (exit 2) |
| **PostCompact** | After conversation compaction | `trigger` | No |

### Notification & Configuration
| Event | Fires When | Matcher Field | Can Block |
|---|---|---|---|
| **Notification** | System notifications sent | `notification_type` | No |
| **ConfigChange** | Configuration files change mid-session | `source` | Yes (exit 2) |
| **InstructionsLoaded** | CLAUDE.md or rule file loaded | `load_reason` | No (observability only) |

### Team & Task Lifecycle
| Event | Fires When | Matcher Field | Can Block |
|---|---|---|---|
| **TeammateIdle** | Teammate about to go idle | (none) | Yes (exit 2 prevents idle) |
| **TaskCreated** | Task is being created | (none) | Yes (exit 2 blocks creation) |
| **TaskCompleted** | Task being marked completed | (none) | Yes (exit 2 blocks completion) |

### MCP Elicitation
| Event | Fires When | Matcher Field | Can Block |
|---|---|---|---|
| **Elicitation** | MCP server requests user input | `mcp_server_name` | Yes (decline action) |
| **ElicitationResult** | User responds to MCP elicitation | `mcp_server_name` | Yes (override response) |

### File System & Worktree
| Event | Fires When | Matcher Field | Can Block |
|---|---|---|---|
| **CwdChanged** | Working directory changes | (none) | No |
| **FileChanged** | Watched file changes | `file_path` basename | No |
| **WorktreeCreate** | Worktree creation requested | (none) | No (returns path) |
| **WorktreeRemove** | Worktree removal requested | (none) | No |

---

## 4. Hook Execution Model

### Exit Code Protocol (Command Hooks)

| Exit Code | Meaning |
|---|---|
| **0** | Success. stdout behavior varies by event (shown to model, shown in transcript, or hidden). |
| **2** | Blocking error. stderr is shown to the model and may block the operation. |
| **Other** | Non-blocking error. stderr is shown to the user only; execution continues. |

### JSON Output Protocol

Command hooks can output JSON to stdout for structured control. The JSON must be the first `{`-starting content in stdout.

```json
{
  "continue": false,
  "stopReason": "Custom stop message",
  "decision": "approve" | "block",
  "reason": "Explanation text",
  "systemMessage": "Warning shown to user",
  "suppressOutput": true,
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow" | "deny" | "ask",
    "permissionDecisionReason": "...",
    "updatedInput": { "command": "modified command" },
    "additionalContext": "Extra context for the model"
  }
}
```

### Async Hook Protocol

A hook can signal it wants to run asynchronously by outputting `{"async": true}` as its **first line** of stdout. The hook process is then backgrounded and its result is collected later by the `AsyncHookRegistry`. Alternatively, the `async: true` or `asyncRewake: true` flags can be set in the hook config.

### Input Delivery

Hook input is delivered as JSON via **stdin**. The input always includes base fields:
- `session_id`, `transcript_path`, `cwd`, `permission_mode`, `hook_event_name`
- Event-specific fields (e.g., `tool_name`, `tool_input`, `prompt`, etc.)

The `$ARGUMENTS` placeholder in prompt/agent hooks is replaced with the JSON input string.

### Prompt Elicitation Protocol

Command hooks can request interactive prompts from the user by writing a JSON `{"prompt": "<id>", "message": "...", "options": [...]}` line to stdout. The system responds on stdin with `{"prompt_response": "<id>", "selected": "<key>"}`.

---

## 5. How Hook Results Influence Tool Execution

### PreToolUse: Permission Decisions

PreToolUse hooks can return permission decisions with precedence: **deny > ask > allow**.

- **allow** -- Bypasses the interactive permission prompt, BUT settings.json deny/ask rules still apply (defense-in-depth). If the hook provides `updatedInput`, the modified input is used.
- **deny** -- Blocks the tool call. The model sees the blocking error message.
- **ask** -- Forces the permission dialog even if the tool would normally be auto-allowed.
- **passthrough** -- Does not affect permissions; `updatedInput` still applies.

The `resolveHookPermissionDecision()` function in `toolHooks.ts` encapsulates the invariant that hook "allow" does NOT bypass settings.json deny/ask rules.

### PostToolUse: Output Modification & Continuation Control

- `additionalContext` -- Injected as context for the model's next response.
- `updatedMCPToolOutput` -- Replaces the output of MCP tool calls.
- `continue: false` + `stopReason` -- Prevents the model from continuing.

### Stop Hooks: Verification Gates

Stop hooks fire right before Claude concludes. Exit code 2 sends stderr to the model and forces it to continue working. Agent-type stop hooks spawn a full multi-turn verification sub-agent that can use tools to inspect the codebase.

### UserPromptSubmit: Input Filtering

Exit code 2 blocks the prompt from being processed and shows stderr to the user only. Exit code 0 with `additionalContext` injects extra context that the model will see alongside the user's prompt.

---

## 6. Security Architecture

### Workspace Trust

All hooks require workspace trust acceptance in interactive mode. The `shouldSkipHookDueToTrust()` function is a centralized security check that prevents RCE vulnerabilities:
- In non-interactive (SDK) mode, trust is implicit.
- In interactive mode, the trust dialog must have been accepted.
- Historical vulnerabilities (SessionEnd hooks on trust decline, SubagentStop before trust) prompted this centralized check.

### Hook Configuration Snapshot

Hooks config is captured once at startup via `captureHooksConfigSnapshot()` and served from that snapshot. This prevents mid-session configuration injection attacks. The snapshot is updated when hooks are modified through the `/hooks` command.

### Managed Hooks Policy

Enterprise administrators can use `policySettings` to:
- `allowManagedHooksOnly: true` -- Only admin-defined hooks execute; user/project hooks are blocked.
- `disableAllHooks: true` -- No hooks run at all (when in policySettings). When set in non-managed settings, it only disables non-managed hooks.
- `allowedHttpHookUrls` -- URL allowlist for HTTP hooks (wildcard patterns).
- `httpHookAllowedEnvVars` -- Allowlist of env vars that HTTP hook headers can reference.

### SSRF Protection

HTTP hooks include an SSRF guard (`ssrfGuard.ts`) that validates resolved IPs and blocks private/link-local ranges (but allows loopback for local development). This is bypassed when a proxy is in use.

### Header Injection Prevention

HTTP hook header values are sanitized to strip CR/LF/NUL bytes, preventing CRLF injection attacks.

---

## 7. Insights: Workflow Automation Patterns

### Automated Code Quality Gates
```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": "bash -c 'echo $ARGUMENTS | jq -r .tool_input.command | grep -q \"rm -rf\" && echo \"{\\\"decision\\\":\\\"block\\\",\\\"reason\\\":\\\"Dangerous command blocked\\\"}\" || true'",
        "if": "Bash(rm *)"
      }]
    }],
    "Stop": [{
      "hooks": [{
        "type": "agent",
        "prompt": "Verify that all modified files have corresponding test coverage. $ARGUMENTS",
        "timeout": 120
      }]
    }]
  }
}
```

### Environment Setup via SessionStart
```json
{
  "hooks": {
    "SessionStart": [{
      "hooks": [{
        "type": "command",
        "command": "source .envrc && echo '{\"hookSpecificOutput\":{\"hookEventName\":\"SessionStart\",\"additionalContext\":\"Environment loaded from .envrc\"}}'",
        "statusMessage": "Loading environment..."
      }]
    }]
  }
}
```

### File Change Monitoring
```json
{
  "hooks": {
    "FileChanged": [{
      "matcher": ".envrc|.env",
      "hooks": [{
        "type": "command",
        "command": "direnv export bash > $CLAUDE_ENV_FILE"
      }]
    }]
  }
}
```

### Cost Tracking via HTTP Hooks
```json
{
  "hooks": {
    "SessionEnd": [{
      "hooks": [{
        "type": "http",
        "url": "https://metrics.internal/claude-usage",
        "headers": {"Authorization": "Bearer $METRICS_TOKEN"},
        "allowedEnvVars": ["METRICS_TOKEN"]
      }]
    }]
  }
}
```

### Automated Permission Handling
```json
{
  "hooks": {
    "PermissionRequest": [{
      "matcher": "Read",
      "hooks": [{
        "type": "command",
        "command": "echo '{\"hookSpecificOutput\":{\"hookEventName\":\"PermissionRequest\",\"decision\":{\"behavior\":\"allow\"}}}'"
      }]
    }]
  }
}
```

---

## 8. Relationships: Hooks in the Architecture

### Hooks and Tools
- PreToolUse/PostToolUse hooks wrap every tool execution (`toolHooks.ts` -> `toolExecution.ts`).
- `runPreToolUseHooks()` and `runPostToolUseHooks()` are async generators that yield results back to the tool orchestration layer.
- Tools provide `preparePermissionMatcher()` for the `if` condition evaluation.

### Hooks and Permissions
- PreToolUse hooks can return `allow`/`deny`/`ask` permission decisions.
- Hook "allow" does NOT override settings.json deny/ask rules -- `checkRuleBasedPermissions()` still applies.
- The `PermissionRequest` event lets hooks programmatically handle permission dialogs (headless/CI use case).
- The `PermissionDenied` event allows hooks to signal retry after classifier denial.

### Hooks and Settings
- Settings files are the primary persistent storage for hook configuration.
- The `HooksSchema` Zod schema validates the hooks portion of settings.
- `captureHooksConfigSnapshot()` freezes the config at startup for security.
- `updateHooksConfigSnapshot()` refreshes when the user modifies hooks via `/hooks`.

### Hooks and Plugins
- Plugins register hooks via `loadPluginHooks.ts`, converting plugin hook configs to `PluginHookMatcher` objects.
- Plugin hooks receive `CLAUDE_PLUGIN_ROOT`, `CLAUDE_PLUGIN_DATA`, and `CLAUDE_PLUGIN_OPTION_*` env vars.
- `${CLAUDE_PLUGIN_ROOT}` and `${user_config.X}` are substituted in command strings.

### Hooks and Agents/Skills
- Agent frontmatter can define hooks that are registered as session hooks via `registerFrontmatterHooks()`.
- Skill frontmatter hooks are registered via `registerSkillHooks()`.
- Agent `Stop` hooks are automatically converted to `SubagentStop` since subagents fire SubagentStop, not Stop.
- Session hooks are scoped to agent IDs to prevent cross-agent leakage.

### Hooks and the Event System
- `hookEvents.ts` provides a generic event broadcasting system separate from the message stream.
- Events: `started`, `progress`, `response` -- emitted for SDK consumers.
- SessionStart and Setup events are always emitted; others require `includeHookEvents` option.

### Hooks and Cost Tracking
- `costHook.ts` is a React hook (`useCostSummary`) that displays cost on process exit and saves session costs. It uses the standard React `useEffect` pattern, not the hooks system described above.
- The hooks system itself tracks execution duration via `addToTurnHookDuration()` and `getStatsStore().observe('hook_duration_ms', ...)`.

---

## 9. costHook.ts -- Cost Tracking

The file `costHook.ts` is **not** part of the hooks lifecycle system. It is a React hook that:
1. Registers a `process.on('exit')` handler.
2. On exit, writes the formatted total cost to stdout (if the user has console billing access).
3. Calls `saveCurrentSessionCosts()` to persist cost data.

This is a UI-level concern using React's `useEffect`, not a configurable lifecycle hook.

---

## 10. Internal Hook Mechanisms

### Post-Sampling Hooks (`postSamplingHooks.ts`)
An internal programmatic registry for hooks that run after model sampling completes. Not exposed in settings.json. Used for analytics, auto-dream consolidation, and internal feedback mechanisms.

### API Query Hook Helper (`apiQueryHookHelper.ts`)
A factory for creating post-sampling hooks that make secondary API calls. Provides a standardized pattern for: should-run checks, message building, model selection, response parsing, and result logging.

### Session File Access Hooks
Internal callbacks registered for PostToolUse to track file access patterns for analytics. Marked as `internal: true` to exclude from `tengu_run_hook` metrics.

### Collapse Hook Summaries (`collapseHookSummaries.ts`)
UI utility that collapses consecutive hook summary messages with the same label (e.g., parallel PostToolUse hooks) into a single summary for cleaner display.
