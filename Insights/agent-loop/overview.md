# Agent Loop Architecture in Claude Code CLI

## Overview

The agent loop is the beating heart of Claude Code — the core cycle that drives all interaction between the model, tools, and user. It is implemented as a **three-layer async generator architecture** where a single universal `query()` function powers both the main REPL session and every sub-agent, with context isolation being the only differentiator.

```
┌──────────────────────────────────────────────────────────────────┐
│                     OUTER REPL LAYER                             │
│  User Input → onQuery() → onQueryImpl() → iterate generator     │
│  React/Ink UI ◄── onQueryEvent() ◄── yield from query()         │
│  (REPL.tsx, replLauncher.tsx, interactiveHelpers.tsx)            │
└──────────────────────────┬───────────────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────────────┐
│                     QUERY LOOP LAYER                             │
│  while(true) {                                                   │
│    1. Pre-process (compact, microcompact, token checks)          │
│    2. API call + stream response                                 │
│    3. Post-sampling hooks                                        │
│    4. Abort check                                                │
│    5. If no tool_use → recovery/stop/return                      │
│    6. Execute tools (streaming or batch)                         │
│    7. Collect attachments (memory, skills, MCP refresh)          │
│    8. Turn limit check → continue or return                      │
│  }                                                               │
│  (query.ts ~1730 lines)                                          │
└──────────────────────────┬───────────────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────────────┐
│                   TOOL EXECUTION LAYER                           │
│  ┌─────────────────────┐  ┌──────────────────────────────────┐   │
│  │ toolOrchestration   │  │ StreamingToolExecutor            │   │
│  │ (non-streaming)     │  │ (streaming — tools start before  │   │
│  │ partition → batch   │  │  response completes)             │   │
│  │ → concurrent/serial │  │ queued→executing→completed→yield │   │
│  └─────────────────────┘  └──────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

---

## 1. The Universal Query Loop (`query.ts`)

### Core Design

The loop is an `async function* queryLoop()` — an async generator that yields messages/events and returns a `Terminal` reason when done. This design enables:

- **Lazy evaluation** — messages yielded as they become available
- **Composability** — `yield*` delegates to sub-generators (stop hooks, tool execution)
- **Backpressure** — the consumer (REPL or parent agent) controls the pace
- **Clean cancellation** — generators clean up on `.return()`

### Loop State

A `State` object carries all cross-iteration state and is **replaced wholesale** (never mutated) at each continue site:

| Field | Purpose |
|-------|---------|
| `messages` | Full conversation history (grows each iteration) |
| `toolUseContext` | Tools, abort controller, app state, file cache |
| `turnCount` | Current turn number (starts at 1) |
| `autoCompactTracking` | Compaction state (flag, failures, turn counter) |
| `maxOutputTokensRecoveryCount` | Recovery attempts for max-output-tokens (max 3) |
| `hasAttemptedReactiveCompact` | One-shot guard for reactive compaction |
| `maxOutputTokensOverride` | Escalated token cap (8k → 64k) |
| `pendingToolUseSummary` | Async summary from previous turn |
| `stopHookActive` | Whether stop hooks have fired |
| `transition` | Why the previous iteration continued |

### The 8-Phase Iteration

Each iteration of the `while(true)` loop executes these phases in strict order:

**Phase 1 — Pre-processing:**
1. Skill discovery prefetch (async, non-blocking)
2. Query chain tracking (depth + UUID)
3. Tool result budget trimming (oversized results)
4. Snip compaction (feature-gated)
5. Microcompact (trim old tool results, no API call)
6. Context collapse (feature-gated)
7. Auto-compact (proactive — tries session memory compact first, then full compact)
8. Blocking token limit check (synthetic early exit)

**Phase 2 — API Call + Streaming:**
- Calls `deps.callModel()` inside a `while(attemptWithFallback)` retry loop
- For each streamed `assistant` message: push to array, extract `tool_use` blocks
- If streaming execution is enabled: tools start running via `StreamingToolExecutor.addTool()` while the response is still arriving
- Recoverable errors (413 prompt-too-long, max-output-tokens, media errors) are **withheld** from yield for recovery handling
- Model fallback: on `FallbackTriggeredError` or 529 exhaustion, clears state, yields tombstones, retries with fallback model

**Phase 3 — Post-Sampling Hooks:**
- Fire-and-forget `executePostSamplingHooks()` (session memory extraction, analytics)

**Phase 4 — Abort Check:**
- If `abortController.signal.aborted`: drain streaming results, yield interruption message, return `aborted_streaming`

**Phase 5 — No Tool Use Path (model wants to stop):**
- Recovery escalation chain for withheld errors (see section 3)
- Stop hooks — can inject blocking errors that force the loop to continue
- Token budget continuation — if budget not exhausted, inject nudge message
- Otherwise: `return { reason: 'completed' }`

**Phase 6 — Tool Execution:**
- If tools were invoked: consume results from `StreamingToolExecutor.getRemainingResults()` or `runTools()`
- Each tool result is yielded to the consumer and accumulated in `toolResults[]`
- Context modifiers from tools applied to `toolUseContext`

**Phase 7 — Attachment Collection:**
- Drain queued commands
- Collect file change attachments, nested memory, conditional rules
- Consume memory and skill discovery prefetch results
- Refresh MCP tools (server reconnections, tool list changes)

**Phase 8 — Turn Limit + Continue:**
- If `turnCount > maxTurns`: return `{ reason: 'max_turns' }`
- Otherwise: build new `State` with appended messages, incremented turn count, and `transition: { reason: 'next_turn' }`, then implicitly continue

### Prefetch-During-Stream Pattern

Multiple expensive operations are kicked off in parallel with model streaming to hide latency:

| Prefetch | Started | Consumed |
|----------|---------|----------|
| Relevant memory | Once per user turn | After tool execution |
| Skill discovery | Per iteration | After tool execution |
| Tool use summary | After tool batch | Next iteration start |

---

## 2. Continuation vs Termination

### 9 Continue Sites (transition reasons)

| Reason | Trigger |
|--------|---------|
| `next_turn` | Normal: tool results collected, loop again |
| `collapse_drain_retry` | Context collapse recovered from 413 |
| `reactive_compact_retry` | Reactive compact succeeded after 413 or media error |
| `max_output_tokens_escalate` | Escalation from 8k to 64k default |
| `max_output_tokens_recovery` | Inject "resume directly" meta-message (up to 3x) |
| `stop_hook_blocking` | Stop hook returned blocking errors |
| `token_budget_continuation` | Token budget not yet exhausted |
| Model fallback retry | 529 exhaustion triggers fallback model |
| Streaming fallback | Mid-stream fallback during response |

### 10 Terminal Conditions (return reasons)

| Reason | Condition |
|--------|-----------|
| `completed` | Model stopped naturally (no tool_use blocks) |
| `max_turns` | Turn count exceeded `maxTurns` |
| `aborted_streaming` | User cancelled during streaming (Escape/Ctrl+C) |
| `aborted_tools` | User cancelled during tool execution |
| `blocking_limit` | Pre-flight token check failed, auto-compact off |
| `prompt_too_long` | 413 recovery exhausted |
| `model_error` | Unhandled API error |
| `image_error` | Image size/resize error |
| `stop_hook_prevented` | Stop hook explicitly prevented continuation |
| `hook_stopped` | Tool-level hook prevented continuation |

### Recovery Escalation Chains

**For prompt-too-long (413):**
1. Context collapse drain (cheap, keeps granular context)
2. Reactive compact (full API summarization)
3. Surface error and return

**For max-output-tokens:**
1. Escalate token cap 8k → 64k (single retry, no meta-message)
2. Inject "resume directly" message (up to 3 times)
3. Surface the withheld error

---

## 3. Tool Execution Mechanics

### Non-Streaming Path (`toolOrchestration.ts`)

Used when streaming tool execution is disabled:

1. **Partition** tool calls via `partitionToolCalls()`:
   - Consecutive concurrency-safe tools → concurrent batch
   - Any non-safe tool → serial batch of one
   - Example: `[Read, Read, Write, Read, Read]` → `[{concurrent: [R,R]}, {serial: [W]}, {concurrent: [R,R]}]`

2. **Execute batches** in order:
   - Concurrent: up to 10 tools in parallel (`MAX_TOOL_USE_CONCURRENCY`)
   - Serial: one at a time, context modifiers applied between runs

### Streaming Path (`StreamingToolExecutor.ts`)

Enables tools to start executing **before the model response completes**:

**Tool lifecycle:** `queued` → `executing` → `completed` → `yielded`

**Concurrency control:** A tool can execute if:
- Nothing is currently executing, OR
- Both the tool and all currently executing tools are concurrency-safe

**Two consumption points:**
1. **During streaming** (`getCompletedResults()`, synchronous): Yields completed results that maintain ordering. If a non-concurrent tool is executing, stops yielding to preserve order.
2. **After streaming** (`getRemainingResults()`, async): Waits for all tools to complete. Uses `Promise.race()` on executing promises + progress-available promise for real-time progress updates.

**Sibling abort (error cascading):**
- When a **Bash tool** errors, a `siblingAbortController` fires, cancelling all queued tools
- Only Bash errors cascade (implicit dependencies). Read/WebFetch errors are independent
- Each tool has its own `toolAbortController` (child of sibling controller)

---

## 4. The Outer REPL Layer

### React/Ink Integration (`REPL.tsx`)

The REPL is a React component — the outer "loop" is event-driven, not imperative:

1. User types in `PromptInput` component
2. `handlePromptSubmit` processes input (slash commands, bash mode, etc.)
3. `onQuery()` acquires the query guard (prevents concurrent queries), appends messages, calls `onQueryImpl()`
4. `onQueryImpl()` loads system prompt + user context + system context in parallel, builds `ToolUseContext`, then iterates the `query()` generator:

```
for await (const event of query({ messages, systemPrompt, ... })) {
    onQueryEvent(event)  // Routes to React state updates
}
```

5. `onQueryEvent()` bridges generator events to React state:
   - Compact boundaries reset the messages array
   - Progress messages replace previous (avoid array bloat)
   - All other messages are appended

### Queue System

The `useQueueProcessor` hook bridges React rendering and the query loop:
- Subscribes to both `QueryGuard` (is a query active?) and the command queue
- When no query is active and queue has items, drains commands
- `useCancelRequest` handles Escape (cancel active task) and Ctrl+C (abort or exit)

---

## 5. Sub-Agent Loops

### Key Insight: Same `query()`, Different Context

Sub-agents reuse the **exact same `query()` function**. The loop is not adapted or duplicated. What differs is the **context** passed in:

| Aspect | Main Loop | Sub-Agent |
|--------|-----------|-----------|
| System prompt | Full (CLAUDE.md, git status, tools) | Agent-specific or parent's (fork) |
| Tool set | All available tools | Filtered by `resolveAgentTools()` |
| Message history | User conversation | Fresh (or forked from parent) |
| Abort controller | User-controlled | Independent (async) or shared (sync) |
| Thinking | User's preference | Disabled (normal) or inherited (fork) |
| UI callbacks | Full terminal rendering | All null/undefined |
| Permission mode | User's mode | Agent-defined or `bubble` (fork) |
| maxTurns | None or user-set | From agent definition |

### `runAgent()` — The Sub-Agent Bridge

`runAgent()` is an `AsyncGenerator<Message, void>` that wraps `query()`:

1. **Constructs isolated context**: Resolves model, creates `agentId`, builds system prompt, optionally strips CLAUDE.md/gitStatus for lightweight agents (Explore, Plan)
2. **Creates `ToolUseContext`** via `createSubagentContext()` with isolation (cloned file state, fresh denial tracking, no-op `setAppState`)
3. **Calls `query()` identically**: Same parameters structure, same generator protocol
4. **Yields messages** back to the parent (forwarding TTFT metrics, recording to sidechain transcript)
5. **Cleans up in `finally`**: MCP servers, session hooks, file state cache, bash tasks, perfetto registry

### `AgentTool.call()` — The Dispatch Layer

The Agent tool's `call()` method decides the execution mode:

```
AgentTool.call()
  ├── team_name + name? → spawnTeammate() (separate process)
  ├── isolation: 'remote'? → teleportToRemote() (cloud container)
  ├── shouldRunAsync? → registerAsyncAgent() + runAgent() in background
  │     └── On completion: enqueueAgentNotification() → <task-notification>
  ├── sync? → iterate runAgent() in foreground
  │     └── Can be backgrounded mid-flight → relaunches as async
  └── fork? → inherits parent's prompt/tools/messages for cache sharing
```

### Fork Sub-Agents (Prompt Cache Optimization)

Fork children maximize prompt cache hits by keeping the API request prefix **byte-identical** to the parent:
- Inherit parent's exact system prompt
- Inherit parent's exact tool array
- Clone parent's assistant message with placeholder tool_results
- Inherit thinking config
- Append a per-child directive as the final text block

### Coordinator Mode

Not a different loop — just a **configuration overlay**:
- Main agent's system prompt replaced with coordinator instructions
- Only `Agent`, `SendMessage`, `TaskStop` tools available
- All spawned workers are forced async
- Worker results arrive as `<task-notification>` XML injected into the coordinator's conversation

---

## 6. Error Recovery and Resilience

### Rate Limiting & Retry (`withRetry.ts`)
- Default 10 retries with exponential backoff (500ms base, 32s cap)
- Honors `retry-after` header
- 529 errors: max 3 consecutive before triggering model fallback
- **Persistent retry mode** (`CLAUDE_CODE_UNATTENDED_RETRY`): retries 429/529 indefinitely with 5-min max backoff, heartbeat messages every 30s

### Network Errors
- `ECONNRESET`/`EPIPE`: disables keep-alive, refreshes client, retries
- OAuth 403: forces token refresh
- AWS/GCP credential errors: clears credential cache and retries

### User Abort
- Checked at 4 points: before retry, after streaming, after tool execution, during stop hooks
- Submit-interrupt (user typed while streaming): skips interruption message, queued message provides context
- Streaming executor: drains remaining results as synthetic errors for in-progress tools

### Auto-Compact Integration
- Runs at the **top of each iteration**, before the API call
- On success: yields post-compact messages, replaces message array, loop continues
- Circuit breaker: stops after 3 consecutive failures (`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES`)
- Reactive compact: fires **after** a 413 error, single-shot guard

---

## 7. Relationships to Other Subsystems

### → Tools System
The query loop drives tool execution via Phase 6. Each tool call flows through the full 10-phase tool execution pipeline (validation → hooks → permissions → execution → result mapping). The `StreamingToolExecutor` enables overlapping tool execution with model streaming.

### → Permission System
Permissions are checked inside tool execution (Phase 6), not in the loop itself. The loop passes `canUseTool` into `query()` which threads it through to tool orchestration. Sub-agents can have different permission modes affecting how tools are gated.

### → Hooks System
Hooks integrate at multiple loop phases:
- **Post-sampling hooks** (Phase 3): session memory extraction, analytics
- **Stop hooks** (Phase 5): verification gates that can force continuation
- **PreToolUse/PostToolUse** (Phase 6): wrapped around each tool execution
- **UserPromptSubmit**: fires before the loop starts (in REPL layer)

### → CLAUDE.md / Context
Context assembly happens **once** at the start of each `onQueryImpl()` call, then cached. CLAUDE.md content enters via `getUserContext()`. The loop itself doesn't reload CLAUDE.md — but lazy loading via `attachments.ts` can add conditional rules after tool execution (Phase 7).

### → Session/Memory
- **Session memory extraction**: fires as a post-sampling hook (Phase 3), writing running notes used by Tier 2 compaction
- **Auto-compact**: runs at the top of each iteration (Phase 1), using session memory notes or full API summarization
- **Memory prefetch**: kicked off during streaming, consumed after tool execution (Phase 7)
- **Extract memories**: runs at query loop end via stop hooks

### → Agent Orchestration
Sub-agents are spawned during tool execution (Phase 6) when the model calls `AgentTool`. The spawned agent runs its own `query()` loop with isolated context. Results flow back as either:
- **Sync**: tool_result content from the last assistant message
- **Async**: `<task-notification>` XML injected into the parent's conversation

---

## 8. Design Insights

### Why Async Generators?
The generator architecture is the key design choice. It allows:
- **Incremental processing**: REPL renders messages as they arrive, not buffered
- **Natural cancellation**: `AbortController` + generator `.return()` = clean teardown
- **Composable phases**: Stop hooks, tool execution, attachment collection are all sub-generators delegated via `yield*`
- **Backpressure**: The consumer controls the pace, preventing runaway tool execution

### Why State Replacement?
The loop creates a new `State` at each continue site rather than mutating fields. This makes it easy to audit what changes between iterations and prevents subtle mutation bugs across the 1730-line function.

### Why One `query()` for Everything?
By parameterizing everything (tools, prompt, permissions, abort, messages), the same loop serves main sessions, sub-agents, coordinators, fork children, and SDK consumers. This eliminates code duplication and ensures all agents benefit from the same recovery logic, compaction, and hook integration.
