# Claude Code Permission System -- Detailed Code-Level Analysis

## 1. Core Type Definitions

### 1.1 Permission Modes (`src/types/permissions.ts`, lines 16-38)

```typescript
export const EXTERNAL_PERMISSION_MODES = [
  'acceptEdits', 'bypassPermissions', 'default', 'dontAsk', 'plan',
] as const

export type InternalPermissionMode = ExternalPermissionMode | 'auto' | 'bubble'
export type PermissionMode = InternalPermissionMode
```

The `auto` mode is conditionally included based on the `TRANSCRIPT_CLASSIFIER` build feature flag. The `bubble` mode is an internal-only mode not exposed externally. The `isExternalPermissionMode()` guard in `src/utils/permissions/PermissionMode.ts` (lines 97-105) filters out `auto` and `bubble` for non-Anthropic users.

### 1.2 Permission Rules (`src/types/permissions.ts`, lines 44-79)

```typescript
export type PermissionBehavior = 'allow' | 'deny' | 'ask'

export type PermissionRuleSource =
  | 'userSettings' | 'projectSettings' | 'localSettings'
  | 'flagSettings' | 'policySettings'
  | 'cliArg' | 'command' | 'session'

export type PermissionRule = {
  source: PermissionRuleSource
  ruleBehavior: PermissionBehavior
  ruleValue: PermissionRuleValue  // { toolName: string; ruleContent?: string }
}
```

### 1.3 ToolPermissionContext (`src/types/permissions.ts`, lines 427-441)

This is the central state object that carries all permission configuration:

```typescript
export type ToolPermissionContext = {
  readonly mode: PermissionMode
  readonly additionalWorkingDirectories: ReadonlyMap<string, AdditionalWorkingDirectory>
  readonly alwaysAllowRules: ToolPermissionRulesBySource
  readonly alwaysDenyRules: ToolPermissionRulesBySource
  readonly alwaysAskRules: ToolPermissionRulesBySource
  readonly isBypassPermissionsModeAvailable: boolean
  readonly strippedDangerousRules?: ToolPermissionRulesBySource
  readonly shouldAvoidPermissionPrompts?: boolean
  readonly awaitAutomatedChecksBeforeDialog?: boolean
  readonly prePlanMode?: PermissionMode
}
```

Key fields:
- `strippedDangerousRules` -- When auto mode strips dangerous permissions, the originals are stashed here for restoration on exit.
- `shouldAvoidPermissionPrompts` -- Set for headless/background subagents. Converts "ask" to "deny" unless a hook intervenes.
- `awaitAutomatedChecksBeforeDialog` -- When true, automated checks (coordinator handler) run before showing the interactive dialog.
- `prePlanMode` -- Remembers the mode before plan was entered, so exit-plan restores it.

### 1.4 Permission Decision Types (`src/types/permissions.ts`, lines 174-266)

The system uses a discriminated union for decisions:

- `PermissionAllowDecision` -- Contains optional `updatedInput` (tools can modify input as part of allowing) and `decisionReason`
- `PermissionAskDecision` -- Contains `message`, optional `suggestions` (proposed permission updates for the SDK), and optional `pendingClassifierCheck`
- `PermissionDenyDecision` -- Contains `message` and `decisionReason`
- `PermissionResult` adds `passthrough` (no opinion from the tool, converted to `ask` by the pipeline)

### 1.5 Decision Reasons (`src/types/permissions.ts`, lines 271-324)

```typescript
export type PermissionDecisionReason =
  | { type: 'rule'; rule: PermissionRule }
  | { type: 'mode'; mode: PermissionMode }
  | { type: 'subcommandResults'; reasons: Map<string, PermissionResult> }
  | { type: 'permissionPromptTool'; permissionPromptToolName: string; toolResult: unknown }
  | { type: 'hook'; hookName: string; hookSource?: string; reason?: string }
  | { type: 'asyncAgent'; reason: string }
  | { type: 'sandboxOverride'; reason: 'excludedCommand' | 'dangerouslyDisableSandbox' }
  | { type: 'classifier'; classifier: string; reason: string }
  | { type: 'workingDir'; reason: string }
  | { type: 'safetyCheck'; reason: string; classifierApprovable: boolean }
  | { type: 'other'; reason: string }
```

The `classifierApprovable` flag on `safetyCheck` determines whether auto mode sends the check to the classifier (true for sensitive files like `.claude/`, `.git/`) or forces a prompt (false for Windows path attacks).

---

## 2. Permission Rule Parsing

### 2.1 Rule String Format (`src/utils/permissions/permissionRuleParser.ts`)

Rules are parsed from strings like `"Bash(npm install:*)"`:

```typescript
// Line 93
export function permissionRuleValueFromString(ruleString: string): PermissionRuleValue {
  const openParenIndex = findFirstUnescapedChar(ruleString, '(')
  // ...
  const toolName = ruleString.substring(0, openParenIndex)
  const rawContent = ruleString.substring(openParenIndex + 1, closeParenIndex)
  // Empty content or "*" -> tool-wide rule (no ruleContent)
  if (rawContent === '' || rawContent === '*') {
    return { toolName: normalizeLegacyToolName(toolName) }
  }
  const ruleContent = unescapeRuleContent(rawContent)
  return { toolName: normalizeLegacyToolName(toolName), ruleContent }
}
```

Key behaviors:
- `Bash()` and `Bash(*)` are equivalent to just `Bash` (tool-wide)
- Escaped parentheses `\(` `\)` are supported for commands containing parens
- Legacy tool names are normalized: `Task` -> `Agent`, `KillShell` -> `TaskStop`

### 2.2 Legacy Name Mapping (`src/utils/permissions/permissionRuleParser.ts`, lines 21-29)

```typescript
const LEGACY_TOOL_NAME_ALIASES: Record<string, string> = {
  Task: AGENT_TOOL_NAME,
  KillShell: TASK_STOP_TOOL_NAME,
  AgentOutputTool: TASK_OUTPUT_TOOL_NAME,
  BashOutputTool: TASK_OUTPUT_TOOL_NAME,
}
```

---

## 3. Settings Loading Pipeline

### 3.1 Loading from Disk (`src/utils/permissions/permissionsLoader.ts`)

`loadAllPermissionRulesFromDisk()` (line 120) is the entry point:

```typescript
export function loadAllPermissionRulesFromDisk(): PermissionRule[] {
  // If enterprise policy says managed-only, ONLY use policySettings
  if (shouldAllowManagedPermissionRulesOnly()) {
    return getPermissionRulesForSource('policySettings')
  }
  // Otherwise, load from all enabled sources
  const rules: PermissionRule[] = []
  for (const source of getEnabledSettingSources()) {
    rules.push(...getPermissionRulesForSource(source))
  }
  return rules
}
```

Settings JSON is parsed via `settingsJsonToRules()` (line 91):
```json
{ "permissions": { "allow": ["Bash(git:*)"], "deny": ["Bash(rm -rf:*)"], "ask": [] } }
```

### 3.2 Managed-Only Mode (`src/utils/permissions/permissionsLoader.ts`, lines 31-36)

```typescript
export function shouldAllowManagedPermissionRulesOnly(): boolean {
  return getSettingsForSource('policySettings')?.allowManagedPermissionRulesOnly === true
}
```

When `true`, `syncPermissionRulesFromDisk()` in `permissions.ts` (lines 1419-1471) clears all non-policy rule sources before applying the policy rules.

### 3.3 Adding Rules to Settings (`src/utils/permissions/permissionsLoader.ts`, lines 229-296)

`addPermissionRulesToSettings()` handles deduplication via roundtrip normalization:
```typescript
const existingRulesSet = new Set(
  existingRules.map(raw => permissionRuleValueToString(permissionRuleValueFromString(raw)))
)
const newRules = ruleStrings.filter(rule => !existingRulesSet.has(rule))
```

This ensures legacy names (e.g., `KillShell`) are normalized to canonical form before comparison.

---

## 4. The Core Permission Pipeline

### 4.1 Entry Point (`src/utils/permissions/permissions.ts`, lines 473-956)

`hasPermissionsToUseTool` is the exported entry point (a `CanUseToolFn`):

```typescript
export const hasPermissionsToUseTool: CanUseToolFn = async (
  tool, input, context, assistantMessage, toolUseID
): Promise<PermissionDecision> => {
  const result = await hasPermissionsToUseToolInner(tool, input, context)
  // Post-pipeline transformations for auto mode, dontAsk, headless agents
  // ...
}
```

### 4.2 Inner Pipeline (`src/utils/permissions/permissions.ts`, lines 1158-1319)

Step-by-step execution:

**Step 1a -- Deny rule check** (line 1171):
```typescript
const denyRule = getDenyRuleForTool(appState.toolPermissionContext, tool)
if (denyRule) { return { behavior: 'deny', ... } }
```

**Step 1b -- Ask rule check** (line 1184):
```typescript
const askRule = getAskRuleForTool(appState.toolPermissionContext, tool)
if (askRule) {
  // Special case: sandbox auto-allow for Bash
  const canSandboxAutoAllow = tool.name === BASH_TOOL_NAME
    && SandboxManager.isSandboxingEnabled()
    && SandboxManager.isAutoAllowBashIfSandboxedEnabled()
    && shouldUseSandbox(input)
  if (!canSandboxAutoAllow) { return { behavior: 'ask', ... } }
}
```

**Step 1c -- Tool-specific check** (line 1210):
```typescript
let toolPermissionResult: PermissionResult = { behavior: 'passthrough', ... }
try {
  const parsedInput = tool.inputSchema.parse(input)
  toolPermissionResult = await tool.checkPermissions(parsedInput, context)
} catch (e) { /* ... */ }
```

**Step 1f -- Content-specific ask rules immune to bypass** (line 1244):
```typescript
if (toolPermissionResult?.behavior === 'ask'
    && toolPermissionResult.decisionReason?.type === 'rule'
    && toolPermissionResult.decisionReason.rule.ruleBehavior === 'ask') {
  return toolPermissionResult  // bypass-immune
}
```

**Step 1g -- Safety checks immune to bypass** (line 1255):
```typescript
if (toolPermissionResult?.behavior === 'ask'
    && toolPermissionResult.decisionReason?.type === 'safetyCheck') {
  return toolPermissionResult  // bypass-immune
}
```

**Step 2a -- Bypass mode** (line 1267):
```typescript
const shouldBypassPermissions =
  appState.toolPermissionContext.mode === 'bypassPermissions' ||
  (appState.toolPermissionContext.mode === 'plan'
   && appState.toolPermissionContext.isBypassPermissionsModeAvailable)
if (shouldBypassPermissions) { return { behavior: 'allow', ... } }
```

**Step 2b -- Allow rules** (line 1284):
```typescript
const alwaysAllowedRule = toolAlwaysAllowedRule(appState.toolPermissionContext, tool)
if (alwaysAllowedRule) { return { behavior: 'allow', ... } }
```

**Step 3 -- Passthrough to ask** (line 1300):
```typescript
const result: PermissionDecision = toolPermissionResult.behavior === 'passthrough'
  ? { ...toolPermissionResult, behavior: 'ask' as const, ... }
  : toolPermissionResult
```

### 4.3 Post-Pipeline: dontAsk Mode (`permissions.ts`, lines 505-517)

```typescript
if (appState.toolPermissionContext.mode === 'dontAsk') {
  return {
    behavior: 'deny',
    decisionReason: { type: 'mode', mode: 'dontAsk' },
    message: DONT_ASK_REJECT_MESSAGE(tool.name),
  }
}
```

### 4.4 Post-Pipeline: Auto Mode Classifier (`permissions.ts`, lines 519-927)

The auto mode section is extensive. Key stages:

1. **Safety check exemption** (line 530): Non-classifier-approvable safety checks stay immune.
2. **User interaction requirement** (line 549): Tools requiring interaction skip the classifier.
3. **PowerShell guard** (line 567): PowerShell requires explicit permission unless `POWERSHELL_AUTO_MODE` is on.
4. **acceptEdits fast-path** (line 600): Checks if the action would pass in acceptEdits mode to avoid classifier API call.
5. **Safe tool allowlist** (line 660): Hardcoded tools skip the classifier.
6. **Classifier API call** (line 693): `classifyYoloAction()` sends the transcript to a side-query.
7. **Denial limit check** (line 890): After the classifier blocks, checks if limits are exceeded.

### 4.5 Post-Pipeline: Headless Agents (`permissions.ts`, lines 930-952)

```typescript
if (appState.toolPermissionContext.shouldAvoidPermissionPrompts) {
  const hookDecision = await runPermissionRequestHooksForHeadlessAgent(
    tool, input, toolUseID, context, /* ... */
  )
  if (hookDecision) { return hookDecision }
  return { behavior: 'deny', message: AUTO_REJECT_MESSAGE(tool.name) }
}
```

---

## 5. Tool-Level Rule Matching

### 5.1 Tool Name Matching (`src/utils/permissions/permissions.ts`, lines 238-269)

```typescript
function toolMatchesRule(tool, rule): boolean {
  if (rule.ruleValue.ruleContent !== undefined) return false  // Content rules don't match tools
  const nameForRuleMatch = getToolNameForPermissionCheck(tool)
  if (rule.ruleValue.toolName === nameForRuleMatch) return true
  // MCP server-level matching
  const ruleInfo = mcpInfoFromString(rule.ruleValue.toolName)
  const toolInfo = mcpInfoFromString(nameForRuleMatch)
  return ruleInfo !== null && toolInfo !== null
    && (ruleInfo.toolName === undefined || ruleInfo.toolName === '*')
    && ruleInfo.serverName === toolInfo.serverName
}
```

### 5.2 Content-Level Rule Lookup (`src/utils/permissions/permissions.ts`, lines 349-390)

`getRuleByContentsForToolName()` builds a `Map<string, PermissionRule>` of all content-specific rules for a given tool name and behavior. For example, for tool `Bash` and behavior `allow`, it might contain:
```
"git:*" -> { source: 'localSettings', ruleBehavior: 'allow', ruleValue: { toolName: 'Bash', ruleContent: 'git:*' } }
"npm install" -> { source: 'projectSettings', ... }
```

---

## 6. File Path Permission Checks

### 6.1 Write Permission (`src/utils/permissions/filesystem.ts`, lines 1205-1412)

`checkWritePermissionForTool()` implements:

1. **Deny rules** for edit patterns (line 1220)
2. **Internal editable paths** -- plan files, scratchpad, job dirs, agent memory, memdir (line 1243)
3. **Session-scoped .claude/ allow rules** -- Only session-level rules can bypass .claude/ safety (line 1262). The rule must match `/.claude/**` or `~/.claude/**` patterns with no `..` and ending in `/**`.
4. **Safety checks** via `checkPathSafetyForAutoEdit()` (line 1305)
5. **Ask rules** for edit patterns (line 1341)
6. **acceptEdits mode** allows writes in working directory (line 1366)
7. **Allow rules** for edit patterns (line 1378)
8. **Default: ask** with suggestions (line 1396)

### 6.2 Read Permission (`src/utils/permissions/filesystem.ts`, lines 1030-1194)

`checkReadPermissionForTool()` is similar but with additional steps:

- Read-specific deny/ask rules are checked BEFORE the "edit implies read" shortcut
- Working directory reads are always allowed (no mode restriction)
- Internal readable paths include: session memory, project dir, plan files, tool results, scratchpad, project temp, agent memory, memdir, task/team dirs, bundled skill files

### 6.3 Path Safety Validation (`src/utils/permissions/filesystem.ts`, lines 620-665)

`checkPathSafetyForAutoEdit()` checks all resolved forms of the path:
```typescript
export function checkPathSafetyForAutoEdit(path, precomputedPathsToCheck?) {
  const pathsToCheck = precomputedPathsToCheck ?? getPathsForPermissionCheck(path)
  // 1. Windows patterns (classifierApprovable: false)
  // 2. Claude config files (classifierApprovable: true)
  // 3. Dangerous files/dirs (classifierApprovable: true)
}
```

### 6.4 Pattern-Based File Rules (`src/utils/permissions/filesystem.ts`, lines 955-1025)

`matchingRuleForInput()` uses the `ignore` library (gitignore-compatible) to match file paths against rules:

```typescript
export function matchingRuleForInput(path, toolPermissionContext, toolType, behavior) {
  const patternsByRoot = getPatternsByRoot(toolPermissionContext, toolType, behavior)
  for (const [root, patternMap] of patternsByRoot.entries()) {
    const ig = ignore().add(patterns)
    const relativePathStr = relativePath(root ?? getCwd(), fileAbsolutePath)
    const igResult = ig.test(relativePathStr)
    if (igResult.ignored && igResult.rule) { /* found matching rule */ }
  }
}
```

Patterns are rooted according to their prefix:
- `//...` -- absolute path from filesystem root (line 860)
- `~/...` -- relative to home directory (line 893)
- `/...` -- relative to the settings file's project root (line 898)
- No prefix -- matches anywhere (line 907)

---

## 7. Shell Command Permission Matching

### 7.1 Rule Parsing (`src/utils/permissions/shellRuleMatching.ts`)

Three rule types:

```typescript
export type ShellPermissionRule =
  | { type: 'exact'; command: string }       // "git status"
  | { type: 'prefix'; prefix: string }       // "git:*" (legacy)
  | { type: 'wildcard'; pattern: string }     // "git * --verbose"
```

### 7.2 Wildcard Matching (`src/utils/permissions/shellRuleMatching.ts`, lines 90-154)

`matchWildcardPattern()` converts patterns to regex:
- `\*` in pattern -> literal asterisk
- `\\` in pattern -> literal backslash
- Unescaped `*` -> `.*` regex
- Special: `git *` (trailing space+wildcard) also matches bare `git` (optional args)
- Uses `dotAll` flag so wildcards match newlines (heredoc commands)

### 7.3 Dangerous Pattern Detection (`src/utils/permissions/dangerousPatterns.ts`)

Cross-platform code execution entry points:
```typescript
export const CROSS_PLATFORM_CODE_EXEC = [
  'python', 'python3', 'python2', 'node', 'deno', 'tsx', 'ruby',
  'perl', 'php', 'lua', 'npx', 'bunx', 'npm run', 'yarn run',
  'pnpm run', 'bun run', 'bash', 'sh', 'ssh',
]
```

Anthropic-internal additions: `coo`, `fa run`, `gh`, `gh api`, `curl`, `wget`, `git`, `kubectl`, `aws`, `gcloud`, `gsutil`

### 7.4 Dangerous Rule Detection (`src/utils/permissions/permissionSetup.ts`, lines 93-233)

`isDangerousBashPermission()` checks multiple rule shape variants:
- Exact match: `python`
- Prefix: `python:*`
- Trailing wildcard: `python*`
- Space wildcard: `python *`
- Flag wildcard: `python -`...`*`

`isDangerousPowerShellPermission()` additionally checks `.exe` variants (e.g., `npm.exe run`).

---

## 8. Permission Mode Cycling and Transitions

### 8.1 Mode Cycling (`src/utils/permissions/getNextPermissionMode.ts`, lines 34-79)

```typescript
export function getNextPermissionMode(toolPermissionContext): PermissionMode {
  switch (toolPermissionContext.mode) {
    case 'default':
      // Ants: skip to bypassPermissions or auto
      // External: go to acceptEdits
    case 'acceptEdits': return 'plan'
    case 'plan':
      // bypassPermissions (if available) -> auto (if available) -> default
    case 'bypassPermissions':
      // auto (if available) -> default
    case 'auto': return 'default'
    case 'dontAsk': return 'default'
  }
}
```

### 8.2 Mode Transition Logic (`src/utils/permissions/permissionSetup.ts`, lines 597-646)

`transitionPermissionMode()` handles side effects:

```typescript
export function transitionPermissionMode(fromMode, toMode, context) {
  if (fromMode === toMode) return context
  handlePlanModeTransition(fromMode, toMode)  // Attachments
  handleAutoModeTransition(fromMode, toMode)  // Transcript markers

  if (toMode === 'plan') return prepareContextForPlanMode(context)

  if (toUsesClassifier && !fromUsesClassifier) {
    setAutoModeActive(true)
    context = stripDangerousPermissionsForAutoMode(context)
  } else if (fromUsesClassifier && !toUsesClassifier) {
    setAutoModeActive(false)
    setNeedsAutoModeExitAttachment(true)
    context = restoreDangerousPermissions(context)
  }
}
```

### 8.3 Dangerous Permission Stripping (`src/utils/permissions/permissionSetup.ts`, lines 510-553)

`stripDangerousPermissionsForAutoMode()`:
1. Collects all allow rules from context
2. Identifies dangerous ones via `findDangerousClassifierPermissions()`
3. Stashes them in `strippedDangerousRules`
4. Removes them from the active context via `removeDangerousPermissions()`

`restoreDangerousPermissions()` (line 561) reverses this on auto mode exit.

---

## 9. Path Validation Security

### 9.1 Tilde and Shell Expansion Blocking (`src/utils/permissions/pathValidation.ts`, lines 373-463)

`validatePath()` implements multiple security checks:

1. **UNC paths**: Block `\\server\share` to prevent credential leaks
2. **Tilde variants**: Block `~root`, `~+`, `~-` (shell expands differently than our code)
3. **Shell expansion**: Block `$VAR`, `${VAR}`, `$(cmd)`, `%VAR%`, `=cmd` (Zsh equals expansion)
4. **Glob in write ops**: Block `*.txt` in write operations (literal interpretation vs shell expansion mismatch)
5. **Dangerous removal paths**: Block `*`, `/*`, `/`, `~`, direct children of root, Windows drive roots

### 9.2 Windows Path Attack Detection (`src/utils/permissions/filesystem.ts`, lines 537-602)

`hasSuspiciousWindowsPathPattern()` detects:
- NTFS ADS: colon after position 2 (Windows/WSL only)
- 8.3 short names: `~\d` pattern
- Long path prefixes: `\\?\`, `\\.\`, `//?/`, `//./`
- Trailing dots/spaces
- DOS device names: `.git.CON`, `settings.json.PRN`
- Three+ consecutive dots as path components

---

## 10. Remote Managed Settings

### 10.1 Fetch and Cache (`src/services/remoteManagedSettings/index.ts`)

Architecture:
- `initializeRemoteManagedSettingsLoadingPromise()` -- Creates a promise other systems await
- `loadRemoteManagedSettings()` -- Cache-first: applies disk cache immediately, then fetches
- `fetchAndLoadRemoteManagedSettings()` -- Handles fetch + file caching + security check
- `pollRemoteSettings()` -- Background hourly polling
- `computeChecksumFromSettings()` -- SHA-256 of sorted JSON for ETag caching

### 10.2 Security Dialog (`src/services/remoteManagedSettings/securityCheck.tsx`)

```typescript
export async function checkManagedSettingsSecurity(
  cachedSettings, newSettings
): Promise<SecurityCheckResult> {
  if (!hasDangerousSettings(extractDangerousSettings(newSettings))) return 'no_check_needed'
  if (!hasDangerousSettingsChanged(cachedSettings, newSettings)) return 'no_check_needed'
  if (!getIsInteractive()) return 'no_check_needed'
  // Show blocking dialog -> 'approved' | 'rejected'
}
```

If rejected, `handleSecurityCheckResult()` calls `gracefulShutdownSync(1)`.

---

## 11. Denial Tracking (`src/utils/permissions/denialTracking.ts`)

Simple state machine:
```typescript
export const DENIAL_LIMITS = { maxConsecutive: 3, maxTotal: 20 }

export function shouldFallbackToPrompting(state): boolean {
  return state.consecutiveDenials >= 3 || state.totalDenials >= 20
}
```

For async subagents, denial state is stored in `context.localDenialTracking` (mutated in-place via `Object.assign`) instead of `appState.denialTracking` because subagent `setAppState` is a no-op.

---

## 12. Permission Update System (`src/utils/permissions/PermissionUpdate.ts`)

### 12.1 Update Types

Six update operations:
- `addRules` / `replaceRules` / `removeRules` -- Modify allow/deny/ask rule sets
- `setMode` -- Change the permission mode
- `addDirectories` / `removeDirectories` -- Modify working directory set

### 12.2 Applying Updates (`src/utils/permissions/PermissionUpdate.ts`, lines 55-100)

`applyPermissionUpdate()` creates a new context (immutable updates):
```typescript
case 'addRules': {
  const ruleKind = update.behavior === 'allow' ? 'alwaysAllowRules'
    : update.behavior === 'deny' ? 'alwaysDenyRules' : 'alwaysAskRules'
  return {
    ...context,
    [ruleKind]: {
      ...context[ruleKind],
      [update.destination]: [
        ...(context[ruleKind][update.destination] || []),
        ...ruleStrings,
      ],
    },
  }
}
```

### 12.3 Persisting Updates

`persistPermissionUpdates()` is called after user approval. For `addRules`, it calls `addPermissionRulesToSettings()` which writes to the appropriate settings file.

---

## 13. The UI/React Integration

### 13.1 useCanUseTool Hook (`src/hooks/useCanUseTool.tsx`)

This is the React-side bridge. It:

1. Creates a `PermissionContext` with queue operations for the UI
2. Calls `hasPermissionsToUseTool()` to get the pipeline decision
3. On `allow` -- resolves immediately with the allow decision
4. On `deny` -- logs the decision, records auto-mode denials, shows notification
5. On `ask` -- routes to one of three handlers:
   - `handleCoordinatorPermission()` -- For automated checks before dialog
   - `handleInteractivePermission()` -- Shows the permission dialog to the user
   - `handleSwarmWorkerPermission()` -- For swarm/team worker agents

### 13.2 Permission Suggestions

When a tool is denied/asked, the system generates `PermissionUpdate[]` suggestions:
- For file writes outside working dir: suggest `addDirectories` + `setMode:acceptEdits`
- For file reads outside working dir: suggest `addRules` for Read with the directory pattern
- For .claude/skills/ paths: suggest session-scoped `Edit(/.claude/skills/{name}/**)` rule

These suggestions are consumed by the SDK host to implement "Always allow" button behavior.

---

## 14. Key File Index

| File | Purpose |
|------|---------|
| `src/types/permissions.ts` | All type definitions (modes, rules, decisions, context) |
| `src/utils/permissions/permissions.ts` | Core pipeline: `hasPermissionsToUseTool()` |
| `src/utils/permissions/permissionSetup.ts` | Initialization, mode transitions, dangerous rule detection |
| `src/utils/permissions/permissionsLoader.ts` | Load/save rules from/to settings files |
| `src/utils/permissions/permissionRuleParser.ts` | Parse `"Tool(content)"` strings, legacy name normalization |
| `src/utils/permissions/PermissionMode.ts` | Mode metadata (titles, symbols, colors) |
| `src/utils/permissions/PermissionResult.ts` | Re-exports of decision types |
| `src/utils/permissions/PermissionRule.ts` | Re-exports of rule types |
| `src/utils/permissions/PermissionUpdate.ts` | Apply/persist permission updates |
| `src/utils/permissions/filesystem.ts` | File path permission checks, safety validation, dangerous files |
| `src/utils/permissions/pathValidation.ts` | Path expansion, glob handling, security blocks |
| `src/utils/permissions/shellRuleMatching.ts` | Shell command matching (exact, prefix, wildcard) |
| `src/utils/permissions/dangerousPatterns.ts` | Lists of dangerous command patterns |
| `src/utils/permissions/yoloClassifier.ts` | Auto mode AI classifier |
| `src/utils/permissions/bashClassifier.ts` | Bash-specific classifier (stub in external builds) |
| `src/utils/permissions/denialTracking.ts` | Consecutive/total denial state machine |
| `src/utils/permissions/getNextPermissionMode.ts` | Mode cycling logic (Shift+Tab) |
| `src/utils/permissions/bypassPermissionsKillswitch.ts` | Async gate check for bypass mode |
| `src/utils/permissions/classifierShared.ts` | Shared classifier utilities |
| `src/utils/permissions/classifierDecision.ts` | Auto mode decision logic (ant-only) |
| `src/utils/permissions/autoModeState.ts` | Auto mode activation state (ant-only) |
| `src/utils/permissions/permissionExplainer.ts` | Risk level explanation for permissions |
| `src/utils/permissions/shadowedRuleDetection.ts` | Detect rules that are overridden by others |
| `src/utils/permissions/PermissionPromptToolResultSchema.ts` | Schema for permission prompt tool results |
| `src/hooks/useCanUseTool.tsx` | React hook bridging pipeline to UI |
| `src/hooks/toolPermission/handlers/interactiveHandler.ts` | Interactive permission dialog |
| `src/hooks/toolPermission/handlers/coordinatorHandler.ts` | Automated pre-dialog checks |
| `src/hooks/toolPermission/handlers/swarmWorkerHandler.ts` | Swarm worker permission handling |
| `src/hooks/toolPermission/PermissionContext.ts` | Permission context for React integration |
| `src/services/tools/toolExecution.ts` | Tool execution orchestration |
| `src/services/remoteManagedSettings/index.ts` | Remote settings fetch/cache/poll |
| `src/services/remoteManagedSettings/securityCheck.tsx` | Security dialog for dangerous managed settings |
| `src/services/remoteManagedSettings/types.ts` | Remote settings API response schema |
| `src/services/remoteManagedSettings/syncCache.ts` | Cache eligibility and state |
| `src/services/remoteManagedSettings/syncCacheState.ts` | Session cache state management |
