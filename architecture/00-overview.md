# Claude Code Architecture Overview

## What Is Claude Code?

Claude Code is Anthropic's official CLI tool for interacting with Claude directly from the terminal. It is a production-grade TypeScript application running on the Bun runtime, spanning ~512,000 lines of source across ~1,900 files. It ships as a single ESM bundle (`dist/cli.mjs`) built with esbuild.

## High-Level Architecture

```
                          CLI Entry (cli.tsx)
                               |
                          Initialization
                        (init.ts + main.tsx)
                               |
                 +-------------+-------------+
                 |                           |
           Interactive REPL            Non-Interactive
          (React/Ink TUI)            (--print pipeline)
                 |                           |
                 +-------------+-------------+
                               |
                         Query Engine
                      (QueryEngine.ts)
                               |
              +----------------+----------------+
              |                |                |
        Claude API       Tool Execution    Context Assembly
       (services/api)   (services/tools)   (context.ts)
              |                |
              |     +----------+----------+
              |     |          |          |
              |  Built-in   MCP Tools  Plugin Tools
              |   Tools    (MCPTool)   (plugins/)
              |  (tools/)
              |
        +-----+------+
        |      |      |
      Direct  Bedrock  Vertex
       API     AWS    Google
```

## Core Modules

| Module | Path | Purpose |
|--------|------|---------|
| Entry Point | `src/entrypoints/cli.tsx` | Bootstrap, flag parsing, early init |
| Main Application | `src/main.tsx` | CLI parser, React renderer, REPL loop |
| Query Engine | `src/QueryEngine.ts` | Orchestrates LLM API calls and tool execution |
| Query Pipeline | `src/query.ts` | Streaming, token budget, compaction, retry |
| Tool System | `src/Tool.ts` + `src/tools/` | 44 built-in tools with schema validation |
| Command System | `src/commands.ts` + `src/commands/` | 89 slash commands |
| Skill System | `src/skills/` | Reusable prompt workflows |
| Terminal UI | `src/ink/` | Custom React reconciler for terminal rendering |
| API Client | `src/services/api/` | Anthropic SDK integration, multi-cloud |
| MCP Client | `src/services/mcp/` | Model Context Protocol integration |
| State | `src/state/` | Immutable AppState tree |
| Hooks | `src/hooks/` | React hooks for permissions, keybindings, settings |
| Components | `src/components/` | 140+ Ink/React UI components |
| Bridge | `src/bridge/` | IDE integration (VS Code, JetBrains) |
| Web Frontend | `web/` | Next.js web interface |
| Build | `scripts/` | esbuild bundler configuration |

## Technology Stack

- **Language:** TypeScript (strict mode, ES modules)
- **Runtime:** Bun (>=1.1.0)
- **Bundler:** esbuild (single-file ESM output)
- **UI Framework:** Custom React reconciler (Ink-style terminal rendering)
- **API Client:** `@anthropic-ai/sdk`
- **Multi-Cloud:** AWS Bedrock, Google Vertex AI, Azure Foundry
- **Protocols:** MCP (Model Context Protocol), LSP (Language Server Protocol)
- **Database:** Drizzle ORM (PostgreSQL + SQLite dual-dialect)
- **Validation:** Zod
- **Telemetry:** OpenTelemetry, Sentry
- **Analytics:** GrowthBook feature flags
- **Testing:** Vitest (unit/integration), Playwright (E2E)
- **Linting:** Biome
- **Web Frontend:** Next.js 14, Radix UI, zustand, xterm.js

## Key Design Decisions

1. **Single-bundle distribution.** The entire CLI compiles to one `.mjs` file via esbuild. Native addons (node-pty, better-sqlite3) are externalized. This simplifies distribution through npm while keeping the dependency graph manageable.

2. **Custom terminal renderer.** Rather than using the `ink` npm package, Claude Code ships its own React reconciler with a full flexbox layout engine (ported from yoga-layout), ANSI protocol handling, object pooling, and frame diffing. This gives complete control over rendering performance and terminal capabilities.

3. **Tool-first architecture.** Every capability the model can invoke is a `Tool` with a Zod input schema, permission metadata, concurrency classification, and progress streaming. This uniform interface enables consistent permission checking, telemetry, and error handling.

4. **Compile-time feature gating.** The `feature()` function from `bun:bundle` enables dead-code elimination at build time. Internal-only features (KAIROS, PROACTIVE, BRIDGE_MODE, etc.) are stripped from external builds, keeping the public bundle lean.

5. **Multi-source configuration.** Settings merge from 5+ sources (user, project, local, CLI flags, enterprise policy) with clear precedence rules. Enterprise policies cannot be overridden by users.

6. **Permission-first security model.** Every tool invocation passes through a permission check that evaluates wildcard rules from multiple sources. The system supports plan mode, auto mode (with ML classifier), and bubble mode (delegate to parent).

## Repository Statistics

| Metric | Value |
|--------|-------|
| Source files | ~1,900 |
| Lines of code | ~512,000 |
| Built-in tools | 44 |
| Slash commands | 89 |
| UI components | 140+ |
| Services | 23+ |
| Feature flags | 25+ |
