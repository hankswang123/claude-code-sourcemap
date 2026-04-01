# Skills System and MCP Architecture in Claude Code CLI

## 1. Architecture Overview

Claude Code CLI has a layered extensibility system composed of three pillars:

1. **Skills** -- Markdown-based prompt templates that inject specialized instructions into the conversation
2. **MCP (Model Context Protocol)** -- External tool servers that expose tools, resources, and prompts over a standardized protocol
3. **Plugins** -- Installable packages from a marketplace that can bundle skills, hooks, and MCP servers together

All three systems converge into the same tool/command registries, allowing the model to use them interchangeably through the `Skill` tool and MCP-bridged tools.

---

## 2. Skills System

### 2.1 What Is a Skill?

A skill is a prompt-based command -- a Markdown document (typically `SKILL.md`) with YAML frontmatter that configures behavior and a body that becomes the system/user prompt injected when the skill is invoked. Skills are the primary mechanism for adding specialized domain knowledge and workflows to Claude Code.

### 2.2 Skill Sources (Loading Hierarchy)

Skills are loaded from multiple locations with defined precedence:

| Priority | Source | Location | `loadedFrom` value |
|----------|--------|----------|-------------------|
| 1 (highest) | Policy/Managed | `<managed-path>/.claude/skills/` | `managed` |
| 2 | User | `~/.claude/skills/` | `skills` |
| 3 | Project | `.claude/skills/` (walked up to home) | `skills` |
| 4 | Additional dirs | `--add-dir <path>/.claude/skills/` | `skills` |
| 5 | Legacy commands | `.claude/commands/` | `commands_DEPRECATED` |
| 6 | Bundled | Compiled into CLI binary | `bundled` |
| 7 | MCP | From connected MCP servers | `mcp` |
| 8 (lowest) | Plugin | From installed plugins | `plugin` |

### 2.3 Skill File Format

Directory-based: `skill-name/SKILL.md`

```yaml
---
description: Brief description of what this skill does
when_to_use: Triggers for the model to invoke this skill
allowed-tools: Edit, Write, Bash
model: sonnet
effort: high
context: fork          # 'inline' (default) or 'fork' (runs as sub-agent)
agent: Bash            # Agent type when forked
user-invocable: true   # Whether user can type /skill-name
disable-model-invocation: false
argument-hint: <file-path>
arguments: file, description
paths: src/**/*.ts     # Conditional activation on matching file paths
shell: bash            # Shell for !`cmd` blocks: 'bash' or 'powershell'
hooks:
  PreToolUse:
    - matcher: Edit
      hooks:
        - type: command
          command: echo "pre-edit hook"
---

Skill instructions in Markdown...

Use $ARGUMENTS for user-provided arguments.
Use ${CLAUDE_SKILL_DIR} for the skill's own directory path.
Use ${CLAUDE_SESSION_ID} for the current session ID.
Use !`command` for inline shell execution.
```

### 2.4 Skill Invocation Flow

1. The model sees available skills listed in system-reminder messages
2. Model calls the `Skill` tool with `{ skill: "skill-name", args: "..." }`
3. `SkillTool.validateInput()` verifies the skill exists and is prompt-based
4. `SkillTool.checkPermissions()` checks allow/deny rules
5. `SkillTool.call()` routes to either:
   - **Inline execution**: Processes the skill's markdown, substitutes variables, executes shell commands, and injects the result as new messages into the conversation. The skill can also modify the context (allowed tools, model, effort level).
   - **Forked execution** (when `context: fork`): Runs the skill in a sub-agent with its own context and token budget via `runAgent()`, returning the result as a tool output.

### 2.5 Bundled Skills

Bundled skills are compiled into the CLI binary and registered at startup via `registerBundledSkill()`. They include:

- `verify` -- Verify code changes by running the app
- `update-config` -- Configure Claude Code settings
- `debug` -- Debug issues
- `simplify` -- Review code for quality
- `remember` -- Save information to memory
- `stuck` -- Help when stuck on a problem
- `batch` -- Batch operations
- `loop` -- Recurring tasks (feature-gated)
- `claude-api` -- Build apps with Claude API (feature-gated)

### 2.6 Conditional Skills

Skills with `paths:` frontmatter are stored as "conditional" and only activated when the model touches files matching the glob patterns. Once activated, they move to the dynamic skills set and remain available.

### 2.7 Dynamic Skill Discovery

When the model reads/writes/edits files, `discoverSkillDirsForPaths()` walks up from the file path to CWD looking for `.claude/skills/` directories in subdirectories. Discovered skills are dynamically added to the session.

---

## 3. MCP Integration

### 3.1 MCP Server Configuration

MCP servers are configured in multiple scopes:

| Scope | Location | Notes |
|-------|----------|-------|
| Enterprise | `<managed-path>/managed-mcp.json` | Exclusive control when present |
| Local | `.claude/settings.local.json` mcpServers | Per-project-instance |
| Project | `.mcp.json` (walked up dirs) | Shared project config, requires approval |
| User | `~/.claude/settings.json` mcpServers | User-global |
| Plugin | Via installed plugins | Prefixed `plugin:<name>:<server>` |
| Claude.ai | Fetched from claude.ai account | Connector integrations |
| Dynamic | SDK-provided at runtime | For IDE extensions |

Supported transport types:
- `stdio` -- Spawns a subprocess (command + args + env)
- `sse` -- Server-Sent Events over HTTP
- `http` -- Streamable HTTP (MCP over HTTP with session management)
- `ws` -- WebSocket
- `sse-ide` / `ws-ide` -- IDE-specific transports
- `sdk` -- SDK-controlled transport (for VS Code extension)
- `claudeai-proxy` -- Claude.ai proxy connections

### 3.2 MCP Server Lifecycle

1. **Configuration loading** (`config.ts`): Merges configs from all scopes, applies policy filters (allowlist/denylist), deduplicates plugin and claude.ai servers
2. **Connection** (`client.ts`): Creates transport, instantiates `@modelcontextprotocol/sdk/client`, connects with timeout
3. **Tool/Resource/Prompt discovery**: Fetches `tools/list`, `resources/list`, `prompts/list` from each connected server
4. **Tool bridging**: Each MCP tool becomes a `Tool` object by spreading `MCPTool` template properties and overriding name, description, call, permissions
5. **Ongoing management** (`MCPConnectionManager.tsx` / `useManageMCPConnections.ts`): Handles reconnection, notifications (tools/list_changed, resources/list_changed), auth flows

### 3.3 How MCP Tools Appear to the Model

Each MCP tool is named `mcp__<serverName>__<toolName>` (built by `buildMcpToolName()`). The MCPTool template provides the base structure, and each tool instance overrides:

- `name` -- Fully qualified MCP name
- `description` / `prompt` -- From MCP server (capped at 2048 chars)
- `inputJSONSchema` -- From MCP server's tool definition
- `call()` -- Calls the MCP server via `callMCPTool()`
- `checkPermissions()` -- Returns `passthrough` (defers to standard permission system)
- `isConcurrencySafe()` / `isReadOnly()` -- Based on `annotations.readOnlyHint`
- `isDestructive()` -- Based on `annotations.destructiveHint`

### 3.4 MCP Resource Tools

When any connected server supports resources, two built-in tools are added:

- **ListMcpResourcesTool** (`mcp__list_resources`): Lists resources from all connected servers, optionally filtered by server name
- **ReadMcpResourceTool** (`mcp__read_resource`): Reads a specific resource by URI from a named server

### 3.5 MCP Auth Tool

When a server is in `needs-auth` state (HTTP 401), a pseudo-tool `mcp__<server>__authenticate` is created via `createMcpAuthTool()`. This lets the model start an OAuth flow on behalf of the user, returning an authorization URL. Once OAuth completes, the real tools replace this pseudo-tool automatically.

### 3.6 MCP Prompts as Skills

MCP servers that support `prompts/list` have their prompts converted to `Command` objects named `mcp__<server>__<prompt>`. These are available via the Skill tool. The `getPromptForCommand()` implementation calls the MCP server's `getPrompt()` method.

When the `MCP_SKILLS` feature flag is enabled, servers with resources support can also expose skills via `fetchMcpSkillsForClient()`, which loads skill definitions from MCP resources.

---

## 4. Plugin System

### 4.1 Plugin Types

- **Marketplace plugins**: Installed from git repositories, declared as `name@marketplace`
- **Built-in plugins**: Compiled into CLI, registered via `registerBuiltinPlugin()`, use `name@builtin` format
- **Official marketplace plugins**: From Anthropic-sanctioned marketplaces, given higher trust

### 4.2 Plugin Components

A plugin can provide:
- **Skills** (markdown command files)
- **Hooks** (PreToolUse, PostToolUse, Stop, etc.)
- **MCP servers** (via manifest `mcpServers` field, `.mcp.json`, or `.mcpb` files)
- **Output styles** (custom rendering)
- **LSP integration** (language server features)

### 4.3 Plugin MCP Integration

Plugin MCP servers are:
1. Loaded from the plugin's manifest (`mcpServers` field) or `.mcp.json` file
2. Environment variables resolved (`${CLAUDE_PLUGIN_ROOT}`, `${user_config.X}`, env vars)
3. Namespaced with `plugin:<pluginName>:<serverName>` prefix
4. Deduplicated against manually-configured servers (by command array or URL signature)
5. Merged into the global MCP server pool with lowest precedence

---

## 5. Key Relationships

### Skills <-> Tools

- Skills can specify `allowed-tools:` to auto-allow certain tools during their execution
- The Skill tool itself is a standard tool that appears in the model's tool list
- Skills can override the model (`model:`), effort level (`effort:`), and execution context

### Skills <-> System Prompt

- Available skills are listed in `system-reminder` messages with name + description (budget: 1% of context window)
- Skill descriptions are truncated to fit within the character budget
- Bundled skills get priority (never truncated); user/plugin skills may be trimmed
- When a skill is invoked inline, its content becomes new user messages in the conversation

### Skills <-> Permissions

- The Skill tool has its own permission layer checking allow/deny rules by skill name
- Skills with only "safe properties" are auto-allowed without user prompt
- Deny rules can block specific skills or prefixes (e.g., `review:*`)
- MCP tools use `passthrough` permissions (deferred to the standard tool permission system)

### MCP <-> Plugins

- Plugins can bundle MCP servers as first-class components
- Plugin servers get automatic environment variable injection (`CLAUDE_PLUGIN_ROOT`, `CLAUDE_PLUGIN_DATA`)
- Plugin servers are deduplicated against user-configured servers to prevent conflicts
- Built-in plugins can also provide MCP servers

### Skill Change Detection

- File watchers (chokidar) monitor skill directories for changes
- Changes trigger cache invalidation, skill reload, and listener notification
- Debounced to prevent cascading reloads during batch operations

---

## 6. How to Extend Claude Code

### Creating a Custom Skill

1. Create `.claude/skills/my-skill/SKILL.md`
2. Add YAML frontmatter with at minimum a `description`
3. Write markdown instructions in the body
4. Use `$ARGUMENTS` for user input, `${CLAUDE_SKILL_DIR}` for file references
5. Optionally add `when_to_use:` to help the model know when to invoke it

### Adding an MCP Server

```bash
claude mcp add my-server --type stdio -- npx my-mcp-server
```

Or add to `.mcp.json`:
```json
{
  "mcpServers": {
    "my-server": {
      "command": "npx",
      "args": ["my-mcp-server"],
      "env": { "API_KEY": "${MY_API_KEY}" }
    }
  }
}
```

### Creating a Plugin

1. Create a git repository with a plugin manifest
2. Include skills in a `skills/` directory, hooks config, or MCP server definitions
3. Publish to a marketplace or install directly via repository URL
4. Users install with `/plugin add name@marketplace`

### Enterprise MCP Control

- Set `managed-mcp.json` for exclusive server control
- Use `allowedMcpServers` / `deniedMcpServers` in policy settings for allowlist/denylist
- Support name-based, command-based, and URL-pattern-based filtering
