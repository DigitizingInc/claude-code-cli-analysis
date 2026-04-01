# Claude Code Architecture Documentation

Comprehensive architectural analysis of the Claude Code codebase. Each section covers a distinct subsystem with its design decisions, key files, and interaction patterns.

## Sections

| # | Section | Description |
|---|---------|-------------|
| 00 | [Overview](00-overview.md) | High-level architecture, technology stack, key design decisions, repository statistics |
| 01 | [Entry Points & Bootstrap](01-entry-and-bootstrap.md) | CLI entry, initialization sequence, deferred prefetching, session lifecycle |
| 02 | [Query Engine](02-query-engine.md) | Core execution pipeline, API integration, token budgeting, retry strategy |
| 03 | [Tool System](03-tool-system.md) | Tool abstraction, 44 built-in tools, concurrency model, BashTool and AgentTool deep dives |
| 04 | [Commands & Skills](04-commands-and-skills.md) | 89 slash commands, skill system, loading pipeline, bundled skills |
| 05 | [Permissions & Security](05-permissions-and-security.md) | Permission model, security layers, hook-based enforcement, enterprise policies |
| 06 | [Terminal Renderer](06-terminal-renderer.md) | Custom React reconciler, flexbox layout engine, ANSI protocol, performance optimizations |
| 07 | [MCP Integration](07-mcp-integration.md) | MCP client/server, 7 transports, authentication, tool normalization |
| 08 | [Agent Coordination](08-agent-coordination.md) | Sub-agents, fork optimization, worktree isolation, multi-agent teams |
| 09 | [State Management](09-state-management.md) | AppState, session persistence, file caching, React context, memory system |
| 10 | [Build System](10-build-system.md) | esbuild pipeline, feature flags, MACRO constants, Docker, CI/CD |
| 11 | [Web Frontend](11-web-frontend.md) | Next.js app, browser terminal, collaboration, API routes |
| 12 | [Configuration](12-configuration.md) | Settings hierarchy, environment variables, hot-reloading, CLAUDE.md |
| 13 | [Services & Integrations](13-services-and-integrations.md) | API client, OAuth, LSP, telemetry, plugins, voice, bridge, monitoring |

## Reading Guide

**Start here:** [00-overview.md](00-overview.md) for the big picture.

**Core runtime:** Sections 01-03 cover the execution spine (bootstrap → query engine → tool execution).

**Extension points:** Sections 04, 07, 13 cover how to extend Claude Code (commands, skills, MCP, plugins).

**Security model:** Section 05 is essential for understanding how Claude Code manages trust and permissions.

**UI internals:** Section 06 covers the custom terminal renderer; Section 11 covers the web frontend.

**Infrastructure:** Sections 10, 12 cover build, configuration, and deployment.
