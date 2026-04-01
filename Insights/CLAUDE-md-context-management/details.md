# CLAUDE.md Context Management -- Detailed Code-Level Analysis

This document contains file paths, line references, and detailed code-level analysis of how CLAUDE.md files are handled in the Claude Code CLI source code.

---

## Core Files

| File | Role |
|---|---|
| `src/utils/claudemd.ts` | Central module: discovery, loading, parsing, merging, caching |
| `src/utils/config.ts` | Config management, `getMemoryPath()`, `getManagedClaudeRulesDir()`, `getUserClaudeRulesDir()` |
| `src/context.ts` | Builds user context for system prompt injection (`getUserContext()`) |
| `src/constants/prompts.ts` | System prompt assembly (`getSystemPrompt()`) |
| `src/utils/frontmatterParser.ts` | YAML frontmatter parsing, path splitting, brace expansion |
| `src/utils/attachments.ts` | Lazy nested/conditional rule loading during conversation |
| `src/utils/hooks.ts` | `InstructionsLoaded` hook event (lines 4303-4369) |
| `src/memdir/memdir.ts` | Auto-memory prompt building (`loadMemoryPrompt()`, `buildMemoryPrompt()`) |
| `src/memdir/paths.ts` | Auto-memory path resolution, `isAutoMemoryEnabled()` |
| `src/memdir/memoryTypes.ts` | Memory type taxonomy for auto-memory entries |
| `src/utils/memory/types.ts` | `MemoryType` union: `'User' \| 'Project' \| 'Local' \| 'Managed' \| 'AutoMem' \| 'TeamMem'` |
| `src/utils/memoryFileDetection.ts` | Detection of auto-managed memory files vs user-managed instruction files |
| `src/utils/settings/types.ts` | Settings schema including `claudeMdExcludes` (line 1053) |
| `src/utils/settings/constants.ts` | `isSettingSourceEnabled()` (line 174) |
| `src/utils/permissions/yoloClassifier.ts` | Auto-permission classifier using CLAUDE.md content |
| `src/components/ClaudeMdExternalIncludesDialog.tsx` | UI for approving external @include paths |
| `src/skills/bundled/remember.ts` | `/remember` skill for reviewing and promoting memory entries |

---

## 1. Discovery Logic

### Entry point: `getMemoryFiles()` (claudemd.ts:790-1075)

This is the main memoized function that discovers all memory files. It returns `MemoryFileInfo[]`.

**Loading order (lines 803-1007):**

```
1. Managed: getMemoryPath('Managed')  -->  /etc/claude-code/CLAUDE.md
   + getManagedClaudeRulesDir()       -->  /etc/claude-code/.claude/rules/*.md

2. User (if userSettings enabled):
   getMemoryPath('User')              -->  ~/.claude/CLAUDE.md
   + getUserClaudeRulesDir()          -->  ~/.claude/rules/*.md

3. Project + Local (directory walk):
   For each dir from root to CWD:
     - CLAUDE.md          (Project, if projectSettings enabled)
     - .claude/CLAUDE.md  (Project, if projectSettings enabled)
     - .claude/rules/*.md (Project, if projectSettings enabled, unconditional only)
     - CLAUDE.local.md    (Local, if localSettings enabled)

4. Additional directories (if CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD):
   Same as #3 but for --add-dir paths

5. AutoMem entrypoint (if isAutoMemoryEnabled()):
   getAutoMemEntrypoint()  -->  ~/.claude/projects/<key>/memory/MEMORY.md

6. TeamMem entrypoint (if feature('TEAMMEM') && isTeamMemoryEnabled()):
   getTeamMemEntrypoint()  -->  team memory path
```

### Path construction: `getMemoryPath()` (config.ts:1779-1799)

```typescript
switch (memoryType) {
  case 'User':    return join(getClaudeConfigHomeDir(), 'CLAUDE.md')      // ~/.claude/CLAUDE.md
  case 'Local':   return join(cwd, 'CLAUDE.local.md')
  case 'Project': return join(cwd, 'CLAUDE.md')
  case 'Managed': return join(getManagedFilePath(), 'CLAUDE.md')          // /etc/claude-code/CLAUDE.md
  case 'AutoMem': return getAutoMemEntrypoint()                           // ~/.claude/projects/.../memory/MEMORY.md
}
```

### Rules directory paths (config.ts:1801-1807)

```typescript
getManagedClaudeRulesDir()  // join(getManagedFilePath(), '.claude', 'rules')
getUserClaudeRulesDir()     // join(getClaudeConfigHomeDir(), 'rules')  -->  ~/.claude/rules/
```

### Directory walk logic (claudemd.ts:850-934)

The walk starts from CWD and collects ancestor directories up to root:
```typescript
let currentDir = originalCwd
while (currentDir !== parse(currentDir).root) {
  dirs.push(currentDir)
  currentDir = dirname(currentDir)
}
// Process from root downward to CWD
for (const dir of dirs.reverse()) { ... }
```

This reverse order ensures root directories are loaded first (lower priority) and CWD directories last (higher priority).

### Git worktree deduplication (claudemd.ts:867-884)

```typescript
const isNestedWorktree =
  gitRoot !== null &&
  canonicalRoot !== null &&
  normalizePathForComparison(gitRoot) !== normalizePathForComparison(canonicalRoot) &&
  pathInWorkingPath(gitRoot, canonicalRoot)

// Skip checked-in files from main repo's working tree when in nested worktree
const skipProject =
  isNestedWorktree &&
  pathInWorkingPath(dir, canonicalRoot) &&
  !pathInWorkingPath(dir, gitRoot)
```

### Setting source gating (claudemd.ts:826, 887, 923)

Each memory type respects its corresponding setting source:
- User: `isSettingSourceEnabled('userSettings')` (line 826)
- Project: `isSettingSourceEnabled('projectSettings')` (line 887)
- Local: `isSettingSourceEnabled('localSettings')` (line 923)
- Managed: always loaded (not gated)

---

## 2. File Processing Pipeline

### `processMemoryFile()` (claudemd.ts:618-685)

Per-file processing with cycle detection and depth limiting:

```
1. Normalize path for comparison (handles Windows drive letter casing)
2. Check processedPaths set (cycle prevention)
3. Check claudeMdExcludes patterns (isClaudeMdExcluded)
4. Resolve symlinks (safeResolvePath)
5. Read file async (safelyReadMemoryFileAsync)
6. If parent specified, set parent field
7. Process @include references recursively (depth < MAX_INCLUDE_DEPTH=5)
8. Return [mainFile, ...includedFiles]
```

### `parseMemoryFileContent()` (claudemd.ts:343-400)

Pure function handling content transformation:

```
1. Check file extension against TEXT_FILE_EXTENSIONS whitelist (96-227)
2. Parse frontmatter paths (parseFrontmatterPaths)
3. Lex markdown with marked (gfm:false for @include compatibility)
4. Strip HTML comments (if content contains '<!--')
5. Extract @include paths from tokens (if includeBasePath provided)
6. Truncate AutoMem/TeamMem entrypoints (truncateEntrypointContent)
7. Return MemoryFileInfo with contentDiffersFromDisk flag
```

### `MemoryFileInfo` type (claudemd.ts:229-243)

```typescript
type MemoryFileInfo = {
  path: string                    // Absolute file path
  type: MemoryType                // User, Project, Local, Managed, AutoMem, TeamMem
  content: string                 // Processed content (comments stripped, frontmatter removed)
  parent?: string                 // Path of the file that @included this one
  globs?: string[]                // Glob patterns from frontmatter paths (conditional rules)
  contentDiffersFromDisk?: boolean // True when auto-injection modified content
  rawContent?: string             // Original disk bytes when contentDiffersFromDisk=true
}
```

---

## 3. Frontmatter Parsing

### File: `src/utils/frontmatterParser.ts`

### `parseFrontmatter()` (line 130)

Regex-based extraction: `FRONTMATTER_REGEX = /^---\s*\n([\s\S]*?)---\s*\n?/`

The parser uses `parseYaml()` with a fallback that quotes problematic YAML values (glob patterns with `{}[]` characters). This means frontmatter like:

```yaml
---
paths: src/**/*.{ts,tsx}
---
```

Works correctly even though `{ts,tsx}` would normally break YAML parsing.

### `splitPathInFrontmatter()` (line 189)

Handles comma-separated path patterns with brace-awareness:
- Commas inside braces are NOT treated as separators
- Brace patterns are expanded: `{a,b}/{c,d}` becomes `a/c`, `a/d`, `b/c`, `b/d`
- Accepts both comma-separated strings and YAML arrays

### `parseFrontmatterPaths()` (claudemd.ts:254-279)

Strips `/**` suffixes from patterns (the `ignore` library treats `path` as matching both the path itself and everything inside it), and filters out `**` match-all patterns.

---

## 4. @include Directive Resolution

### `extractIncludePathsFromTokens()` (claudemd.ts:451-535)

Processes pre-lexed markdown tokens to find `@path` references.

**Regex**: `(?:^|\s)@((?:[^\s\\]|\\ )+)` (line 459)

**Rules:**
- Skips `code` and `codespan` tokens entirely
- For `html` tokens: strips comment spans, checks residual text for @paths
- Processes `text` tokens for @path extraction
- Recurses into child `tokens` and `items` (for lists)
- Strips `#fragment` identifiers from paths
- Unescapes `\ ` (escaped spaces) in paths
- Validates paths: must start with `./`, `~/`, `/`, or `[a-zA-Z0-9._-]`
- Resolves relative paths using `expandPath(path, dirname(basePath))`

### External include handling (claudemd.ts:666-670)

```typescript
const isExternal = !pathInOriginalCwd(resolvedIncludePath)
if (isExternal && !includeExternal) {
  continue  // Skip unless user approved external includes
}
```

External includes require approval via `ClaudeMdExternalIncludesDialog` component, stored in project config as `hasClaudeMdExternalIncludesApproved`.

---

## 5. HTML Comment Stripping

### `stripHtmlComments()` (claudemd.ts:292-301)

Uses the `marked` lexer to identify block-level HTML comments only:

```typescript
// Early return if no comment marker
if (!content.includes('<!--')) {
  return { content, stripped: false }
}
return stripHtmlCommentsFromTokens(new Lexer({ gfm: false }).lex(content))
```

### `stripHtmlCommentsFromTokens()` (claudemd.ts:303-334)

Only strips tokens where `token.type === 'html'` AND the trimmed content starts with `<!--` and contains `-->`. This preserves:
- Comments inside code blocks (different token type)
- Inline HTML that isn't a comment
- Unclosed comments (safety measure)

Residual content after comment removal is preserved:
```markdown
<!-- note --> Use bun
```
becomes: ` Use bun`

---

## 6. Exclusion Mechanism

### `isClaudeMdExcluded()` (claudemd.ts:547-573)

Only applies to User, Project, and Local types. Managed files are never excluded.

Uses `picomatch.isMatch()` with `dot: true` option. Patterns come from `settings.claudeMdExcludes`.

### `resolveExcludePatterns()` (claudemd.ts:581-612)

Handles symlink resolution for exclude patterns. For each absolute pattern, resolves the static prefix directory via `realpathSync` and adds the resolved version to the pattern list. This handles cases like `/tmp` -> `/private/tmp` on macOS.

---

## 7. Conditional Rules (Path-based Activation)

### `processMdRules()` (claudemd.ts:697-788)

Processes all `.md` files in `.claude/rules/` directories recursively:
- `conditionalRule: false` -> returns files WITHOUT frontmatter paths (unconditional)
- `conditionalRule: true` -> returns files WITH frontmatter paths (conditional)

### `processConditionedMdRules()` (claudemd.ts:1354-1397)

Filters conditional rules to only include files whose glob patterns match the target path:

```typescript
return ignore().add(file.globs).ignores(relativePath)
```

For Project rules, globs are relative to the directory containing `.claude/`. For Managed/User rules, globs are relative to the original CWD.

### Lazy loading during conversation (attachments.ts:1792-1878)

When Claude touches a file, `getNestedMemoryAttachmentsForFile()` performs:

1. **Phase 1** (line 1809): Load managed and user conditional rules matching the file
2. **Phase 2** (line 1822-1840): Walk directories from CWD to the target file, loading CLAUDE.md + all rules
3. **Phase 3** (line 1843-1857): Walk CWD-level directories (root to CWD), loading conditional rules only

Results are deduplicated via `loadedNestedMemoryPaths` (a non-evicting Set) and `readFileState` (a 100-entry LRU cache).

---

## 8. System Prompt Injection

### `getUserContext()` (context.ts:155-189)

```typescript
const claudeMd = shouldDisableClaudeMd
  ? null
  : getClaudeMds(filterInjectedMemoryFiles(await getMemoryFiles()))
setCachedClaudeMdContent(claudeMd || null)  // Cache for yoloClassifier

return {
  ...(claudeMd && { claudeMd }),
  currentDate: `Today's date is ${getLocalISODate()}.`,
}
```

### Disable conditions (context.ts:163-167)

```typescript
const shouldDisableClaudeMd =
  isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_CLAUDE_MDS) ||
  (isBareMode() && getAdditionalDirectoriesForClaudeMd().length === 0)
```

### `getClaudeMds()` (claudemd.ts:1153-1195)

Builds the final string with the instruction prompt header:

```typescript
const MEMORY_INSTRUCTION_PROMPT =
  'Codebase and user instructions are shown below. Be sure to adhere to these instructions. ' +
  'IMPORTANT: These instructions OVERRIDE any default behavior and you MUST follow them exactly as written.'
```

Each file gets a type description annotation:
- Project: `(project instructions, checked into the codebase)`
- Local: `(user's private project instructions, not checked in)`
- User: `(user's private global instructions for all projects)`
- AutoMem: `(user's auto-memory, persists across conversations)`
- TeamMem: `(shared team memory, synced across the organization)` -- wrapped in `<team-memory-content>` tags

### Feature flag filtering (claudemd.ts:1142-1151)

```typescript
// tengu_paper_halyard: skip Project and Local types
if (skipProjectLevel && (file.type === 'Project' || file.type === 'Local')) continue

// tengu_moth_copse: skip AutoMem and TeamMem from system prompt injection
// (they arrive via attachments instead)
```

### System prompt placement (prompts.ts:491-576)

CLAUDE.md content is part of the dynamic (uncached) section of the system prompt:

```typescript
const dynamicSections = [
  systemPromptSection('session_guidance', ...),
  systemPromptSection('memory', () => loadMemoryPrompt()),  // Auto-memory prompt
  systemPromptSection('ant_model_override', ...),
  systemPromptSection('env_info_simple', ...),
  systemPromptSection('language', ...),
  ...
]
```

Note that `loadMemoryPrompt()` is the auto-memory system prompt (MEMORY.md instructions), NOT the CLAUDE.md content. The CLAUDE.md content arrives via `getUserContext()` which is injected separately as a user message context block.

---

## 9. Caching and Invalidation

### Memoization (claudemd.ts:790)

`getMemoryFiles` is memoized with lodash `memoize`. The cache is keyed on the `forceIncludeExternal` boolean parameter.

### Cache clearing

Two functions with different semantics:

```typescript
// Just clears the cache (for correctness)
clearMemoryFileCaches()  // line 1119

// Clears cache AND arms the InstructionsLoaded hook for the next load
resetGetMemoryFilesCache(reason)  // line 1124
```

`clearMemoryFileCaches()` is used for: worktree enter/exit, settings sync, /memory dialog.
`resetGetMemoryFilesCache()` is used for: compaction (with reason='compact'), session start.

### InstructionsLoaded hook firing (claudemd.ts:1054-1071)

The hook fires once per cache miss for non-forceIncludeExternal loads. A one-shot flag (`shouldFireHook`) prevents double-firing. The hook reports each loaded file with its type and load reason.

---

## 10. YOLO Classifier Integration

### File: `src/utils/permissions/yoloClassifier.ts`

### `buildClaudeMdMessage()` (lines ~445-472)

The classifier reads the cached CLAUDE.md content (set by `getUserContext()` via `setCachedClaudeMdContent()`) and wraps it for the classifier model:

```typescript
const claudeMd = getCachedClaudeMdContent()
if (claudeMd === null) return null

// Wrapped in XML tags for the classifier
`<user_claude_md>\n${claudeMd}\n</user_claude_md>`
```

This is injected as a prefix message in the classifier's conversation, meaning CLAUDE.md instructions directly influence auto-permission decisions.

---

## 11. Nested Memory Attachment Flow

### File: `src/utils/attachments.ts`

### `memoryFilesToAttachments()` (line 1710)

Converts `MemoryFileInfo[]` to attachment objects, with deduplication:

```typescript
// Two-layer dedup:
// 1. loadedNestedMemoryPaths: non-evicting Set (survives LRU eviction)
// 2. readFileState: 100-entry LRU (provides cross-function dedup)
if (toolUseContext.loadedNestedMemoryPaths?.has(memoryFile.path)) continue
if (!toolUseContext.readFileState.has(memoryFile.path)) {
  attachments.push({
    type: 'nested_memory',
    path: memoryFile.path,
    content: memoryFile,
    displayPath: relative(getCwd(), memoryFile.path),
  })
}
```

When `contentDiffersFromDisk` is true (frontmatter stripped, comments stripped, or MEMORY.md truncated), the raw disk bytes are cached with `isPartialView: true`. This means Edit/Write tools will still require an explicit Read before modifying these files.

---

## 12. Auto-Memory System (MEMORY.md)

### Path resolution (memdir/paths.ts:223-235)

```
Resolution order:
1. CLAUDE_COWORK_MEMORY_PATH_OVERRIDE (env var, full path override)
2. autoMemoryDirectory in settings.json (trusted sources only: policy/local/user)
3. <memoryBaseDir>/projects/<sanitized-git-root>/memory/
```

### Enable/disable (memdir/paths.ts:30-55)

Priority chain:
1. `CLAUDE_CODE_DISABLE_AUTO_MEMORY` env var
2. `CLAUDE_CODE_SIMPLE` (--bare) mode
3. CCR without persistent storage
4. `autoMemoryEnabled` in settings.json
5. Default: enabled

### Entrypoint truncation (memdir/memdir.ts:57-100)

MEMORY.md is capped at 200 lines and 25,000 bytes. Truncation appends a warning message guiding the model to keep entries concise.

### Memory prompt in system prompt (memdir/memdir.ts:419-485)

`loadMemoryPrompt()` dispatches based on enabled memory systems:
- KAIROS mode: daily-log append-only paradigm
- TEAMMEM + auto: combined prompt with private/team directories
- Auto only: single-directory memory prompt
- Disabled: returns null

The prompt includes instructions for memory types (user, feedback, project, reference), what not to save, when to access, and how to trust recalled information.

---

## 13. Text File Extension Whitelist

### `TEXT_FILE_EXTENSIONS` (claudemd.ts:96-227)

A comprehensive Set of ~100 extensions organized by category:
- Markdown/text: `.md`, `.txt`, `.text`
- Data formats: `.json`, `.yaml`, `.yml`, `.toml`, `.xml`, `.csv`
- Web: `.html`, `.css`, `.scss`, etc.
- Programming languages: `.js`, `.ts`, `.py`, `.go`, `.rs`, `.java`, etc.
- Config: `.env`, `.ini`, `.cfg`, `.conf`, etc.
- Build: `.cmake`, `.make`, `.gradle`, `.sbt`
- Documentation: `.rst`, `.adoc`, `.org`, `.tex`

Any file with an extension not in this set is silently skipped during @include processing, preventing binary files from entering the prompt.

---

## 14. Key Constants

| Constant | Value | Location |
|---|---|---|
| `MAX_MEMORY_CHARACTER_COUNT` | 40,000 | claudemd.ts:93 |
| `MAX_INCLUDE_DEPTH` | 5 | claudemd.ts:537 |
| `MAX_ENTRYPOINT_LINES` | 200 | memdir.ts:36 |
| `MAX_ENTRYPOINT_BYTES` | 25,000 | memdir.ts:38 |
| `MEMORY_INSTRUCTION_PROMPT` | "Codebase and user instructions..." | claudemd.ts:89 |
| `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` | `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__` | prompts.ts:114 |

---

## 15. Error Handling

### `handleMemoryFileReadError()` (claudemd.ts:402-416)

- `ENOENT` (file doesn't exist): silently ignored (expected for probed paths)
- `EISDIR` (is a directory): silently ignored
- `EACCES` (permission denied): logged as analytics event with metadata about whether it's in the config home dir

### Frontmatter parse failure (frontmatterParser.ts:153-168)

Two-pass parsing: first with raw YAML, then with problematic values auto-quoted. If both fail, logs a warning and returns empty frontmatter (file content still loaded without frontmatter stripping).

### Corrupted config recovery (config.ts:1421-1585)

The global config file (`~/.claude.json`) has extensive corruption recovery: backup creation, backup restoration suggestions, and guard against auth state loss during concurrent writes.
