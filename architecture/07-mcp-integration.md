# MCP (Model Context Protocol) Integration

## Overview

Claude Code is both an MCP client (connecting to external MCP servers for additional tools) and an MCP server (exposing its own tools to other clients). The MCP integration is one of the most complex subsystems, spanning ~300 KB of source across authentication, connection management, tool normalization, and configuration parsing.

## Architecture

```
src/services/mcp/
  ├── client.ts          Main MCP client (119 KB)
  ├── auth.ts            OAuth/XAA authentication (88 KB)
  ├── config.ts          Configuration parsing (51 KB)
  ├── types.ts           Type definitions (7 KB)
  └── useManageMCPConnections.ts  React hook for lifecycle (44 KB)

mcp-server/              Claude Code as MCP server
  └── dist/index.js      Server implementation
```

## MCP Server Transports

Claude Code supports 7 transport types for connecting to MCP servers:

```typescript
type McpServerConfig =
  | McpStdioServerConfig       // Spawn process, communicate via stdin/stdout
  | McpSSEServerConfig         // Server-Sent Events over HTTP
  | McpHTTPServerConfig        // HTTP request/response
  | McpWebSocketServerConfig   // WebSocket connection
  | McpSdkServerConfig         // In-process SDK integration
  // Internal variants:
  | McpSSEIdeServerConfig      // IDE extension SSE
  | McpWSIdeServerConfig       // IDE extension WebSocket
  | McpClaudeAIProxyConfig     // Claude.ai proxy
```

## Connection States

```typescript
type MCPServerConnection =
  | ConnectedMCPServer    // Active, tools available
  | FailedMCPServer       // Connection failed (with error)
  | NeedsAuthMCPServer    // Requires authentication
  | PendingMCPServer      // Connecting...
  | DisabledMCPServer     // Explicitly disabled
```

## Configuration Sources

MCP servers are configured from multiple sources, merged in priority order:

```
1. Local:    ./.claude/mcp.json (project-specific, gitignored)
2. Project:  ./.claude/mcp.json (shared via git)
3. User:     ~/.claude/mcp.json (global)
4. Policy:   Enterprise managed settings
5. Claude.ai: Proxy servers (when authenticated via claude.ai)
6. IDE:      VS Code/JetBrains extension servers
```

### Configuration Format

```json
{
  "mcpServers": {
    "server-name": {
      "command": "node",
      "args": ["path/to/server.js"],
      "env": {
        "API_KEY": "..."
      }
    },
    "remote-server": {
      "url": "https://mcp.example.com",
      "transport": "sse",
      "headers": {
        "Authorization": "Bearer $API_TOKEN"
      }
    }
  }
}
```

### Environment Variable Interpolation

Headers support `$VAR` syntax for environment variable interpolation, with allowlisting via `allowedEnvVars` in enterprise policies.

## Client Connection Flow

```
1. Parse configuration from all sources
2. For each configured server:
   │
   ├── Check policy allowlist/denylist
   │   └── Enterprise can restrict which servers are available
   │
   ├── Resolve transport type
   │
   ├── Check if authentication needed
   │   ├── OAuth flow (browser-based)
   │   └── XAA (Cross-App Access) flow
   │
   ├── Establish connection
   │   ├── stdio: Spawn child process
   │   ├── sse: Open SSE connection
   │   ├── http: Send initialization request
   │   ├── ws: Open WebSocket
   │   └── sdk: Initialize in-process
   │
   ├── Discover tools
   │   ├── Call tools/list
   │   ├── Normalize tool names (prefix with server name)
   │   └── Register as MCPTool instances
   │
   ├── Discover resources
   │   ├── Call resources/list
   │   └── Store for context attachment
   │
   └── Set connection state → Connected
```

## Authentication

### OAuth Flow (`auth.ts`, 88 KB)

```
1. Server returns 401 with OAuth metadata URL
2. Client fetches /.well-known/oauth-authorization-server
3. Client initiates PKCE authorization code flow
   ├── Generate code_verifier + code_challenge
   ├── Open browser to authorization URL
   ├── Local HTTP server receives callback
   └── Exchange code for tokens
4. Store tokens (refresh token persisted)
5. Auto-refresh on expiration
```

### XAA (Cross-App Access)

For Anthropic-internal services, XAA provides cross-application authentication without browser-based OAuth.

## Tool Integration

MCP tools are wrapped as `MCPTool` instances that conform to Claude Code's `Tool` interface:

```
MCP Server "github"
  └── Tool: "create_issue"
       │
       ▼
MCPTool instance
  name: "mcp__github__create_issue"
  isMcp: true
  inputSchema: (from MCP server's tool definition)
  call(): Forwards to MCP server via client
```

### Tool Name Normalization

To avoid conflicts between servers, tool names are prefixed:

```
Raw name:     "search"
Normalized:   "mcp__server_name__search"
```

### Tool Filtering

Enterprise policies can filter MCP tools:

```json
{
  "mcpServers": {
    "github": {
      "allowedTools": ["create_issue", "list_issues"],
      "deniedTools": ["delete_repo"]
    }
  }
}
```

## Resource Integration

MCP servers can expose resources (files, data) that Claude Code can attach as context:

```
MCP Server "docs"
  └── Resource: "api-spec.yaml"
       │
       ▼
Available as context attachment
  └── Model can read resource content
```

## MCP as Skills

When `feature('MCP_SKILLS')` is enabled, MCP prompt-type tools are exposed as skills:

```
MCP Server "workflow"
  └── Prompt: "review-code"
       │
       ▼
Slash command: /mcp:workflow:review-code
  └── Model-invocable via SkillTool
```

## Lifecycle Management

The `useManageMCPConnections` React hook manages the full lifecycle:

```
Mount:
  ├── Parse configuration
  ├── Connect to all enabled servers
  └── Register tools/resources in AppState

Runtime:
  ├── Health checks on list/get commands
  ├── Auto-reconnect on disconnection
  ├── Handle authentication challenges
  └── Dynamic server registration (registerMcpAddCommand)

Unmount:
  ├── Disconnect all servers
  ├── Kill child processes (stdio servers)
  └── Clean up resources
```

## Project Approval System

For project-scoped `.mcp.json` files, Claude Code requires user approval:

```
1. Parse .mcp.json from project directory
2. Check if servers are approved
3. If not approved:
   ├── Display server list with details
   ├── Prompt user for approval
   └── Store approval state
4. Only connect to approved servers
```

This prevents malicious repositories from auto-connecting to arbitrary MCP servers.

## Claude Code as MCP Server

Claude Code itself can run as an MCP server (`src/entrypoints/mcp.ts`):

```json
// .mcp.json in another tool
{
  "mcpServers": {
    "claude-code-explorer": {
      "command": "node",
      "args": ["mcp-server/dist/index.js"],
      "env": {
        "CLAUDE_CODE_SRC_ROOT": "./src"
      }
    }
  }
}
```

This exposes Claude Code's file reading, searching, and analysis capabilities to other MCP-compatible clients.
