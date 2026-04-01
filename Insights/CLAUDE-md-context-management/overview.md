# CLAUDE.md Context Management in Claude Code CLI

## Overview

CLAUDE.md is the primary mechanism for users to inject persistent instructions into every Claude Code conversation. It acts as a "memory file" that Claude reads at session start and treats as high-priority directives. The system supports a layered hierarchy of instruction files, from enterprise-managed policies down to personal local overrides, all merged into the system prompt with explicit priority ordering.

The key design principle: **later-loaded files have higher priority** -- the model pays more attention to instructions that appear later in the prompt. This means local overrides beat project rules, which beat user-global rules, which beat managed/enterprise rules.

---

## 1. Discovery: How CLAUDE.md Files Are Found

### Hierarchy (loaded in this order, lowest priority first)

| Layer | Path(s) | Type | Checked In? | Purpose |
|---|---|---|---|---|
| **Managed** | `/etc/claude-code/CLAUDE.md` + `/etc/claude-code/.claude/rules/*.md` | `Managed` | N/A | Enterprise/admin-mandated instructions for all users |
| **User** | `~/.claude/CLAUDE.md` + `~/.claude/rules/*.md` | `User` | No | Personal global instructions across all projects |
| **Project** | `CLAUDE.md`, `.claude/CLAUDE.md`, `.claude/rules/*.md` in each directory from root to CWD | `Project` | Yes | Project-level instructions checked into the repo |
| **Local** | `CLAUDE.local.md` in each directory from root to CWD | `Local` | No (gitignored) | Private project-specific overrides |
| **AutoMem** | `~/.claude/projects/<sanitized-path>/memory/MEMORY.md` | `AutoMem` | No | Auto-generated persistent memory index |
| **TeamMem** | (team memory directory, feature-flagged) | `TeamMem` | No | Shared team memory synced across org |

### Directory Traversal

For Project and Local types, discovery walks **upward** from CWD to the filesystem root, collecting files from each directory. The files are then injected in **root-to-CWD order**, so files closer to CWD (more specific) have higher priority.

For example, in `/home/user/myorg/myproject/src/`, it would look for:
1. `/home/user/myorg/CLAUDE.md` (loaded first, lowest project priority)
2. `/home/user/myorg/myproject/CLAUDE.md`
3. `/home/user/myorg/myproject/src/CLAUDE.md` (loaded last, highest project priority)

### Additional Directories

The `--add-dir` CLI flag adds extra directories whose CLAUDE.md files are included. This is controlled by the `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD` environment variable.

### Git Worktree Deduplication

When running from a git worktree nested inside the main repo (e.g., `.claude/worktrees/<name>/`), the system detects the overlap and skips Project-type files from the main repo's working tree above the worktree, since the worktree already has its own checkout of those files.

---

## 2. Loading and Parsing

### File Reading

Each candidate path is read asynchronously (`safelyReadMemoryFileAsync`). Missing files (ENOENT) and directories (EISDIR) are silently ignored -- this allows the system to probe many paths without errors. Permission errors (EACCES) are logged for diagnostics.

### Frontmatter Parsing

Files can have YAML frontmatter delimited by `---`. The parser extracts:

- **`paths`**: Glob patterns that restrict when this rule file applies. Only files with frontmatter `paths` that match the current target file are included (conditional rules). Patterns support brace expansion (e.g., `src/*.{ts,tsx}`).

Example:
```markdown
---
paths: src/**/*.ts, lib/**/*.js
---
Only apply these rules when editing TypeScript or JavaScript files.
```

### HTML Comment Stripping

Block-level HTML comments (`<!-- ... -->`) are stripped from the content before injection. This allows authors to leave notes in CLAUDE.md files that Claude never sees. Comments inside code blocks and inline code spans are preserved. Unclosed comments are left intact to avoid silently swallowing content.

### @include Directives

CLAUDE.md files can include other files using `@` notation:

```markdown
@./relative/path.md
@~/home-relative/path.md
@/absolute/path.md
@path-without-prefix.md  (treated as relative)
```

Rules:
- Works only in text nodes (not inside code blocks or code spans)
- Included files are processed recursively up to **5 levels deep**
- Circular references are prevented by tracking processed paths
- Non-existent files are silently ignored
- External includes (outside project CWD) require explicit user approval via a dialog
- Only text file extensions are allowed (a comprehensive whitelist of ~100 extensions prevents binary files from being loaded)
- `@path#heading` fragment identifiers are stripped before resolution

### Content Limits

- Recommended max: **40,000 characters** per memory file
- MEMORY.md entrypoints: max **200 lines** and **25,000 bytes** (truncated with warning if exceeded)

---

## 3. Merging and Priority

### Merge Strategy

There is no complex merge logic. All loaded files are concatenated into a single prompt block with headers identifying each file. The priority system relies on the well-known LLM recency bias: content placed later in the prompt receives more attention.

### The Final Prompt Block

The `getClaudeMds()` function builds the merged content:

```
Codebase and user instructions are shown below. Be sure to adhere to these instructions.
IMPORTANT: These instructions OVERRIDE any default behavior and you MUST follow them exactly as written.

Contents of /etc/claude-code/CLAUDE.md (user's private global instructions for all projects):
[content]

Contents of /home/user/project/CLAUDE.md (project instructions, checked into the codebase):
[content]

Contents of /home/user/project/CLAUDE.local.md (user's private project instructions, not checked in):
[content]
```

Each file is annotated with its type description in parentheses so the model understands the source and scope.

---

## 4. Injection into System Prompt

### Two-Phase Injection

CLAUDE.md content enters the conversation through two mechanisms:

**Phase 1: Eager Loading (Session Start)**
The `getUserContext()` function in `context.ts` calls `getMemoryFiles()` then `getClaudeMds()` to build a `claudeMd` string. This is included in the system prompt's dynamic section as a user context block containing all unconditional memory files.

**Phase 2: Lazy Loading (During Conversation)**
When Claude touches files (via Read, Edit, Write tools), the `attachments.ts` system loads:
- **Conditional rules**: `.claude/rules/*.md` files with `paths:` frontmatter that match the file being accessed
- **Nested memory**: CLAUDE.md files from directories between CWD and the target file that weren't in the initial load

These arrive as `nested_memory` attachments on tool results, injected just-in-time.

### System Prompt Architecture

The system prompt has a static/dynamic boundary marked by `SYSTEM_PROMPT_DYNAMIC_BOUNDARY`. The CLAUDE.md content sits in the dynamic section (after the boundary) via the `memory` system prompt section, which calls `loadMemoryPrompt()`.

The auto-memory prompt (from `memdir/memdir.ts`) is a separate section that includes the MEMORY.md index plus instructions for how the model should read/write to the memory system.

---

## 5. Special Syntax and Sections

### No Special Markdown Sections

CLAUDE.md does **not** have special section syntax like `# commands` or `# rules`. It is treated as free-form markdown. The entire content is injected as-is (after frontmatter stripping and HTML comment removal). Any markdown structure is simply passed to the model, which interprets it naturally.

### Frontmatter Fields (for .claude/rules/*.md)

| Field | Purpose |
|---|---|
| `paths` | Glob patterns restricting when the rule applies |
| `description` | One-line description for display |
| `allowed-tools` | Tool allowlist (for agents/skills) |
| `model` | Model override |
| `hooks` | Hook definitions |
| `context` | Execution context (inline/fork) |
| `shell` | Shell for command blocks |

### HTML Comments as Author Notes

```markdown
<!-- This is a note for the CLAUDE.md author that Claude will never see -->
Use bun instead of npm for all package management.
```

### @include for Modular Instructions

```markdown
# Project Rules
@./coding-standards.md
@./testing-requirements.md
@./deployment-guidelines.md
```

---

## 6. Interaction with Other Config Sources

### settings.json

- **`settingSources`**: Controls which memory layers are loaded. If `projectSettings` is disabled, no Project-type CLAUDE.md files are loaded. If `userSettings` is disabled, the user-level `~/.claude/CLAUDE.md` is skipped.
- **`claudeMdExcludes`**: Array of glob patterns to exclude specific CLAUDE.md files from loading. Only affects User, Project, and Local types (Managed/policy files cannot be excluded).
- **`autoMemoryEnabled`**: Controls whether the auto-memory system (MEMORY.md) is active.
- **`autoMemoryDirectory`**: Custom path for auto-memory storage.

### .claude/ Directory

The `.claude/` directory in a project serves multiple purposes:
- `.claude/CLAUDE.md` -- alternative location for project instructions (equivalent to root CLAUDE.md)
- `.claude/rules/*.md` -- granular rule files with optional conditional `paths:` frontmatter
- `.claude/settings.json` -- project settings (controls permissions, tools, etc.)
- `.claude/worktrees/` -- git worktree session data

### Permission System (YOLO/Auto-mode Classifier)

The YOLO permission classifier reads CLAUDE.md content to make auto-approval decisions. It wraps the content in `<user_claude_md>` tags and passes it to a smaller model (Haiku) that determines if a tool action aligns with user instructions. This means CLAUDE.md content directly influences which tool calls get auto-approved.

### Hooks System

The `InstructionsLoaded` hook event fires whenever a CLAUDE.md or rules file is loaded into context. It reports:
- `file_path`: The instruction file path
- `memory_type`: User, Project, Local, or Managed
- `load_reason`: `session_start`, `compact`, `include`, `path_glob_match`, or `nested_traversal`
- Optional: `globs`, `trigger_file_path`, `parent_file_path`

This enables enterprise audit trails and custom processing of instruction files.

### Environment Variables

| Variable | Effect |
|---|---|
| `CLAUDE_CODE_DISABLE_CLAUDE_MDS` | Hard disable all CLAUDE.md loading |
| `CLAUDE_CODE_SIMPLE` (--bare mode) | Skip CLAUDE.md auto-discovery (but honors --add-dir) |
| `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD` | Enable CLAUDE.md loading from --add-dir directories |
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY` | Disable auto-memory (MEMORY.md) |
| `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` | Override auto-memory directory path |

---

## 7. Practical Tips for Using CLAUDE.md Effectively

### For Coding Tasks

1. **Project-level CLAUDE.md** (checked in): Put build commands, test commands, coding standards, preferred libraries, and architectural decisions here. Every team member benefits.
   ```markdown
   # Build & Test
   - Use `bun test` to run tests
   - Use `bun run build` to build

   # Code Style
   - Prefer functional style over classes
   - Use TypeScript strict mode
   - All API routes use kebab-case
   ```

2. **Conditional rules** (`.claude/rules/*.md` with `paths:` frontmatter): Target specific file types or directories.
   ```markdown
   ---
   paths: src/api/**/*.ts
   ---
   All API handlers must validate input with zod schemas.
   Always return proper HTTP status codes.
   ```

3. **Local overrides** (`CLAUDE.local.md`): Personal preferences that shouldn't be imposed on the team.
   ```markdown
   Run tests before every commit.
   Use verbose git commit messages.
   I prefer detailed code comments.
   ```

### For Documentation Handling

1. Use `@include` to reference style guides: `@./docs/style-guide.md`
2. Keep CLAUDE.md under 40K characters; split into `.claude/rules/` files if needed
3. Use HTML comments for internal notes that Claude shouldn't see

### For Daily Jobs and Workflows

1. **Auto-memory** handles cross-session learning automatically -- it remembers user preferences, project context, and corrections
2. **The `/remember` skill** (ant-only) reviews memory layers and proposes promotions from auto-memory to CLAUDE.md or CLAUDE.local.md
3. **Conditional rules with glob paths** activate only when relevant files are touched, keeping the prompt lean for unrelated work

### Architecture Best Practices

- **Keep CLAUDE.md concise**: The model has limited attention; shorter, more focused instructions work better
- **Use the hierarchy**: Global rules in `~/.claude/CLAUDE.md`, project rules in `CLAUDE.md`, personal rules in `CLAUDE.local.md`
- **Conditional rules scale**: Instead of one giant CLAUDE.md, use `.claude/rules/` with `paths:` frontmatter for domain-specific rules
- **Don't duplicate what's in code**: CLAUDE.md should contain instructions, not code patterns (those are derivable from the codebase)
- **Frontmatter paths support brace expansion**: `paths: src/**/*.{ts,tsx}` works correctly

---

## 8. Relationships Diagram

```
                    Enterprise Admin
                         |
                    /etc/claude-code/CLAUDE.md  (Managed)
                    /etc/claude-code/.claude/rules/*.md
                         |
                    ~/.claude/CLAUDE.md  (User)
                    ~/.claude/rules/*.md
                         |
              +----------+-----------+
              |                      |
    CLAUDE.md (Project)     CLAUDE.local.md (Local)
    .claude/CLAUDE.md
    .claude/rules/*.md
    [per directory, root->CWD]
              |
              +--- @include directives --> external files
              |
              +--- paths: frontmatter --> conditional activation
              |
    +---------+---------+
    |                   |
System Prompt      YOLO Classifier
(getUserContext)   (auto-approvals)
    |                   |
    +--- loadMemoryPrompt() --> auto-memory (MEMORY.md)
    |
    +--- attachments.ts --> nested/conditional rules (lazy)
    |
    +--- InstructionsLoaded hooks --> audit/enterprise
```
