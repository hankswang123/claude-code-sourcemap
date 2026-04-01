# Sub-Agents and Orchestration: Detailed Code-Level Analysis

This document provides a file-by-file, line-by-line analysis of the agent orchestration system in Claude Code CLI. All paths are relative to `restored-src/src/`.

---

## 1. AgentTool.tsx -- The Entry Point

**File:** `tools/AgentTool/AgentTool.tsx` (~1350 lines)

This is the main tool definition file that ties together all agent spawning logic. It uses `buildTool()` to define the `Agent` tool.

### Input Schema (Lines 82-125)

```typescript
const baseInputSchema = lazySchema(() => z.object({
  description: z.string().describe('A short (3-5 word) description of the task'),
  prompt: z.string().describe('The task for the agent to perform'),
  subagent_type: z.string().optional(),
  model: z.enum(['sonnet', 'opus', 'haiku']).optional(),
  run_in_background: z.boolean().optional()
}));
```

The full schema adds multi-agent parameters (`name`, `team_name`, `mode`) and isolation (`worktree`, `remote`, `cwd`). Key design choices:

- **Line 99**: The `isolation` field uses conditional enum -- internal builds support `'worktree' | 'remote'`, external only `'worktree'`.
- **Lines 110-125**: Schema stripping via `.omit()` -- when `FORK_SUBAGENT` is enabled or background tasks disabled, `run_in_background` is removed from the schema entirely so the model never sees it. When `KAIROS` is off, `cwd` is omitted.

### Output Schema (Lines 141-155)

Two discriminated union variants:
- `syncOutputSchema`: `status: 'completed'` with `agentToolResultSchema` fields (agentId, content, totalToolUseCount, totalDurationMs, totalTokens, usage).
- `asyncOutputSchema`: `status: 'async_launched'` with agentId, description, prompt, outputFile, canReadOutputFile.

Additionally, internal-only types `TeammateSpawnedOutput` (lines 161-176) and `RemoteLaunchedOutput` (lines 183-190) exist but are excluded from the exported schema for dead code elimination.

### The `call()` Method (Lines 239-1262)

This is the core execution method. It follows this flow:

**Step 1: Validation and routing (lines 250-280)**
```
- Extract permissionMode from appState
- Resolve rootSetAppState (setAppStateForTasks for in-process teammates)
- Guard: team_name without ENABLE_AGENT_SWARMS throws
- Guard: Teammates cannot spawn other teammates (flat roster)
- Guard: In-process teammates cannot spawn background agents
```

**Step 2: Multi-agent spawn path (lines 284-316)**
When `teamName && name` are both set, the call routes to `spawnTeammate()` from `tools/shared/spawnMultiAgent.ts`. The result is cast through `unknown` to `Output` because `TeammateSpawnedOutput` is intentionally excluded from the exported union.

**Step 3: Agent resolution (lines 318-356)**
```
effectiveType = subagent_type ?? (isForkSubagentEnabled() ? undefined : GENERAL_PURPOSE_AGENT.agentType)
```
- If `effectiveType` is undefined, fork path activates
- Fork recursive guard: checks `querySource === 'agent:builtin:fork'` and `isInForkChild(messages)` (lines 332-334)
- Otherwise, finds the agent in `agentDefinitions.activeAgents`, respecting `filterDeniedAgents` and `allowedAgentTypes`

**Step 4: MCP server validation (lines 369-410)**
If `selectedAgent.requiredMcpServers` is set, waits up to 30s for pending MCP connections (`POLL_INTERVAL_MS = 500`), then verifies tools are available. Early exits if a required server has already failed.

**Step 5: System prompt construction (lines 492-541)**
Two branches:
- **Fork path**: Inherits parent's rendered system prompt via `toolUseContext.renderedSystemPrompt` (byte-identical for cache). Falls back to `buildEffectiveSystemPrompt()` recomputation. Prompt messages built by `buildForkedMessages()`.
- **Normal path**: Calls `selectedAgent.getSystemPrompt({ toolUseContext })` and enhances with environment details. Prompt is a single `createUserMessage({ content: prompt })`.

**Step 6: Async determination (lines 557-567)**
```typescript
const shouldRunAsync = (run_in_background === true
  || selectedAgent.background === true
  || isCoordinator
  || forceAsync          // fork experiment forces all async
  || assistantForceAsync // KAIROS forces all async
  || proactiveModule?.isProactiveActive()
) && !isBackgroundTasksDisabled;
```

**Step 7: Worker tool pool assembly (lines 573-577)**
```typescript
const workerPermissionContext = {
  ...appState.toolPermissionContext,
  mode: selectedAgent.permissionMode ?? 'acceptEdits'
};
const workerTools = assembleToolPool(workerPermissionContext, appState.mcp.tools);
```
Workers always get their tools from `assembleToolPool` with their own permission mode -- the parent's tool restrictions do not leak.

**Step 8: Worktree isolation setup (lines 582-601)**
If `effectiveIsolation === 'worktree'`:
- Creates a slug like `agent-${earlyAgentId.slice(0, 8)}`
- Calls `createAgentWorktree(slug)` returning `{ worktreePath, worktreeBranch, headCommit, gitRoot, hookBased }`
- For fork+worktree: appends a `buildWorktreeNotice()` message telling the child to translate paths

**Step 9a: Async launch path (lines 686-764)**
```typescript
const agentBackgroundTask = registerAsyncAgent({ ... });
void runWithAgentContext(asyncAgentContext, () =>
  wrapWithCwd(() => runAsyncAgentLifecycle({ ... }))
);
return { data: { status: 'async_launched', ... } };
```
- `registerAsyncAgent` creates the task in `AppState.tasks`
- `runAsyncAgentLifecycle` drives the background agent loop
- If `name` is set, registers in `agentNameRegistry` for SendMessage routing
- Returns immediately with `async_launched` status

**Step 9b: Sync execution path (lines 765-1261)**
```typescript
return runWithAgentContext(syncAgentContext, () => wrapWithCwd(async () => {
  // Register as foreground task with auto-background timer
  const registration = registerAgentForeground({ ... });

  // Get async iterator
  const agentIterator = runAgent({ ... })[Symbol.asyncIterator]();

  // Race loop: message vs background signal
  while (true) {
    const raceResult = await Promise.race([nextMessagePromise, backgroundPromise]);
    if (raceResult.type === 'background') { /* transition to async */ }
    // Process messages...
  }

  return { data: { status: 'completed', ...agentResult } };
}));
```

Key sync-specific features:
- **Background hint UI** (line 873): After `PROGRESS_THRESHOLD_MS = 2000ms`, shows `<BackgroundHint />` via `setToolJSX`
- **Foreground-to-background transition** (lines 896-1051): When user backgrounds the agent, the iterator is returned, a new `runAgent()` starts in background mode, and an `async_launched` result is returned immediately
- **Progress forwarding** (lines 1082-1089): Bash/PowerShell progress events from sub-agent are forwarded to parent SDK

### Permission Checking (lines 1281-1297)

```typescript
async checkPermissions(input, context): Promise<PermissionResult> {
  // Only route through auto mode classifier in auto mode
  if ("external" === 'ant' && appState.toolPermissionContext.mode === 'auto') {
    return { behavior: 'passthrough', message: '...' };
  }
  return { behavior: 'allow', updatedInput: input };
}
```

In non-auto modes, agent spawning is always auto-approved (the actual tool executions within the agent are separately permissioned).

---

## 2. runAgent.ts -- The Agent Execution Engine

**File:** `tools/AgentTool/runAgent.ts` (974 lines)

This is the async generator that creates an agent's isolated execution context and drives the `query()` loop.

### MCP Server Initialization (Lines 95-218)

`initializeAgentMcpServers()` handles agent-specific MCP servers:
- **String references** (line 140-143): Looked up via `getMcpConfigByName()`, shared/memoized with parent. NOT cleaned up on agent exit.
- **Inline definitions** (lines 153-169): Agent-specific servers with `scope: 'dynamic'`. These ARE cleaned up when the agent finishes.
- **Plugin-only policy** (lines 117-127): Under `strictPluginOnlyCustomization`, user-agent MCP servers are skipped. Plugin/built-in/policy agents are exempt.

### runAgent() Parameters (Lines 248-329)

Notable parameters:
- `useExactTools` (line 313): When true (fork path), skips `resolveAgentTools()` filtering. Also inherits `thinkingConfig` and `isNonInteractiveSession` from parent for byte-identical API prefixes.
- `onCacheSafeParams` (line 304): Callback for background summarization to fork the agent's conversation.
- `contentReplacementState` (line 308): For resume, reconstructed from sidechain transcript.
- `canShowPermissionPrompts` (line 277): Overrides the default `!isAsync` heuristic. True for in-process teammates that share the terminal.
- `onQueryProgress` (line 328): Fires on every query yield including stream_events. Used for liveness detection during long thinking blocks.

### Context Construction (Lines 330-498)

**User context stripping (lines 390-398)**:
```typescript
const shouldOmitClaudeMd = agentDefinition.omitClaudeMd
  && !override?.userContext
  && getFeatureValue_CACHED_MAY_BE_STALE('tengu_slim_subagent_claudemd', true)
```
Explore and Plan agents have `omitClaudeMd: true`. This saves ~5-15 Gtok/week across 34M+ Explore spawns.

**Git status stripping (lines 404-410)**:
Explore and Plan agents also drop `gitStatus` from system context. Saves ~1-3 Gtok/week.

**Permission mode override (lines 415-498)**:
`agentGetAppState()` wraps the real `getAppState()` to:
1. Override `mode` to agent's `permissionMode` (unless parent is `bypassPermissions`, `acceptEdits`, or `auto`)
2. Set `shouldAvoidPermissionPrompts: true` for async agents that cannot show UI
3. Set `awaitAutomatedChecksBeforeDialog: true` for background agents that CAN show prompts (bubble mode)
4. Scope tool permissions: when `allowedTools` provided, replaces session-level allow rules while preserving `cliArg` rules from SDK's `--allowedTools`
5. Override `effortValue` if agent defines one

### Tool Resolution (Lines 500-502)

```typescript
const resolvedTools = useExactTools
  ? availableTools
  : resolveAgentTools(agentDefinition, availableTools, isAsync).resolvedTools
```

### Agent Options (Lines 667-695)

```typescript
const agentOptions = {
  isNonInteractiveSession: useExactTools
    ? toolUseContext.options.isNonInteractiveSession
    : isAsync ? true : (toolUseContext.options.isNonInteractiveSession ?? false),
  thinkingConfig: useExactTools
    ? toolUseContext.options.thinkingConfig
    : { type: 'disabled' },
  // Fork children inherit querySource for recursive-fork guard
  ...(useExactTools && { querySource }),
};
```
- Regular sub-agents have thinking **disabled** to control output token costs
- Fork children **inherit** thinking config for prompt cache identity

### Subagent Context Creation (Lines 700-714)

```typescript
const agentToolUseContext = createSubagentContext(toolUseContext, {
  options: agentOptions,
  agentId,
  agentType: agentDefinition.agentType,
  messages: initialMessages,
  readFileState: agentReadFileState,
  abortController: agentAbortController,
  getAppState: agentGetAppState,
  shareSetAppState: !isAsync,  // Sync agents share, async agents don't
  shareSetResponseLength: true,
  criticalSystemReminder_EXPERIMENTAL: agentDefinition.criticalSystemReminder_EXPERIMENTAL,
  contentReplacementState,
});
```

### Skill Preloading (Lines 578-646)

Agent frontmatter `skills` array is resolved via `resolveSkillName()` (line 945-973) which tries:
1. Exact match via `hasCommand()`
2. Plugin-prefixed: `"my-skill"` to `"my-plugin:my-skill"`
3. Suffix match: any command ending with `":skillName"`

Skills are loaded concurrently via `Promise.all()` and added as user messages to `initialMessages`.

### Hook Registration (Lines 557-575)

Agent frontmatter hooks are registered via `registerFrontmatterHooks()` with `isAgent=true`, which converts `Stop` hooks to `SubagentStop` (since subagents trigger SubagentStop, not Stop).

### The Query Loop (Lines 747-860)

```typescript
for await (const message of query({ messages, systemPrompt, ... })) {
  // Forward TTFT metrics to parent display
  // Handle max_turns_reached attachment
  // Record sidechain transcript incrementally (O(1) per message)
  // Yield recordable messages (assistant, user, progress, compact_boundary)
}
```

### Cleanup (Lines 816-859, finally block)

Comprehensive cleanup includes:
1. MCP server cleanup (`mcpCleanup()`)
2. Session hooks cleanup (`clearSessionHooks()`)
3. Prompt cache tracking cleanup (`cleanupAgentTracking()`)
4. File state cache release (`.clear()`)
5. Initial messages release (`.length = 0`)
6. Perfetto agent unregister
7. Transcript subdir mapping release
8. Todos entry removal from AppState
9. Background bash task killing (`killShellTasksForAgent()`)
10. Monitor MCP task killing (if `MONITOR_TOOL` feature enabled)

### filterIncompleteToolCalls (Lines 866-904)

Used when forking to prevent API errors from orphaned tool_use blocks:
```typescript
// Build set of tool_use_ids that have corresponding tool_results
// Filter out assistant messages with tool_use blocks missing results
```

---

## 3. agentToolUtils.ts -- Tool Filtering and Lifecycle Utilities

**File:** `tools/AgentTool/agentToolUtils.ts` (687 lines)

### filterToolsForAgent (Lines 70-116)

Three-tier filtering:
1. `ALL_AGENT_DISALLOWED_TOOLS`: Tools never available to any agent
2. `CUSTOM_AGENT_DISALLOWED_TOOLS`: Additional tools blocked for non-built-in agents
3. `ASYNC_AGENT_ALLOWED_TOOLS`: Whitelist for async agents (more restrictive)

Special cases:
- MCP tools (`mcp__*`) always pass (line 83)
- `ExitPlanMode` passes if agent is in `plan` mode (lines 88-93)
- In-process teammates get `AgentTool` (for sync subagents) and `IN_PROCESS_TEAMMATE_ALLOWED_TOOLS` even when async (lines 101-111)

### resolveAgentTools (Lines 122-225)

Handles wildcard and explicit tool lists:
- Wildcard (`['*']` or undefined): Returns all tools after filtering
- Explicit list: Validates each tool spec, extracts `allowedAgentTypes` from `Agent(worker, researcher)` syntax
- `disallowedTools`: Parsed via `permissionRuleValueFromString()` for pattern matching
- `isMainThread` flag: Skips `filterToolsForAgent` entirely (main thread's pool is already correct)

### finalizeAgentTool (Lines 276-357)

Extracts the final result from agent messages:
```typescript
let content = lastAssistantMessage.message.content.filter(_ => _.type === 'text')
if (content.length === 0) {
  // Fall back to most recent assistant message with text content
  for (let i = agentMessages.length - 1; i >= 0; i--) { ... }
}
```
Also emits `tengu_cache_eviction_hint` with the last request ID so the inference service can evict the subagent's cache chain.

### runAsyncAgentLifecycle (Lines 508-686)

Shared between:
- `AgentTool.call()` async-from-start path
- `resumeAgentBackground()`

Flow:
```
1. Create progress tracker
2. For each message from makeStream():
   a. Push to agentMessages
   b. Append to AppState task if retained
   c. Update progress
   d. Emit task_progress SDK event
3. On completion:
   a. stopSummarization()
   b. finalizeAgentTool()
   c. completeAsyncAgent() -- status transition FIRST (unblocks TaskOutput)
   d. classifyHandoffIfNeeded() -- safety check (can hang)
   e. getWorktreeResult() -- worktree cleanup (can hang)
   f. enqueueAgentNotification() -- injects <task-notification> into parent
4. On AbortError:
   a. killAsyncAgent()
   b. extractPartialResult()
   c. enqueueAgentNotification() with status 'killed'
5. On other errors:
   a. failAsyncAgent()
   b. enqueueAgentNotification() with status 'failed'
```

### classifyHandoffIfNeeded (Lines 389-481)

Safety classifier for auto mode:
```typescript
if (toolPermissionContext.mode !== 'auto') return null;
const agentTranscript = buildTranscriptForClassifier(agentMessages, tools);
const classifierResult = await classifyYoloAction(agentMessages, ..., tools, ...);
```
If classifier flags output: returns a `SECURITY WARNING` string prepended to the result.
If classifier unavailable: returns a softer "please verify" warning.

---

## 4. forkSubagent.ts -- Fork Sub-agent System

**File:** `tools/AgentTool/forkSubagent.ts` (211 lines)

### Feature Gate (Lines 32-39)

```typescript
export function isForkSubagentEnabled(): boolean {
  if (feature('FORK_SUBAGENT')) {
    if (isCoordinatorMode()) return false;
    if (getIsNonInteractiveSession()) return false;
    return true;
  }
  return false;
}
```
Mutually exclusive with coordinator mode and non-interactive sessions.

### FORK_AGENT Definition (Lines 60-71)

```typescript
export const FORK_AGENT = {
  agentType: 'fork',
  tools: ['*'],
  maxTurns: 200,
  model: 'inherit',
  permissionMode: 'bubble',
  source: 'built-in',
  getSystemPrompt: () => '',  // Unused -- parent's rendered prompt is passed via override
} satisfies BuiltInAgentDefinition
```

### Recursive Fork Guard (Lines 78-89)

```typescript
export function isInForkChild(messages: MessageType[]): boolean {
  return messages.some(m => {
    if (m.type !== 'user') return false;
    return content.some(block =>
      block.type === 'text' && block.text.includes(`<${FORK_BOILERPLATE_TAG}>`)
    );
  });
}
```
Scans messages for the `<fork-boilerplate>` XML tag to prevent recursive forking.

### buildForkedMessages (Lines 100+)

Creates byte-identical API request prefixes for prompt cache sharing:
1. Clones the parent's full assistant message (all tool_use blocks, thinking, text)
2. Creates placeholder `tool_result` blocks with `FORK_PLACEHOLDER_RESULT = 'Fork started -- processing in background'`
3. Appends a per-child directive with the fork's specific task

---

## 5. forkedAgent.ts -- Subagent Context Isolation

**File:** `utils/forkedAgent.ts` (690 lines)

### CacheSafeParams Type (Lines 57-68)

```typescript
export type CacheSafeParams = {
  systemPrompt: SystemPrompt
  userContext: { [k: string]: string }
  systemContext: { [k: string]: string }
  toolUseContext: ToolUseContext
  forkContextMessages: Message[]
}
```
Carries the parameters that must be identical between fork and parent for prompt cache hits. The API cache key is: system prompt + tools + model + message prefix + thinking config.

### Global CacheSafeParams Slot (Lines 73-81)

```typescript
let lastCacheSafeParams: CacheSafeParams | null = null;
export function saveCacheSafeParams(params: CacheSafeParams | null): void { ... }
export function getLastCacheSafeParams(): CacheSafeParams | null { ... }
```
Written by `handleStopHooks` after each turn so post-turn forks (promptSuggestion, postTurnSummary, `/btw`) can share the main loop's prompt cache without threading params.

### createSubagentContext()

Creates an isolated `ToolUseContext` for the agent:
- **readFileState**: Cloned from parent (independent reads)
- **AbortController**: Child controller (linked or independent based on `isAsync`)
- **setAppState**: Shared for sync agents, no-op for async agents
- **setAppStateForTasks**: Always reaches root store
- **denialTracking**: Fresh `createDenialTrackingState()`
- **contentReplacementState**: Cloned or provided from resume
- **messages**: Independent conversation
- **nestedMemoryTriggers**: Fresh per-agent
- **toolDecisions**: Fresh per-agent

---

## 6. SendMessageTool -- Inter-Agent Communication

**File:** `tools/SendMessageTool/SendMessageTool.ts` (918 lines)

### Input Schema (Lines 67-87)

```typescript
z.object({
  to: z.string(),    // teammate name, "*", "uds:<path>", "bridge:<session-id>"
  summary: z.string().optional(),
  message: z.union([z.string(), StructuredMessage()])
})
```

### Structured Messages (Lines 46-65)

Three protocol message types:
- `shutdown_request`: Request teammate to shut down
- `shutdown_response`: Accept/reject shutdown request (with `request_id`, `approve`, `reason`)
- `plan_approval_response`: Accept/reject plan mode request (with `request_id`, `approve`, `feedback`)

### Message Routing

The `call()` method routes based on the `to` field:

1. **Broadcast** (`to: "*"`): Calls `handleBroadcast()` which sends to all team members except sender
2. **Direct teammate**: Calls `handleMessage()` which writes to file-based mailbox at `~/.claude/teams/{team}/mailboxes/{name}/`
3. **In-process agent**: Checks `agentNameRegistry` for the target, queues messages or auto-resumes stopped agents via `resumeAgentBackground()`
4. **UDS socket** (`to: "uds:<path>"`): Cross-session messaging via Unix Domain Sockets
5. **Remote Control bridge** (`to: "bridge:<session_id>"`): Via `getReplBridgeHandle()`

### In-Process Agent Resume (Referenced from `resumeAgent.ts`)

When `SendMessage` targets a completed/stopped agent:
```typescript
// From SendMessageTool.ts -- routes to:
resumeAgentBackground({ agentId, prompt, toolUseContext, canUseTool })
```
This loads the sidechain transcript from disk, reconstructs replacement state, and re-runs the agent.

---

## 7. TeamCreateTool and TeamDeleteTool

### TeamCreateTool

**File:** `tools/TeamCreateTool/TeamCreateTool.ts` (241 lines)

Creates a multi-agent team:
- Team file: `~/.claude/teams/{name}/config.json`
- Task directory: `~/.claude/tasks/{name}/`
- Lead agent ID: `team-lead@{team-name}` (deterministic format via `formatAgentId()`)
- Registers cleanup handler for session exit

### TeamDeleteTool

**File:** `tools/TeamDeleteTool/TeamDeleteTool.ts` (140 lines)

Disbands a team:
- Refuses if active (non-idle) members remain
- Cleans up: team directory, task directory, worktrees
- Clears `teamContext` and inbox from AppState

---

## 8. Coordinator Mode

**File:** `coordinator/coordinatorMode.ts` (369 lines)

### Mode Detection

```typescript
export function isCoordinatorMode(): boolean {
  return feature('COORDINATOR_MODE') && isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE);
}
```

### Coordinator System Prompt

Defines a workflow with four phases:
1. **Research**: Spawn research workers to understand the codebase
2. **Synthesis**: Combine findings, identify patterns
3. **Implementation**: Spawn implementation workers with precise instructions
4. **Verification**: Verify changes with read-only verification agent

### Coordinator Tool Restrictions

The coordinator uses only:
- `Agent` (to spawn workers)
- `SendMessage` (to communicate)
- `TaskStop` (to terminate agents)
- Subscription tools

Workers get the full async tool set via `assembleToolPool()`.

### Continue vs Spawn Decision Matrix

The coordinator system prompt includes a decision matrix for when to send follow-up messages to existing workers vs spawning new ones, based on task continuity and context overlap.

---

## 9. Swarm Utilities

### constants.ts

**File:** `utils/swarm/constants.ts` (34 lines)

```typescript
export const TEAM_LEAD_NAME = 'team-lead'
export const SWARM_SESSION_NAME = 'claude-swarm'
export function getSwarmSocketName(): string {
  return `claude-swarm-${process.pid}`  // PID isolation for multiple instances
}
```

### teamHelpers.ts

**File:** `utils/swarm/teamHelpers.ts` (684 lines)

**TeamFile structure** tracks members with:
```typescript
type TeamMember = {
  agentId: string
  name: string
  agentType?: string
  model?: string
  color?: string
  cwd: string
  worktreePath?: string
  backendType: BackendType   // 'tmux' | 'iterm2' | 'in-process'
  isActive: boolean
  mode?: PermissionMode
}
```

Key operations:
- `readTeamFile()` / `writeTeamFileAsync()`: File-based CRUD with JSON serialization
- `removeMemberByAgentId()`: Atomic member removal
- `setMemberMode()` / `setMemberActive()`: Status updates
- `cleanupTeamDirectories()`: Removes team dir, task dir, and worktrees
- `destroyWorktree()`: Uses `git worktree remove --force` with `rm -rf` fallback
- `cleanupSessionTeams()`: Handles orphaned teams on session exit

### spawnInProcess.ts

**File:** `utils/swarm/spawnInProcess.ts` (329 lines)

**spawnInProcessTeammate()** (lines 104-216):
1. Generates deterministic `agentId` via `formatAgentId(name, teamName)` (format: `name@team`)
2. Creates independent `AbortController` (not linked to parent)
3. Creates `TeammateContext` for `AsyncLocalStorage` isolation
4. Registers in Perfetto trace for hierarchy visualization
5. Creates `InProcessTeammateTaskState` with:
   - `spinnerVerb` from random `getSpinnerVerbs()` sample
   - `pastTenseVerb` from random `TURN_COMPLETION_VERBS` sample
   - `permissionMode`: `'plan'` if `planModeRequired`, else `'default'`
   - `pendingUserMessages: []` for message queuing
   - `messages: []` for transcript display
6. Registers cleanup handler via `registerCleanup()`
7. Registers task via `registerTask(taskState, setAppState)`

**killInProcessTeammate()** (lines 227-328):
1. Aborts the controller
2. Calls pending idle callbacks (unblocks waiters like `engine.waitForIdle`)
3. Removes from `teamContext.teammates` in AppState
4. Sets status to `'killed'`, clears non-essential state
5. Outside state updater: removes from team file, evicts task output
6. Emits `task_terminated` SDK event
7. Schedules `evictTerminalTask` after `STOPPED_DISPLAY_MS` timeout

### leaderPermissionBridge.ts

**File:** `utils/swarm/leaderPermissionBridge.ts` (55 lines)

Module-level bridge using getter/setter pattern:
```typescript
let registeredSetter: SetToolUseConfirmQueueFn | null = null;
let registeredPermissionContextSetter: SetToolPermissionContextFn | null = null;

export function registerLeaderToolUseConfirmQueue(setter): void { ... }
export function getLeaderToolUseConfirmQueue(): ... | null { ... }
export function unregisterLeaderToolUseConfirmQueue(): void { ... }
// Same pattern for SetToolPermissionContext
```

The REPL registers its queue setter at startup. In-process teammates call the getters to push permission prompts to the leader's terminal UI.

### permissionSync.ts

**File:** `utils/swarm/permissionSync.ts` (929 lines)

Two permission synchronization approaches:

**File-based** (primary for tmux/iTerm2):
- Pending: `~/.claude/teams/{team}/permissions/pending/{requestId}.json`
- Resolved: `~/.claude/teams/{team}/permissions/resolved/{requestId}.json`
- Uses file locking for atomic writes
- Workers write pending requests; leader reads, surfaces to UI, writes resolution
- Workers poll for resolution file

**Mailbox-based** (for in-process teammates):
- `sendPermissionRequestViaMailbox()`: Sends structured permission request as a message
- `sendPermissionResponseViaMailbox()`: Sends resolution back

Also handles sandbox permission requests for network access approval.

### backends/types.ts

**File:** `utils/swarm/backends/types.ts` (312 lines)

```typescript
export type BackendType = 'tmux' | 'iterm2' | 'in-process'

export interface PaneBackend {
  // Low-level pane operations (split, kill, hide, show, resize, etc.)
}

export interface TeammateExecutor {
  spawn(config: TeammateSpawnConfig): Promise<SpawnResult>
  sendMessage(recipientName: string, content: string): Promise<void>
  terminate(name: string): Promise<void>
  kill(name: string): Promise<void>
  isActive(name: string): Promise<boolean>
}
```

---

## 10. Built-in Agent Definitions

### generalPurposeAgent.ts

**File:** `tools/AgentTool/built-in/generalPurposeAgent.ts` (73 lines)

```typescript
export const GENERAL_PURPOSE_AGENT = {
  agentType: 'general-purpose',
  tools: ['*'],           // Full capability
  source: 'built-in',
  // No model override (uses default sub-agent model)
}
```

### exploreAgent.ts

**File:** `tools/AgentTool/built-in/exploreAgent.ts`

```typescript
{
  agentType: 'Explore',
  disallowedTools: ['Agent', 'ExitPlanMode', 'FileEdit', 'FileWrite', 'NotebookEdit'],
  model: 'haiku' (external) / 'inherit' (internal),
  omitClaudeMd: true,   // Saves ~5-15 Gtok/week
  // Read-only -- no write tools
}
```

### planAgent.ts

**File:** `tools/AgentTool/built-in/planAgent.ts`

```typescript
{
  agentType: 'Plan',
  disallowedTools: ['Agent', 'ExitPlanMode', 'FileEdit', 'FileWrite', 'NotebookEdit'],
  model: 'inherit',
  omitClaudeMd: true,
  // Read-only -- returns structured plan
}
```

### verificationAgent.ts

**File:** `tools/AgentTool/built-in/verificationAgent.ts`

```typescript
{
  agentType: 'verification',
  disallowedTools: ['Agent', 'ExitPlanMode', 'FileEdit', 'FileWrite', 'NotebookEdit'],
  model: 'inherit',
  color: 'red',
  background: true,       // Always runs in background
  criticalSystemReminder_EXPERIMENTAL: '...',  // Adversarial verification prompt
  // Read-only for project directory
}
```

### claudeCodeGuideAgent.ts

**File:** `tools/AgentTool/built-in/claudeCodeGuideAgent.ts`

```typescript
{
  agentType: 'claude-code-guide',
  model: 'haiku',
  permissionMode: 'dontAsk',
  tools: ['Glob', 'Grep', 'Read', 'WebFetch', 'WebSearch'],
  // Dynamic system prompt includes user's custom skills, agents, MCP servers, settings
}
```

### statuslineSetup.ts

**File:** `tools/AgentTool/built-in/statuslineSetup.ts`

```typescript
{
  agentType: 'statusline-setup',
  model: 'sonnet',
  tools: ['Read', 'Edit'],
  color: 'orange',
}
```

---

## 11. Agent Memory System

**File:** `tools/AgentTool/agentMemory.ts` (178 lines)

Three memory scopes:

| Scope | Path | Shared Across |
|---|---|---|
| `user` | `~/.claude/agent-memory/{type}/` | All projects for this user |
| `project` | `.claude/agent-memory/{type}/` | All users of this project (checked in) |
| `local` | `.claude/agent-memory-local/{type}/` | Only this user + project (gitignored) |

`loadAgentMemoryPrompt()` builds a memory prompt with scope-specific guidelines and ensures the memory directory exists. The memory file is a Markdown file named after the agent type.

---

## 12. Resume Agent System

**File:** `tools/AgentTool/resumeAgent.ts` (266 lines)

### resumeAgentBackground() (Lines 42-100+)

Resume flow:
1. Load sidechain transcript and agent metadata from disk
2. Filter problematic messages: whitespace-only assistants, orphaned thinking-only, unresolved tool_uses
3. Reconstruct content replacement state for prompt cache stability
4. Validate worktree path still exists (falls back to parent cwd if removed)
5. Resolve agent definition: try metadata's `agentType`, fall back to `GENERAL_PURPOSE_AGENT`
6. For fork resume: reconstruct parent's system prompt via `buildEffectiveSystemPrompt()`
7. Register as async agent, run via `runAsyncAgentLifecycle()`

---

## 13. Agent Loading and Definition Types

**File:** `tools/AgentTool/loadAgentsDir.ts` (756 lines)

### Type Hierarchy

```
AgentDefinition (union)
  ├── BuiltInAgentDefinition
  │     └── getSystemPrompt(params: { toolUseContext })  // function
  ├── CustomAgentDefinition
  │     └── getSystemPrompt()  // no params, body is markdown
  └── PluginAgentDefinition
        └── getSystemPrompt()  // plugin metadata, namespaced
```

### Base Fields (from BaseAgentDefinition)

```typescript
{
  agentType: string
  whenToUse: string
  tools?: string[]
  disallowedTools?: string[]
  skills?: string[]
  mcpServers?: Array<string | Record<string, object>>
  requiredMcpServers?: string[]
  hooks?: { SubagentStart?: ..., SubagentStop?: ..., ... }
  color?: string
  model?: string
  effort?: number
  permissionMode?: PermissionMode
  maxTurns?: number
  memory?: 'user' | 'project' | 'local'
  isolation?: 'worktree' | 'remote'
  background?: boolean
  omitClaudeMd?: boolean
  criticalSystemReminder_EXPERIMENTAL?: string
  source: 'built-in' | 'plugin' | 'user' | 'project' | 'flag' | 'managed' | 'policySettings'
  baseDir: string
  callback?: () => void
}
```

### Loading Priority

```
built-in < plugin < user < project < flag < managed
```
Later sources override earlier ones for the same `agentType`.

### parseAgentFromMarkdown()

Parses `.claude/agents/*.md` files with YAML frontmatter:
```yaml
---
name: my-agent
model: sonnet
tools:
  - Read
  - Grep
  - Agent(worker, researcher)
disallowedTools:
  - FileWrite
permissionMode: plan
maxTurns: 50
background: true
memory: project
isolation: worktree
skills:
  - my-custom-skill
mcpServers:
  - my-existing-server
  - my-inline-server:
      command: node
      args: [server.js]
hooks:
  SubagentStart: echo "started"
---

You are a specialized research agent...
```

---

## 14. Key Architectural Observations

### Prompt Cache Economics

The fork system is explicitly designed around prompt cache economics:
- Fork children use `useExactTools: true` to inherit the parent's exact tool definitions (byte-identical)
- `forkParentSystemPrompt` carries the parent's already-rendered system prompt bytes
- `FORK_PLACEHOLDER_RESULT` is identical across all fork children so the message prefix matches
- Thinking config is inherited (not disabled) for fork children

### Memory Safety Patterns

The codebase is careful about memory leaks in long-running sessions:
- `runAgent.ts` finally block (lines 816-859) releases 10+ resources
- `initialMessages.length = 0` explicitly releases fork context messages
- `readFileState.clear()` releases cloned file state cache
- `clearInvokedSkillsForAgent()` prevents global map growth
- Todo entries are cleaned from `AppState.todos` on agent exit

### Status Transition Ordering

A pattern appears multiple times (tagged `gh-20236`):
```
// Mark task completed FIRST so TaskOutput(block=true) unblocks immediately.
// classifyHandoffIfNeeded and cleanupWorktreeIfNeeded can hang.
completeAsyncAgent(agentResult, rootSetAppState);
// ... then do potentially hanging operations
```

### Permission Model Hierarchy

```
bypassPermissions > acceptEdits > auto > agent's permissionMode > default
```
Agent-level `permissionMode` never overrides `bypassPermissions`, `acceptEdits`, or `auto` from the parent. This is enforced in `agentGetAppState()` (runAgent.ts lines 420-434).

### Analytics Instrumentation

Every agent lifecycle event is instrumented:
- `tengu_agent_tool_selected` -- at spawn time
- `tengu_agent_tool_completed` -- on success
- `tengu_agent_tool_terminated` -- on abort/kill (with reason: `user_cancel_sync`, `user_cancel_background`, `user_kill_async`)
- `tengu_cache_eviction_hint` -- signals cache chain eviction
- `tengu_auto_mode_decision` -- handoff classifier result

### Error Recovery in Sync Path

The sync agent path (AgentTool.tsx lines 1220-1234) attempts graceful degradation:
```typescript
if (syncAgentError) {
  const hasAssistantMessages = agentMessages.some(msg => msg.type === 'assistant');
  if (!hasAssistantMessages) throw syncAgentError;
  // Has some messages -- return partial result
}
```
This allows the parent to see partial progress even when the agent errors mid-execution.

### Foreground-to-Background Transition

The sync path includes a sophisticated transition mechanism (lines 896-1051):
1. Foreground agent runs with `registerAgentForeground()` which includes a `backgroundSignal` promise
2. Each iteration races `nextMessagePromise` against `backgroundSignal`
3. When backgrounded: the iterator is returned (releasing resources), a fresh `runAgent()` starts async, and `async_launched` is returned immediately
4. The backgrounded closure has its own independent `stopBackgroundedSummarization` -- not shared with foreground
