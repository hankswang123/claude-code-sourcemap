# Claude Code Hooks System -- Detailed Code-Level Analysis

## File Map

```
src/
  costHook.ts                                    -- React hook for cost display on exit
  schemas/hooks.ts                               -- Zod schemas for hook configuration
  types/hooks.ts                                 -- TypeScript types and response schemas
  entrypoints/sdk/coreTypes.ts                   -- HOOK_EVENTS constant (27 events)
  utils/
    hooks.ts                                     -- Main engine (~5000 lines): execution, matching, all execute*Hooks exports
    hooks/
      hookEvents.ts                              -- Event broadcasting system (started/progress/response)
      hookHelpers.ts                             -- Shared utilities (hookResponseSchema, $ARGUMENTS substitution)
      hooksSettings.ts                           -- Hook listing, equality, source display, getAllHooks()
      hooksConfigManager.ts                      -- Hook event metadata, grouping, UI config management
      hooksConfigSnapshot.ts                     -- Security snapshot: capture/update/get frozen config
      sessionHooks.ts                            -- Session-scoped hook storage in AppState
      registerFrontmatterHooks.ts                -- Register agent/skill frontmatter hooks
      registerSkillHooks.ts                      -- Register skill-defined hooks with once: support
      execPromptHook.ts                          -- Execute prompt-type hooks via LLM
      execAgentHook.ts                           -- Execute agent-type hooks via multi-turn query
      execHttpHook.ts                            -- Execute HTTP-type hooks via POST
      AsyncHookRegistry.ts                       -- Global registry for backgrounded async hooks
      postSamplingHooks.ts                       -- Internal post-sampling hook registry
      apiQueryHookHelper.ts                      -- Factory for API-query-based post-sampling hooks
      fileChangedWatcher.ts                      -- chokidar-based file watcher for FileChanged hooks
      ssrfGuard.ts                               -- SSRF protection for HTTP hooks
      skillImprovement.ts                        -- Skill improvement hook (internal)
    collapseHookSummaries.ts                     -- UI: collapse consecutive hook summary messages
    classifierApprovalsHook.ts                   -- React hook for classifier approval state
    sessionFileAccessHooks.ts                    -- Internal PostToolUse callback for file access tracking
  services/tools/
    toolHooks.ts                                 -- Bridge: runPreToolUseHooks, runPostToolUseHooks, resolveHookPermissionDecision
    toolExecution.ts                             -- Calls toolHooks during tool execution pipeline
  plugins/
    loadPluginHooks.ts                           -- Convert plugin hooks to PluginHookMatcher and register
  commands/hooks/
    hooks.tsx                                    -- /hooks slash command UI
  components/hooks/
    HooksConfigMenu.tsx                          -- Interactive hook configuration menu
    SelectHookMode.tsx                           -- Hook selection UI
    ViewHookMode.tsx                             -- Hook viewing UI
  hooks/
    useDeferredHookMessages.ts                   -- React hook for deferred hook message rendering
```

---

## Core Engine: `src/utils/hooks.ts`

This is the central file (~5000 lines) containing all hook execution logic.

### Constants and Timeouts (lines 166-182)

```typescript
const TOOL_HOOK_EXECUTION_TIMEOUT_MS = 10 * 60 * 1000  // 10 minutes default
const SESSION_END_HOOK_TIMEOUT_MS_DEFAULT = 1500        // 1.5 seconds for teardown
```

`getSessionEndHookTimeoutMs()` (line 176) allows override via `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS` env var.

### Trust Check: `shouldSkipHookDueToTrust()` (line 286)

Central security gate. Returns `true` (skip) when:
- Interactive mode AND trust dialog not accepted.
- Returns `false` (execute) in non-interactive (SDK) mode.

### Base Hook Input: `createBaseHookInput()` (line 301)

Builds the common JSON payload for all hooks:
```typescript
{
  session_id, transcript_path, cwd, permission_mode, agent_id, agent_type
}
```

### Pattern Matching: `matchesPattern()` (line 1346)

Three matching modes:
1. **Exact match** -- Simple alphanumeric string (e.g., `"Write"`)
2. **Pipe-separated** -- Multiple exact matches (e.g., `"Write|Edit"`)
3. **Regex** -- Full regex patterns (e.g., `"^Bash.*"`, `".*"`)

Legacy tool names are normalized via `normalizeLegacyToolName()`.

### If-Condition Matching: `prepareIfConditionMatcher()` (line 1390)

For tool events (PreToolUse, PostToolUse, PostToolUseFailure, PermissionRequest), the `if` field uses permission-rule syntax. The function:
1. Parses the `if` string via `permissionRuleValueFromString()`.
2. Validates the tool name matches.
3. If there is rule content (e.g., `Bash(git *)`), uses the tool's `preparePermissionMatcher()` for pattern matching.

### Hook Config Assembly: `getHooksConfig()` (line 1492)

Merges hooks from all sources in order:
1. Config snapshot (user/project/local settings, post-policy filtering)
2. Registered hooks (SDK callbacks, plugin native hooks) -- skipped if `managedOnly`
3. Session hooks for current session ID -- skipped if `managedOnly`
4. Session function hooks for current session ID -- skipped if `managedOnly`

### Hook Matching: `getMatchingHooks()` (line 1603)

Main matching algorithm:
1. Determines `matchQuery` based on event type (tool_name, source, trigger, etc.)
2. Filters matchers by pattern matching
3. Extracts hooks with plugin/skill context
4. Deduplicates by type-specific key (command+shell, prompt, url)
5. Applies `if` conditions
6. Filters HTTP hooks from SessionStart/Setup (deadlock prevention)

### Main Execution Loop: `executeHooks()` (line 1952)

Internal async generator that orchestrates hook execution:

1. **Guard checks**: `shouldDisableAllHooksIncludingManaged()`, `CLAUDE_CODE_SIMPLE`, trust check
2. **Hook matching**: Calls `getMatchingHooks()`
3. **Fast path**: If all hooks are internal callbacks, runs them directly without span/progress overhead (-70% latency)
4. **Progress emission**: Yields progress messages for UI spinners
5. **Parallel execution**: All hooks run in parallel via `all(hookPromises)`
6. **Result aggregation**: Permission precedence (deny > ask > allow), context collection, input updates

### Command Hook Execution: `execCommandHook()` (line 747)

Full shell execution pipeline:
1. **Shell selection**: `hook.shell` or default `'bash'`. PowerShell uses `pwsh -NoProfile -NonInteractive -Command`.
2. **Windows support**: Git Bash for bash hooks, native paths for PowerShell.
3. **Plugin variable substitution**: `${CLAUDE_PLUGIN_ROOT}`, `${CLAUDE_PLUGIN_DATA}`, `${user_config.X}`
4. **Environment variables**: `CLAUDE_PROJECT_DIR`, `CLAUDE_ENV_FILE`, plugin-specific vars.
5. **Stdin delivery**: JSON input + newline written to child process stdin.
6. **Async detection**: First line checked for `{"async": true}` -- if found, process is backgrounded.
7. **Prompt elicitation**: stdout scanned for prompt request JSON; responses sent on stdin.
8. **Output collection**: stdout/stderr collected, streams awaited before close.

### Hook Output Processing: `processHookJSONOutput()` (line 489)

Parses sync JSON output and extracts:
- `continue: false` -> `preventContinuation`
- `decision: "approve"/"block"` -> permission behavior
- `hookSpecificOutput` -> Event-specific fields (permission decisions, updated input, additional context, watch paths, etc.)

---

## Type System: `src/types/hooks.ts`

### Core Types (lines 203-290)

- **`HookCallback`** -- Internal callback hook with async function signature. Receives `HookInput`, toolUseID, AbortSignal, hookIndex, and optional `HookCallbackContext`.
- **`HookCallbackMatcher`** -- Associates a matcher pattern with callback hooks.
- **`HookProgress`** -- Progress data for UI spinners during hook execution.
- **`HookResult`** -- Per-hook execution result with outcome, messages, permissions, context.
- **`AggregatedHookResult`** -- Merged result from multiple hooks for a single event.
- **`PermissionRequestResult`** -- Allow (with optional updatedInput/updatedPermissions) or deny (with optional message/interrupt).

### Response Schema: `syncHookResponseSchema` (lines 50-166)

Zod schema validating JSON output from hooks. The `hookSpecificOutput` is a discriminated union on `hookEventName`, with each event having its own allowed fields:

| Event | Specific Fields |
|---|---|
| PreToolUse | `permissionDecision`, `permissionDecisionReason`, `updatedInput`, `additionalContext` |
| UserPromptSubmit | `additionalContext` |
| SessionStart | `additionalContext`, `initialUserMessage`, `watchPaths` |
| PostToolUse | `additionalContext`, `updatedMCPToolOutput` |
| PermissionDenied | `retry` |
| PermissionRequest | `decision` (allow with updatedInput/updatedPermissions, or deny with message/interrupt) |
| Elicitation | `action` (accept/decline/cancel), `content` |
| CwdChanged | `watchPaths` |
| FileChanged | `watchPaths` |
| WorktreeCreate | `worktreePath` |

---

## Schema Layer: `src/schemas/hooks.ts`

### Hook Command Schema: `HookCommandSchema` (line 176)

Discriminated union on `type`:

```typescript
z.discriminatedUnion('type', [
  BashCommandHookSchema,   // type: 'command'
  PromptHookSchema,        // type: 'prompt'
  AgentHookSchema,         // type: 'agent'
  HttpHookSchema,          // type: 'http'
])
```

### HooksSchema (line 211)

```typescript
z.partialRecord(z.enum(HOOK_EVENTS), z.array(HookMatcherSchema()))
```

This makes hooks optional per event and validates the full configuration structure.

### If-Condition Schema (line 19)

```typescript
const IfConditionSchema = z.string().optional().describe(
  'Permission rule syntax to filter when this hook runs (e.g., "Bash(git *)").'
)
```

---

## Configuration Snapshot: `src/utils/hooks/hooksConfigSnapshot.ts`

### Security Model (lines 1-133)

The snapshot system prevents runtime injection:

1. **`captureHooksConfigSnapshot()`** (line 95) -- Called once at startup, freezes hooks config.
2. **`getHooksConfigFromSnapshot()`** (line 119) -- Returns the frozen config (lazy-initializes if null).
3. **`updateHooksConfigSnapshot()`** (line 104) -- Called on `/hooks` command; resets settings cache and recaptures.

### Source Filtering: `getHooksFromAllowedSources()` (line 18)

Priority logic:
1. If `policySettings.disableAllHooks === true` -> return `{}`
2. If `policySettings.allowManagedHooksOnly === true` -> return only `policySettings.hooks`
3. If `strictPluginOnlyCustomization('hooks')` -> return only `policySettings.hooks`
4. If `mergedSettings.disableAllHooks === true` (from non-managed) -> return only `policySettings.hooks`
5. Otherwise -> return fully merged hooks from all sources

---

## Session Hooks: `src/utils/hooks/sessionHooks.ts`

### Storage Architecture (lines 42-62)

Session hooks are stored in `AppState.sessionHooks` as a `Map<string, SessionStore>`, keyed by session/agent ID. The Map is deliberately mutated in-place (not replaced) to avoid O(N^2) copying costs with parallel agents.

### Key Functions

- **`addSessionHook()`** (line 68) -- Adds a command/prompt/agent hook to a session. Finds or creates a matcher entry.
- **`addFunctionHook()`** (line 93) -- Adds an in-memory function hook. Returns hook ID for later removal.
- **`removeFunctionHook()`** (line 120) -- Removes a function hook by ID.
- **`removeSessionHook()`** (line 225) -- Removes a specific hook by equality check (`isHookEqual()`).
- **`clearSessionHooks()`** (line 437) -- Deletes all hooks for a session (used during cleanup).
- **`getSessionHooks()`** (line 302) -- Returns session hooks excluding function hooks.
- **`getSessionFunctionHooks()`** (line 345) -- Returns only function hooks (kept separate since they are not persistable).

---

## Hook Executors

### Prompt Hook: `src/utils/hooks/execPromptHook.ts`

Execution flow (line 21):
1. Substitute `$ARGUMENTS` in the prompt string.
2. Call `queryModelWithoutStreaming()` with the processed prompt.
3. System prompt instructs the model to return `{"ok": true}` or `{"ok": false, "reason": "..."}`.
4. Uses `outputFormat: { type: 'json_schema' }` for structured output.
5. Model defaults to `getSmallFastModel()`, overridable via `hook.model`.
6. Default timeout: 30 seconds.

### Agent Hook: `src/utils/hooks/execAgentHook.ts`

Multi-turn verification flow (line 36):
1. Creates a unique `hookAgentId` for isolation.
2. Configures `ToolUseContext` with `dontAsk` permission mode.
3. Registers `StructuredOutput` tool and enforcement hook.
4. Runs the `query()` loop with max 50 turns.
5. Filters out disallowed agent tools (no nested subagents, no plan mode).
6. System prompt directs the agent to verify conditions and use the `StructuredOutput` tool.
7. Model defaults to `getSmallFastModel()`, overridable via `hook.model`.
8. Default timeout: 60 seconds.
9. On completion, clears session hooks for the agent and returns result.

### HTTP Hook: `src/utils/hooks/execHttpHook.ts`

HTTP POST execution (line 123):
1. **URL allowlist check**: `allowedHttpHookUrls` patterns from settings. If defined and no match, request is blocked.
2. **Header interpolation**: `$VAR_NAME` and `${VAR_NAME}` patterns resolved from env vars, constrained by `allowedEnvVars` (hook-level) intersected with `httpHookAllowedEnvVars` (policy-level).
3. **Header sanitization**: CR/LF/NUL stripped to prevent CRLF injection.
4. **Sandbox proxy**: Routes through sandbox network proxy when sandboxing is enabled.
5. **Env var proxy**: Respects `HTTP_PROXY`/`HTTPS_PROXY`.
6. **SSRF guard**: `ssrfGuardedLookup` validates resolved IPs (skipped behind proxy).
7. Default timeout: 10 minutes.

### Callback Hook: `executeHookCallback()` (in hooks.ts)

Internal callback execution:
1. Creates combined abort signal with timeout.
2. Calls `hook.callback(hookInput, toolUseID, signal, hookIndex, context)`.
3. Returns the `HookJSONOutput` from the callback.

### Function Hook: `executeFunctionHook()` (in hooks.ts)

Session-scoped function execution:
1. Calls `hook.callback(messages, signal)`.
2. If returns `true`, hook passes (success).
3. If returns `false`, creates a blocking error with `hook.errorMessage`.

---

## Async Hook Registry: `src/utils/hooks/AsyncHookRegistry.ts`

### Global State (line 28)

```typescript
const pendingHooks = new Map<string, PendingAsyncHook>()
```

### Registration: `registerPendingAsyncHook()` (line 30)

Stores the hook with its `ShellCommand`, timeout (default 15s), and starts a progress interval for SDK event emission.

### Polling: `checkForAsyncHookResponses()` (line 113)

Called by the query loop to collect completed async hook results:
1. Iterates all pending hooks.
2. For completed processes, reads stdout/stderr.
3. Parses JSON response from output lines.
4. Emits hook response events.
5. Invalidates session env cache if a SessionStart hook completed.

### Background with Rewake: `executeInBackground()` (in hooks.ts, line 184)

`asyncRewake` hooks bypass the registry entirely. On completion:
- Exit code 2 -> Enqueues a `task-notification` that wakes the model.
- Other codes -> Silently completes.

---

## Tool Integration: `src/services/tools/toolHooks.ts`

### `runPreToolUseHooks()` (line 435)

Async generator bridging `executePreToolHooks()` to the tool execution pipeline. Yields typed results:
- `hookPermissionResult` -- Permission decisions from hooks
- `hookUpdatedInput` -- Modified tool input
- `preventContinuation` / `stopReason` -- Halt signals
- `additionalContext` -- Extra context for the model
- `stop` -- Abort tool execution

### `runPostToolUseHooks()` (line 39)

Processes post-tool results:
- Blocking errors -> attachment messages
- `preventContinuation` -> `hook_stopped_continuation` attachment
- `additionalContext` -> `hook_additional_context` attachment
- `updatedMCPToolOutput` -> replaces MCP tool output

### `resolveHookPermissionDecision()` (line 332)

Critical security function. Resolves hook permission results against rule-based permissions:

```
Hook says "allow" -> Still check checkRuleBasedPermissions()
  -> If rule says deny -> Override to deny
  -> If rule says ask -> Override to ask (dialog required)
  -> If rule says null (no rule) -> Use hook's allow
Hook says "deny" -> Always deny
Hook says "ask" -> Forward to canUseTool dialog
No hook decision -> Normal permission flow
```

---

## Event Broadcasting: `src/utils/hooks/hookEvents.ts`

### Architecture (lines 1-193)

Separate from the message stream. Three event types:
- **HookStartedEvent** -- Emitted when a hook begins execution
- **HookProgressEvent** -- Periodic stdout/stderr snapshots during execution
- **HookResponseEvent** -- Final result with exit code and outcome

### Emission Control

- `ALWAYS_EMITTED_HOOK_EVENTS` = `['SessionStart', 'Setup']`
- Other events only emitted when `allHookEventsEnabled` is true (set by SDK `includeHookEvents` option or CLAUDE_CODE_REMOTE mode).
- Pending events queue (max 100) buffers events until a handler is registered.

### Progress Interval: `startHookProgressInterval()` (line 124)

Creates an `unref`-ed interval (default 1s) that polls hook output and emits progress events when content changes. Used for SDK real-time hook visibility.

---

## Plugin Hooks: `src/utils/plugins/loadPluginHooks.ts`

### Registration Flow (lines 28-80)

1. `convertPluginHooksToMatchers()` transforms a `LoadedPlugin`'s `hooksConfig` into `PluginHookMatcher[]` per event.
2. Each matcher includes `pluginRoot`, `pluginName`, `pluginId` for context.
3. Registered via `registerHookCallbacks()` into the global bootstrap state.

### Hot Reload (referenced in the file)

Plugin hooks support hot-reload on settings change. A `settingsChangeDetector` subscription monitors `enabledPlugins` changes and re-registers hooks.

---

## Frontmatter & Skill Hooks

### `registerFrontmatterHooks()` (`src/utils/hooks/registerFrontmatterHooks.ts`)

Registers hooks from agent or skill YAML/JSON frontmatter as session hooks:
- For agents, `Stop` hooks are converted to `SubagentStop` (line 39-44).
- Uses `addSessionHook()` to store in session-scoped state.

### `registerSkillHooks()` (`src/utils/hooks/registerSkillHooks.ts`)

Similar to frontmatter registration, but with `once:` support:
- If `hook.once === true`, an `onHookSuccess` callback is registered that removes the hook after first successful execution (line 36-42).
- Passes `skillRoot` for `CLAUDE_PLUGIN_ROOT` env var.

---

## File Change Watcher: `src/utils/hooks/fileChangedWatcher.ts`

### Architecture (lines 1-192)

Uses `chokidar` for file watching:
1. **Initialization**: `initializeFileChangedWatcher(cwd)` -- reads FileChanged matchers from config, resolves paths, starts watching.
2. **Static paths**: From matcher patterns (pipe-separated filenames in cwd).
3. **Dynamic paths**: From hook output `watchPaths` arrays, updated at runtime.
4. **CWD change handling**: `onCwdChangedForHooks()` clears env files, runs CwdChanged hooks, updates watch paths.
5. **Stability**: `awaitWriteFinish` with 500ms threshold prevents duplicate events.

---

## Hook Helpers: `src/utils/hooks/hookHelpers.ts`

### `addArgumentsToPrompt()` (line 30)

Substitutes `$ARGUMENTS` placeholder in prompt/agent hook strings with the JSON input. Also supports indexed arguments `$ARGUMENTS[0]`, `$ARGUMENTS[1]`, shorthand `$0`, `$1`.

### `createStructuredOutputTool()` (line 41)

Creates a `StructuredOutput` tool for agent hooks with the `hookResponseSchema` (`{ok: boolean, reason?: string}`).

### `registerStructuredOutputEnforcement()` (line 70)

Adds a function hook on `Stop` event that checks if `StructuredOutput` was called. If not, the model receives an error message forcing it to call the tool.

---

## Hook Settings UI: `src/utils/hooks/hooksSettings.ts`

### `getAllHooks()` (line 92)

Collects hooks from:
1. User/project/local settings (with dedup by file path)
2. Session hooks for current session

Returns `IndividualHookConfig[]` with event, config, matcher, and source.

### `isHookEqual()` (line 33)

Compares hooks by type-specific content:
- Command: `command` + `shell` + `if`
- Prompt: `prompt` + `if`
- Agent: `prompt` + `if`
- HTTP: `url` + `if`
- Function: always returns `false` (no stable identifier)

---

## Post-Sampling Hooks: `src/utils/hooks/postSamplingHooks.ts`

### Internal Registry (lines 25-70)

Not exposed in settings.json. Programmatic-only API:
- `registerPostSamplingHook()` -- Adds to internal array.
- `executePostSamplingHooks()` -- Runs all after model sampling, errors logged but not propagated.

### API Query Hook Helper: `src/utils/hooks/apiQueryHookHelper.ts`

Factory pattern `createApiQueryHook()` (line 56) for building post-sampling hooks that:
1. Check `shouldRun()` condition
2. Build messages from context
3. Query model (with disabled thinking, optional tools)
4. Parse and log results

Used by auto-dream, compact warning, and other internal features.

---

## Exit Code Summary Table (All Events)

| Event | Exit 0 | Exit 2 | Other |
|---|---|---|---|
| PreToolUse | stdout hidden | stderr -> model, blocks tool | stderr -> user only |
| PostToolUse | stdout in transcript | stderr -> model immediately | stderr -> user only |
| PostToolUseFailure | stdout in transcript | stderr -> model immediately | stderr -> user only |
| UserPromptSubmit | stdout -> model | blocks processing, stderr -> user | stderr -> user only |
| SessionStart | stdout -> model | (ignored) | stderr -> user only |
| Stop | stdout hidden | stderr -> model, continue conv | stderr -> user only |
| StopFailure | (fire-and-forget) | (fire-and-forget) | (fire-and-forget) |
| SubagentStart | stdout -> subagent | (ignored) | stderr -> user only |
| SubagentStop | stdout hidden | stderr -> subagent, continue | stderr -> user only |
| Notification | stdout hidden | N/A | stderr -> user only |
| PreCompact | stdout -> custom instructions | blocks compaction | stderr -> user only |
| PostCompact | stdout -> user | N/A | stderr -> user only |
| SessionEnd | completes | N/A | stderr -> user only |
| Setup | stdout -> model | (ignored) | stderr -> user only |
| PermissionRequest | use hook decision | N/A | stderr -> user only |
| PermissionDenied | stdout in transcript | N/A | stderr -> user only |
| ConfigChange | allow change | block change | stderr -> user only |
| TeammateIdle | stdout hidden | stderr -> teammate, prevent idle | stderr -> user only |
| TaskCreated | stdout hidden | stderr -> model, prevent creation | stderr -> user only |
| TaskCompleted | stdout hidden | stderr -> model, prevent completion | stderr -> user only |
| Elicitation | use hook response | deny elicitation | stderr -> user only |
| ElicitationResult | use hook response | block response | stderr -> user only |
| CwdChanged | completes | N/A | stderr -> user only |
| FileChanged | completes | N/A | stderr -> user only |
| InstructionsLoaded | completes | N/A | stderr -> user only |
| WorktreeCreate | stdout = worktree path | creation failed | creation failed |
| WorktreeRemove | completes | N/A | stderr -> user only |
