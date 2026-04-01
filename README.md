# Claude Code CLI — Architectural Analysis

A comprehensive reverse-engineering and architectural documentation of **Claude Code**, Anthropic's official CLI tool for interacting with Claude from the terminal.

Claude Code is a production-grade TypeScript application running on the Bun runtime — ~512,000 lines of source across ~1,900 files, shipped as a single ESM bundle built with esbuild.

---

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

## Documentation Map

| Document | Description |
|----------|-------------|
| [architecture.md](architecture.md) | Core architecture overview — pipeline, query engine, tool system, permissions, UI |
| [architecture/](architecture/) | 14-part deep-dive series covering every major subsystem |
| [tools.md](tools.md) | Complete catalog of all ~40 agent tools |
| [commands.md](commands.md) | Complete catalog of all slash commands |
| [subsystems.md](subsystems.md) | Detailed docs for bridge, MCP, permissions, plugins, skills, tasks, memory, voice |
| [bridge.md](bridge.md) | Deep-dive into VS Code / JetBrains IDE integration layer |
| [exploration-guide.md](exploration-guide.md) | How to navigate and study the source code |

## Architecture Deep-Dive Series

The [`architecture/`](architecture/) directory contains a 14-part series:

| # | Section | Topics |
|---|---------|--------|
| 00 | [Overview](architecture/00-overview.md) | Tech stack, design decisions, repo stats |
| 01 | [Entry & Bootstrap](architecture/01-entry-and-bootstrap.md) | CLI entry, init sequence, session lifecycle |
| 02 | [Query Engine](architecture/02-query-engine.md) | LLM API calls, token budgeting, retry strategy |
| 03 | [Tool System](architecture/03-tool-system.md) | 44 built-in tools, concurrency model, BashTool/AgentTool |
| 04 | [Commands & Skills](architecture/04-commands-and-skills.md) | 89 slash commands, skill system, bundled skills |
| 05 | [Permissions & Security](architecture/05-permissions-and-security.md) | Permission model, hooks, enterprise policies |
| 06 | [Terminal Renderer](architecture/06-terminal-renderer.md) | Custom React reconciler, flexbox, ANSI protocol |
| 07 | [MCP Integration](architecture/07-mcp-integration.md) | MCP client/server, 7 transports, tool normalization |
| 08 | [Agent Coordination](architecture/08-agent-coordination.md) | Sub-agents, fork optimization, worktree isolation |
| 09 | [State Management](architecture/09-state-management.md) | AppState, session persistence, memory system |
| 10 | [Build System](architecture/10-build-system.md) | esbuild pipeline, feature flags, Docker, CI/CD |
| 11 | [Web Frontend](architecture/11-web-frontend.md) | Next.js app, browser terminal, collaboration |
| 12 | [Configuration](architecture/12-configuration.md) | Settings hierarchy, env vars, hot-reloading, CLAUDE.md |
| 13 | [Services & Integrations](architecture/13-services-and-integrations.md) | OAuth, LSP, telemetry, plugins, voice, bridge |

## Key Design Patterns

- **React for the terminal** — The entire UI is built with React + Ink, making it a fully reactive CLI with components, hooks, and state management
- **Tool-call loops** — The query engine executes LLM-requested tools and feeds results back in a loop until the model is done
- **Permission layering** — Enterprise MDM policies > project settings > user settings, with hook-based enforcement
- **Single-bundle shipping** — Everything compiles to one `dist/cli.mjs` via esbuild
- **Multi-provider API** — Supports Direct API, AWS Bedrock, and Google Vertex backends

## License

[MIT](LICENSE) — Copyright (c) 2026 DigitizingInc
