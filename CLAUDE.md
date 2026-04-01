# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

This is an **unofficial research reconstruction** of `@anthropic-ai/claude-code` v2.1.88, extracted from the npm package's source map (`cli.js.map`). It contains 4,756 restored files including 1,884 TypeScript/TSX source files. This is a **read-only research archive** — there is no build system, test suite, or development workflow. All source code copyright belongs to Anthropic.

## Repository Layout

```
package/              # Original npm package contents
  cli.js              # Compiled 13MB single-file binary
  cli.js.map          # 58MB source map (extraction source)
  sdk-tools.d.ts      # TypeScript SDK type definitions
  vendor/             # Bundled binaries (ripgrep, audio-capture)
restored-src/src/     # Reconstructed TypeScript source tree
```

## Architecture

Claude Code is a terminal-based AI coding assistant built with **TypeScript**, **React**, and **Ink** (terminal rendering). The compiled output is a single Node.js ES module (`cli.js`).

### Core layers (in `restored-src/src/`):

- **`main.tsx`** — Monolithic CLI entry point (~809KB). Handles startup, auth, feature flags, plugin/skill initialization, and the main REPL loop.
- **`commands.ts` + `commands/`** — Command registry and 100+ slash-command implementations (e.g., `/commit`, `/review`, `/plan`, `/config`).
- **`tools/`** — 40+ tool implementations, each in its own directory. These are the capabilities the AI agent can invoke (BashTool, FileEditTool, GlobTool, GrepTool, AgentTool, MCPTool, WebFetchTool, TaskCreateTool, TeamCreateTool, etc.).
- **`services/`** — Backend services: Anthropic API client (`api/`), MCP server management (`mcp/`), OAuth (`oauth/`), analytics (`analytics/`), memory persistence (`SessionMemory/`), context compaction (`compact/`), MDM settings (`remoteManagedSettings/`).
- **`components/`** — React/Ink UI components for terminal rendering: messages, diffs, task management, settings panels, onboarding wizard.
- **`utils/`** — ~90 utility modules covering git operations (`git/`), model selection (`model/`), permissions (`permissions/`), shell execution (`bash/`, `shell/`), configuration (`config.ts`), task management (`task/`), skill loading (`skills/`), and multi-agent coordination (`swarm/`).

### Feature subsystems:

- **`coordinator/`** — Multi-agent orchestration (COORDINATOR_MODE)
- **`assistant/`** — KAIROS assistant mode (feature-gated)
- **`plugins/` + `skills/`** — Plugin and skill systems with bundled defaults
- **`remote/`** — Remote/distributed session handling
- **`voice/`** — Voice interaction (STT/TTS)
- **`vim/`** — Vim keybinding mode
- **`ink/`** — Custom terminal UI framework layer (components, hooks, I/O abstractions)
- **`entrypoints/`** — Alternative entry points for SDK usage

### Key patterns:

- Each tool in `tools/` follows a consistent structure: tool definition, parameter schema, and execution logic in its own directory.
- The permission system (`utils/permissions/`) gates tool execution based on user-configured policies.
- State management uses React Context (`context/`, `context.ts`) combined with global state modules (`state/`).
- The system prompt and conversation context are assembled dynamically based on environment, tools, plugins, and skills.
