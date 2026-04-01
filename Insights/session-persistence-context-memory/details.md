# Session Persistence, Context Management & Memory Systems -- Detailed Code Analysis

## Table of Contents

1. [Session Memory Service](#1-session-memory-service)
2. [Context Compaction Service](#2-context-compaction-service)
3. [Memory Directory (memdir)](#3-memory-directory-memdir)
4. [Extract Memories Service](#4-extract-memories-service)
5. [History Management](#5-history-management)
6. [Session Storage](#6-session-storage)
7. [Context Assembly](#7-context-assembly)
8. [Settings Sync Service](#8-settings-sync-service)
9. [Team Memory Sync Service](#9-team-memory-sync-service)
10. [Post-Compact Cleanup](#10-post-compact-cleanup)
11. [Token Counting & Context Window](#11-token-counting--context-window)
12. [Cross-File Dependency Map](#12-cross-file-dependency-map)

---

## 1. Session Memory Service

### File: `src/services/SessionMemory/sessionMemory.ts` (496 lines)

**Initialization (`initSessionMemory`, ~line 60-130):**
```
initSessionMemory() → registers post-sampling hook
  ├── Checks: autoCompact enabled, feature gate, GrowthBook flag
  ├── Creates session memory file path
  ├── Registers afterApiResponse hook via addPostSamplingHook()
  └── Hook calls shouldExtractMemory() on each API response
```

- Post-sampling hook registered at line ~95: fires after every API response from the main model.
- Guard: `isAutoCompactEnabled()` must be true (imported from `../compact/autoCompact.js`).
- Feature gate: GrowthBook `tengu_enable_session_memory` checked via `getFeatureValue_CACHED_MAY_BE_STALE`.

**Extraction Decision (`shouldExtractMemory`, ~line 135-175):**
- Dual threshold: both token growth AND tool call count must be met.
- Token threshold: `minimumTokensBetweenUpdate = 5000` -- measured as context window growth since last extraction.
- Tool call threshold: `toolCallsBetweenUpdates = 3` -- counts tool_use blocks in assistant messages since last extraction.
- Default config at ~line 30:
  ```
  DEFAULT_SESSION_MEMORY_CONFIG = {
    minimumMessageTokensToInit: 10000,
    minimumTokensBetweenUpdate: 5000,
    toolCallsBetweenUpdates: 3
  }
  ```

**Extraction Execution (`extractSessionMemory`, ~line 180-300):**
- Wrapped in `sequential()` (from `../../utils/generators.js`) to serialize concurrent calls.
- Calls `runForkedAgent` with:
  - Full conversation transcript as context
  - Session memory template (structured markdown)
  - Tool restriction: only `createMemoryFileCanUseTool` allowing `FileEdit` on the memory file path
- Updates tracking state: `extractionStartedAt`, `tokensAtLastExtraction`, `lastSummarizedMessageId`.

### File: `src/services/SessionMemory/sessionMemoryUtils.ts` (208 lines)

**State Tracking (~line 1-50):**
- Module-level variables: `extractionStartedAt`, `tokensAtLastExtraction`, `lastSummarizedMessageId`, `toolCallsSinceLastExtraction`.
- `hasMetUpdateThreshold()` (~line 55): compares current context window token usage against `tokensAtLastExtraction + threshold`.

**Wait Mechanism (`waitForSessionMemoryExtraction`, ~line 100-140):**
- Polls with `await sleep(500)` in a loop.
- Timeout: 15 seconds max wait.
- Stale threshold: 1 minute -- if extraction started more than 1 minute ago, considers it stale and stops waiting.
- Used by compact system to ensure session memory is fresh before attempting session memory compact.

**Content Access (`getSessionMemoryContent`, ~line 150-175):**
- Reads the session memory markdown file from disk via `readFile`.
- Returns null if file doesn't exist or is empty.
- Used by `trySessionMemoryCompaction()` to get the pre-extracted summary.

### File: `src/services/SessionMemory/prompts.ts` (325 lines)

**Template Definition (`DEFAULT_SESSION_MEMORY_TEMPLATE`, ~line 10-100):**
Nine sections with structured markdown headers:
1. `## Session Title` -- one-line description
2. `## Current State` -- what's happening right now
3. `## Task Specification` -- original user request
4. `## Files and Functions` -- key files being worked on
5. `## Workflow` -- steps taken and planned
6. `## Errors & Corrections` -- issues encountered
7. `## Codebase and System Documentation` -- discovered patterns
8. `## Learnings` -- insights for future reference
9. `## Key Results` -- deliverables and outcomes
10. `## Worklog` -- chronological event log

**Truncation (`truncateSessionMemoryForCompact`, ~line 200-270):**
- `MAX_SECTION_LENGTH = 2000` characters per section.
- `MAX_TOTAL_SESSION_MEMORY_TOKENS = 12000` tokens for compact insertion.
- Parses sections by `##` headers, truncates oversized sections with `[truncated]` marker.
- Token counting via `tokenCountWithEstimation()`.

**Emptiness Check (`isSessionMemoryEmpty`, ~line 280-310):**
- Compares content against template structure.
- If all section bodies are empty (match template placeholders), returns true.
- Used by session memory compact to decide whether to fall back to full compact.

**Custom Templates (~line 115-155):**
- Custom template path: `~/.claude/session-memory/config/template.md`
- Custom prompt path: `~/.claude/session-memory/config/prompt.md`
- Loaded via `readFile` with graceful fallback to defaults.

---

## 2. Context Compaction Service

### File: `src/services/compact/compact.ts` (1706 lines)

**Full Compaction (`compactConversation`, ~line 100-400):**
```
compactConversation(messages, options)
  ├── Pre-compact hooks (snapshot current state)
  ├── stripImagesFromMessages() -- replace images with [image]
  ├── Build compact prompt via getCompactPrompt()
  ├── runForkedAgent() for summarization
  ├── formatCompactSummary() -- strip <analysis>, reformat <summary>
  ├── Post-compact restoration:
  │   ├── File attachments (up to 5 files, 50K token budget)
  │   ├── Plan attachments
  │   ├── Skill attachments (max 5K/skill, 25K total)
  │   ├── Deferred tool deltas
  │   ├── MCP instruction deltas
  │   └── Agent listing deltas
  └── runPostCompactCleanup()
```

- `POST_COMPACT_MAX_FILES_TO_RESTORE = 5` (~line 30): limits file re-attachment after compaction.
- `POST_COMPACT_TOKEN_BUDGET = 50_000` (~line 31): total token budget for restored attachments.
- `stripImagesFromMessages()` (~line 450-500): iterates all content blocks, replaces `image` type with text `[image]` to reduce prompt size before summarization.

**Skill Preservation (`createSkillAttachmentIfNeeded`, ~line 550-650):**
- Scans `invokedSkills` set for skills used during the session.
- Per-skill budget: 5K tokens max.
- Total budget: 25K tokens for all skills combined.
- Creates attachment blocks that are appended to the post-compact message.
- Intentionally survives across multiple compactions (skill names NOT cleared in postCompactCleanup).

**Prompt-Too-Long Recovery (`truncateHeadForPTLRetry`, ~line 700-800):**
- When the compact prompt itself exceeds the context window, drops oldest API-round groups.
- Uses `groupMessagesByApiRound()` to identify boundaries.
- Max 3 retries with progressive head truncation.
- Each retry drops one more group from the start.

**Partial Compaction (`partialCompactConversation`, ~line 900-1100):**
- `direction: 'from'` -- summarizes from a specific message index forward.
- `direction: 'up_to'` -- summarizes prefix up to a specific message index.
- Uses dedicated prompt variants (`PARTIAL`, `PARTIAL_UP_TO`) from `prompt.ts`.

### File: `src/services/compact/autoCompact.ts` (352 lines)

**Threshold Calculation (~line 30-80):**
```typescript
getEffectiveContextWindowSize() = contextWindow - min(maxOutputTokens, COMPACT_MAX_OUTPUT_TOKENS)
getAutoCompactThreshold() = effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS

// where:
COMPACT_MAX_OUTPUT_TOKENS = 20_000  (from utils/context.ts)
AUTOCOMPACT_BUFFER_TOKENS = 13_000
WARNING_THRESHOLD_BUFFER_TOKENS = 20_000
```

**Guard Conditions (`shouldAutoCompact`, ~line 100-160):**
- Rejects `session_memory` source (prevent recursion from session memory extraction).
- Rejects `compact` source (prevent recursion from compaction).
- Checks `isReactiveOnlyMode()` -- some configurations only allow reactive compaction.
- Checks `isContextCollapseMode()` -- context collapse is an alternative to traditional compaction.
- `isAutoCompactEnabled()` (~line 170-220): checks `DISABLE_COMPACT`, `DISABLE_AUTO_COMPACT` env vars, and user config `autoCompact` setting.

**Main Entry (`autoCompactIfNeeded`, ~line 230-330):**
```
autoCompactIfNeeded(messages, tokenUsage)
  ├── Check shouldAutoCompact()
  ├── Check token usage > threshold
  ├── Try session memory compact first:
  │   ├── waitForSessionMemoryExtraction() (up to 15s)
  │   └── trySessionMemoryCompaction()
  ├── If SM compact fails → full compactConversation()
  └── Circuit breaker: MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3
```

### File: `src/services/compact/sessionMemoryCompact.ts` (631 lines)

**Core Logic (`trySessionMemoryCompaction`, ~line 50-200):**
1. Gets session memory content via `getSessionMemoryContent()`.
2. Checks `isSessionMemoryEmpty()` -- returns null to signal fallback if empty.
3. `truncateSessionMemoryForCompact()` to fit within token budget.
4. `calculateMessagesToKeepIndex()` to determine split point.
5. Builds replacement messages: summary + kept recent messages.

**Split Point Calculation (`calculateMessagesToKeepIndex`, ~line 220-350):**
- Starts from `lastSummarizedMessageId` (the message ID when session memory was last extracted).
- Expands backwards to meet minimum thresholds:
  - `minTokens: 10_000` -- keep at least 10K tokens of recent messages.
  - `minTextBlockMessages: 5` -- keep at least 5 messages with text blocks.
  - `maxTokens: 40_000` -- don't keep more than 40K tokens (leave room for compact savings).
- Config at ~line 20: `DEFAULT_SM_COMPACT_CONFIG = { minTokens: 10_000, minTextBlockMessages: 5, maxTokens: 40_000 }`.

**API Invariant Preservation (`adjustIndexToPreserveAPIInvariants`, ~line 380-450):**
- Cannot split between `tool_use` (assistant) and `tool_result` (user) -- must keep paired.
- Cannot split inside a thinking block sequence.
- Adjusts the keep index forward or backward to maintain valid message sequences.

**Resumed Session Handling (~line 460-530):**
- When `lastSummarizedMessageId` is not set (e.g., resumed sessions), falls back to keeping a fixed number of recent messages.
- Uses `DEFAULT_RESUMED_KEEP_COUNT = 10` as the fallback.

### File: `src/services/compact/microCompact.ts` (531 lines)

**Entry Point (`microcompactMessages`, ~line 50-120):**
```
microcompactMessages(messages, options)
  ├── Path selection:
  │   ├── Time-based: clearOldToolResults()
  │   ├── Cached: buildCacheEdits()
  │   └── Legacy: (removed)
  └── Return modified messages
```

**Time-Based Path (~line 130-250):**
- `clearOldToolResults()`: iterates messages in reverse, clearing tool result content blocks for eligible tools when the time gap since the last assistant message exceeds a threshold.
- `COMPACTABLE_TOOLS` list at ~line 25: `['FileRead', 'Bash', 'Grep', 'Glob', 'WebSearch', 'WebFetch', 'FileEdit', 'FileWrite']`.
- Replaces content with `[content cleared by microcompact]` marker.

**Cached Path (~line 260-400):**
- Uses `cache_edits` API blocks -- a model API feature that allows editing cached prompt content without invalidating the cache prefix.
- Builds edit operations that remove tool result blocks from the cached representation.
- More efficient than time-based because it preserves the prompt cache.

**Token Estimation (`estimateMessageTokens`, ~line 420-500):**
- Walks content blocks: text blocks count characters / 4, tool blocks add overhead.
- 4/3 padding factor applied to final estimate.
- Used for threshold decisions without requiring API token counting calls.

### File: `src/services/compact/prompt.ts` (375 lines)

**Full Compact Prompt (`getCompactPrompt`, ~line 20-150):**
Nine required output sections:
1. Primary Request and Intent
2. Key Technical Concepts
3. Files and Code Sections (with paths and line references)
4. Errors and Fixes
5. Problem Solving Approaches
6. All User Messages (verbatim if possible)
7. Pending Tasks
8. Current Work State
9. Optional Next Step

- `NO_TOOLS_PREAMBLE` (~line 160-180): aggressive instruction block that prevents the summarization agent from invoking any tools during compaction.

**Summary Formatting (`formatCompactSummary`, ~line 200-250):**
- Strips `<analysis>` scratchpad blocks (model's internal reasoning).
- Extracts content from `<summary>` tags.
- Reformats for clean insertion into conversation continuation.

**Continuation Message (`getCompactUserSummaryMessage`, ~line 260-330):**
- Wraps the formatted summary with instructions for the model to continue the conversation.
- Includes the transcript file path for reference.
- Adds "Continue the conversation from where it left off" framing.

### File: `src/services/compact/grouping.ts` (63 lines)

**API Round Grouping (`groupMessagesByApiRound`, ~line 5-60):**
- Splits message arrays at boundaries where a new assistant message ID appears.
- Each "group" represents one API request-response cycle (user message + assistant response + tool results).
- Used by `truncateHeadForPTLRetry` to drop complete rounds, and by reactive compact for boundary detection.

---

## 3. Memory Directory (memdir)

### File: `src/memdir/memdir.ts` (508 lines)

**Constants (~line 1-10):**
```
ENTRYPOINT_NAME = 'MEMORY.md'
MAX_ENTRYPOINT_LINES = 200
MAX_ENTRYPOINT_BYTES = 25_000
```

**Memory Prompt Construction (`buildMemoryLines`, ~line 50-150):**
- Constructs behavioral instructions for typed-memory system.
- Instructions vary by memory type (user, feedback, project, reference).
- Includes file naming conventions, content format requirements.
- Output is an array of markdown lines joined into the system prompt.

**Entrypoint Loading (`loadMemoryPrompt`, ~line 160-250):**
- Dispatches based on mode:
  - **Auto mode:** Reads MEMORY.md, builds standard memory prompt.
  - **Team mode:** Delegates to team memory prompt builder.
  - **KAIROS mode:** Uses `buildAssistantDailyLogPrompt()` with date-named files.
- Returns formatted string for system prompt injection.

**KAIROS Mode (`buildAssistantDailyLogPrompt`, ~line 260-350):**
- Uses date-formatted filenames (e.g., `2026-04-01.md`) for daily logs.
- Maintains a rolling set of recent log files.
- Designed for chronological memory organization rather than type-based.

**Search Context (`buildSearchingPastContextSection`, ~line 370-450):**
- Generates instructions teaching the model how to grep the memory directory.
- Includes transcript log paths for searching past session context.
- Enables the model to self-serve historical context retrieval.

**Directory Setup (`ensureMemoryDirExists`, ~line 460-500):**
- Idempotent creation of the memory directory structure.
- Creates subdirectories for each memory type.
- Handles race conditions with `recursive: true` on `mkdir`.

### File: `src/memdir/findRelevantMemories.ts` (142 lines)

**AI-Powered Relevance (`findRelevantMemories`, ~line 10-100):**
1. Scans memory directory for files with frontmatter headers.
2. Builds a manifest of file paths + frontmatter summaries.
3. Sends manifest to Sonnet via `sideQuery` asking: "Which of these memories are relevant to the current conversation?"
4. Model returns up to 5 file paths.
5. Returns absolute paths + mtime for freshness display.

**Exclusions (~line 60-75):**
- Excludes `MEMORY.md` (already in system prompt).
- Excludes paths already surfaced in the current conversation.
- Deduplicates by normalized path.

### File: `src/memdir/memoryTypes.ts` (272 lines)

**Type Definitions (~line 1-100):**
Four memory types with save criteria and examples:

| Type | Save When | Don't Save |
|------|-----------|------------|
| `user` | Personal preferences, named workflows, environment setup | One-time commands, temporary preferences |
| `feedback` | Explicit corrections, "remember this", style preferences | Temporary context, session-specific notes |
| `project` | Architecture decisions, API patterns, file structure | Code snippets, debugging steps |
| `reference` | External API docs, config schemas, rate limits | Frequently-changing data, large datasets |

**Anti-Patterns (`WHAT_NOT_TO_SAVE_SECTION`, ~line 120-170):**
- Code patterns and implementations (they live in code)
- Git history (it's in git)
- Debugging solutions (too specific)
- Temporary state (expires quickly)

**Trust Verification (`TRUSTING_RECALL_SECTION`, ~line 180-220):**
- Memories should be verified against current codebase state before recommending.
- Stale memories can lead to incorrect suggestions.

**Drift Warning (`MEMORY_DRIFT_CAVEAT`, ~line 230-270):**
- Memories can become stale as code evolves.
- Model instructed to cross-reference memory claims with actual file contents.

### File: `src/memdir/paths.ts` (279 lines)

**Path Resolution (`getAutoMemPath`, ~line 20-80):**
Memoized resolution chain:
1. `CLAUDE_CODE_MEMORY_DIR` environment variable override.
2. `memoryDir` in settings.json.
3. Default: `<memoryBase>/projects/<sanitized-git-root>/memory/`
   - `memoryBase` defaults to `~/.claude/memory/`
   - `sanitized-git-root`: git repository root path with path separators replaced.

**Enable Check (`isAutoMemoryEnabled`, ~line 90-150):**
Priority chain:
1. `CLAUDE_CODE_AUTO_MEMORY` env var (explicit override).
2. `--bare` flag (disables if set).
3. CCR check (disabled in headless CCR mode).
4. Settings.json `autoMemory` value.
5. Default: `true`.

**Extract Mode (`isExtractModeActive`, ~line 160-200):**
- Gates the background extraction agent.
- Additional checks beyond `isAutoMemoryEnabled()`: GrowthBook flag, feature flag.
- Returns false if auto-memory is disabled.

**Security Validation (`validateMemoryPath`, ~line 210-260):**
- Rejects paths containing `..` (relative traversal).
- Rejects root paths (`/` or drive root on Windows).
- Rejects UNC paths (`\\server\share`).
- Rejects null bytes in path.
- Normalizes path before validation.

**Write Carve-Out (`isAutoMemPath`, ~line 265-279):**
- Normalized prefix match against the configured memory directory.
- Used by tool permission systems to allow writes within the memory directory.

---

## 4. Extract Memories Service

### File: `src/services/extractMemories/extractMemories.ts` (616 lines)

**Initialization Pattern (`initExtractMemories`, ~line 30-100):**
- Returns a closure capturing extraction state.
- State variables: `lastExtractionTurn`, `pendingExtraction`, `extractionCount`.
- Registered as a stop hook via `handleStopHooks` -- runs once at the end of each query loop iteration.

**Skip Conditions (~line 110-180):**
- `hasMemoryWritesSince(lastTurn)`: checks if the main agent already wrote to memory dir since the last check. If so, skips extraction to avoid duplicating work.
- Turn throttling: GrowthBook `tengu_bramble_lintel` config controls how many turns between extractions (default: 1 = every turn).
- `isExtractModeActive()` must be true.

**Tool Permissions (`createAutoMemCanUseTool`, ~line 200-300):**
```
Tool Permission Matrix:
  Read   → unrestricted
  Grep   → unrestricted
  Glob   → unrestricted
  Bash   → read-only mode (specific allowed commands)
  Write  → only within getAutoMemPath() directory
  Edit   → only within getAutoMemPath() directory
  All others → denied
```
- Path validation via `isAutoMemPath()` for write operations.
- Bash restrictions: only `cat`, `ls`, `head`, `tail`, `wc` and similar read-only commands.

**Execution (`runExtraction`, ~line 320-450):**
- Forks a subagent via `runForkedAgent` with extraction prompt.
- Passes current conversation messages as context.
- Subagent writes to memory directory using allowed tools.
- `pendingExtraction` promise tracked for drain support.

**Lifecycle Management (`drainPendingExtraction`, ~line 500-530):**
- Awaits `pendingExtraction` promise if one exists.
- Called during graceful shutdown to ensure in-flight extractions complete.
- Timeout: waits up to 30 seconds before abandoning.

### File: `src/services/extractMemories/prompts.ts` (155 lines)

**Individual Extraction (`buildExtractAutoOnlyPrompt`, ~line 10-70):**
- Instructs agent to identify durable memories worth persisting.
- Pre-injects existing memory manifest (file listing) to avoid needing `ls` turn.
- Specifies memory type taxonomy for categorization.
- Efficient pattern: read existing → identify new → write only new/updated.

**Combined Extraction (`buildExtractCombinedPrompt`, ~line 80-140):**
- Handles both auto-memory and team-memory in a single extraction pass.
- Additional instructions for team memory format and server sync considerations.
- Pre-injects both auto and team memory manifests.

**Tool Budget (~line 145-155):**
- Instructions limit tool usage to efficient read-then-write pattern.
- Discourages exploration loops; agent should know what to write from conversation context.

---

## 5. History Management

### File: `src/history.ts` (465 lines)

**Storage Location (~line 10-20):**
- Path: `~/.claude/history.jsonl`
- Format: newline-delimited JSON (one entry per line).
- File locking via `lock()` (from `proper-lockfile` or similar) for concurrent access safety.

**Entry Schema (`LogEntry` type, ~line 25-40):**
```typescript
type LogEntry = {
  display: string           // Visible text in history list
  pastedContents?: string   // Inline if <1024 chars, hash ref otherwise
  timestamp: number         // Unix timestamp
  project?: string          // Project path for grouping
  sessionId: string         // Links entries to sessions
}
```

**Large Content Handling (~line 100-140):**
- Paste content exceeding 1024 characters stored via content-addressed hash.
- Hash stored in `pastedContents` field as reference (e.g., `hash:sha256:abc123`).
- Paste store at `~/.claude/paste-store/` with hash-named files.
- Keeps JSONL entries small for fast history scanning.

**History Retrieval (`getHistory`, ~line 150-230):**
- Reads JSONL file, parses each line.
- Yields current session entries first (matching `sessionId`), then others.
- `MAX_HISTORY_ITEMS = 100` -- limits total entries returned.
- Skips malformed entries gracefully.

**Undo Support (`removeLastFromHistory`, ~line 350-400):**
- Removes the last entry matching the current session.
- Used by auto-restore-on-interrupt: when the user interrupts and the system auto-restores, the interrupted entry is removed.
- Requires file lock for safe concurrent modification.

---

## 6. Session Storage

### File: `src/utils/sessionStorage.ts`

**Core Mechanism:**
- JSONL-based transcript storage, one file per session.
- Each message appended as a JSON line as the conversation progresses.
- File path derived from session ID.

**Session Switching (`switchSession`):**
- Loads a previous session's transcript for `--resume` and `--continue` flags.
- Validates session ID format and file existence.

**Metadata Management (`reAppendSessionMetadata`):**
- Session metadata (model, project, timestamp, etc.) maintained in a 16KB tail window.
- After each significant state change, metadata is re-appended to the end of the file.
- The `--resume` display reads only this tail window to show session list without parsing full transcripts.
- 16KB chosen as a balance between metadata richness and file scanning performance.

**Cache (`clearSessionMessagesCache`):**
- Module-level cache of parsed session messages.
- Cleared during post-compact cleanup to prevent stale message references.
- Import path: `../../utils/sessionStorage.js` (used by `postCompactCleanup.ts` at line 8).

**CCR v2 Support:**
- Internal event readers for headless session resume in CCR (Claude Code Remote) environments.
- Enables session continuity across CCR restarts.

---

## 7. Context Assembly

### File: `src/context.ts` (190 lines)

**System Context (`getSystemContext`, ~line 20-80):**
```typescript
// Memoized -- computed once per session
getSystemContext() → {
  gitStatus: {
    branch: string
    mainBranch: string
    user: string
    status: string       // truncated at 2000 chars
    recentCommits: string
  }
}
```
- Uses `execFileSync` or similar to run `git status`, `git branch`, `git log`.
- Status truncated at 2000 characters to prevent context bloat from large working trees.
- Memoized: computed once and cached for session lifetime.
- Cache cleared via `setSystemPromptInjection()` for cache breaking.

**User Context (`getUserContext`, ~line 90-140):**
```typescript
// Memoized -- recomputed after cache clear
getUserContext() → {
  claudeMd: string       // Concatenated CLAUDE.md content
  currentDate: string    // ISO date string
}
```
- CLAUDE.md discovery: walks up directory tree + checks `~/.claude/CLAUDE.md`.
- Respects `--bare` mode: skips CLAUDE.md loading entirely.
- Respects `CLAUDE_CODE_DISABLE_CLAUDE_MDS` env var.
- Cache explicitly cleared in `runPostCompactCleanup()` (line 59 of postCompactCleanup.ts): `getUserContext.cache.clear?.()`.

**Cache Injection (`setSystemPromptInjection`, ~line 150-180):**
- Clears both `getSystemContext` and `getUserContext` caches.
- Called when system prompt content needs to change (e.g., after settings sync download).
- Forces re-computation on next access.

### File: `src/utils/context.ts` (222 lines)

**Context Window Constants:**
```typescript
MODEL_CONTEXT_WINDOW_DEFAULT = 200_000    // line 9
COMPACT_MAX_OUTPUT_TOKENS = 20_000        // line 12
MAX_OUTPUT_TOKENS_DEFAULT = 32_000        // line 15
MAX_OUTPUT_TOKENS_UPPER_LIMIT = 64_000    // line 16
CAPPED_DEFAULT_MAX_TOKENS = 8_000         // line 24
ESCALATED_MAX_TOKENS = 64_000             // line 25
```

**Context Window Resolution (`getContextWindowForModel`, line 51-98):**
Priority chain:
1. `CLAUDE_CODE_MAX_CONTEXT_TOKENS` env var (ant-only, line 62-67).
2. `has1mContext(model)` -- `[1m]` suffix detection (line 70-72).
3. `getModelCapability(model).max_input_tokens` -- registry lookup (line 74-83).
4. Beta header check: `CONTEXT_1M_BETA_HEADER` + `modelSupports1M()` (line 85-87).
5. GrowthBook experiment: `getSonnet1mExpTreatmentEnabled()` (line 88-90).
6. Ant-internal model registry: `resolveAntModel()` (line 91-96).
7. Default: `MODEL_CONTEXT_WINDOW_DEFAULT = 200_000` (line 97).

**1M Context Detection:**
- `has1mContext()` (line 35-40): regex `/\[1m\]/i` on model string.
- `modelSupports1M()` (line 43-49): canonical name includes `claude-sonnet-4` or `opus-4-6`.
- `is1mContextDisabled()` (line 31-33): checks `CLAUDE_CODE_DISABLE_1M_CONTEXT` env var.
- `getSonnet1mExpTreatmentEnabled()` (line 100-112): GrowthBook `coral_reef_sonnet` experiment for sonnet-4-6.

**Output Token Limits (`getModelMaxOutputTokens`, line 149-210):**
Per-model defaults:
| Model | Default | Upper Limit |
|-------|---------|-------------|
| opus-4-6 | 64,000 | 128,000 |
| sonnet-4-6 | 32,000 | 128,000 |
| opus-4-5, sonnet-4, haiku-4 | 32,000 | 64,000 |
| opus-4-1, opus-4 | 32,000 | 32,000 |
| claude-3-opus | 4,096 | 4,096 |
| claude-3-sonnet | 8,192 | 8,192 |
| 3-5-sonnet, 3-5-haiku | 8,192 | 8,192 |
| 3-7-sonnet | 32,000 | 64,000 |
| Default | 32,000 | 64,000 |

Override: `getModelCapability(model).max_tokens` can override upper limit if >= 4096.

**Percentage Calculation (`calculateContextPercentages`, line 118-144):**
- Input: token usage object (input_tokens, cache_creation, cache_read) + context window size.
- Output: `{ used: 0-100, remaining: 0-100 }`.
- Clamped to [0, 100] range.

---

## 8. Settings Sync Service

### File: `src/services/settingsSync/index.ts` (582 lines)

**Constants (~line 51-53):**
```
SETTINGS_SYNC_TIMEOUT_MS = 10000
DEFAULT_MAX_RETRIES = 3
MAX_FILE_SIZE_BYTES = 500 * 1024  // 500 KB
```

**Upload Path (`uploadUserSettingsInBackground`, line 60-111):**
1. Guard: `UPLOAD_USER_SETTINGS` feature + `tengu_enable_settings_sync_push` GrowthBook + interactive mode + OAuth.
2. `fetchUserSettings()` to get current remote state.
3. `buildEntriesFromLocalFiles(projectId)` to read local files.
4. Diff: `pickBy(localEntries, (value, key) => remoteEntries[key] !== value)`.
5. Upload only changed entries via `uploadUserSettings(changedEntries)`.

**Download Path (`downloadUserSettings`, line 129-202):**
- Cached promise pattern: `downloadPromise` prevents duplicate fetches.
- `doDownloadUserSettings()`: fetch → parse → `applyRemoteEntriesToLocal()`.
- `redownloadUserSettings()` (line 152): bypasses cache for mid-session refresh (used by `/reload-plugins`).

**File Building (`buildEntriesFromLocalFiles`, line 418-459):**
Sync keys:
- `SYNC_KEYS.USER_SETTINGS` → `getSettingsFilePathForSource('userSettings')`
- `SYNC_KEYS.USER_MEMORY` → `getMemoryPath('User')`
- `SYNC_KEYS.projectSettings(projectId)` → `getSettingsFilePathForSource('localSettings')`
- `SYNC_KEYS.projectMemory(projectId)` → `getMemoryPath('Local')`

**File Application (`applyRemoteEntriesToLocal`, line 488-581):**
1. For each sync key, check if entry exists in remote data.
2. Validate against `MAX_FILE_SIZE_BYTES` (defense-in-depth).
3. `markInternalWrite(filePath)` before writing to suppress change detection.
4. `writeFileForSync()` creates parent directories and writes content.
5. Post-write cache invalidation:
   - `resetSettingsCache()` if any settings file was written.
   - `clearMemoryFileCaches()` if any memory file was written.

**Authentication (`isUsingOAuth`, line 212-221):**
- Requires `firstParty` API provider + first-party Anthropic base URL.
- Checks for valid access token with `user:inference` scope.
- Note: does NOT require `user:profile` scope -- CCR's fd token only has `user:inference`.

---

## 9. Team Memory Sync Service

### File: `src/services/teamMemorySync/index.ts` (1257 lines)

**Scoping:**
- Per-repository via `getGithubRepo()` slug (e.g., `owner/repo`).
- Team memory directory: `<memoryBase>/team/<repo-slug>/`.

**State Management (`SyncState`):**
```typescript
type SyncState = {
  lastKnownChecksum: string        // ETag for conditional requests
  serverChecksums: Map<string, string>  // Per-entry checksums
  serverMaxEntries: number         // Server-imposed entry limit
}
```

**Pull Protocol (`pullTeamMemory`):**
1. Conditional GET with `If-None-Match: <lastKnownChecksum>`.
2. 304 Not Modified → no changes, skip.
3. 200 OK → parse entries, write to disk.
4. Update `lastKnownChecksum` and `serverChecksums`.

**Push Protocol (`pushTeamMemory`):**
1. Read local team memory files.
2. Compute deltas against `serverChecksums`.
3. `batchDeltaByBytes()` splits uploads under 200KB gateway limit.
4. Upload batches sequentially.
5. On 412 Conflict:
   - Hash probe: check if server already has the content (hash match).
   - If hash matches: update local checksums, skip upload.
   - If hash differs: re-pull, re-compute delta, retry push.

**Security (`scanForSecrets`):**
- Scans content before upload using pattern matching.
- Detects: API keys, tokens, passwords, private keys.
- Rejects upload if secrets found, logs warning.

**Orchestration (`syncTeamMemory`):**
```
syncTeamMemory()
  ├── pullTeamMemory()     // Server wins
  ├── pushTeamMemory()     // Local wins on conflict
  └── Return sync result
```

---

## 10. Post-Compact Cleanup

### File: `src/services/compact/postCompactCleanup.ts` (78 lines)

**Function: `runPostCompactCleanup(querySource?)` (line 31-77)**

Cleanup operations:
1. `resetMicrocompactState()` -- clear microcompact tracking (always).
2. Context collapse reset -- only if `CONTEXT_COLLAPSE` feature flag AND main-thread compact (line 42-49).
3. `getUserContext.cache.clear?.()` -- clear memoized user context (main-thread only, line 59).
4. `resetGetMemoryFilesCache('compact')` -- clear CLAUDE.md file cache (main-thread only, line 60).
5. `clearSystemPromptSections()` -- reset system prompt section tracking (always, line 62).
6. `clearClassifierApprovals()` -- reset classifier approval cache (always, line 63).
7. `clearSpeculativeChecks()` -- reset speculative bash permission checks (always, line 64).
8. `clearBetaTracingState()` -- reset telemetry tracing state (always, line 70).
9. File content cache sweep -- `sweepFileContentCache()` for commit attribution (conditional on feature flag, line 71-74).
10. `clearSessionMessagesCache()` -- clear session message parse cache (always, line 76).

**Main-Thread Detection (line 36-39):**
```typescript
const isMainThreadCompact =
  querySource === undefined ||
  querySource.startsWith('repl_main_thread') ||
  querySource === 'sdk'
```
- Subagents (`agent:*`) share module-level state with main thread.
- Resetting getUserContext or memory file caches from a subagent would corrupt main thread state.
- Only main-thread compacts reset these shared caches.

**Intentional Omissions:**
- `resetSentSkillNames()` NOT called -- skill listing must survive compaction for cache efficiency (noted in comment, line 65-69).
- Invoked skill content NOT cleared -- skills survive multiple compactions for `createSkillAttachmentIfNeeded()`.

---

## 11. Token Counting & Context Window

### Token Counting Strategy

The codebase uses a hybrid approach:
1. **API usage data:** `input_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens` from API responses.
2. **Estimation:** `tokenCountWithEstimation()` for pre-API-call decisions using character-based heuristics.
3. **`estimateMessageTokens()`** in microcompact: walks content blocks with 4/3 padding factor.

### Effective Context Window Calculation

```
Raw context window (e.g., 200K or 1M)
  - max(maxOutputTokens, COMPACT_MAX_OUTPUT_TOKENS=20K)
  = effectiveContextWindow

effectiveContextWindow
  - AUTOCOMPACT_BUFFER_TOKENS (13K)
  = autoCompactThreshold

effectiveContextWindow
  - WARNING_THRESHOLD_BUFFER_TOKENS (20K)
  = warningThreshold (shows warning before auto-compact fires)
```

### Output Token Escalation

The `CAPPED_DEFAULT_MAX_TOKENS = 8_000` optimization reserves less slot capacity for most requests. The BQ p99 output is 4,911 tokens, so the 8K cap covers >99% of requests. Requests that hit the cap get one retry at `ESCALATED_MAX_TOKENS = 64_000`.

---

## 12. Cross-File Dependency Map

### Import Relationships (key paths)

```
compact/autoCompact.ts
  → compact/compact.ts (compactConversation)
  → compact/sessionMemoryCompact.ts (trySessionMemoryCompaction)
  → SessionMemory/sessionMemoryUtils.ts (waitForSessionMemoryExtraction)
  → utils/context.ts (COMPACT_MAX_OUTPUT_TOKENS)

compact/compact.ts
  → compact/prompt.ts (getCompactPrompt, formatCompactSummary)
  → compact/grouping.ts (groupMessagesByApiRound)
  → compact/postCompactCleanup.ts (runPostCompactCleanup)
  → compact/microCompact.ts (microcompactMessages)

compact/sessionMemoryCompact.ts
  → SessionMemory/sessionMemoryUtils.ts (getSessionMemoryContent)
  → SessionMemory/prompts.ts (truncateSessionMemoryForCompact, isSessionMemoryEmpty)

SessionMemory/sessionMemory.ts
  → SessionMemory/sessionMemoryUtils.ts (state tracking)
  → SessionMemory/prompts.ts (template, truncation)
  → compact/autoCompact.ts (isAutoCompactEnabled)

extractMemories/extractMemories.ts
  → memdir/paths.ts (getAutoMemPath, isAutoMemPath, isExtractModeActive)
  → extractMemories/prompts.ts (buildExtractAutoOnlyPrompt, buildExtractCombinedPrompt)

memdir/memdir.ts
  → memdir/paths.ts (getAutoMemPath)
  → memdir/memoryTypes.ts (type definitions)
  → memdir/findRelevantMemories.ts (AI relevance selection)

settingsSync/index.ts
  → utils/config.ts (getMemoryPath)
  → utils/settings/settings.ts (getSettingsFilePathForSource)
  → utils/settings/settingsCache.ts (resetSettingsCache)
  → utils/claudemd.ts (clearMemoryFileCaches)
  → utils/settings/internalWrites.ts (markInternalWrite)

teamMemorySync/index.ts
  → utils/git.ts (getGithubRepo)
  → memdir/paths.ts (team memory paths)

context.ts (root)
  → utils/claudemd.ts (CLAUDE.md loading)
  → utils/git.ts (git status)

utils/context.ts
  → utils/model/model.ts (getCanonicalName)
  → utils/model/modelCapabilities.ts (getModelCapability)
  → utils/config.ts (getGlobalConfig)
  → utils/envUtils.ts (isEnvTruthy)
```

### Shared Patterns

**Forked Agent Pattern:**
Used by: `sessionMemory.ts`, `compact.ts`, `extractMemories.ts`
- `runForkedAgent()` shares prompt cache prefix between main and sub-agents.
- Each caller provides a custom `canUseTool` function to restrict tool access.

**Feature Gating Pattern:**
Used by: All services
- `feature('FLAG_NAME')` for compile-time feature flags (bun:bundle).
- `getFeatureValue_CACHED_MAY_BE_STALE(key, default)` for runtime GrowthBook flags.
- Environment variables for admin overrides.

**Fail-Open Pattern:**
Used by: `settingsSync`, `teamMemorySync`, `extractMemories`
- All background services catch top-level errors and continue.
- Logged for diagnostics but never block the main conversation flow.
- Comment pattern: `// Fail-open: log error but don't block [operation]`.

**Cache Invalidation Chain:**
After any write to settings or memory files:
```
Write file → markInternalWrite() → resetSettingsCache() / clearMemoryFileCaches()
                                  → getUserContext.cache.clear?()
                                  → resetGetMemoryFilesCache()
```
