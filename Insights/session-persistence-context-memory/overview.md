# Session Persistence, Context Management & Memory Systems

## Architecture Overview

Claude Code implements a layered memory and context management architecture that spans volatile (in-session) and durable (cross-session) persistence. The system is designed around a central tension: LLM context windows are finite, but user sessions can run for hours with thousands of tool invocations. The architecture resolves this through multi-tier compaction, background memory extraction, and file-based persistence.

```
+------------------------------------------------------------------+
|                     USER SESSION (volatile)                       |
|                                                                   |
|  Messages[] ──> Token Counting ──> Auto-Compact Decision          |
|       │                                    │                      |
|       │              ┌─────────────────────┼──────────┐           |
|       │              │                     │          │           |
|       ▼              ▼                     ▼          ▼           |
|  Microcompact   Session Memory     Full Compact   Partial        |
|  (tool result    Compact (use      (API call to    Compact       |
|   trimming)      pre-extracted     summarize all   (directional  |
|                  notes)            messages)        subset)       |
+------------------------------------------------------------------+
        │                │                  │
        ▼                ▼                  ▼
+------------------------------------------------------------------+
|                   PERSISTENT STORAGE (durable)                    |
|                                                                   |
|  history.jsonl     session-memory/    memdir/     teamMemory/     |
|  (JSONL transcript  (markdown notes   (typed      (server-synced  |
|   with metadata)    per session)      memories)   per-repo)       |
|                                                                   |
|  settingsSync/     extractMemories/                               |
|  (OAuth cloud      (background agent                              |
|   bidirectional)    writes to memdir)                             |
+------------------------------------------------------------------+
```

---

## 1. Session Memory

**Purpose:** Maintain a running summary of the current session as structured markdown notes, updated incrementally by a background subagent.

**Lifecycle:**
1. `initSessionMemory()` registers a post-sampling hook that fires after every API response.
2. The hook checks `shouldExtractMemory()` -- both token growth (5K+ tokens since last extraction) AND tool call count (3+ calls) must be met.
3. When triggered, `extractSessionMemory` (wrapped in `sequential()` to prevent concurrency) forks a subagent via `runForkedAgent`.
4. The forked agent receives the full conversation transcript and a structured template, and writes/updates a session memory markdown file.
5. The forked agent is restricted to a single tool: `FileEdit` on the designated memory file.

**Template Structure (9 sections):**
- Session Title, Current State, Task Specification
- Files and Functions, Workflow, Errors & Corrections
- Codebase and System Documentation, Learnings, Key Results, Worklog

**Key Constraints:**
- `MAX_SECTION_LENGTH = 2000` characters per section
- `MAX_TOTAL_SESSION_MEMORY_TOKENS = 12000` for compact insertion
- Custom templates loadable from `~/.claude/session-memory/config/template.md`
- Feature-gated via GrowthBook (`tengu_enable_session_memory`)

**Relationship to Compaction:** Session memory is the primary input for "session memory compact" -- a cheaper alternative to full API-based compaction that reuses pre-extracted notes instead of making an additional summarization API call.

---

## 2. Context Compaction (Multi-Tier)

The compaction system is Claude Code's primary mechanism for managing context window pressure. It operates in four tiers, each with increasing cost and scope.

### Tier 1: Microcompact (In-Place Tool Result Trimming)

**Trigger:** Runs inline during message processing, no API call needed.

**Mechanism:** Removes tool result content from older messages that are unlikely to be referenced again. Three implementation paths:
- **Time-based:** Clears tool results when the gap since the last assistant message exceeds a time threshold.
- **Cached (cache_edits API):** Uses cache editing blocks to remove tool results without invalidating the cached prompt prefix.
- **Legacy:** Removed in current version.

**Eligible Tools:** FileRead, Bash, Grep, Glob, WebSearch, WebFetch, FileEdit, FileWrite.

**Token Estimation:** `estimateMessageTokens()` walks content blocks with a 4/3 padding factor to approximate token count without API calls.

### Tier 2: Session Memory Compact (Pre-Extracted Summary)

**Trigger:** `autoCompactIfNeeded()` tries this tier first before falling back to full compact.

**Mechanism:** Uses the already-extracted session memory notes (from the background session memory agent) as the summary, avoiding an additional API summarization call.

**Key Logic:**
- `calculateMessagesToKeepIndex()` expands backwards from `lastSummarizedMessageId` to determine how many recent messages to preserve.
- `adjustIndexToPreserveAPIInvariants()` prevents splitting tool_use/tool_result pairs or thinking blocks.
- Falls back to full compact if session memory is empty (matches template with no actual content).

**Configuration:** `DEFAULT_SM_COMPACT_CONFIG = { minTokens: 10_000, minTextBlockMessages: 5, maxTokens: 40_000 }`

### Tier 3: Full Compact (API Summarization)

**Trigger:** When session memory compact is unavailable or insufficient.

**Mechanism:** Sends the entire conversation to a forked agent with a detailed summarization prompt. The agent produces a structured 9-section summary (Primary Request, Key Technical Concepts, Files and Code, Errors, Problem Solving, All User Messages, Pending Tasks, Current Work, Optional Next Step).

**Post-Compact Restoration:**
- File attachments (up to 5 files, 50K token budget)
- Plan attachments, skill attachments (max 5K tokens/skill, 25K total)
- Deferred tool deltas, MCP instruction deltas, agent listing deltas
- Images stripped and replaced with `[image]` markers before summarization

**Prompt-Too-Long Recovery:** `truncateHeadForPTLRetry()` drops oldest API-round groups when the compact prompt itself exceeds the context window (max 3 retries).

**Forked Agent Pattern:** Uses `runForkedAgent` for prompt cache sharing between the main conversation and the summarization agent (enabled by default for first-party API).

### Tier 4: Partial Compact (Directional Subset)

**Trigger:** Explicit request or specific conditions requiring targeted summarization.

**Mechanism:** Two modes:
- `'from'` direction: Summarizes messages from a specific point forward.
- `'up_to'` direction: Summarizes a prefix of the conversation up to a point.

### Auto-Compact Decision Logic

```
shouldAutoCompact()
  ├── Check: not in recursion (source != session_memory, compact)
  ├── Check: not in reactive-only mode
  ├── Check: not in context-collapse mode
  └── Check: isAutoCompactEnabled()
        ├── DISABLE_COMPACT env var
        ├── DISABLE_AUTO_COMPACT env var
        └── User config autoCompact setting

Threshold: effectiveContextWindow - 13K buffer
  where effectiveContextWindow = contextWindow - min(maxOutputTokens, 20K)

Circuit breaker: MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3
```

### Post-Compact Cleanup

`runPostCompactCleanup()` resets caches and state after any compaction:
- Microcompact state, context collapse, getUserContext cache
- Memory file cache, system prompt sections, classifier approvals
- Speculative checks, beta tracing state, session messages cache
- Distinguishes main-thread vs subagent compacts to avoid corrupting shared module-level state

---

## 3. Memory Directory (memdir)

**Purpose:** File-based persistent memory across sessions, organized by type with an entrypoint index file.

**Structure:**
```
<memory-base>/projects/<sanitized-git-root>/memory/
  MEMORY.md              (entrypoint, max 200 lines / 25KB)
  user/                  (user preferences, workflows)
  feedback/              (corrections, explicit preferences)
  project/               (architecture, patterns, conventions)
  reference/             (API docs, configs, environment)
```

**Memory Type Taxonomy:**
| Type | Purpose | Example |
|------|---------|---------|
| `user` | Personal preferences, workflows | "User prefers TDD approach" |
| `feedback` | Corrections from user | "Don't use `any` type in this project" |
| `project` | Architecture, patterns | "Auth uses JWT with refresh tokens" |
| `reference` | External docs, configs | "API rate limit is 100 req/min" |

**Path Resolution:** `getAutoMemPath()` resolves via: env override > settings.json > `<memoryBase>/projects/<sanitized-git-root>/memory/`

**Security:** `validateMemoryPath()` rejects relative paths, root paths, UNC paths, and null-byte paths.

**Relevance Surfacing:** `findRelevantMemories()` uses a Sonnet sideQuery to scan memory file frontmatter headers and select up to 5 relevant files for the current context.

**Modes:**
- **Auto mode:** Standard file-per-memory with MEMORY.md entrypoint
- **Team mode:** Server-synced shared memories (see Team Memory Sync)
- **KAIROS mode:** Date-named daily log files for chronological organization

---

## 4. Extract Memories (Background Agent)

**Purpose:** Automatically extract durable memories from conversation transcripts and write them to the memory directory.

**Lifecycle:**
1. `initExtractMemories()` creates closure-scoped state.
2. Runs once per query loop end via `handleStopHooks`.
3. Checks `hasMemoryWritesSince()` -- skips if main agent already wrote memories.
4. Turn throttling via GrowthBook config (default: every 1 turn).
5. Forks a subagent with restricted tool access.

**Tool Permissions:**
- **Unrestricted:** Read, Grep, Glob (read-only exploration)
- **Read-only Bash:** Allowed but restricted
- **Write/Edit:** Only within the memory directory path
- `createAutoMemCanUseTool()` enforces these boundaries

**Prompt Variants:**
- `buildExtractAutoOnlyPrompt()`: Individual memory extraction
- `buildExtractCombinedPrompt()`: Auto + team memory extraction in single pass
- Pre-injects existing memory manifest to avoid needing `ls` turns

**Lifecycle Management:** `drainPendingExtraction()` awaits in-flight extractions before shutdown.

---

## 5. History Management

**Purpose:** Append-only log of user messages and conversation events, used for session display and resume.

**Storage:** `~/.claude/history.jsonl` with proper file locking via `lock()`.

**Entry Schema:**
```typescript
type LogEntry = {
  display: string          // Display text
  pastedContents?: string  // Inline paste or hash reference
  timestamp: number
  project?: string
  sessionId: string
}
```

**Large Content Handling:** Pasted content exceeding 1024 characters is stored via hash reference in a separate paste store, keeping the JSONL entries small.

**Session Ordering:** `getHistory()` yields current session entries first, then others. `MAX_HISTORY_ITEMS = 100`.

**Undo Support:** `removeLastFromHistory()` supports undo for auto-restore-on-interrupt scenarios.

---

## 6. Session Storage & Resumption

**Purpose:** JSONL transcript persistence enabling `--resume` and `--continue` session recovery.

**Mechanism:**
- Messages appended to per-session JSONL files
- Session metadata stored in a 16KB tail window via `reAppendSessionMetadata()`
- `switchSession()` enables resume support by loading a previous session's transcript
- CCR v2 internal event readers for headless session resume

**Metadata Tail Window:** The last 16KB of each session file is reserved for metadata that `--resume` display reads without parsing the entire transcript. `reAppendSessionMetadata()` keeps this window current.

**Cache Management:** `clearSessionMessagesCache()` is called during post-compact cleanup to ensure stale cached messages are evicted.

---

## 7. Context Assembly

**Purpose:** Build the system prompt and user context that frames every API request.

### System Context (`getSystemContext()`)
- Memoized function returning git status, branch info, user info
- Status output truncated at 2000 characters
- Includes recent commits for repository awareness

### User Context (`getUserContext()`)
- Memoized function returning CLAUDE.md content + current date
- CLAUDE.md discovery respects `--bare` mode and `CLAUDE_CODE_DISABLE_CLAUDE_MDS`
- Cache cleared during compaction via `getUserContext.cache.clear?.()`

### Context Window Resolution (`getContextWindowForModel()`)
Resolution priority:
1. `CLAUDE_CODE_MAX_CONTEXT_TOKENS` env override (ant-only)
2. `[1m]` suffix in model name (explicit 1M opt-in)
3. Model capability registry (`max_input_tokens`)
4. Beta header (`CONTEXT_1M_BETA_HEADER`)
5. GrowthBook experiment (`coral_reef_sonnet`)
6. Ant-internal model registry
7. `MODEL_CONTEXT_WINDOW_DEFAULT = 200_000`

### Output Token Management
- Per-model defaults and upper limits in `getModelMaxOutputTokens()`
- Capped default optimization: `CAPPED_DEFAULT_MAX_TOKENS = 8_000` with escalation to 64K on retry
- Thinking budget: `getMaxThinkingTokensForModel()` = upperLimit - 1

---

## 8. Settings Sync (Cloud Bidirectional)

**Purpose:** Sync user settings and memory files across Claude Code environments via OAuth-authenticated API.

**Two Modes:**
- **Upload (Interactive CLI):** Detects changed local entries, uploads delta to server
- **Download (CCR):** Fetches remote settings before plugin installation

**Synced Content:**
- `USER_SETTINGS`: Global user settings file
- `USER_MEMORY`: Global user memory (CLAUDE.md)
- `projectSettings(<projectId>)`: Per-project local settings
- `projectMemory(<projectId>)`: Per-project local memory

**Guards:**
- OAuth with `user:inference` scope required
- 500KB file size limit per file
- Feature-gated via GrowthBook (`tengu_enable_settings_sync_push`, `tengu_strap_foyer`)
- Retry with exponential backoff (default 3 retries)

**Cache Invalidation:** After download, `resetSettingsCache()` and `clearMemoryFileCaches()` ensure subsequent reads pick up synced content.

---

## 9. Team Memory Sync

**Purpose:** Server-synced shared memory for team collaboration on a per-repository basis.

**Scoping:** Per-repo via `getGithubRepo()` slug derived from git remote URL.

**Sync Protocol:**
1. **Pull (server wins):** Fetch with ETag conditional GET, write entries to disk
2. **Push (local wins on conflict):** Delta upload with 412 conflict resolution

**Conflict Resolution:**
- Pull: Server content overwrites local
- Push: On 412 conflict, performs hash probe to check if server already has the content, then retries
- `batchDeltaByBytes()` splits uploads under 200KB API gateway limit

**Security:** `scanForSecrets()` scans content before upload to prevent accidental secret leakage.

**State Tracking:**
```typescript
type SyncState = {
  lastKnownChecksum: string
  serverChecksums: Map<string, string>
  serverMaxEntries: number
}
```

---

## 10. Cross-System Relationships

### Compaction triggers Memory Extraction
When auto-compact fires, it first attempts session memory compact (Tier 2). This creates a dependency on the session memory agent having extracted notes. If session memory is empty, it falls back to full compact (Tier 3).

### Extract Memories writes to Memdir
The background extraction agent reads conversation transcripts and writes durable memories to the memdir structure. It respects the memory type taxonomy and path security constraints.

### Settings Sync propagates Memory and Settings
Cloud sync ensures that user memory (which includes CLAUDE.md content) and settings are available in CCR environments. The download path invalidates both settings and memory caches.

### Team Memory Sync complements Extract Memories
Both systems write to persistent storage, but team memory is server-authoritative and shared across team members, while extract memories is local and per-user.

### Post-Compact Cleanup resets all caches
After any compaction tier completes, `runPostCompactCleanup()` invalidates caches across the system: context assembly caches, memory file caches, session message caches, and various tracking state.

### Context Assembly consumes Memory outputs
`getUserContext()` reads CLAUDE.md content which may have been updated by settings sync or memory extraction. The memoization cache must be cleared after any write to these files.

### Forked Agent Pattern (shared infrastructure)
Session memory extraction, full compaction, and extract memories all use `runForkedAgent` for prompt cache sharing. This means the main conversation's cached prompt prefix is reusable by these background operations.

---

## 11. Feature Gating Summary

| Feature | Gate | Default |
|---------|------|---------|
| Session Memory | `tengu_enable_session_memory` | Off |
| Settings Sync Upload | `UPLOAD_USER_SETTINGS` + `tengu_enable_settings_sync_push` | Off |
| Settings Sync Download | `DOWNLOAD_USER_SETTINGS` + `tengu_strap_foyer` | Off |
| Auto Memory (memdir) | `isAutoMemoryEnabled()` chain | On |
| Extract Memories | `isExtractModeActive()` | Gated |
| Context Collapse | `CONTEXT_COLLAPSE` feature flag | Gated |
| Commit Attribution | `COMMIT_ATTRIBUTION` feature flag | Gated |
| Microcompact (cached) | `cache_edits` API support | Conditional |

---

## 12. Data Flow Summary

```
User Message
    │
    ▼
API Request ──────────────────────────────────┐
    │                                          │
    ▼                                          ▼
Post-Sampling Hook                      Token Counting
    │                                          │
    ├── Session Memory extraction              │
    │   (if threshold met)                     │
    │                                          ▼
    ▼                                   shouldAutoCompact?
Extract Memories                               │
    (at query loop end)               ┌────────┼────────┐
    │                                 │        │        │
    ▼                                 ▼        ▼        ▼
memdir/                          Micro    SM Compact  Full
  MEMORY.md                      compact     │       Compact
  user/                             │        │        │
  feedback/                         └────────┴────────┘
  project/                                   │
  reference/                                 ▼
                                    Post-Compact Cleanup
                                             │
                                    ┌────────┼────────┐
                                    ▼        ▼        ▼
                               Clear     Clear    Clear
                               caches    state    session
                                                  messages

Persistent Storage:
  history.jsonl ◄── append per message
  session.jsonl ◄── append per message (resume support)
  settingsSync  ◄── upload/download via OAuth API
  teamMemory    ◄── pull/push via server API
```
