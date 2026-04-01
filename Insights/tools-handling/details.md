# Claude Code CLI - Tools Handling System: Detailed Code Analysis

This document provides a code-level reference for the tools handling system, organized by subsystem with file paths and line references.

---

## 1. Core Type Definitions

### `Tool.ts` - The Tool Interface

**File**: `restored-src/src/Tool.ts`

#### `Tool<Input, Output, P>` type (line 362-695)

The main interface. Key method signatures:

- **`call(args, context, canUseTool, parentMessage, onProgress?)`** (line 379-385): Executes the tool. Returns `ToolResult<Output>` with optional `newMessages`, `contextModifier`, and `mcpMeta`.

- **`validateInput?(input, context)`** (line 489-493): Pre-execution validation. Returns `ValidationResult = { result: true } | { result: false, message, errorCode }` (line 95-101).

- **`checkPermissions(input, context)`** (line 500-503): Returns `PermissionResult` with behavior `allow | deny | ask | passthrough`.

- **`inputSchema`** (line 394): A Zod schema for input validation. MCP tools can alternatively provide `inputJSONSchema` (line 397).

- **`mapToolResultToToolResultBlockParam(content, toolUseID)`** (line 557-559): Converts typed output to Anthropic API `ToolResultBlockParam`.

- **`backfillObservableInput?(input)`** (line 481): Mutates a clone of input to add legacy/derived fields before hooks see it. The original input is never mutated (preserves prompt cache).

- **`preparePermissionMatcher?(input)`** (line 514-516): Returns a closure `(pattern: string) => boolean` for efficient hook `if` condition matching.

#### `ToolResult<T>` type (line 321-336)

```typescript
type ToolResult<T> = {
  data: T
  newMessages?: (UserMessage | AssistantMessage | AttachmentMessage | SystemMessage)[]
  contextModifier?: (context: ToolUseContext) => ToolUseContext
  mcpMeta?: { _meta?: Record<string, unknown>; structuredContent?: Record<string, unknown> }
}
```

#### `ToolUseContext` type (line 158-300)

The execution context passed to every tool call. Key fields:
- `options.tools` (line 163): Available tools
- `options.mcpClients` (line 167): MCP server connections
- `abortController` (line 179): For cancellation
- `readFileState` (line 180): `FileStateCache` for dedup
- `getAppState()` / `setAppState()` (line 181-183): State access
- `messages` (line 251): Current conversation messages
- `toolDecisions` (line 258-264): Map of tool-use-ID to permission decisions
- `fileReadingLimits` (line 251-254): Optional overrides for Read tool limits
- `contentReplacementState` (line 287-292): Tool result budget tracking

#### `buildTool()` function (line 783-792)

```typescript
export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name,
    ...def,
  } as BuiltTool<D>
}
```

Default values defined at lines 757-769:
- `isEnabled: () => true`
- `isConcurrencySafe: () => false`
- `isReadOnly: () => false`
- `isDestructive: () => false`
- `checkPermissions: () => Promise.resolve({ behavior: 'allow', updatedInput: input })`
- `toAutoClassifierInput: () => ''`

#### `toolMatchesName()` / `findToolByName()` (lines 348-360)

Support for tool aliases: `toolMatchesName` checks primary name and `aliases` array. Used throughout the codebase for tool lookup.

---

## 2. Tool Registration

### `tools.ts` - Central Registry

**File**: `restored-src/src/tools.ts`

#### `getAllBaseTools()` (line 193-251)

Returns the complete list of all built-in tools. This array is the authoritative source of all tools that can exist in the system:

- Lines 195-213: Core tools always included (AgentTool, BashTool, FileReadTool, FileEditTool, FileWriteTool, etc.)
- Lines 199-200: Conditional exclusion: `GlobTool`/`GrepTool` omitted when `hasEmbeddedSearchTools()` (ant builds with bfs/ugrep)
- Lines 214-232: Feature-gated tools using `feature()` and `process.env` checks
- Lines 226-231: Lazy require tools (`getSendMessageTool()`, `getTeamCreateTool()`, `getTeamDeleteTool()`) to break circular dependencies
- Lines 247-250: `ToolSearchTool` included only when `isToolSearchEnabledOptimistic()`

#### `getTools(permissionContext)` (line 271-327)

Filters `getAllBaseTools()` for the current context:
- Lines 273-298: Simple mode (`CLAUDE_CODE_SIMPLE`): returns only Bash/Read/Edit (or REPL)
- Lines 301-306: Excludes special tools (`ListMcpResources`, `ReadMcpResource`, `SyntheticOutput`)
- Line 310: `filterToolsByDenyRules()` removes tools with blanket deny rules
- Lines 314-323: REPL mode: hides `REPL_ONLY_TOOLS` from direct model access
- Lines 325-326: `isEnabled()` final filter

#### `assembleToolPool(permissionContext, mcpTools)` (line 345-367)

Merges built-in and MCP tools:
- Line 349: Gets built-in tools via `getTools()`
- Line 352: Filters MCP tools by deny rules
- Lines 362-366: Sorts each partition by name (built-ins first), deduplicates with `uniqBy('name')` (built-ins win)

#### `filterToolsByDenyRules()` (line 262-269)

Uses `getDenyRuleForTool()` to check if a tool has a blanket deny rule (deny rule with no `ruleContent`). Handles MCP server-prefix rules like `mcp__server` that strip all tools from a server.

---

## 3. Individual Tool Anatomy

### FileReadTool - Canonical Example

**File**: `restored-src/src/tools/FileReadTool/FileReadTool.ts`

#### Schema Definition (lines 227-332)

Input schema using `lazySchema()` wrapper for deferred initialization:
```typescript
const inputSchema = lazySchema(() =>
  z.strictObject({
    file_path: z.string().describe('The absolute path to the file to read'),
    offset: semanticNumber(z.number().int().nonnegative().optional()).describe(...),
    limit: semanticNumber(z.number().int().positive().optional()).describe(...),
    pages: z.string().optional().describe(...),
  }),
)
```

Output schema (lines 248-332): Discriminated union with types `text`, `image`, `notebook`, `pdf`, `parts`, `file_unchanged`.

#### Tool Definition (lines 337-718)

```typescript
export const FileReadTool = buildTool({
  name: FILE_READ_TOOL_NAME,           // 'Read'
  searchHint: 'read files, images, PDFs, notebooks',
  maxResultSizeChars: Infinity,         // Never persist (circular read)
  strict: true,                         // API strict mode
  ...
} satisfies ToolDef<InputSchema, Output>)
```

Key methods implemented:
- `description()` (line 343): Returns `DESCRIPTION` from prompt.ts
- `prompt()` (lines 347-359): Dynamically renders with file size limits
- `isConcurrencySafe()` (line 373): Returns `true` (read-only)
- `isReadOnly()` (line 376): Returns `true`
- `isSearchOrReadCommand()` (line 382): `{ isSearch: false, isRead: true }`
- `getPath()` (line 385): Returns `file_path` for path-based operations
- `backfillObservableInput()` (lines 389-393): Expands `file_path` with `expandPath()`
- `preparePermissionMatcher()` (lines 395-397): Returns `pattern => matchWildcardPattern(pattern, file_path)`
- `checkPermissions()` (lines 398-404): Calls `checkReadPermissionForTool()`
- `validateInput()` (lines 418-494): Checks pages parameter, deny rules, UNC paths, binary extensions, blocked device paths
- `call()` (lines 496-651): Main execution with file dedup, skill discovery, ENOENT handling
- `mapToolResultToToolResultBlockParam()` (lines 652-717): Type-specific serialization (image base64, notebook cells, PDF documents, text with line numbers)

#### Deduplication Logic (lines 536-573)

FileReadTool tracks file reads in `readFileState` (a `FileStateCache`). If the same file/range was read before and the file's `mtime` hasn't changed, returns `file_unchanged` type instead of re-sending content. Controlled by `tengu_read_dedup_killswitch` GrowthBook flag.

#### `prompt.ts` (FileReadTool)

**File**: `restored-src/src/tools/FileReadTool/prompt.ts`

- Line 5: `FILE_READ_TOOL_NAME = 'Read'`
- Line 7-8: `FILE_UNCHANGED_STUB` message for dedup responses
- Lines 27-49: `renderPromptTemplate()` builds the full prompt with conditional PDF support, line format, and offset instructions

#### `UI.tsx` (FileReadTool)

**File**: `restored-src/src/tools/FileReadTool/UI.tsx`

- `renderToolUseMessage()` (line 30): Shows file path with optional line range/page info
- `renderToolUseTag()` (line 66): Shows agent task ID for agent output files
- `renderToolResultMessage()` (line 77): Displays summary ("Read N lines", "Read image (42KB)"), NOT the actual content

### BashTool - Complex Permission Model

**File**: `restored-src/src/tools/BashTool/BashTool.tsx`

The most complex tool, with extensive submodules:

| File | Purpose |
|------|---------|
| `bashPermissions.ts` | Command-level permission checking, wildcard matching, speculative classifier |
| `bashSecurity.ts` | Security analysis of commands |
| `commandSemantics.ts` | Interprets command results (exit code, output patterns) |
| `sedEditParser.ts` | Parses sed commands into simulated file edits |
| `sedValidation.ts` | Validates sed edit safety |
| `shouldUseSandbox.ts` | Determines if sandboxing is needed |
| `modeValidation.ts` | Validates tool use against current mode |
| `pathValidation.ts` | Path-based validation |
| `readOnlyValidation.ts` | Enforces read-only constraints |
| `destructiveCommandWarning.ts` | Warns about destructive commands |

Key characteristics:
- Lines 60-72: Command classification sets (`BASH_SEARCH_COMMANDS`, `BASH_READ_COMMANDS`, `BASH_LIST_COMMANDS`, `BASH_SILENT_COMMANDS`)
- `isSearchOrReadBashCommand()` (line 95): Analyzes pipelines -- ALL parts must be search/read for the whole command to be collapsible
- `isConcurrencySafe()`: Returns true only for pure read/search commands
- Complex `checkPermissions()`: Uses classifier for auto-mode, speculative checks, sandbox decisions

### GlobTool - Minimal Read-only Tool

**File**: `restored-src/src/tools/GlobTool/GlobTool.ts`

Demonstrates the minimal read-only tool pattern:
- Lines 26-36: Strict input schema with `pattern` (required) and `path` (optional)
- Lines 39-53: Output schema with `durationMs`, `numFiles`, `filenames`, `truncated`
- Lines 57-80: `buildTool()` with `isConcurrencySafe: true`, `isReadOnly: true`
- `searchHint: 'find files by name pattern or wildcard'`
- `getActivityDescription()`: "Finding {summary}" for spinner display

---

## 4. MCP Tool Creation

### `services/mcp/client.ts` - MCP Tool Factory

**File**: `restored-src/src/services/mcp/client.ts`

#### Tool Creation (lines 1765-1980)

MCP tools are created by spreading `MCPTool` base and overriding:

```typescript
return toolsToProcess.map((tool): Tool => {
  const fullyQualifiedName = buildMcpToolName(client.name, tool.name)
  return {
    ...MCPTool,                                    // Base template from MCPTool.ts
    name: skipPrefix ? tool.name : fullyQualifiedName,
    mcpInfo: { serverName: client.name, toolName: tool.name },
    isMcp: true,
    searchHint: tool._meta?.['anthropic/searchHint'],    // Lines 1779-1784
    alwaysLoad: tool._meta?.['anthropic/alwaysLoad'],    // Line 1785
    description() { return tool.description ?? '' },      // Line 1786-1788
    prompt() { ... truncate to MAX_MCP_DESCRIPTION_LENGTH }, // Lines 1789-1794
    isConcurrencySafe() { return tool.annotations?.readOnlyHint ?? false },
    isReadOnly() { return tool.annotations?.readOnlyHint ?? false },
    isDestructive() { return tool.annotations?.destructiveHint ?? false },
    isOpenWorld() { return tool.annotations?.openWorldHint ?? false },
    inputJSONSchema: tool.inputSchema,
    checkPermissions() { return { behavior: 'passthrough', ... } },
    call(args, context, ...) {
      // Lines 1833-1970: MCP call with retry, progress, error handling
      // Uses callMCPToolWithUrlElicitationRetry
      // Handles session retries, auth errors, URL elicitation
    },
    userFacingName() { return `${client.name} - ${displayName} (MCP)` },
  }
})
```

### `MCPTool.ts` - Base Template

**File**: `restored-src/src/tools/MCPTool/MCPTool.ts`

The base MCPTool (lines 27-77) is a minimal tool definition where most methods are marked "Overridden in mcpClient.ts". It provides:
- `isMcp: true`
- Passthrough input schema: `z.object({}).passthrough()` (line 15)
- String output schema (line 17-19)
- Default `checkPermissions()` returning `passthrough`
- `renderToolUseMessage`, `renderToolUseProgressMessage`, `renderToolResultMessage` from its `UI.tsx`

---

## 5. Tool Execution Service Layer

### `services/tools/toolExecution.ts`

**File**: `restored-src/src/services/tools/toolExecution.ts`

#### `runToolUse()` (line 337-490)

Entry point generator function. Yields `MessageUpdateLazy` objects:

1. Lines 345-353: Tool lookup with alias fallback
2. Lines 369-410: Unknown tool handling
3. Lines 414-453: Abort check
4. Lines 455-466: Delegates to `streamedCheckPermissionsAndCallTool()`

#### `checkPermissionsAndCallTool()` (line 599-1745)

The monolithic execution function (~1150 lines). Key phases:

**Phase 1: Input Validation** (lines 615-733)
- Line 615: `tool.inputSchema.safeParse(input)` -- Zod validation
- Lines 618-629: Deferred tool schema hint (`buildSchemaNotSentHint`)
- Lines 683-733: `tool.validateInput()` call

**Phase 2: Pre-hooks** (lines 800-891)
- Lines 800-862: `runPreToolUseHooks()` yields messages, permission decisions, input updates, stop signals
- Lines 874-891: Hook timing summary (ant-only, > 500ms threshold)

**Phase 3: Permission Resolution** (lines 918-946)
- Line 921-928: `resolveHookPermissionDecision()` combines hook result with rule-based + interactive checks
- Lines 936-946: Slow permission logging for auto-mode

**Phase 4: Tool Execution** (lines 1176-1222)
- Lines 1189-1205: Input convergence logic (backfill clone vs hook-modified vs original)
- Lines 1207-1222: `tool.call()` invocation with context enrichment

**Phase 5: Result Processing** (lines 1223-1588)
- Lines 1292-1301: `mapToolResultToToolResultBlockParam()` + size measurement
- Lines 1403-1474: `addToolResult()` helper: processes result block, appends accept feedback/content blocks
- Lines 1477-1479: Non-MCP tools: result added immediately
- Lines 1483-1531: PostToolUse hooks (MCP tools can have output modified)
- Lines 1540-1541: MCP tools: result added after hooks
- Lines 1566-1569: New messages from tool result appended

**Phase 6: Error Handling** (lines 1589-1745)
- Lines 1599-1629: MCP auth error -> update client status
- Lines 1691-1692: `formatError()` for user-facing error
- Lines 1697-1712: `runPostToolUseFailureHooks()`

#### `buildSchemaNotSentHint()` (lines 577-597)

When a deferred tool fails Zod validation, checks if its schema was actually sent. If not, appends a hint telling the model to call `ToolSearch` first with `select:<tool_name>`.

### `services/tools/toolOrchestration.ts`

**File**: `restored-src/src/services/tools/toolOrchestration.ts`

#### `runTools()` (lines 19-82)

Main generator that orchestrates multiple tool calls:
- Line 26: `partitionToolCalls()` splits into concurrent/serial batches
- Lines 28-63: Concurrent batch: `runToolsConcurrently()` with queued context modifiers
- Lines 64-81: Serial batch: `runToolsSerially()` with inline context updates

#### `partitionToolCalls()` (lines 91-116)

Reduces tool blocks into `Batch[]`. A batch is `{ isConcurrencySafe: boolean, blocks: ToolUseBlock[] }`. Consecutive concurrency-safe tools merge into one batch; anything else gets its own.

### `services/tools/StreamingToolExecutor.ts`

**File**: `restored-src/src/services/tools/StreamingToolExecutor.ts`

#### `StreamingToolExecutor` class (line 40+)

Manages tool execution during streaming API responses:
- `TrackedTool` type (lines 21-32): Tracks status (`queued | executing | completed | yielded`), results, progress, context modifiers
- `addTool()` (line 76+): Adds tool to queue, determines concurrency safety
- `discard()` (line 69): Abandons all pending tools (streaming fallback)
- Uses sibling abort controller (line 48-49): kills parallel tools on Bash error without aborting the query

### `services/tools/toolHooks.ts`

**File**: `restored-src/src/services/tools/toolHooks.ts`

#### `runPostToolUseHooks()` (lines 39-49)

Generator that yields `PostToolUseHooksResult<Output>`:
- Delegates to `executePostToolHooks()` from `utils/hooks.ts`
- For MCP tools: yields `{ updatedMCPToolOutput }` when hooks modify output
- For non-MCP tools: yields message updates (attachments, progress)

#### `resolveHookPermissionDecision()` (referenced from toolExecution.ts)

Combines hook permission results with rule-based checks:
- If hook made a decision: use it
- If hook provided updated input but no decision: flow through normal permission path
- Falls through to `checkRuleBasedPermissions()` then `canUseTool()`

---

## 6. ToolSearchTool

### `tools/ToolSearchTool/ToolSearchTool.ts`

**File**: `restored-src/src/tools/ToolSearchTool/ToolSearchTool.ts`

#### Input Schema (lines 21-34)
- `query`: String for search or `select:<tool_name>` for direct selection
- `max_results`: Number, default 5

#### Output Schema (lines 37-45)
- `matches`: Array of tool names
- `query`: The search query
- `total_deferred_tools`: Count
- `pending_mcp_servers`: Optional array (servers still connecting)

#### `searchToolsWithKeywords()` (lines 186-302)

Keyword search algorithm:
1. **Exact name match** (lines 199-204): Fast path for bare tool names
2. **MCP prefix match** (lines 208-216): `mcp__server` prefix matching
3. **Term parsing** (lines 218-232): Split query into required (`+prefixed`) and optional terms
4. **Scoring** (lines 259-295): Score per tool based on:
   - Exact part match in name: 10-12 points (MCP gets higher)
   - Partial part match: 5-6 points
   - Full name fallback: 3 points
   - `searchHint` match: 4 points (curated capability phrase)
   - Description word-boundary match: 2 points

#### `mapToolResultToToolResultBlockParam()` (lines 444-470)

Returns `tool_reference` blocks (lines 465-468):
```typescript
content: content.matches.map(name => ({
  type: 'tool_reference' as const,
  tool_name: name,
}))
```

### `tools/ToolSearchTool/prompt.ts`

**File**: `restored-src/src/tools/ToolSearchTool/prompt.ts`

#### `isDeferredTool()` (lines 62-108)

Decision logic for tool deferral:
1. Line 65: `alwaysLoad === true` -> never defer
2. Line 68: `isMcp === true` -> always defer
3. Line 71: ToolSearch itself -> never defer
4. Lines 76-81: `AgentTool` in fork-first mode -> never defer
5. Lines 88-94: `BriefTool` -> never defer (communication channel)
6. Lines 98-105: `SendUserFileTool` in REPL bridge -> never defer
7. Line 107: `shouldDefer === true` -> defer

---

## 7. Shared Infrastructure

### `tools/shared/gitOperationTracking.ts`

**File**: `restored-src/src/tools/shared/gitOperationTracking.ts`

#### `detectGitOperation()` (lines 135-186)

Scans command + output for git operations. Returns structured result:
```typescript
{
  commit?: { sha: string; kind: 'committed' | 'amended' | 'cherry-picked' }
  push?: { branch: string }
  branch?: { ref: string; action: 'merged' | 'rebased' }
  pr?: { number: number; url?: string; action: PrAction }
}
```

Used by `toolUseSummaryGenerator` and the collapsed tool-use summary display.

#### `trackGitOperations()` (lines 189-277)

Fires analytics events and increments OTLP counters for successful git operations. Detects:
- `git commit` (with `--amend` variant)
- `git push`
- `gh pr create/edit/merge/comment/close/ready`
- `glab mr create`
- `curl` to PR REST APIs (Bitbucket, GitHub, GitLab)

Auto-links session to PR on `gh pr create` success (lines 226-246).

### `tools/shared/spawnMultiAgent.ts`

**File**: `restored-src/src/tools/shared/spawnMultiAgent.ts`

Handles teammate spawning with multiple backends:
- In-process (direct function call)
- Tmux pane
- iTerm2 pane
- Configurable model selection with leader-follow or explicit override

### `tools/utils.ts`

**File**: `restored-src/src/tools/utils.ts`

- `tagMessagesWithToolUseID()` (lines 12-25): Tags user messages with `sourceToolUseID` for UI dedup
- `getToolUseIDFromParentMessage()` (lines 30-40): Extracts tool_use block ID from assistant message

---

## 8. Tool Use Summary Generation

### `services/toolUseSummary/toolUseSummaryGenerator.ts`

**File**: `restored-src/src/services/toolUseSummary/toolUseSummaryGenerator.ts`

#### `generateToolUseSummary()` (lines 45-97)

Non-critical SDK feature that calls Haiku to produce one-line summaries:
- System prompt (lines 15-24): Instructs Haiku to write git-commit-style labels (<30 chars)
- Truncates tool input/output to 300 chars each (line 59-60)
- Includes last assistant text for intent context (lines 64-66)
- Catches all errors gracefully (lines 90-96)

---

## 9. Tool Constants and Groupings

### `constants/tools.ts`

**File**: `restored-src/src/constants/tools.ts`

Defines tool access sets:

- **`ALL_AGENT_DISALLOWED_TOOLS`** (lines 36-46): Tools forbidden in all subagents (TaskOutput, ExitPlanMode, EnterPlanMode, AskUserQuestion, TaskStop, Workflow)
- **`CUSTOM_AGENT_DISALLOWED_TOOLS`** (lines 48-50): Same as above (currently identical)
- **`ASYNC_AGENT_ALLOWED_TOOLS`** (lines 55-71): Whitelist for async agents (file I/O, search, web, skills, worktrees)
- **`IN_PROCESS_TEAMMATE_ALLOWED_TOOLS`** (lines 77-88): Additional tools for in-process teammates (Task CRUD, SendMessage, Cron tools)
- **`COORDINATOR_MODE_ALLOWED_TOOLS`** (lines 107-112): Minimal set for coordinators (Agent, TaskStop, SendMessage, SyntheticOutput)

---

## 10. Key Patterns and Conventions

### `lazySchema()` Pattern

All Zod schemas use `lazySchema()` wrapper to defer initialization. This avoids circular dependency issues and reduces startup cost:
```typescript
const inputSchema = lazySchema(() => z.strictObject({ ... }))
type InputSchema = ReturnType<typeof inputSchema>
```

### `satisfies ToolDef<InputSchema, Output>` Pattern

All tool definitions end with `} satisfies ToolDef<InputSchema, Output>)`. The `satisfies` operator provides type checking without widening, ensuring the object literal matches the tool definition type.

### UI Rendering Convention

Each tool's `UI.tsx` exports:
- `renderToolUseMessage(input, options)` -> what shows when the tool starts
- `renderToolResultMessage(output, progressMessages, options)` -> what shows when the tool completes
- `renderToolUseErrorMessage(result, options)` -> what shows on error
- `userFacingName(input)` -> display name (e.g., "Read", "Bash")
- `getToolUseSummary(input)` -> compact summary for collapsed views

### Permission Flow Convention

Most file-based tools follow this pattern:
1. `validateInput()`: Check deny rules, path validation (no I/O)
2. `checkPermissions()`: Call `checkReadPermissionForTool()` or similar
3. `preparePermissionMatcher()`: Return `matchWildcardPattern` closure
4. `backfillObservableInput()`: Expand paths with `expandPath()`

### Analytics Convention

All tool operations emit analytics events via `logEvent()`:
- `tengu_tool_use_success`: Successful execution with timing
- `tengu_tool_use_error`: Error with classification
- `tengu_tool_use_can_use_tool_allowed/rejected`: Permission outcomes
- `tengu_tool_use_cancelled`: Abort handling
- `tengu_tool_use_progress`: Progress updates
- Tool-specific events: `tengu_file_read_dedup`, `tengu_pdf_page_extraction`, `tengu_git_operation`, etc.

### Security Conventions

- `CYBER_RISK_MITIGATION_REMINDER` (FileReadTool line 729-730): Appended to every text file read result, instructing the model not to improve malware
- `shouldIncludeFileReadMitigation()` (lines 735-738): Skips for certain models
- `toAutoClassifierInput()`: Returns compact representation for auto-mode security classifier
- Bash tool has the most elaborate security: AST parsing, sed validation, sandbox decisions, speculative classifier checks
