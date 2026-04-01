# Skills and MCP: Detailed Code-Level Analysis

## 1. SkillTool Implementation

### File: `src/tools/SkillTool/SkillTool.ts`

**Tool Registration** (line 331-869):
The SkillTool is built via `buildTool()` with name `'Skill'` (from `constants.ts` line 1). Its input schema accepts `{ skill: string, args?: string }`.

**Validation** (lines 354-430):
- `validateInput()` normalizes the skill name (strips leading `/`), looks it up in `getAllCommands()`, and checks:
  - The command exists
  - `disableModelInvocation` is false
  - The command is prompt-type (not a built-in CLI command)
  - For experimental remote skills, checks `_canonical_<slug>` prefix against discovered session state

**Permission Checking** (lines 432-578):
- `checkPermissions()` implements a deny-first, allow-next, safe-property-auto-allow system
- Rule matching supports exact match and prefix wildcards (e.g., `review:*`)
- Skills with only "safe properties" (listed in `SAFE_SKILL_PROPERTIES` set, lines 875-908) are auto-allowed
- The safe properties allowlist includes: type, progressMessage, model, effort, source, pluginInfo, name, description, etc.

**Call Execution** (lines 580-841):
Two execution paths:
1. **Forked** (line 622-628): When `command.context === 'fork'`, delegates to `executeForkedSkill()` which runs a sub-agent via `runAgent()` (imported from `AgentTool/runAgent.js`)
2. **Inline** (lines 635-841): Processes via `processPromptSlashCommand()`, extracts metadata (allowedTools, model, effort), tags messages with tool use ID, and returns `newMessages` plus a `contextModifier` function

**Context Modifier** (lines 775-839):
The inline path returns a `contextModifier` that:
- Adds allowed tools to `toolPermissionContext.alwaysAllowRules`
- Overrides `mainLoopModel` if skill specifies a model
- Overrides `effortValue` if skill specifies effort

**Forked Execution** (lines 122-289):
`executeForkedSkill()` creates a sub-agent via:
- `prepareForkedCommandContext()` to build modified appState and prompt messages
- `runAgent()` iterator to execute the skill in isolation
- Progress reporting via `onProgress` callback
- Cleanup via `clearInvokedSkillsForAgent(agentId)` in finally block

### File: `src/tools/SkillTool/prompt.ts`

**Skill Listing Budget** (lines 21-24):
- `SKILL_BUDGET_CONTEXT_PERCENT = 0.01` (1% of context window)
- `CHARS_PER_TOKEN = 4`
- `DEFAULT_CHAR_BUDGET = 8_000` (fallback for 200k context)
- `MAX_LISTING_DESC_CHARS = 250` per entry

**Budget-Aware Formatting** (lines 70-171):
`formatCommandsWithinBudget()` implements progressive degradation:
1. Try full descriptions for all commands
2. If over budget, partition into bundled (never truncated) and rest
3. Calculate available space per non-bundled command
4. If less than 20 chars each, show names-only for non-bundled
5. Otherwise truncate descriptions proportionally

**Prompt Content** (lines 173-196):
The prompt instructs the model to:
- Check available skills in system-reminder messages
- Treat `/something` references as skill invocations
- Call the Skill tool BEFORE generating other responses
- Not invoke skills that are already running
- Recognize `<command-name>` tags as already-loaded skills

---

## 2. Skill Loading System

### File: `src/skills/loadSkillsDir.ts`

**Frontmatter Parsing** (lines 185-265):
`parseSkillFrontmatterFields()` extracts all skill metadata from YAML frontmatter:
- `description` with fallback to markdown extraction
- `allowed-tools` parsed via `parseSlashCommandToolsFromFrontmatter()`
- `model` parsed via `parseUserSpecifiedModel()` (with 'inherit' meaning no override)
- `effort` parsed via `parseEffortValue()`
- `context` supports 'fork' for sub-agent execution
- `hooks` validated against `HooksSchema`
- `shell` for `!`-block execution ('bash' or 'powershell')
- `paths` parsed as gitignore-style glob patterns
- `user-invocable` controls visibility
- `disable-model-invocation` prevents model from calling via Skill tool

**Skill Command Creation** (lines 270-401):
`createSkillCommand()` builds a `Command` object with a `getPromptForCommand()` method that:
1. Prepends "Base directory for this skill: ..." if baseDir is set
2. Substitutes `$ARGUMENTS` with user args
3. Replaces `${CLAUDE_SKILL_DIR}` with the skill directory path
4. Replaces `${CLAUDE_SESSION_ID}` with current session ID
5. Executes inline shell commands (`!`cmd``) -- but NOT for MCP skills (security, line 374)

**Loading from Skills Dir** (lines 407-480):
`loadSkillsFromSkillsDir()` reads directories, expects `skill-name/SKILL.md` format only. Single `.md` files are NOT supported in the `/skills/` directory.

**Master Loading Function** (lines 638-804):
`getSkillDirCommands()` (memoized) loads from all sources in parallel:
- Managed skills dir
- User skills dir
- Project skills dirs (walked up to home)
- Additional dirs (--add-dir)
- Legacy commands dir

Then deduplicates by resolved file path (symlink-aware via `realpath()`), separates conditional skills (with `paths:` frontmatter), and returns unconditional skills.

**Dynamic Discovery** (lines 861-975):
`discoverSkillDirsForPaths()` walks up from file paths to CWD looking for `.claude/skills/` dirs. Skips gitignored paths. `addSkillDirectories()` loads the discovered skills and merges them into the dynamic skills map.

**Conditional Activation** (lines 997-1058):
`activateConditionalSkillsForPaths()` uses the `ignore` library (gitignore-style matching) to test file paths against skill `paths:` patterns. Matching skills move from conditional storage to dynamic skills.

**MCP Skill Builder Registration** (lines 1083-1086):
At module init, `registerMCPSkillBuilders()` is called to expose `createSkillCommand` and `parseSkillFrontmatterFields` to the MCP skill discovery system without creating import cycles.

### File: `src/skills/bundledSkills.ts`

**BundledSkillDefinition Type** (lines 15-41):
Defines the interface for compiled-in skills, including optional `files` record for extracting reference files to disk on first invocation.

**Registration** (lines 53-100):
`registerBundledSkill()` creates a `Command` object and pushes it to the internal registry. If `files` is provided, wraps `getPromptForCommand()` to lazily extract files via `extractBundledSkillFiles()` and prepend a "Base directory" header.

**File Extraction** (lines 131-220):
Uses `O_NOFOLLOW | O_EXCL` flags for safe file writes (symlink protection). Files are extracted to a per-process nonce directory under `getBundledSkillsRoot()`.

### File: `src/skills/bundled/index.ts`

**Initialization** (lines 24-79):
`initBundledSkills()` registers all bundled skills. Feature-gated skills use `require()` for lazy loading:
- KAIROS: `registerDreamSkill()`
- REVIEW_ARTIFACT: `registerHunterSkill()`
- AGENT_TRIGGERS: `registerLoopSkill()`
- BUILDING_CLAUDE_APPS: `registerClaudeApiSkill()`
- RUN_SKILL_GENERATOR: `registerRunSkillGeneratorSkill()`

### File: `src/skills/bundled/verify.ts` (Example)

Shows the pattern for bundled skills:
1. Parse frontmatter from embedded content (`SKILL_MD`)
2. Extract description
3. Call `registerBundledSkill()` with name, description, optional `files`, and `getPromptForCommand()`

### File: `src/skills/mcpSkillBuilders.ts`

**Cycle-Breaking Registry** (lines 1-44):
A write-once registry that holds references to `createSkillCommand` and `parseSkillFrontmatterFields`. This breaks the dependency cycle between `client.ts -> mcpSkills.ts -> loadSkillsDir.ts -> ... -> client.ts`. Registration happens at `loadSkillsDir.ts` module init time.

---

## 3. MCP Tool Integration

### File: `src/tools/MCPTool/MCPTool.ts`

**Template Tool** (lines 27-77):
MCPTool is a template/prototype -- most properties are placeholder stubs ("Overridden in mcpClient.ts"). The actual MCP tool instances are created by spreading this template and overriding specific methods. Key properties:
- `isMcp: true`
- `name: 'mcp'` (overridden per tool)
- `maxResultSizeChars: 100_000`
- `checkPermissions()` returns `passthrough` (defers to standard system)

### File: `src/services/mcp/client.ts`

**Tool Building** (lines 1750-1998):
`fetchToolsForClient` (memoized with LRU cache) creates Tool objects for each MCP tool:

```typescript
return toolsToProcess.map((tool): Tool => ({
    ...MCPTool,  // Spread template
    name: buildMcpToolName(client.name, tool.name),  // mcp__server__tool
    mcpInfo: { serverName, toolName },
    description: () => tool.description,
    prompt: () => truncated description,
    inputJSONSchema: tool.inputSchema,
    call: async (args, context) => { /* MCP tool call with retry */ },
    // ... other overrides
}))
```

Tool annotations are mapped:
- `readOnlyHint` -> `isConcurrencySafe()`, `isReadOnly()`
- `destructiveHint` -> `isDestructive()`
- `openWorldHint` -> `isOpenWorld()`
- `title` -> `userFacingName()` (display: "server - title (MCP)")
- `anthropic/searchHint` -> `searchHint` (for deferred tool loading)
- `anthropic/alwaysLoad` -> `alwaysLoad` (prevents deferral)

**Tool Call Implementation** (lines 1833-1971):
The `call()` method:
1. Emits "started" progress
2. Calls `ensureConnectedClient()` (reconnects if needed)
3. Calls `callMCPToolWithUrlElicitationRetry()` (handles elicitation flows)
4. On success, emits "completed" progress, returns `{ data: content, mcpMeta }`
5. On `McpSessionExpiredError`, retries once with fresh client
6. On failure, emits "failed" progress, wraps errors for telemetry

**MCP Prompts as Commands** (lines 2033-2107):
`fetchCommandsForClient` converts MCP prompts to Command objects:
- Name: `mcp__<server>__<prompt>`
- `source: 'mcp'`
- `getPromptForCommand()` calls `client.getPrompt()` with parsed arguments

**Server Connection** (lines 2137-2210):
`reconnectMcpServerImpl()` handles full server lifecycle:
1. Clear keychain cache and server connection cache
2. Connect to server (transport creation, Client initialization)
3. Fetch tools, commands, MCP skills, and resources in parallel
4. Add ListMcpResourcesTool and ReadMcpResourceTool if server supports resources
5. Return combined tools, commands, and resources

**Batch Connection** (lines 2226+):
`getMcpToolsCommandsAndResources()` processes all configured servers:
1. Partitions into disabled and active entries
2. Disabled servers emit `DisabledMCPServer` connections immediately
3. Active servers are processed with concurrency control via `pMap`
4. Auth-cached servers get `NeedsAuthMCPServer` + `McpAuthTool` pseudo-tool

---

## 4. MCP Resource and Auth Tools

### File: `src/tools/ListMcpResourcesTool/ListMcpResourcesTool.ts`

- Name: `LIST_MCP_RESOURCES_TOOL_NAME` (from prompt.ts)
- Input: `{ server?: string }` (optional filter)
- Calls `fetchResourcesForClient()` for each matching connected client
- Returns array of `{ uri, name, mimeType, description, server }`
- Flags: `isConcurrencySafe: true`, `isReadOnly: true`, `shouldDefer: true`

### File: `src/tools/ReadMcpResourceTool/ReadMcpResourceTool.ts`

- Name: `ReadMcpResourceTool`
- Input: `{ server: string, uri: string }`
- Calls `client.request({ method: 'resources/read', params: { uri } })`
- Handles binary content: decodes base64 blobs, writes to disk via `persistBinaryContent()`
- Returns `{ contents: [{ uri, mimeType, text?, blobSavedTo? }] }`

### File: `src/tools/McpAuthTool/McpAuthTool.ts`

- Created dynamically by `createMcpAuthTool(serverName, config)` (line 49)
- Name: `mcp__<server>__authenticate`
- Description explains the server needs authentication
- `checkPermissions()` auto-allows (line 83)
- `call()` implementation (lines 85-205):
  1. Rejects claude.ai proxy and non-HTTP transports
  2. Starts `performMCPOAuthFlow()` with `skipBrowserOpen: true`
  3. Returns the authorization URL for user to open
  4. Background: on OAuth completion, reconnects server and swaps tools in appState
  5. Prefix-based replacement (`mcp__<server>__*`) removes this pseudo-tool automatically

---

## 5. MCP Configuration

### File: `src/services/mcp/config.ts`

**Configuration Scopes** (lines 888-1026):
`getMcpConfigsByScope()` loads from specific scope:
- `project`: Walks directories from CWD to root reading `.mcp.json` files
- `user`: From `getGlobalConfig().mcpServers`
- `local`: From `getCurrentProjectConfig().mcpServers`
- `enterprise`: From `managed-mcp.json`

**Merged Configuration** (lines 1071-1251):
`getClaudeCodeMcpConfigs()` merges all scopes:
1. Enterprise config has exclusive control when present
2. Policy filtering via `allowedMcpServers` / `deniedMcpServers`
3. Project servers require explicit approval (`getProjectMcpServerStatus()`)
4. Plugin servers deduplicated against manual servers by signature
5. Precedence: plugin < user < project < local

**Server Signatures** (lines 202-212):
`getMcpServerSignature()` creates dedup keys:
- stdio: `stdio:["command","arg1","arg2"]`
- remote: `url:<original-url>` (unwraps CCR proxy URLs)

**Policy Filtering** (lines 364-508):
Three-level policy:
1. Denylist always blocks (name, command, or URL pattern matching)
2. Allowlist gates access when defined (empty = block all)
3. Both support name-based, command-based, and URL-pattern-based entries

**Environment Variable Expansion** (lines 556-616):
`expandEnvVars()` handles `${VAR}` substitution in all config fields:
- stdio: command, args, env values
- remote: url, headers

### File: `src/services/mcp/types.ts`

**Server Config Types** (lines 28-161):
Zod schemas for each transport type:
- `McpStdioServerConfigSchema`: command, args, env
- `McpSSEServerConfigSchema`: url, headers, oauth config
- `McpHTTPServerConfigSchema`: url, headers, oauth config
- `McpWebSocketServerConfigSchema`: url, headers
- `McpSdkServerConfigSchema`: name only
- `McpClaudeAIProxyServerConfigSchema`: url, id

**Connection State Types** (lines 180-227):
Server connections have five states:
- `connected`: Active connection with client, capabilities, instructions
- `failed`: Connection failed with error message
- `needs-auth`: Requires OAuth authentication
- `pending`: Connection in progress (with reconnect attempt tracking)
- `disabled`: Explicitly disabled by user

---

## 6. Plugin-MCP Integration

### File: `src/utils/plugins/mcpPluginIntegration.ts`

**Loading Plugin MCP Servers** (lines 131-212):
`loadPluginMcpServers()` handles multiple config formats:
1. `.mcp.json` file in plugin directory (lowest priority)
2. `manifest.mcpServers` as:
   - String path to JSON file
   - String path to `.mcpb` (DXT manifest) file
   - Array of paths/configs
   - Direct inline config object

**Environment Resolution** (lines 465-582):
`resolvePluginMcpEnvironment()` resolves:
1. `${CLAUDE_PLUGIN_ROOT}` -> plugin installation path
2. `${user_config.X}` -> saved user configuration values
3. `${VAR}` -> general environment variables
For stdio servers, auto-injects `CLAUDE_PLUGIN_ROOT` and `CLAUDE_PLUGIN_DATA` env vars.

**Server Scoping** (lines 341-360):
`addPluginScopeToServers()` prefixes server names with `plugin:<pluginName>:<serverName>` and sets scope to `'dynamic'`.

### File: `src/plugins/builtinPlugins.ts`

**Built-in Plugin Registry** (lines 1-159):
- Plugins registered via `registerBuiltinPlugin()` with `name@builtin` ID format
- User can enable/disable via `/plugin` UI (persisted to settings)
- `isAvailable()` callback for conditional availability
- `defaultEnabled` controls initial state
- Skills from built-in plugins use `source: 'bundled'` to maintain listing/analytics compatibility

---

## 7. Skill Change Detection

### File: `src/utils/skills/skillChangeDetector.ts`

**File Watching** (lines 85-141):
Uses chokidar to watch skill directories:
- `~/.claude/skills/` and `~/.claude/commands/`
- `.claude/skills/` and `.claude/commands/` (project)
- `--add-dir` additional paths
- Depth 2 (for `skill-name/SKILL.md` format)
- Uses polling under Bun (workaround for FSWatcher deadlock bug)

**Debounced Reload** (lines 255-278):
`scheduleReload()` batches rapid changes:
1. Collects changed paths in a Set
2. After 300ms debounce, fires `ConfigChange` hooks
3. If not blocked by hooks, clears all skill caches
4. Emits `skillsChanged` signal for listener notification

---

## 8. Frontmatter Format

### File: `src/utils/frontmatterParser.ts`

**FrontmatterData Type** (lines 10-59):
All recognized frontmatter keys:
- `allowed-tools`: string | string[] (tool names to auto-allow)
- `description`: string (skill description)
- `type`: string (memory type for memory files)
- `argument-hint`: string (usage hint for arguments)
- `when_to_use`: string (triggers for model invocation)
- `version`: string
- `model`: string ('inherit', 'haiku', 'sonnet', 'opus', or specific model)
- `skills`: string (comma-separated skill names for agents)
- `user-invocable`: string (boolean-like: 'true'/'false')
- `hooks`: HooksSettings (event handlers)
- `effort`: string ('low'/'medium'/'high'/'max' or integer)
- `context`: 'inline' | 'fork'
- `agent`: string (agent type for forked context)
- `paths`: string | string[] (gitignore-style glob patterns)
- `shell`: string ('bash' or 'powershell')
- `disable-model-invocation`: string (boolean-like)
- Plus any additional `[key: string]: unknown`

---

## 9. How Skills Modify Behavior

### Inline Skill Injection

When a skill is invoked inline:
1. The skill's markdown content (after variable substitution and shell execution) becomes a new `UserMessage` with `isMeta: true`
2. The `contextModifier` returned by SkillTool.call() modifies the `ToolUseContext`:
   - Adds `allowedTools` to `alwaysAllowRules.command` (auto-approves specified tools)
   - Overrides `mainLoopModel` if skill specifies a model
   - Overrides `effortValue` if skill specifies effort
3. The skill content is registered with `addInvokedSkill()` for compaction-survival

### Forked Skill Execution

When a skill runs forked (`context: fork`):
1. A sub-agent is created with its own message history and token budget
2. The skill's effort is merged into the agent definition
3. Progress is reported via `onProgress` callbacks showing tool uses
4. The final result text is extracted and returned as the tool result
5. The agent state is cleaned up via `clearInvokedSkillsForAgent()`

### Skill Hooks

Skills can define hooks in frontmatter that are registered when the skill is invoked:
```yaml
hooks:
  PreToolUse:
    - matcher: Edit
      hooks:
        - type: command
          command: "run-my-linter $FILE"
```
These hooks are registered via `registerSkillHooks()` during skill processing and fire for matching tool events.

---

## 10. Key Data Flow Summary

```
User types /skill or model calls Skill tool
    |
    v
SkillTool.validateInput() -- finds command in getAllCommands()
    |                          (local skills + MCP skills from AppState)
    v
SkillTool.checkPermissions() -- deny/allow rules + safe-property auto-allow
    |
    v
SkillTool.call()
    |
    +-- context='fork' --> executeForkedSkill() --> runAgent() --> result text
    |
    +-- context='inline' --> processPromptSlashCommand()
            |
            v
        getPromptForCommand(args, context)
            |
            +-- substituteArguments()
            +-- ${CLAUDE_SKILL_DIR} / ${CLAUDE_SESSION_ID} replacement
            +-- executeShellCommandsInPrompt() (NOT for MCP skills)
            |
            v
        Return newMessages + contextModifier
```

```
MCP Server Connection:
    config.ts (load + merge + policy-filter configs)
        |
        v
    client.ts connectToServer() (create transport + Client)
        |
        v
    fetchToolsForClient() --> Tool[] (spread MCPTool + overrides)
    fetchCommandsForClient() --> Command[] (MCP prompts)
    fetchResourcesForClient() --> ServerResource[]
        |
        v
    AppState.mcp.{tools, commands, clients, resources}
        |
        v
    Model sees mcp__server__tool in available tools
    Model sees mcp__server__prompt in available skills
```
