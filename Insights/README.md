# Claude Code CLI Internals -- Cross-Cutting Architecture Summary

This document synthesizes findings from 8 deep-dive investigations into the Claude Code CLI (v2.1.88) source code. Each topic has its own subfolder with `overview.md` (architecture + insights) and `details.md` (code-level references).

## Index

| # | Topic | Folder | Key File Count |
|---|-------|--------|----------------|
| 1 | [CLAUDE.md Context Management](#1-claudemd-context-management) | `CLAUDE-md-context-management/` | ~15 source files |
| 2 | [Sub-Agents & Orchestration](#2-sub-agents--orchestration) | `sub-agents-orchestration/` | ~20 source files |
| 3 | [Tools Handling](#3-tools-handling) | `tools-handling/` | ~40+ tool directories |
| 4 | [Permission System](#4-permission-system) | `permission-system/` | ~25 source files |
| 5 | [Skills & MCP](#5-skills--mcp) | `skills-and-MCP/` | ~30 source files |
| 6 | [Hooks System](#6-hooks-system) | `hooks-system/` | ~30 source files |
| 7 | [Session/Context/Memory](#7-sessioncontextmemory) | `session-persistence-context-memory/` | ~22 source files |
| 8 | [Agent Loop](#8-agent-loop) | `agent-loop/` | ~25 source files |

---

## How Everything Connects

The seven subsystems form a layered architecture where each layer both consumes and produces context for the others:

```
┌─────────────────────────────────────────────────────────────────┐
│                      USER INTERFACE LAYER                       │
│  Terminal (React/Ink) ◄──► Permission Dialogs ◄──► Cost Display │
└───────────────┬─────────────────────┬───────────────────────────┘
                │                     │
┌───────────────▼─────────────────────▼───────────────────────────┐
│                    ORCHESTRATION LAYER                           │
│  Agent Tool ◄──► Team Management ◄──► Coordinator Mode          │
│       │              │                     │                     │
│  SendMessage    Task System           Worker Agents              │
│       │              │                     │                     │
│  Fork/Async     Shared Tasks          Pure Delegation            │
└───────────────┬─────────────────────┬───────────────────────────┘
                │                     │
┌───────────────▼─────────────────────▼───────────────────────────┐
│                     EXECUTION LAYER                              │
│  Tool Registry ◄──► Tool Execution Pipeline ◄──► MCP Bridge     │
│       │                    │                         │           │
│  40+ Built-in         Permission Check          External Servers │
│  Tools                     │                         │           │
│       │              Hooks Pipeline              Tool Discovery  │
│  buildTool()        (Pre/Post/Stop)              (ToolSearch)    │
└───────────────┬─────────────────────┬───────────────────────────┘
                │                     │
┌───────────────▼─────────────────────▼───────────────────────────┐
│                      CONTEXT LAYER                               │
│  CLAUDE.md ◄──► System Prompt Assembly ◄──► Skills/Commands      │
│       │              │                         │                 │
│  Multi-layer      Dynamic Sections          Skill Frontmatter    │
│  Hierarchy        (memory, tools,           (paths, hooks,       │
│  (managed→        context, rules)            allowed-tools)      │
│   user→project→                                                  │
│   local→automem)                                                 │
└───────────────┬─────────────────────┬───────────────────────────┘
                │                     │
┌───────────────▼─────────────────────▼───────────────────────────┐
│                    PERSISTENCE LAYER                             │
│  Session Memory ◄──► Context Compaction ◄──► Memory Extraction   │
│       │                    │                       │             │
│  Running Notes       4-Tier System              memdir/          │
│  (background         (micro→SM→full→            (user, feedback, │
│   extraction)         partial)                   project, ref)   │
│       │                    │                       │             │
│  history.jsonl        Session JSONL            Settings Sync     │
│  (append-only)        (resume support)         (cloud bidir)     │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1. CLAUDE.md Context Management

**What it is:** The primary mechanism for users to inject persistent instructions into every conversation. A layered hierarchy of markdown files merged into the system prompt.

**Key discoveries:**
- **6-layer hierarchy** with positional priority (later = higher attention): Managed → User → Project → Local → AutoMem → TeamMem
- **Two injection paths**: eagerly at session start, lazily when file operations match `paths:` globs in `.claude/rules/*.md`
- **No special sections**: Free-form markdown. Only YAML frontmatter (`paths:`, `description`) and `@include` directives are parsed
- **Directly influences auto-permissions**: Content is wrapped in `<user_claude_md>` tags and fed to the YOLO permission classifier
- **HTML comments stripped**: `<!-- -->` blocks are invisible to the model -- useful for author notes
- **@include**: 5-level depth, circular reference prevention, external includes require user approval
- **Limits**: ~40K chars recommended per file; MEMORY.md capped at 200 lines / 25KB

**Practical guidance for daily use:**
- Put build/test commands and coding standards in project `CLAUDE.md` (checked in)
- Use `.claude/rules/*.md` with `paths:` frontmatter for domain-specific rules that only activate when relevant
- Use `CLAUDE.local.md` for personal preferences not shared with the team
- Use `@include` to reference style guides without duplicating content
- Keep instructions concise -- the model has limited attention within the context window

**Connects to:** System Prompt Assembly, Permission System (auto-classifier), Memory System (MEMORY.md), Hooks (`InstructionsLoaded` event)

---

## 2. Sub-Agents & Orchestration

**What it is:** A multi-layered system supporting simple sub-agents, background workers, team-based swarms, and pure coordinator mode.

**Key discoveries:**
- **5 routing paths** in AgentTool: multi-agent spawn, fork sub-agent, remote isolation, async launch, sync execution
- **8 built-in agent types** with distinct tool sets and models (general-purpose, Explore, Plan, verification, claude-code-guide, statusline-setup, worker, fork)
- **Fork sub-agents** optimize prompt cache economics by keeping system prompt bytes identical between parent and child
- **In-process teammates** use `AsyncLocalStorage` for context isolation within the same Node.js process
- **3 backend modes** for teammates: tmux panes, iTerm2 splits, in-process
- **Permission sync** across swarm: workers write to pending/, leader resolves, workers poll
- **Coordinator Mode** transforms the main agent into a pure orchestrator that never uses tools directly

**When to use what:**
| Scenario | Pattern |
|----------|---------|
| Quick search | Synchronous `Explore` agent |
| Multi-file work | Background `general-purpose` agent |
| Parallel tasks | Multiple background agents in one message |
| Large project | Team with `TeamCreate` + teammates |
| Context preservation | Fork sub-agent (inherits full context) |

**Connects to:** Tools (filtered tool pools per agent type), Permissions (inherited + scoped rules), Memory (agents get isolated context), Hooks (SubagentStart/SubagentStop events)

---

## 3. Tools Handling

**What it is:** The execution backbone -- every model action flows through a unified `Tool` interface with ~40 methods, constructed via `buildTool()` with fail-closed defaults.

**Key discoveries:**
- **Single `Tool` type** with ~40 members covering identity, schema, behavior flags, execution, rendering, and classification
- **`buildTool()` factory** provides safe defaults (not concurrency-safe, not read-only, no classifier input)
- **Registration pipeline**: `getAllBaseTools()` → `getTools()` (filter by mode/rules/enabled) → `assembleToolPool()` (merge MCP, sort, deduplicate)
- **10-phase execution lifecycle** in `checkPermissionsAndCallTool()`: lookup → Zod validation → tool validation → pre-hooks → permissions → backfill → execution → post-hooks → result mapping → error handling
- **Concurrency**: consecutive safe tools batched for parallel execution; `StreamingToolExecutor` handles tools as they stream in
- **MCP tools**: created by spreading base `MCPTool` template, named `mcp__<server>__<tool>`, always deferred unless `alwaysLoad`
- **ToolSearch**: deferred tools load on-demand via keyword scoring over names, hints, and descriptions

**Connects to:** Permissions (per-tool `checkPermissions()`), Hooks (PreToolUse/PostToolUse wrapping), Agents (filtered tool pools), System Prompt (tool `prompt()` methods), UI (dual rendering path: terminal + API)

---

## 4. Permission System

**What it is:** A defense-in-depth system spanning 25+ files / ~8000 lines that gates every tool invocation through multiple independent checks.

**Key discoveries:**
- **6 permission modes**: default, plan, acceptEdits, bypassPermissions, auto (AI classifier), dontAsk
- **3-step pipeline**: (1) Rule-based checks (bypass-immune) → (2) Mode-based resolution → (3) Default behavior, with post-pipeline transforms for dontAsk/auto/headless
- **Bypass-immune safety**: `.git/`, `.claude/`, `.vscode/`, `.idea/`, shell configs always require approval
- **Windows attack detection** (NTFS ADS, 8.3 names, DOS devices, UNC paths) enforced on ALL platforms
- **Auto mode classifier**: AI-based (Haiku) allow/deny that reads CLAUDE.md content, with denial tracking (3 consecutive → fallback to prompting)
- **Enterprise MDM**: `allowManagedPermissionRulesOnly` ignores all local rules; `disableBypassPermissionsMode` prevents bypass
- **Rule format**: `ToolName(content)` with prefix, exact, and wildcard matching; gitignore-style globs for file paths

**Connects to:** Tools (per-tool `checkPermissions()`), Hooks (PreToolUse can return permission decisions; PermissionRequest hook for headless), CLAUDE.md (fed to auto-classifier), Agents (permission inheritance + scoping), Settings (multi-source rule loading)

---

## 5. Skills & MCP

**What it is:** A 3-pillar extensibility system -- Skills (markdown prompts), MCP (external tool servers), and Plugins (bundled packages).

**Key discoveries:**
- **Skills**: Markdown + YAML frontmatter defining prompt templates. 8-level loading hierarchy. Support `$ARGUMENTS`, `${CLAUDE_SKILL_DIR}`, `!`command`` shell execution
- **Two execution modes**: inline (injects into conversation) and fork (runs as sub-agent with own context)
- **Conditional skills**: `paths:` frontmatter activates skills only when matching files are touched
- **MCP servers**: 7 configuration scopes from enterprise to dynamic. Supported transports: stdio, sse, http, ws, sdk
- **MCP tool bridging**: Each tool becomes `mcp__<server>__<tool>` via spread-and-override of base MCPTool template
- **MCP prompts become skills**: `prompts/list` from MCP servers are converted to slash commands
- **Plugins**: Bundle skills + hooks + MCP servers. Marketplace or built-in. Namespaced with `plugin:<name>:<server>`
- **Skill listing budget**: 1% of context window, bundled skills prioritized, progressive truncation

**Connects to:** Tools (SkillTool invokes skills; MCP tools registered alongside built-ins), Permissions (skill-level allow/deny; MCP uses passthrough), Hooks (skill/agent frontmatter can register session hooks), System Prompt (skill listing in system-reminder), Plugins (bundle all three pillars)

---

## 6. Hooks System

**What it is:** User-defined lifecycle actions (shell commands, LLM prompts, HTTP requests, agentic verifiers) that customize, guard, and automate CLI behavior without modifying source.

**Key discoveries:**
- **6 hook types**: command, prompt, agent, http, callback, function
- **27 lifecycle events** across tool, session, model response, subagent, compaction, notification, team/task, MCP, and filesystem categories
- **Exit code protocol**: 0 = success, 2 = blocking error, other = non-blocking error
- **JSON output protocol**: hooks return structured decisions (allow/deny/ask, updated input, stop signals, additional context)
- **Async hook protocol**: output `{"async": true}` to background the hook; `asyncRewake` to re-engage the model
- **Security**: config snapshot frozen at startup (prevents mid-session injection); workspace trust required; SSRF protection for HTTP hooks; CRLF injection prevention
- **Enterprise controls**: `allowManagedHooksOnly`, `disableAllHooks`, `allowedHttpHookUrls`

**Key patterns:**
- PreToolUse hooks can block dangerous commands, modify input, or make permission decisions
- Stop hooks act as verification gates (agent-type hooks spawn multi-turn verifiers)
- UserPromptSubmit hooks can filter input (e.g., PII detection)
- SessionStart hooks can set up environment
- FileChanged hooks can react to file modifications

**Connects to:** Tools (PreToolUse/PostToolUse wrapping every execution), Permissions (hook decisions feed into permission pipeline; but deny/ask rules always override hook "allow"), Skills (frontmatter can register session hooks), Agents (SubagentStart/Stop events; agent frontmatter hooks), Settings (primary persistent storage for hook config)

---

## 7. Session/Context/Memory

**What it is:** A layered persistence system managing volatile (in-session) and durable (cross-session) state, resolving the tension between finite context windows and long sessions.

**Key discoveries:**
- **4-tier compaction**: Microcompact (trim tool results, no API) → Session Memory Compact (reuse pre-extracted notes) → Full Compact (API summarization) → Partial Compact (directional subset)
- **Session Memory**: Background agent extracts running notes after every API response (threshold: 5K tokens + 3 tool calls). 9-section template. Primary input for Tier 2 compaction
- **Memory Directory (memdir)**: File-based cross-session memory with typed taxonomy (user/feedback/project/reference). MEMORY.md entrypoint (200 lines max). Security validation on paths
- **Extract Memories**: Background agent writes durable memories from transcripts to memdir. Restricted tool access (read-only + write only to memory dir)
- **Forked Agent Pattern**: Shared infrastructure (`runForkedAgent`) enables prompt cache sharing across session memory, compaction, and memory extraction
- **Fail-open pattern**: All background services log errors but never block the main conversation
- **History**: Append-only JSONL with file locking. Large pastes stored by hash reference
- **Settings Sync**: OAuth-authenticated cloud bidirectional sync of settings and memory
- **Team Memory Sync**: Server-synced shared memory per-repository with conflict resolution

**Auto-compact decision**: Triggers when tokens approach `effectiveContextWindow - 13K buffer`. Circuit breaker after 3 consecutive failures.

**Connects to:** CLAUDE.md (consumed during context assembly; cache cleared after compaction), Agents (forked agent pattern shared; subagent compacts isolated from main thread), Tools (tool results eligible for microcompact trimming), Hooks (PreCompact/PostCompact events), Skills (skill attachments restored after compaction)

---

## 8. Agent Loop

**What it is:** The core cycle that drives all interaction — a universal `async function* query()` generator implementing a `while(true)` loop (~1730 lines) that powers both the main REPL and every sub-agent.

**Key discoveries:**
- **Three-layer architecture**: Outer REPL (React/Ink event-driven) → Query Loop (`while(true)` async generator) → Tool Execution (streaming or batch orchestration)
- **8-phase iteration**: pre-processing → API call + streaming → post-sampling hooks → abort check → no-tool recovery/stop → tool execution → attachment collection → turn limit + continue
- **9 continue sites**: normal tool follow-up, context collapse recovery, reactive compact, token escalation (8k→64k), output recovery ("resume directly" ×3), stop hook blocking, token budget continuation, model fallback, streaming fallback
- **10 terminal conditions**: completed, max_turns, aborted_streaming, aborted_tools, blocking_limit, prompt_too_long, model_error, image_error, stop_hook_prevented, hook_stopped
- **One `query()` for everything**: Main sessions, sub-agents, coordinators, fork children, and SDK consumers all use the same function — only the context parameters differ
- **Streaming tool execution**: `StreamingToolExecutor` starts tools before the model response completes, with sibling abort (Bash errors cascade to cancel queued tools)
- **State replacement, not mutation**: Each continue site creates a fresh `State` object — makes it easy to audit cross-iteration changes in the 1730-line function
- **Prefetch-during-stream pattern**: Memory, skill discovery, and tool summaries are kicked off in parallel with model streaming to hide latency
- **Recovery escalation chains**: 413 → collapse drain → reactive compact → surface error; max-output-tokens → escalate 8k→64k → inject "resume" ×3 → surface error

**Connects to:** Tools (Phase 6 drives tool execution pipeline), Permissions (checked inside tool execution, not loop itself), Hooks (post-sampling Phase 3, stop hooks Phase 5, PreToolUse/PostToolUse Phase 6), CLAUDE.md (loaded once via `getUserContext()`, lazy rules via Phase 7 attachments), Memory (session extraction via post-sampling, auto-compact at Phase 1 top), Agents (AgentTool in Phase 6 spawns child `query()` loops with isolated context)

---

## The Request Pipeline (End-to-End)

Here's how a single user message flows through all 7 subsystems:

```
User types message
    │
    ├─── [Hooks] UserPromptSubmit hooks fire (can block or inject context)
    │
    ▼
Context Assembly
    ├─── [CLAUDE.md] Load hierarchy (managed→user→project→local→automem)
    ├─── [Skills] List available skills (1% context budget)
    ├─── [Tools] Generate tool prompts and schemas
    ├─── [Memory] Load MEMORY.md + relevant memory files
    └─── [Session] Include compacted history + recent messages
    │
    ▼
API Request → Model Response (tool_use blocks)
    │
    ▼
For each tool call:
    ├─── [Tools] Lookup + Zod validation + tool-specific validation
    ├─── [Hooks] PreToolUse hooks fire (can block/modify/decide permissions)
    ├─── [Permissions] Pipeline: rules → mode → safety → classifier
    │         ├─── [CLAUDE.md] Fed to auto-classifier for context
    │         └─── [Settings] Rules from all sources evaluated
    ├─── [Tools] Execute tool.call()
    │         ├─── [MCP] If MCP tool → bridge to external server
    │         ├─── [Agents] If AgentTool → spawn sub-agent (fork/async/sync)
    │         └─── [Skills] If SkillTool → load and inject/fork skill
    ├─── [Hooks] PostToolUse hooks fire (can modify output)
    └─── [Tools] Map result → API format, handle large results
    │
    ▼
Post-Response Processing
    ├─── [Memory] Session memory extraction (if threshold met)
    ├─── [Memory] Extract durable memories (background)
    ├─── [Memory] Auto-compact check (4-tier system)
    ├─── [Hooks] Stop hooks fire (verification gates)
    └─── [Session] Append to history.jsonl + session.jsonl
```

---

## Practical Insights for Power Users

### 1. Context Window Management
- Your CLAUDE.md content, loaded skills, and tool schemas all compete for the same context window
- Use `.claude/rules/*.md` with `paths:` frontmatter to keep instructions lean -- they only load when relevant
- The 4-tier compaction system is automatic, but keeping tool results small (avoiding massive file reads) helps defer compaction
- Session memory extraction happens in the background -- long sessions benefit from the incremental summary

### 2. Permission Optimization
- `settings.json` rules persist across sessions; use `Bash(git:*)` style prefix rules for commonly-used safe commands
- CLAUDE.md instructions influence auto-mode permission decisions -- clear instructions reduce friction
- For CI/headless: use PermissionRequest hooks to programmatically handle permission dialogs

### 3. Extensibility Strategy
- **Quick customization**: CLAUDE.md + `.claude/rules/` (no code needed)
- **Workflow automation**: Hooks in `settings.json` (shell commands, HTTP webhooks)
- **Domain knowledge**: Skills in `.claude/skills/` (markdown + frontmatter)
- **External tools**: MCP servers in `.mcp.json` (standardized protocol)
- **Full packages**: Plugins (bundle skills + hooks + MCP servers)

### 4. Multi-Agent Best Practices
- Use `Explore` agents for quick read-only research (fast, uses haiku model)
- Use fork sub-agents when you need prompt cache sharing with the parent
- Use teams (`TeamCreate`) for coordinated multi-step projects with shared task lists
- Background agents deliver results via `<task-notification>` -- launch multiple in one message for parallelism
- Worktree isolation (`isolation: "worktree"`) prevents agents from interfering with each other's file changes
