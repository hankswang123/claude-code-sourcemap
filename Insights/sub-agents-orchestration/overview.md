# Sub-Agents and Orchestration Architecture

## 1. Overview

Claude Code CLI implements a multi-layered agent orchestration system that supports everything from simple synchronous sub-agents to fully parallel multi-agent swarms with team coordination. The architecture centers around three execution modes:

1. **Synchronous sub-agents** -- spawned via the `Agent` tool, run in the foreground, block the parent until completion, and return a result directly.
2. **Asynchronous (background) sub-agents** -- spawned via the `Agent` tool with `run_in_background: true`, run independently with their own AbortController, and deliver results as `<task-notification>` messages.
3. **Swarm teammates** -- long-lived agents spawned into a team, coordinated via mailbox-based messaging (`SendMessage`), task lists, and permission synchronization. They can run in separate terminal panes (tmux/iTerm2) or in-process with AsyncLocalStorage isolation.

There is also a **Coordinator Mode** that transforms the main agent into a pure orchestrator that never uses tools directly, delegating all work to "worker" sub-agents.

---

## 2. Agent Types and Capabilities

### Built-in Agents

| Agent Type | Model | Purpose | Tools | Read-Only | Key Trait |
|---|---|---|---|---|---|
| `general-purpose` | Default sub-agent model | Research, code search, multi-step tasks | All tools (`*`) | No | Full capability workhorse |
| `Explore` | `haiku` (external) / `inherit` (internal) | Fast codebase search and analysis | All except Agent, ExitPlanMode, FileEdit, FileWrite, NotebookEdit | Yes | Omits CLAUDE.md, omits gitStatus |
| `Plan` | `inherit` | Architecture planning, implementation design | Same as Explore | Yes | Omits CLAUDE.md, omits gitStatus |
| `verification` | `inherit` | Post-implementation verification | All except Agent, ExitPlanMode, FileEdit, FileWrite, NotebookEdit | Yes (project dir) | Critical system reminder, red color, always background |
| `claude-code-guide` | `haiku` | Help with Claude Code features and docs | Glob, Grep, Read, WebFetch, WebSearch | Yes | `dontAsk` permission mode |
| `statusline-setup` | `sonnet` | Configure Claude Code status line | Read, Edit | No | Orange color, minimal tool set |
| `worker` (Coordinator) | Default | Coordinator-mode workers | Configurable async-allowed set | No | Only exists in coordinator mode |
| `fork` (Implicit) | `inherit` | Forked copy of parent context | Parent's exact tool pool | No | `bubble` permission mode, shares prompt cache |

### Custom Agents

Custom agents are defined in `.claude/agents/` as Markdown files with YAML frontmatter, or via JSON in settings. They support:
- Custom system prompts (the markdown body)
- Tool allowlists and denylists
- Custom models (or `inherit` from parent)
- Permission modes (`default`, `plan`, `dontAsk`, `bubble`, `acceptEdits`, `bypassPermissions`)
- Max turn limits
- Agent-specific MCP servers
- Session-scoped hooks
- Skill preloading
- Persistent memory (user/project/local scope)
- Isolation modes (`worktree` or `remote`)
- Background-only execution flag

### Plugin Agents

Plugin agents are loaded from plugin packages and function like custom agents but with plugin metadata and namespacing (e.g., `my-plugin:my-agent`).

---

## 3. Agent Spawning and Lifecycle

### Spawning Flow

1. **Parent calls `Agent` tool** with `subagent_type`, `prompt`, and optional parameters (`run_in_background`, `name`, `team_name`, `model`, `isolation`).
2. **Agent definition resolved** from `agentDefinitions.activeAgents` by matching `agentType`.
3. **Tool pool assembled** via `resolveAgentTools()` -- applies allowlists, denylists, and filters out tools disallowed for agents.
4. **`runAgent()` generator invoked** -- creates an isolated `ToolUseContext` via `createSubagentContext()`, builds agent-specific system prompt, initializes MCP servers, registers hooks, preloads skills, and enters the `query()` loop.
5. **Messages streamed back** to the parent as the agent works.
6. **Cleanup on completion** -- MCP servers cleaned up, session hooks cleared, cache tracking released, file state cache cleared, bash tasks killed, perfetto agent unregistered.

### Foreground (Synchronous) Agents

- Share parent's `AbortController` -- parent abort kills the sub-agent.
- Share parent's `setAppState` callback -- state changes propagate immediately.
- Block the parent until all messages are yielded.
- Result is extracted from the agent's last assistant message and returned as `tool_result`.

### Background (Asynchronous) Agents

- Get their own `AbortController` -- run independently of parent.
- `setAppState` is a no-op (isolated).
- `setAppStateForTasks` still reaches the root store for task registration.
- Registered as `LocalAgentTask` in `AppState.tasks` for UI tracking.
- On completion, `enqueueAgentNotification()` delivers a `<task-notification>` XML message to the parent's conversation.
- Support periodic summarization (`startAgentSummarization`) for progress updates.

### Fork Sub-agents

When the fork feature is enabled, omitting `subagent_type` creates a **fork** -- a child that inherits the parent's full conversation context and system prompt for maximum prompt cache sharing. Key properties:
- The fork shares the parent's rendered system prompt bytes (byte-identical for cache hits).
- All tool_use blocks in the current assistant message get placeholder results.
- A per-child directive is appended as the final text block.
- Forks are always background (async).
- Recursive forking is prevented by detecting `<fork-boilerplate>` tags in conversation history.
- Forks use `permissionMode: 'bubble'` to surface permission prompts to the parent terminal.

---

## 4. Inter-Agent Communication

### SendMessage Tool

The `SendMessage` tool provides multiple communication channels:

1. **Teammate messaging** -- writes to a teammate's file-based mailbox at `~/.claude/teams/{team}/mailboxes/{name}/`.
2. **Broadcast** -- `to: "*"` sends to all team members except self.
3. **In-process agent messaging** -- routes to `AppState.agentNameRegistry` for named agents; queues messages or auto-resumes stopped agents.
4. **Cross-session messaging** -- via UDS sockets (`uds:/path`) or Remote Control bridge (`bridge:session_id`).
5. **Structured protocol messages** -- `shutdown_request`, `shutdown_response`, `plan_approval_response` for lifecycle management.

### Message Delivery

- Messages from teammates arrive automatically as conversation turns.
- In-process teammates have messages queued via `pendingUserMessages` and delivered at the next tool round.
- Stopped agents are auto-resumed by `resumeAgentBackground()` when a message targets them.

---

## 5. Team Management

### TeamCreate

Creates a multi-agent team with:
- A team config file at `~/.claude/teams/{team-name}/config.json`
- A shared task directory at `~/.claude/tasks/{team-name}/`
- The creator becomes `team-lead` with a deterministic agent ID (`team-lead@{team-name}`)
- Registered for automatic cleanup on session exit

### TeamDelete

Disbands a team:
- Fails if active (non-idle) members remain -- must send `shutdown_request` first.
- Cleans up team directory, task directory, and worktrees.
- Clears team context from AppState.

### Team File Structure

The `config.json` tracks:
- Team metadata (name, description, creation time)
- Lead agent ID and session ID
- Members array with: agentId, name, agentType, model, color, cwd, worktreePath, backendType, isActive status, permission mode
- Hidden pane IDs (for UI management)
- Team-wide allowed paths (for shared file editing permissions)

---

## 6. Coordination Patterns

### Coordinator Mode

When `CLAUDE_CODE_COORDINATOR_MODE=1`, the main agent becomes a pure orchestrator:
- Uses only `Agent`, `SendMessage`, `TaskStop`, and subscription tools.
- Workers get the full async tool set.
- The coordinator system prompt defines a research-synthesis-implementation-verification workflow.
- Workers are always `subagent_type: "worker"`.
- Results arrive as `<task-notification>` XML in user-role messages.
- The coordinator synthesizes findings and directs implementation -- "never delegate understanding."

### Task-Based Coordination (Swarm Mode)

Teams use a shared task list for work coordination:
1. Team lead creates tasks via `TaskCreate`.
2. Tasks are assigned to teammates via `TaskUpdate` with `owner`.
3. Teammates check `TaskList` after completing work to find available tasks.
4. Tasks can be blocked/unblocked, creating dependency chains.
5. Task directories live at `~/.claude/tasks/{team-name}/`.

### Permission Synchronization

The `permissionSync.ts` module coordinates permissions across swarm agents:
- Workers write permission requests to `~/.claude/teams/{team}/permissions/pending/`.
- The leader reads pending requests and surfaces them through the UI.
- Resolution is written to `permissions/resolved/` and the pending file is removed.
- Workers poll for resolution.
- A mailbox-based approach also exists, sending permission requests as structured messages.
- The `leaderPermissionBridge.ts` module bridges the REPL's permission UI to in-process teammates.

---

## 7. Context Sharing and Isolation

### What Is Shared

- **Root AppState store** -- via `setAppStateForTasks`, even async agents can register tasks and kill bash processes.
- **MCP server connections** -- referenced MCP servers (by string name) are shared/memoized; only inline definitions get per-agent cleanup.
- **File state cache** -- cloned from parent (not shared, each agent reads independently).
- **Permission rules** -- SDK-level (`cliArg`) rules are always preserved; session-level rules can be scoped.

### What Is Isolated

- **AbortController** -- async agents get their own; sync agents share parent's.
- **Messages** -- each agent has its own conversation history.
- **readFileState** -- cloned per-agent.
- **setAppState** -- no-op for async agents (except via `setAppStateForTasks`).
- **Denial tracking** -- fresh `createDenialTrackingState()` for isolated agents.
- **Content replacement state** -- cloned to prevent interference.
- **Nested memory triggers, tool decisions** -- fresh per-agent.

### Worktree Isolation

Agents can run in isolated git worktrees (`isolation: "worktree"`):
- Creates a temporary worktree branch from the current HEAD.
- The agent operates on a separate working copy -- changes do not affect the parent's files.
- A `buildWorktreeNotice()` message tells the fork child to translate paths and re-read files.
- On cleanup, worktrees are removed via `git worktree remove --force` (or fallback `rm -rf`).
- If the agent makes changes, the worktree path and branch are returned in the result.

---

## 8. Result Flow

### Synchronous Agents

1. `runAgent()` yields messages as an AsyncGenerator.
2. The parent's `AgentTool.call()` collects all messages.
3. `finalizeAgentTool()` extracts text content from the last assistant message.
4. Result is returned as `AgentToolResult` with content, usage stats, tool count, and duration.
5. The parent receives this as a `tool_result` block.

### Asynchronous Agents

1. `runAsyncAgentLifecycle()` drives the background agent.
2. Progress is tracked via `ProgressTracker` and written to `AppState.tasks[taskId]`.
3. On completion, `completeAsyncAgent()` transitions task status.
4. `enqueueAgentNotification()` creates a `<task-notification>` XML message injected into the parent's conversation as a user-role message.
5. The notification includes: task-id, status, summary, result text, and usage metrics.

### Resumed Agents

Agents can be resumed via `SendMessage` to a stopped/completed agent:
- `resumeAgentBackground()` loads the sidechain transcript from disk.
- Reconstructs replacement state for prompt cache stability.
- Re-runs `runAgent()` with the transcript as initial messages plus the new prompt.
- Delivers a notification on completion.

---

## 9. Backend Execution Modes for Teammates

### Tmux Backend

- Creates split panes in tmux for each teammate.
- Each teammate runs as a separate Claude process.
- Supports hide/show, border coloring, and rebalancing.
- Works inside existing tmux sessions or creates an external socket.

### iTerm2 Backend

- Uses iTerm2 native split panes via the `it2` CLI.
- Similar capabilities to tmux but leverages iTerm2's API.

### In-Process Backend

- Teammates run in the same Node.js process.
- Uses `AsyncLocalStorage` for context isolation (`runWithTeammateContext()`).
- Each teammate gets:
  - Independent `AbortController`
  - `TeammateContext` with identity (agentId, name, team, color)
  - Task state registered in `AppState.tasks`
- Messages are delivered via `pendingUserMessages` array on the task state.
- The `inProcessRunner.ts` wraps `runAgent()` with teammate context, progress tracking, idle notifications, plan mode approval flow, and auto-compact support.

---

## 10. When to Use Each Pattern

| Scenario | Pattern | Why |
|---|---|---|
| Quick codebase search | Synchronous `Explore` agent | Fast, read-only, uses haiku model |
| Implementation planning | Synchronous `Plan` agent | Read-only, returns structured plan |
| Multi-file implementation | Background `general-purpose` agent | Full tools, runs independently |
| Parallel independent tasks | Multiple background agents in one message | Concurrent execution, notifications on completion |
| Large coordinated project | Team with `TeamCreate` + teammates | Shared task list, mailbox messaging, permission sync |
| Pure orchestration | Coordinator mode | Main agent never touches files, delegates everything to workers |
| Context-preserving delegation | Fork sub-agent (omit `subagent_type`) | Shares prompt cache with parent, inherits full context |
| Post-implementation verification | `verification` agent | Adversarial testing, read-only, produces PASS/FAIL verdict |
| Isolated file changes | Agent with `isolation: "worktree"` | Separate git working copy, no interference with parent |
