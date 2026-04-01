# Claude Code CLI - Tools Handling System: Architecture Overview

## 1. Overview

The tools system is the execution backbone of Claude Code CLI. Every action the model takes -- reading files, running shell commands, editing code, searching the web, interacting with MCP servers -- flows through a unified tool abstraction defined in `Tool.ts`. The architecture follows a pattern where each tool is a self-contained module exporting a `Tool` object constructed via `buildTool()`, with shared infrastructure for execution, permissions, hooks, concurrency, and UI rendering.

### Key Architectural Principles

- **Single type, many implementations**: All tools conform to the `Tool<Input, Output, Progress>` interface.
- **Convention-based directory structure**: Each tool lives in `tools/<ToolName>/` with standardized files for schema, logic, prompt, and UI.
- **Centralized registration**: `tools.ts` is the single source of truth for all available built-in tools.
- **Extension via MCP**: External tools from MCP servers are dynamically created by spreading the base `MCPTool` object and overriding methods.
- **Fail-closed defaults**: `buildTool()` provides safe defaults (not concurrency-safe, not read-only, no classifier input) so new tools cannot accidentally bypass safety checks by omission.
- **Separation of concerns**: Tool definition (`tools/`), execution orchestration (`services/tools/`), permission checking (`utils/permissions/`), hook execution (`utils/hooks.ts`), and UI rendering (`UI.tsx` per tool) are cleanly separated.

---

## 2. Tool Definition and Structure

### The `Tool` Type (`Tool.ts`)

The `Tool` type is a large interface with ~40 members covering:

| Category | Key Members |
|----------|-------------|
| **Identity** | `name`, `aliases`, `searchHint`, `isMcp`, `isLsp`, `mcpInfo` |
| **Schema** | `inputSchema` (Zod), `outputSchema` (Zod), `inputJSONSchema` (JSON Schema for MCP) |
| **Behavior flags** | `isConcurrencySafe()`, `isReadOnly()`, `isDestructive()`, `isEnabled()`, `isOpenWorld()`, `shouldDefer`, `alwaysLoad`, `strict` |
| **Execution** | `call()`, `validateInput()`, `checkPermissions()`, `backfillObservableInput()`, `preparePermissionMatcher()` |
| **Rendering** | `renderToolUseMessage()`, `renderToolResultMessage()`, `renderToolUseProgressMessage()`, `renderToolUseErrorMessage()`, `renderToolUseRejectedMessage()`, `renderGroupedToolUse()`, `renderToolUseTag()` |
| **Summary/Display** | `userFacingName()`, `getToolUseSummary()`, `getActivityDescription()`, `extractSearchText()` |
| **Prompt** | `description()`, `prompt()` |
| **Classification** | `toAutoClassifierInput()`, `isSearchOrReadCommand()` |
| **Result mapping** | `mapToolResultToToolResultBlockParam()` |

### The `buildTool()` Factory

All tools are constructed via `buildTool(def)` which merges safe defaults:
- `isEnabled` -> `true`
- `isConcurrencySafe` -> `false` (assume unsafe)
- `isReadOnly` -> `false` (assume writes)
- `isDestructive` -> `false`
- `checkPermissions` -> allow with passthrough
- `toAutoClassifierInput` -> `''` (skip classifier)
- `userFacingName` -> tool's name

This ensures every tool always has these methods defined, removing the need for `?.()` checks.

### Tool Directory Convention

Each tool follows this pattern within `tools/<ToolName>/`:

| File | Purpose |
|------|---------|
| `<ToolName>.ts` or `<ToolName>.tsx` | Main tool definition using `buildTool()` |
| `prompt.ts` | Tool name constant, description text, prompt template |
| `UI.tsx` | React components for rendering (tool use message, result, error, progress) |
| `constants.ts` | Shared constants (tool name string) |
| `toolName.ts` | (BashTool) Exported name constant |
| Additional files | Tool-specific helpers (e.g., `bashPermissions.ts`, `sedEditParser.ts`, `imageProcessor.ts`) |

---

## 3. Tool Registration and Discovery

### Built-in Tools (`tools.ts`)

`getAllBaseTools()` returns the exhaustive list of all possible tools. Tools are included based on:

1. **Always included**: `AgentTool`, `BashTool`, `FileReadTool`, `FileEditTool`, `FileWriteTool`, `GlobTool`, `GrepTool`, `NotebookEditTool`, `WebFetchTool`, `TodoWriteTool`, `WebSearchTool`, etc.
2. **Feature-gated** (via `feature()` from `bun:bundle`): `REPLTool`, `SleepTool`, `SnipTool`, `WorkflowTool`, `MonitorTool`, `CronTools`, etc.
3. **Environment-gated** (via `process.env`): `ConfigTool`, `TungstenTool` (ant-only), `LSPTool`, `TestingPermissionTool` (test-only).
4. **Conditional**: `GlobTool`/`GrepTool` excluded when embedded search tools are available; `PowerShellTool` on Windows; `ToolSearchTool` when tool search is enabled.

### Tool Filtering Pipeline

```
getAllBaseTools()   -->  getTools(permissionContext)  -->  assembleToolPool()  -->  Final tool set
                        |                                  |
                        |-- simple mode filter             |-- Add MCP tools
                        |-- deny rule filter               |-- Deny rule filter on MCP
                        |-- REPL mode filter               |-- Sort (built-in prefix, MCP suffix)
                        |-- isEnabled() filter             |-- Deduplicate by name
```

`getTools()` applies:
- **Simple mode**: Only `BashTool`, `FileReadTool`, `FileEditTool` (with `CLAUDE_CODE_SIMPLE` env var)
- **Deny rules**: `filterToolsByDenyRules()` removes tools with blanket deny rules
- **REPL mode**: Hides primitive tools when REPL wraps them
- **isEnabled()**: Each tool's runtime enablement check

`assembleToolPool()` merges built-in tools with MCP tools, sorting each partition separately for prompt-cache stability.

### Tool Search / Deferred Loading

When many tools are available (especially with MCP servers), `ToolSearchTool` enables deferred loading:

- Tools marked as deferred (`isDeferredTool()`) have their schemas sent with `defer_loading: true`
- The model sees only tool names, not full schemas
- To use a deferred tool, the model first calls `ToolSearch` with a query
- `ToolSearch` returns `tool_reference` blocks that load the full schema
- MCP tools are always deferred; built-in tools can opt in via `shouldDefer: true`
- Critical tools (`ToolSearch` itself, `AgentTool` in fork mode, `BriefTool`) are never deferred

---

## 4. Tool Execution Lifecycle

### Entry Point: `runToolUse()` in `services/tools/toolExecution.ts`

The full lifecycle for a single tool call:

```
1. Tool Lookup
   - Find tool by name in available tools
   - Fall back to base tools for deprecated aliases

2. Input Validation (Zod)
   - Parse input against tool.inputSchema
   - On failure: return InputValidationError
   - If deferred tool schema wasn't sent: append ToolSearch hint

3. Tool-Specific Validation
   - Call tool.validateInput() for business-logic checks
   - Examples: deny-rule path check, binary file detection, device file blocking

4. Pre-Tool Hooks (PreToolUse)
   - Run user-configured hooks that can inspect/modify/block the call
   - Hooks can return: permission decision, updated input, stop signal, additional context

5. Permission Check
   - resolveHookPermissionDecision() combines hook results with rule-based permissions
   - Calls canUseTool (interactive permission prompt or auto-mode classifier)
   - Decision: allow, deny, or ask (with classifier pending)

6. Backfill Observable Input
   - Clone input, apply backfillObservableInput() for hooks/canUseTool to see derived fields
   - Original input passed to call() to avoid altering transcript hashes

7. Tool Execution
   - tool.call(input, context, canUseTool, parentMessage, onProgress)
   - Returns: { data, newMessages?, contextModifier?, mcpMeta? }

8. Post-Tool Hooks (PostToolUse)
   - Run hooks that can inspect/modify the output
   - MCP tools: hooks can update the output before it's added to messages

9. Result Mapping
   - tool.mapToolResultToToolResultBlockParam() converts output to API format
   - processToolResultBlock() handles large results (persist to disk, return preview)
   - Accept feedback and content blocks from permission decision appended

10. Error Handling
    - Format error, run PostToolUseFailure hooks
    - MCP auth errors: update client status to 'needs-auth'
    - Return tool_result with is_error: true
```

### Concurrency Orchestration (`services/tools/toolOrchestration.ts`)

Multiple tool calls from a single model response are partitioned:

1. **Partition into batches**: Consecutive concurrency-safe tools form a batch; non-safe tools get individual batches
2. **Concurrent batch**: Run all tools in parallel (up to `MAX_TOOL_USE_CONCURRENCY`, default 10)
3. **Serial batch**: Run one at a time; context modifiers applied between runs
4. The `isConcurrencySafe()` flag determines batching (read-only tools like Read, Glob, Grep are typically safe)

### Streaming Executor (`services/tools/StreamingToolExecutor.ts`)

For streaming API responses, `StreamingToolExecutor` manages tools as they arrive:
- Tools added to queue as they stream in
- Concurrent-safe tools start immediately if no exclusive tool is running
- Results buffered and emitted in order
- Sibling abort controller kills parallel tools when one Bash errors

---

## 5. Shared Infrastructure (`tools/shared/`)

### `gitOperationTracking.ts`
Detects git operations (commit, push, cherry-pick, merge, rebase, PR creation) in shell command output. Fires analytics events and increments OTLP counters. Works shell-agnostically (same regex for Bash and PowerShell).

### `spawnMultiAgent.ts`
Shared spawn module for teammate creation. Handles in-process and out-of-process teammate spawning with tmux pane management, model selection, permission inheritance, and task registration.

---

## 6. Tool Results: Rendering and API Serialization

### Dual Rendering Path

Every tool has two rendering paths:

1. **UI Rendering** (React/Ink): `renderToolUseMessage()`, `renderToolResultMessage()`, `renderToolUseProgressMessage()` -- what the user sees in the terminal.
2. **API Serialization**: `mapToolResultToToolResultBlockParam()` -- what goes back to the Claude API as a `tool_result` content block.

These are deliberately different. For example, `FileReadTool`:
- **UI**: Shows "Read N lines from file.ts" (summary chrome only)
- **API**: Sends full file content with line numbers, plus `CYBER_RISK_MITIGATION_REMINDER` system tag

### Large Result Handling

When `tool.maxResultSizeChars` is exceeded, `processToolResultBlock()` persists the result to disk and sends a preview with a file path. Tools can set `maxResultSizeChars: Infinity` to opt out (e.g., Read tool, to avoid circular Read-to-file-to-Read loops).

### Tool Use Summary Generation (`services/toolUseSummary/`)

For SDK clients, `generateToolUseSummary()` calls Haiku to produce a one-line summary of completed tool batches (e.g., "Searched in auth/", "Fixed NPE in UserService"). This is a non-critical, best-effort feature.

---

## 7. MCP Tools: External Extension

### How MCP Tools Are Created

MCP tools are created in `services/mcp/client.ts` by spreading the base `MCPTool` object and overriding:

```
{
  ...MCPTool,                          // Base template
  name: "mcp__server__toolname",       // Prefixed name
  mcpInfo: { serverName, toolName },   // Original names
  isMcp: true,
  searchHint: from _meta,
  alwaysLoad: from _meta,
  description() { return tool.description },
  prompt() { return tool.description },
  isConcurrencySafe() { return annotations.readOnlyHint },
  isReadOnly() { return annotations.readOnlyHint },
  isDestructive() { return annotations.destructiveHint },
  isOpenWorld() { return annotations.openWorldHint },
  inputJSONSchema: tool.inputSchema,   // JSON Schema, not Zod
  checkPermissions() { return { behavior: 'passthrough' } },
  call(args, context, ...) { ... callMCPTool ... },
  userFacingName() { return "server - toolName (MCP)" },
}
```

### MCP Tool Naming

- Standard: `mcp__<normalized_server_name>__<tool_name>` (e.g., `mcp__chrome_devtools__click`)
- SDK mode with `CLAUDE_AGENT_SDK_MCP_NO_PREFIX`: uses original tool name, can override built-ins
- `mcpInfo` always preserves original server/tool names for permission checking

### MCP vs Built-in Differences

| Aspect | Built-in | MCP |
|--------|----------|-----|
| Schema format | Zod (`inputSchema`) | JSON Schema (`inputJSONSchema`) |
| Permission default | Tool-specific | Always `passthrough` (generic permission flow) |
| Deferred by default | Only if `shouldDefer: true` | Always (unless `alwaysLoad: true`) |
| Post-hook output mutation | Not supported | Hooks can update MCP output before adding to messages |
| Result processing | Tool-specific `mapToolResultToToolResultBlockParam` | Generic string content |

---

## 8. Relationships to Other Systems

### Permissions System
- Each tool implements `checkPermissions()` returning a `PermissionResult` (`allow`, `deny`, `ask`, `passthrough`)
- `passthrough` defers to the general permission system (deny/allow rules, interactive prompt, auto-mode classifier)
- File tools use `checkReadPermissionForTool()` and `matchWildcardPattern()` for path-based rules
- `preparePermissionMatcher()` enables efficient hook `if` condition matching

### Hook System
- **PreToolUse hooks**: Run before execution; can block, modify input, make permission decisions
- **PostToolUse hooks**: Run after success; can modify MCP output, add messages
- **PostToolUseFailure hooks**: Run after errors
- **PermissionRequest hooks**: External decision-making for permissions
- **PermissionDenied hooks**: Run when classifier denies in auto mode; can signal retry

### Agent/Subagent System
- `AgentTool` spawns subagents with filtered tool sets
- `ALL_AGENT_DISALLOWED_TOOLS`: Tools forbidden in subagents (plan mode tools, ask user, etc.)
- `ASYNC_AGENT_ALLOWED_TOOLS`: Whitelist for async agents
- `COORDINATOR_MODE_ALLOWED_TOOLS`: Minimal set for coordinator pattern
- Subagents get a `createSubagentContext()` with modified `setAppState` and tool filtering

### System Prompt
- Each tool's `prompt()` method generates its system prompt section
- `description()` provides the short description for the API tool definition
- `searchHint` feeds ToolSearch keyword matching
- Tool schemas are sent in the initial API request (or deferred via ToolSearch)

### UI System
- Tools use React (Ink) components for terminal rendering
- Progress is reported via `onProgress` callbacks that yield `ProgressMessage` objects
- `renderGroupedToolUse()` enables batch rendering of parallel tool calls
- `isSearchOrReadCommand()` determines if tool output should be collapsed in the UI
- `isResultTruncated()` gates click-to-expand in fullscreen mode

---

## 9. Tool Categories

| Category | Tools | Characteristics |
|----------|-------|-----------------|
| **File I/O** | Read, Edit, Write, NotebookEdit | Path-based permissions, file state tracking |
| **Shell** | Bash, PowerShell | Complex permission model (classifier, sandbox), progress streaming |
| **Search** | Glob, Grep, ToolSearch | Read-only, concurrency-safe, collapsible UI |
| **Web** | WebFetch, WebSearch | Network access, result size limits |
| **Agent/Task** | AgentTool, TaskCreate/Get/Update/List/Stop/Output, SendMessage | Subagent management, task framework |
| **Planning** | EnterPlanMode, ExitPlanMode, VerifyPlanExecution | Mode switching, plan verification |
| **MCP** | MCPTool (template), ListMcpResources, ReadMcpResource, McpAuth | External server integration |
| **UI/Interaction** | AskUserQuestion, BriefTool, TodoWrite | User communication |
| **Skills** | SkillTool | Slash command / skill invocation |
| **Infrastructure** | ConfigTool, EnterWorktree, ExitWorktree, SleepTool, ScheduleCron, RemoteTrigger | System configuration, worktrees, scheduling |
| **Internal** | TungstenTool, REPLTool, SyntheticOutput, OverflowTest, CtxInspect, Snip | Ant-only or feature-gated internal tools |

---

## 10. Adding New Tools

### Built-in Tool Checklist

1. Create `tools/<ToolName>/` directory
2. Define input/output schemas with Zod (using `lazySchema()`)
3. Implement `call()` with the tool's logic
4. Implement `checkPermissions()` or rely on the default
5. Implement `validateInput()` for pre-execution validation
6. Create `prompt.ts` with name constant, description, and prompt text
7. Create `UI.tsx` with rendering functions
8. Implement `mapToolResultToToolResultBlockParam()` for API serialization
9. Set behavioral flags: `isConcurrencySafe`, `isReadOnly`, `isDestructive`
10. Export via `buildTool({ ... } satisfies ToolDef<InputSchema, Output>)`
11. Register in `tools.ts` `getAllBaseTools()` (with any feature/env guards)
12. Add tool name to relevant constant sets in `constants/tools.ts` if needed

### MCP Tool Extension

MCP tools are automatically created when an MCP server connects. To customize behavior:
- Set `annotations.readOnlyHint`, `annotations.destructiveHint`, `annotations.openWorldHint` in the MCP tool definition
- Use `_meta['anthropic/searchHint']` for ToolSearch keyword help
- Use `_meta['anthropic/alwaysLoad']` to prevent deferral
- Use `annotations.title` for display name override
