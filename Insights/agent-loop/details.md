# Agent Loop — Code-Level Details

This document provides detailed code-level references for the agent loop architecture. All paths are relative to `restored-src/src/`.

---

## 1. Core Files Map

| File | Lines | Role |
|------|-------|------|
| `query.ts` | ~1730 | Core `while(true)` loop — API calls, tool dispatch, recovery, continuation |
| `QueryEngine.ts` | ~400 | SDK/headless wrapper around `query()` with budget/turn limits |
| `screens/REPL.tsx` | ~3200 | React UI — user input, query invocation, message rendering |
| `replLauncher.tsx` | ~50 | Minimal launcher — renders `<App><REPL /></App>` |
| `interactiveHelpers.tsx` | ~300 | Setup dialogs, render context, onboarding flow |
| `services/tools/toolOrchestration.ts` | ~350 | Non-streaming tool execution with concurrent/serial batching |
| `services/tools/StreamingToolExecutor.ts` | ~550 | Streaming tool execution — tools start before response completes |
| `services/tools/toolExecution.ts` | ~1150 | Per-tool execution pipeline (`checkPermissionsAndCallTool`) |
| `tools/AgentTool/AgentTool.tsx` | ~1350 | Agent dispatch — sync/async/fork/remote/team routing |
| `tools/AgentTool/runAgent.ts` | ~974 | Sub-agent bridge — constructs context, calls `query()`, cleanup |
| `query/deps.ts` | ~80 | Dependency injection (callModel, microcompact, autocompact, uuid) |
| `query/config.ts` | ~60 | Immutable per-query config snapshot (session ID, feature gates) |
| `query/stopHooks.ts` | ~200 | Stop/TeammateIdle/TaskCompleted hook execution |
| `query/tokenBudget.ts` | ~150 | Token budget tracking — continue vs stop based on output spend |
| `state/AppStateStore.ts` | ~570 | AppState type definition (~450 fields) |
| `state/store.ts` | ~34 | Minimal Zustand-like external store |
| `state/AppState.tsx` | ~150 | React provider + `useAppState()` / `useSetAppState()` hooks |
| `state/onChangeAppState.ts` | ~200 | State change side effects (mode sync, model sync) |
| `context.ts` | ~150 | `getSystemContext()` and `getUserContext()` — memoized |
| `Tool.ts` | ~200 | `ToolUseContext` type definition |
| `utils/forkedAgent.ts` | ~400 | `createSubagentContext()` — isolation boundary |
| `services/api/withRetry.ts` | ~300 | Retry/backoff logic (10 retries, exponential, fallback) |
| `services/api/claude.ts` | ~600 | Streaming API client, prompt caching, token budgets |
| `services/compact/autoCompact.ts` | ~250 | Auto-compact decision + execution |
| `hooks/useQueueProcessor.ts` | ~100 | React-to-query bridge |
| `hooks/useCancelRequest.ts` | ~80 | Cancel/interrupt handling |

---

## 2. Query Loop Internals (`query.ts`)

### Entry Point (line ~219)

```typescript
export async function* query(params: QueryParams): AsyncGenerator<
  StreamEvent | RequestStartEvent | Message | TombstoneMessage | ToolUseSummaryMessage,
  Terminal
>
```

Delegates to `queryLoop()` then notifies command lifecycle for consumed commands.

### State Type (line ~204)

```typescript
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState
  maxOutputTokensRecoveryCount: number        // max 3
  hasAttemptedReactiveCompact: boolean         // one-shot guard
  maxOutputTokensOverride: number | undefined  // 8k → 64k escalation
  pendingToolUseSummary: Promise<...> | undefined
  stopHookActive: boolean | undefined
  turnCount: number                           // starts at 1
  transition: Continue | undefined            // why we continued
}
```

### Main Loop Structure (line ~307)

```typescript
while (true) {
  // Phase 1: Pre-processing (lines 307-648)
  //   - skill discovery prefetch
  //   - query chain tracking
  //   - tool result budget
  //   - snip compaction
  //   - microcompact (cached or regular)
  //   - context collapse
  //   - autocompact
  //   - blocking token limit check

  // Phase 2: API Call + Streaming (lines 654-953)
  //   while(attemptWithFallback) {
  //     for await (const message of deps.callModel({...})) {
  //       // stream processing, tool_use extraction
  //       // StreamingToolExecutor.addTool() for each tool block
  //     }
  //   }

  // Phase 3: Post-sampling hooks (lines 999-1009)
  //   executePostSamplingHooks() — fire-and-forget

  // Phase 4: Abort check (lines 1015-1052)
  //   if (abortController.signal.aborted) → drain, yield interruption, return

  // Phase 5: No tool_use path (lines 1062-1358)
  //   Recovery chain: collapse → reactive compact → token escalation → stop hooks
  //   → token budget continuation → return 'completed'

  // Phase 6: Tool execution (lines 1360-1408)
  //   streamingToolExecutor.getRemainingResults() or runTools()

  // Phase 7: Attachment collection (lines 1536-1671)
  //   queued commands, file changes, memory, skills, MCP refresh

  // Phase 8: Turn limit + continue (lines 1679-1728)
  //   if (turnCount > maxTurns) return
  //   state = { ...next }  // continue to top
}
```

### Continue Sites (9 total)

| Line Range | Transition Reason | Trigger |
|-----------|-------------------|---------|
| ~1115 | `collapse_drain_retry` | Context collapse drained after 413 |
| ~1165 | `reactive_compact_retry` | Reactive compact succeeded |
| ~1220 | `max_output_tokens_escalate` | First escalation 8k→64k |
| ~1250 | `max_output_tokens_recovery` | "Resume directly" injection (≤3x) |
| ~1305 | `stop_hook_blocking` | Stop hook blocking errors |
| ~1337 | `token_budget_continuation` | Budget not exhausted |
| ~1725 | `next_turn` | Normal tool use follow-up |
| ~894 | (fallback retry) | `FallbackTriggeredError` caught |
| ~712 | (streaming fallback) | `streamingFallbackOccured` set |

### Abort Handling (lines ~1015-1052)

```typescript
if (abortController.signal.aborted) {
  // If streaming executor active: drain getRemainingResults()
  //   → synthetic errors for in-progress tools
  // Otherwise: yieldMissingToolResultBlocks with "Interrupted by user"
  // Submit-interrupt (reason === 'interrupt'): skip interruption message
  return { reason: 'aborted_streaming' }
}
```

Second check after tool execution (line ~1485):
```typescript
if (abortController.signal.aborted) {
  // yield interruption
  return { reason: 'aborted_tools' }
}
```

### Auto-Compact Integration (lines ~454-543)

```typescript
// Top of each iteration, before API call
const compactResult = await deps.autocompact(messagesForQuery, ...)
if (compactResult) {
  yield* compactResult.postCompactMessages
  messagesForQuery = compactResult.messages
  // tracking state updated
}
```

Threshold: `effectiveContextWindow - 13_000` (AUTOCOMPACT_BUFFER_TOKENS)
Circuit breaker: `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3`

### Reactive Compact (lines ~1119-1175)

```typescript
// After 413 prompt-too-long, single-shot guard
if (!hasAttemptedReactiveCompact) {
  const compactResult = await deps.autocompact(messages, { force: true })
  if (compactResult) {
    state = { ...state, messages: compactResult.messages,
              hasAttemptedReactiveCompact: true,
              transition: { reason: 'reactive_compact_retry' } }
    continue
  }
}
```

---

## 3. REPL Integration (`screens/REPL.tsx`)

### `onQuery` (line ~2855)

```typescript
async function onQuery(newMessages) {
  // Acquire queryGuard (prevents concurrent queries)
  // If already running → enqueue message for later
  // Append messages via setMessages
  // Snapshot token budgets
  // Call onQueryImpl()
  // finally: release guard, reset loading, handle swarm turn completion
}
```

### `onQueryImpl` (line ~2661)

```typescript
async function onQueryImpl() {
  // Parallel load: systemPrompt, userContext, systemContext
  const [systemPrompt, userContext, systemContext] = await Promise.all([...])

  // Build ToolUseContext
  const toolUseContext = { options, abortController, readFileState, ... }

  // Iterate generator
  for await (const event of query({
    messages, systemPrompt, userContext, systemContext,
    canUseTool, toolUseContext, querySource
  })) {
    onQueryEvent(event)
  }

  // Post-loop: log metrics, reset loading, onTurnComplete
}
```

### `onQueryEvent` (line ~2584)

Routes events to React state via `handleMessageFromStream()`:
- Compact boundaries → reset messages array
- Progress messages → replace previous (avoid array bloat)
- All others → append to messages

---

## 4. Tool Orchestration (`services/tools/toolOrchestration.ts`)

### `runTools()` — Non-Streaming Path

```typescript
export async function* runTools(
  toolUseMessages: ToolUseBlock[],
  assistantMessages: AssistantMessage[],
  canUseTool: CanUseToolFn,
  toolUseContext: ToolUseContext
): AsyncGenerator<MessageUpdate, void>
```

### `partitionToolCalls()`

Groups consecutive tool calls:
```
Input:  [Read, Read, Write, Read, Read]
Output: [{concurrent: [Read, Read]}, {serial: [Write]}, {concurrent: [Read, Read]}]
```

Decision: `tool.isConcurrencySafe(parsedInput)` on each tool definition.

### Concurrency Limit

```typescript
getMaxToolUseConcurrency() → parseInt(CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY) || 10
```

Uses `all()` from `utils/generators.js` — merges async generators with concurrency limit.

---

## 5. Streaming Tool Executor (`services/tools/StreamingToolExecutor.ts`)

### Tool State Machine

```typescript
type ToolStatus = 'queued' | 'executing' | 'completed' | 'yielded'

type TrackedTool = {
  id: string
  block: ToolUseBlock
  status: ToolStatus
  isConcurrencySafe: boolean
  promise?: Promise<void>
  results?: Message[]
  pendingProgress: Message[]
  contextModifiers?: Array<(ctx: ToolUseContext) => ToolUseContext>
}
```

### Concurrency Control (`canExecuteTool`, line ~180)

```typescript
canExecuteTool(isConcurrencySafe: boolean): boolean {
  // Can execute if nothing running
  // OR if this tool is safe AND all running tools are safe
}
```

### Queue Processing (`processQueue`, line ~200)

```typescript
processQueue() {
  for (const tool of tools) {
    if (tool.status !== 'queued') continue
    if (canExecuteTool(tool.isConcurrencySafe)) {
      startTool(tool)
    } else if (!tool.isConcurrencySafe) {
      break  // Must wait — ordering matters
    }
    // Concurrent tools that can't execute: skip (don't break)
  }
}
```

### Two Consumption Points

**During streaming** (`getCompletedResults`, line ~412, synchronous generator):
```typescript
*getCompletedResults(): Generator<MessageUpdate> {
  for (const tool of tools) {
    if (tool.status === 'completed') {
      yield* tool.results  // In order
      tool.status = 'yielded'
    }
    yield* tool.pendingProgress  // Real-time progress
    if (tool.status === 'executing' && !tool.isConcurrencySafe) break  // Preserve order
  }
}
```

**After streaming** (`getRemainingResults`, line ~453, async generator):
```typescript
async *getRemainingResults(): AsyncGenerator<MessageUpdate> {
  while (hasUncompletedTools()) {
    // Promise.race: executing tool promises + progress-available promise
    await Promise.race([...executingPromises, progressPromise])
    yield* getCompletedResults()  // Drain completed
  }
}
```

### Sibling Abort (line ~350)

```typescript
// Only Bash errors cascade
if (tool.name === 'Bash' && result.is_error) {
  siblingAbortController.abort('sibling_error')
  // All queued tools get synthetic error messages
}
```

### Discard (line ~500)

```typescript
discard() {
  this.discarded = true
  // Prevents all pending tools from executing
  // All results from being yielded
  // Used during streaming fallback
}
```

---

## 6. Sub-Agent Loop (`tools/AgentTool/`)

### `AgentTool.call()` Decision Tree (AgentTool.tsx, line ~250)

```
call(input, toolUseContext, canUseTool)
  │
  ├── input.team_name + input.name → spawnTeammate()
  │     (line ~284, separate process via tmux/iTerm2/in-process)
  │
  ├── resolve AgentDefinition from subagent_type (lines 322-356)
  │
  ├── input.isolation === 'remote' → teleportToRemote() (line ~435)
  │
  ├── assemble tool pool via assembleToolPool() (lines 573-577)
  │
  ├── shouldRunAsync? (line ~567)
  │     true if: run_in_background || agent.background || coordinator || fork || assistant
  │     │
  │     ├── YES → registerAsyncAgent() + void runAgent() (lines 686-764)
  │     │         On completion: enqueueAgentNotification() → <task-notification>
  │     │         Return immediately: { status: 'async_launched', agentId }
  │     │
  │     └── NO → iterate runAgent() in while(true) (lines 765+)
  │               Race each next() against backgroundPromise
  │               If backgrounded mid-flight: relaunch as async
  │               Collect messages → finalizeAgentTool()
  │
  └── fork path (when subagent_type omitted + fork enabled)
        Inherit parent's prompt/tools/messages for cache sharing
```

### `runAgent()` Flow (runAgent.ts)

```typescript
async function* runAgent(params): AsyncGenerator<Message, void> {
  // 1. Resolve model (lines ~500-520)
  const model = getAgentModel(agentDefinition, parentModel, modelOverride)

  // 2. Create agent ID (line ~530)
  const agentId = createAgentId()

  // 3. Build system prompt (lines ~540-600)
  //    - Agent-specific prompt OR parent's prompt (fork)
  //    - Optionally strip claudeMd/gitStatus for lightweight agents

  // 4. Create isolated context (lines ~610-660)
  const agentToolUseContext = createSubagentContext(toolUseContext, {
    abortController: isAsync ? new AbortController() : parentAbortController,
    readFileState: clone(parent.readFileState),
    setAppState: isAsync ? noop : parentSetAppState,
    // ...
  })

  // 5. Preload skills (lines ~670-700)
  //    Convert frontmatter skills to initial user messages

  // 6. Initialize agent MCP servers (lines ~700-714)

  // 7. Call query() — identical to main loop (lines ~748-806)
  for await (const message of query({
    messages: initialMessages,
    systemPrompt: agentSystemPrompt,
    userContext: resolvedUserContext,
    systemContext: resolvedSystemContext,
    canUseTool,
    toolUseContext: agentToolUseContext,
    querySource,
    maxTurns: maxTurns ?? agentDefinition.maxTurns,
  })) {
    // Forward TTFT metrics
    // Record to sidechain transcript
    yield message  // Forward to parent
  }

  // 8. Cleanup in finally (lines ~816-858)
  //    - Clean up agent MCP servers
  //    - Clear session hooks
  //    - Release file state cache
  //    - Kill background bash tasks
  //    - Unregister from perfetto
  //    - Remove todo entries
}
```

### Context Isolation (`utils/forkedAgent.ts`, `createSubagentContext`, line ~345)

| Field | Main Loop | Sub-Agent (Sync) | Sub-Agent (Async) |
|-------|-----------|------------------|-------------------|
| `readFileState` | Original | Cloned | Cloned |
| `abortController` | User-controlled | **Shared** with parent | **New** independent |
| `getAppState` | Normal | Normal | Wrapped (`shouldAvoidPermissionPrompts: true`) |
| `setAppState` | Normal | Normal | **No-op** |
| `setAppStateForTasks` | Root store | **Shared** with root | **Shared** with root |
| `messages` | User conversation | **Fresh** array | **Fresh** array |
| `agentId` | Session ID | **New** UUID | **New** UUID |
| `queryTracking` | Depth 0 | **Depth N+1** | **Depth N+1** |
| `contentReplacementState` | Original | **Cloned** | **Cloned** |
| UI callbacks | Full terminal | **All undefined** | **All undefined** |

### `finalizeAgentTool()` (agentToolUtils.ts, line ~276)

```typescript
function finalizeAgentTool(messages, agentId, startTime): AgentToolResult {
  // Extract text content from last assistant message
  // Count total tool uses across all messages
  // Sum token usage
  // Log analytics event
  // Return { content, toolCount, usage, duration }
}
```

---

## 7. QueryEngine — SDK/Headless Wrapper (`QueryEngine.ts`)

For non-interactive (SDK, headless) consumers:

```typescript
class QueryEngine {
  async *runQuery(messages, options): AsyncGenerator<Message, QueryResult> {
    // Check budget: getTotalCost() >= maxBudgetUsd → error_max_budget_usd
    // Check turns: numTurns >= maxTurns → error_max_turns
    // Calls query() with same parameters
    // Aggregates messages, handles errors
    // Returns QueryResult with terminal reason
  }
}
```

---

## 8. API Client Layer (`services/api/`)

### Retry Logic (`withRetry.ts`)

```typescript
DEFAULT_MAX_RETRIES = 10
BASE_DELAY_MS = 500
MAX_DELAY_MS = 32_000

// Exponential backoff: BASE * 2^(attempt-1), capped at MAX
// Honors retry-after header
// 529: max 3 consecutive before FallbackTriggeredError
// Persistent mode (CLAUDE_CODE_UNATTENDED_RETRY):
//   - 429/529 retry indefinitely
//   - 5-min max backoff
//   - Heartbeat messages every 30s
```

### Model Fallback Chain

```
Primary model → 529 exhaustion (3x) → FallbackTriggeredError
  → query.ts catches at line ~894
  → Clears assistantMessages, toolResults, toolUseBlocks
  → Yields tombstone messages for orphans
  → Discards streaming executor
  → Retries with fallback model
```

### Streaming Client (`claude.ts`)

```typescript
// Creates Anthropic SDK client (supports direct, Bedrock, Vertex, Foundry)
// Handles: streaming, prompt caching, effort parameters, token budgets
// Both main loop and sub-agents share the same client path
// Prompt cache config: system prompt cached, tools cached, recent messages cached
```

---

## 9. React Hooks Driving the Loop

### `useQueueProcessor` (`hooks/useQueueProcessor.ts`)

```typescript
// Subscribes to QueryGuard (is query active?) + command queue
// When: !queryActive && !localJSXBlocking && queue.hasItems
// → processQueueIfReady() → drains commands → triggers onQuery
```

### `useCancelRequest` (`hooks/useCancelRequest.ts`)

```typescript
// Registers keybindings:
//   'chat:cancel' (Escape) → abort active task or pop queue
//   'app:interrupt' (Ctrl+C) → abort or escalate to exit
// Priority:
//   1. Active task with abortSignal → abort it
//   2. Queued commands → pop from queue
//   3. Fallback → clear confirm queue, call onCancel
```

### `useExitOnCtrlCD` (`hooks/useExitOnCtrlCD.ts`)

```typescript
// Double-press pattern:
//   First Ctrl+C/D → "Press again to exit"
//   Second within timeout → process.exit()
```

---

## 10. Key Constants

| Constant | Value | Location |
|----------|-------|----------|
| `MAX_TOOL_USE_CONCURRENCY` | 10 | `toolOrchestration.ts` |
| `AUTOCOMPACT_BUFFER_TOKENS` | 13,000 | `autoCompact.ts` |
| `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` | 3 | `autoCompact.ts` |
| `MAX_OUTPUT_TOKENS_RECOVERY_LIMIT` | 3 | `query.ts` |
| `ESCALATED_MAX_TOKENS` | 64,000 | `query.ts` |
| `CAPPED_DEFAULT_MAX_TOKENS` | 8,000 | `claude.ts` |
| `DEFAULT_MAX_RETRIES` | 10 | `withRetry.ts` |
| `BASE_DELAY_MS` | 500 | `withRetry.ts` |
| `MAX_DELAY_MS` | 32,000 | `withRetry.ts` |
| `MAX_529_CONSECUTIVE` | 3 | `withRetry.ts` |
| `MODEL_CONTEXT_WINDOW_DEFAULT` | 200,000 | `context.ts` |
