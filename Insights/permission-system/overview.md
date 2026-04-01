# Claude Code Permission System -- Architectural Overview

## 1. Executive Summary

The Claude Code CLI implements a multi-layered permission system that governs which tools the AI model can invoke and under what conditions. The system is designed with a "defense-in-depth" philosophy: multiple independent checks must all agree before a tool can execute. The architecture supports individual developers, enterprise-managed deployments, and headless/SDK integrations with different trust profiles.

The permission system has five core responsibilities:
1. **Gate tool execution** -- decide allow/ask/deny for every tool invocation
2. **Protect dangerous paths and commands** -- hardened rules for shell, filesystem, and configuration files
3. **Support multiple permission modes** -- from fully interactive to fully autonomous
4. **Enable enterprise policy enforcement** -- remotely managed settings override local config
5. **Handle subagent delegation safely** -- headless agents cannot bypass interactive approval requirements

---

## 2. Permission Modes

The system defines six permission modes that control the overall trust level. Users cycle through modes with Shift+Tab.

| Mode | Symbol | Behavior |
|------|--------|----------|
| **default** | (none) | Ask for permission on file writes, shell commands, MCP tools. File reads in working directory are auto-allowed. |
| **plan** | Pause icon | Claude explains what it would do but does not execute. Combined with auto mode for opt-in users. |
| **acceptEdits** | Fast-forward | Auto-allow file writes within the working directory. Still asks for shell commands and MCP tools. |
| **bypassPermissions** | Fast-forward (red) | Auto-allow almost everything. Only safety checks (`.git/`, `.claude/`, shell configs) and explicit ask/deny rules can block. Can be disabled by enterprise policy. |
| **auto** | Fast-forward (warning) | Internal/ant-only mode. Uses an AI classifier to decide allow/deny instead of prompting the user. Falls back to prompting after denial limits. |
| **dontAsk** | Fast-forward (red) | Converts all "ask" decisions into "deny" -- used for non-interactive sessions where prompting is impossible. |

Mode cycling order:
- External users: `default` -> `acceptEdits` -> `plan` -> `bypassPermissions` (if available) -> `auto` (if available) -> `default`
- Anthropic internal: `default` -> `bypassPermissions` (if available) -> `auto` (if available) -> `default`

---

## 3. Permission Pipeline

Every tool invocation passes through a well-defined decision pipeline in `hasPermissionsToUseToolInner()`. The steps execute in strict order with early return:

### Step 1: Rule-based checks (always respected, even in bypass mode)
1. **1a. Deny rules** -- If the entire tool matches a deny rule, immediately deny.
2. **1b. Ask rules** -- If the entire tool matches an ask rule, prompt (unless sandbox auto-allow applies for Bash).
3. **1c. Tool-specific checkPermissions** -- Each tool implements its own `checkPermissions()` method for content-aware checks (e.g., Bash checks each subcommand, file tools check paths).
4. **1d. Tool-denied** -- If the tool's own check returned deny, honor it.
5. **1e. User interaction required** -- Some tools require interactive UI even in bypass mode.
6. **1f. Content-specific ask rules** -- Explicit ask rules on specific commands (e.g., `Bash(npm publish:*)`) are respected even in bypass mode.
7. **1g. Safety checks** -- Hardcoded safety for `.git/`, `.claude/`, `.vscode/`, `.idea/`, shell config files. These are **bypass-immune**.

### Step 2: Mode-based resolution
1. **2a. bypassPermissions** -- If mode is bypassPermissions (or plan mode with bypass available), allow.
2. **2b. Always-allow rules** -- If the tool matches an allow rule, allow.

### Step 3: Default behavior
- If the tool's `checkPermissions()` returned "passthrough" (no opinion), convert to "ask".

### Post-pipeline transformations (outer function)
- **dontAsk mode**: Converts "ask" to "deny" with rejection message.
- **auto mode**: Runs the AI classifier. If it approves, allow. If it blocks, deny (with fallback to prompting after denial limits).
- **Headless agents**: Runs PermissionRequest hooks; if none decide, auto-deny.

---

## 4. Permission Rules Configuration

### 4.1 Rule Structure

Rules follow the format: `ToolName` or `ToolName(content)`.

Examples:
- `Bash` -- matches the entire Bash tool
- `Bash(npm install:*)` -- matches npm install commands (prefix match)
- `Bash(git status)` -- exact command match
- `Edit(/.env)` -- matches .env file edits (gitignore-style pattern)
- `mcp__server1` -- matches all tools from MCP server "server1"
- `mcp__server1__*` -- wildcard for all tools from an MCP server
- `Agent(Explore)` -- matches a specific sub-agent type

Parenthesized content supports escaped characters: `\(`, `\)`, `\\`.

### 4.2 Rule Behaviors

Each rule has one of three behaviors:
- **allow** -- Auto-approve matching tool use
- **deny** -- Block matching tool use entirely
- **ask** -- Force an interactive prompt even in permissive modes

### 4.3 Rule Sources (precedence order)

Rules can come from multiple sources, evaluated by type:

| Source | Location | Persistence | Editable |
|--------|----------|-------------|----------|
| `policySettings` | Remote managed settings API | Server-side | No (enterprise admin only) |
| `flagSettings` | Feature flags | Server-side | No |
| `userSettings` | `~/.claude/settings.json` | Disk | Yes |
| `projectSettings` | `.claude/settings.json` in project | Disk | Yes |
| `localSettings` | `.claude/settings.local.json` | Disk | Yes |
| `cliArg` | `--allowed-tools`, `--disallowed-tools` | Session | No |
| `command` | Slash commands | Session | No |
| `session` | Runtime decisions | Memory | No |

### 4.4 Settings JSON Format

```json
{
  "permissions": {
    "allow": ["Bash(git:*)", "Edit"],
    "deny": ["Bash(rm -rf:*)"],
    "ask": ["Bash(npm publish:*)"],
    "defaultMode": "acceptEdits",
    "disableBypassPermissionsMode": "disable",
    "additionalDirectories": ["/path/to/extra/dir"]
  }
}
```

---

## 5. Path-Based File Access Control

### 5.1 Working Directory Model

Files within the working directory (and additional registered directories) get relaxed permissions:
- **Reads** are auto-allowed in all modes.
- **Writes** are auto-allowed only in `acceptEdits` mode or higher.

Additional working directories can be added via:
- `--add-dir` CLI flag
- `settings.permissions.additionalDirectories`
- Session-level additions (symlink resolution of `$PWD`)

### 5.2 Safety Checks (Bypass-Immune)

The following paths require manual approval even in `bypassPermissions` mode:

**Dangerous directories**: `.git/`, `.vscode/`, `.idea/`, `.claude/`

**Dangerous files**: `.gitconfig`, `.gitmodules`, `.bashrc`, `.bash_profile`, `.zshrc`, `.zprofile`, `.profile`, `.ripgreprc`, `.mcp.json`, `.claude.json`

**Windows attack patterns** (all platforms for defense-in-depth):
- NTFS Alternate Data Streams (e.g., `file.txt::$DATA`)
- 8.3 short names (e.g., `GIT~1`)
- Long path prefixes (`\\?\`, `\\.\`)
- Trailing dots/spaces
- DOS device names (`.git.CON`)
- UNC paths (`\\server\share`)

### 5.3 Internal Paths (Auto-Allowed)

Certain internal paths bypass normal checks:
- Session plan files (`{plansDir}/{planSlug}.md`)
- Scratchpad directory (`/tmp/claude-{uid}/{project}/{session}/scratchpad/`)
- Session memory directory
- Agent memory files
- Job directories (for templates feature)
- `.claude/launch.json` (desktop preview config)

### 5.4 File Permission Rules

File-level rules use gitignore-style glob patterns:
- `/src/**` -- relative to the settings file's project root
- `//home/user/secret/**` -- absolute path (double-slash prefix)
- `~/.ssh/**` -- home directory relative
- `*.env` -- matches anywhere (no root prefix)

Rules apply to tool names `Edit` (for write tools) and `Read` (for read tools).

---

## 6. Bash Command Permissions

### 6.1 Rule Types for Shell Commands

Shell commands support three rule syntaxes:
- **Exact match**: `git status` -- only matches that exact command
- **Prefix match (legacy)**: `git:*` -- matches any command starting with "git"
- **Wildcard match**: `git * --verbose` -- glob-style pattern matching

### 6.2 Dangerous Patterns

Rules that would auto-allow code execution are flagged as "dangerous" and stripped when entering auto mode:
- Tool-level Bash allow (`Bash` or `Bash(*)`)
- Interpreter prefixes: `python:*`, `node:*`, `ruby:*`, `perl:*`, etc.
- Package runners: `npx:*`, `npm run:*`, etc.
- Shell launchers: `bash:*`, `sh:*`, `zsh:*`
- Eval/exec: `eval:*`, `exec:*`, `sudo:*`

PowerShell has its own dangerous pattern list including cmdlets like `Invoke-Expression`, `Start-Process`, `Add-Type`.

### 6.3 Sandbox Integration

When sandboxing is enabled:
- Sandboxed Bash commands can bypass ask rules (the sandbox is the safety mechanism)
- Commands with `dangerouslyDisableSandbox: true` still require normal permission checks
- Sandbox write-allowlist paths are treated as additional allowed write directories

---

## 7. MCP Tool Permissions

MCP (Model Context Protocol) tools follow the same permission pipeline as built-in tools.

- Tool name format: `mcp__<serverName>__<toolName>`
- Server-level rules: `mcp__server1` matches all tools from that server
- Wildcard server rules: `mcp__server1__*` matches all tools from server1
- Individual tool rules: `mcp__server1__tool1` matches a specific tool
- In skip-prefix mode (`CLAUDE_AGENT_SDK_MCP_NO_PREFIX`), MCP tools use their unprefixed names for display but the fully-qualified name for permission matching

---

## 8. Subagent Permission Model

### 8.1 Main Session vs Subagents

Subagents (spawned via the Agent tool) have restricted permissions:

- **Interactive subagents**: Inherit the main session's permission context
- **Headless/background subagents**: Set `shouldAvoidPermissionPrompts: true`, which means any "ask" decision becomes "deny" unless a PermissionRequest hook provides a decision
- **Async subagents**: Use `localDenialTracking` instead of global app state (since `setAppState` is a no-op for them)

### 8.2 Agent-Level Deny Rules

Specific agent types can be blocked: `Agent(Explore)` denies the Explore agent while allowing other agents.

### 8.3 Dangerous Agent Permissions

In auto mode, any Agent allow rule is considered dangerous because it would bypass the classifier's evaluation of sub-agent prompts.

---

## 9. Auto Mode (AI Classifier)

### 9.1 Architecture

Auto mode replaces user prompts with an AI classifier that evaluates each tool invocation:

1. **acceptEdits fast-path**: Before running the classifier, checks if the action would be allowed in acceptEdits mode. This saves API calls for safe operations.
2. **Safe tool allowlist**: Some tools are on a hardcoded allowlist and skip the classifier.
3. **Classifier API call**: Sends the conversation transcript plus the proposed action to a side-query. The classifier returns allow/block with a reason.

### 9.2 Denial Tracking

The system tracks consecutive and total denials:
- After 3 consecutive denials: falls back to interactive prompting
- After 20 total denials in a session: falls back to interactive prompting
- In headless mode: throws `AbortError` instead of prompting

### 9.3 Fail-Closed Behavior

When the classifier API is unavailable:
- By default (iron gate closed): deny with retry guidance
- Iron gate open (feature flag): fall back to normal permission handling
- Transcript too long: fall back to prompting (deterministic error, won't recover)

---

## 10. Enterprise/MDM Management

### 10.1 Remote Managed Settings

Enterprise customers can push settings via API to `policySettings`:
- Fetched from `{BASE_API_URL}/api/claude_code/settings`
- Supports ETag-based caching (304 Not Modified)
- Polled hourly for changes
- Fails open -- if fetch fails, continues without remote settings
- Eligible users: Console API key users (all), OAuth users (Enterprise/Team only)

### 10.2 Policy Controls

Enterprise admins can:
- **`allowManagedPermissionRulesOnly`**: When true, only `policySettings` rules are respected. All user/project/local/CLI rules are ignored.
- **`disableBypassPermissionsMode`**: Prevents users from entering bypass mode (both via CLI flag and settings). Can also be controlled via Statsig gate.
- **`disableAutoMode`**: Prevents auto mode activation.
- **`defaultMode`**: Set the default permission mode for all users (only `acceptEdits` and `plan` are honored in CCR/remote environments).

### 10.3 Security Checks for Managed Settings

When remote settings contain "dangerous" changes (hooks, etc.), a blocking security dialog is shown. Users must approve before the settings are applied. If rejected, the process exits.

---

## 11. Key Design Insights

### 11.1 Defense in Depth
- Multiple independent checks (rules, safety checks, path validation, classifier) must all agree
- Safety checks are immune to bypass mode
- Symlink resolution checks both original and resolved paths
- Case-insensitive path comparison prevents bypass on macOS/Windows

### 11.2 Fail-Safe Defaults
- Unknown modes default to "default" (most restrictive interactive mode)
- Unknown paths default to "ask"
- API failures fail open for availability but closed for security-critical paths
- Empty/invalid settings are handled gracefully

### 11.3 Separation of Concerns
- Each tool implements its own `checkPermissions()` method for content-aware decisions
- The permission pipeline handles mode-based and rule-based logic centrally
- Settings loading, rule parsing, and rule matching are independent subsystems
- Enterprise policy layer sits above all local configuration

### 11.4 Practical Usage Tips
- Use `settings.local.json` for machine-specific rules that should not be committed
- Use `settings.json` in the project for team-shared rules
- `Bash(git:*)` is safer than `Bash` because it restricts to git commands
- Safety checks on `.claude/` paths can be bypassed for skill editing via session-scoped `.claude/skills/{name}/**` rules
- The `--add-dir` flag is useful for monorepos where you work across multiple directories

---

## 12. Relationships Map

```
settings.json (user/project/local/policy)
    |
    v
permissionsLoader.ts  -- Loads rules from all sources
    |
    v
permissionSetup.ts    -- Initializes ToolPermissionContext, handles mode transitions
    |
    v
permissions.ts        -- Core pipeline: hasPermissionsToUseTool()
    |                     Checks rules, mode, safety, classifier
    |
    +-- Tool.checkPermissions()     -- Each tool's own permission logic
    |     |
    |     +-- filesystem.ts         -- File path validation, safety checks
    |     +-- shellRuleMatching.ts  -- Bash/PowerShell command matching
    |     +-- pathValidation.ts     -- Path expansion, glob handling, security
    |
    +-- yoloClassifier.ts           -- Auto mode AI classifier
    +-- denialTracking.ts           -- Consecutive/total denial limits
    +-- hooks.ts                    -- PermissionRequest hooks for headless agents
    |
    v
useCanUseTool.tsx     -- React hook: bridges pipeline to UI
    |
    +-- interactiveHandler.ts   -- Shows permission dialog to user
    +-- coordinatorHandler.ts   -- Handles automated checks before dialog
    +-- swarmWorkerHandler.ts   -- Handles swarm/team worker permissions
    |
    v
toolExecution.ts      -- Executes tool if allowed, handles deny/cancel
```
