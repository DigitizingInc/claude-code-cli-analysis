# Configuration Architecture

## Overview

Claude Code has a sophisticated multi-source configuration system with strict precedence rules, enterprise policy enforcement, schema validation, and hot-reloading. Configuration affects everything from model selection to permission rules to MCP server connections.

## Configuration Hierarchy

```
Priority (lowest → highest):

1. Defaults (hardcoded)
   └── Built into source code

2. User Settings (~/.claude/settings.json)
   └── Global preferences for all projects

3. Project Settings (./.claude/settings.json)
   └── Shared via git, per-project configuration

4. Local Settings (./.claude/settings-local.json)
   └── Per-project, gitignored (personal overrides)

5. CLI Flag Settings (--settings)
   └── Runtime override via flag

6. Enterprise Policy Settings
   └── Managed remotely, cannot be user-overridden
```

Higher-priority sources override lower ones. Enterprise policies are the final authority.

## Configuration Files

### User Settings (`~/.claude/settings.json`)

```json
{
  "model": "claude-sonnet-4-6",
  "permissions": {
    "allow": [
      "Read",
      "Bash(git *)"
    ],
    "deny": [
      "Bash(rm -rf *)"
    ]
  },
  "theme": "dark",
  "hooks": {
    "pre_tool_use": [...]
  },
  "mcpServers": {}
}
```

### Project Settings (`./.claude/settings.json`)

```json
{
  "permissions": {
    "allow": [
      "Bash(npm run *)",
      "Write(src/**)"
    ]
  },
  "mcpServers": {
    "project-tools": {
      "command": "node",
      "args": ["./tools/mcp-server.js"]
    }
  }
}
```

### Local Settings (`./.claude/settings-local.json`)

Same schema as project settings, but gitignored for personal overrides.

## Environment Variables

From `.env.example` (296 lines):

### Authentication

| Variable | Purpose |
|----------|---------|
| `ANTHROPIC_API_KEY` | API key for direct access |
| `ANTHROPIC_AUTH_TOKEN` | Bearer token (bridge/remote) |
| `ANTHROPIC_BASE_URL` | Custom API base URL |
| `ANTHROPIC_CUSTOM_HEADERS` | Custom request headers |

### Model Selection

| Variable | Purpose |
|----------|---------|
| `ANTHROPIC_MODEL` | Override default model |
| `ANTHROPIC_SMALL_FAST_MODEL` | Fast model for internal use |
| `CLAUDE_CODE_SUBAGENT_MODEL` | Model for sub-agents |

### Multi-Cloud Providers

| Variable | Purpose |
|----------|---------|
| `CLAUDE_CODE_USE_BEDROCK` | Enable AWS Bedrock backend |
| `CLAUDE_CODE_USE_VERTEX` | Enable Google Vertex AI |
| `CLAUDE_CODE_USE_FOUNDRY` | Enable Azure Foundry |
| + region/credential vars | Per-provider configuration |

### Feature Toggles

| Variable | Purpose |
|----------|---------|
| `CLAUDE_CODE_SIMPLE` | Simplified/worker mode |
| `CLAUDE_CODE_COORDINATOR_MODE` | Multi-agent mode |
| `CLAUDE_CODE_PROACTIVE` | Proactive agent mode |
| `CLAUDE_CODE_ENABLE_TASKS` | Task management |
| `CLAUDE_CODE_ACCESSIBILITY` | Accessibility mode |

### Performance

| Variable | Purpose |
|----------|---------|
| `CLAUDE_CODE_MAX_OUTPUT_TOKENS` | Token limits |
| `CLAUDE_CODE_IDLE_THRESHOLD_MINUTES` | Session timeout |
| `NODE_OPTIONS` | Node.js heap/memory |

### Debugging

| Variable | Purpose |
|----------|---------|
| `CLAUDE_CODE_DEBUG_LOG_LEVEL` | Log level |
| `CLAUDE_CODE_PROFILE_STARTUP` | Startup profiling |
| `DEBUG` | General debug flag |

## Schema Validation

Configuration is validated at load time using Zod schemas:

```
src/schemas/
  └── Settings schemas (lazy-loaded)

Validation flow:
  1. Read JSON file
  2. Parse with jsonc-parser (supports comments)
  3. Validate against Zod schema
  4. Report errors (don't crash on invalid config)
  5. Use validated values
```

### Why Lazy Schema Loading?

Zod schemas are lazily loaded to break import cycles. The settings system is needed very early in initialization, before all modules are loaded. Lazy evaluation ensures schemas can reference types from modules that haven't been initialized yet.

## Settings Merge Logic

```typescript
// Simplified merge behavior:
function mergeSettings(sources: SettingsSource[]): Settings {
  let result = DEFAULT_SETTINGS

  for (const source of sources) {
    // Simple properties: override
    if (source.model) result.model = source.model

    // Permission rules: concatenate per-source
    // (evaluated in priority order at check time)
    result.permissions = mergePermissions(
      result.permissions,
      source.permissions,
      source.priority
    )

    // MCP servers: merge by name
    result.mcpServers = {
      ...result.mcpServers,
      ...source.mcpServers
    }

    // Hooks: concatenate
    result.hooks = mergeHooks(result.hooks, source.hooks)
  }

  return result
}
```

## Enterprise Policy Enforcement

Organizations can enforce configuration via managed settings:

```
Policy loading:
  1. Check for local managed-settings.json
  2. OR fetch from remote API endpoint
  3. Policies cannot be overridden by any user setting
  4. Policies can restrict:
     ├── Available models
     ├── Available tools
     ├── MCP server allowlist/denylist
     ├── Permission modes
     ├── Feature access
     └── Custom restrictions
```

### Policy Validation

```
src/utils/permissions/permissionValidation.ts
  ├── Validate policy against schema
  ├── Check for conflicting rules
  ├── Detect shadowed permissions
  └── Report violations
```

## Hot-Reloading

Configuration changes are detected and applied without restart:

```
settingsChangeDetector.initialize()
  ├── Watch all settings files (chokidar)
  ├── On change:
  │   ├── Re-parse affected file
  │   ├── Re-validate against schema
  │   ├── Merge into active configuration
  │   ├── Clear memoized command list
  │   ├── Update permission rules
  │   └── Trigger React re-render
  └── Debounced to avoid rapid-fire reloads
```

## CLI Settings Override

The `--settings` flag supports both file paths and inline JSON:

```bash
# File path
claude --settings /path/to/settings.json

# Inline JSON
claude --settings '{"model": "claude-opus-4-6"}'
```

The `--setting-sources` flag filters which sources are loaded:

```bash
# Only load user settings (ignore project/local)
claude --setting-sources user

# Load user and project (ignore local)
claude --setting-sources user,project
```

## CLAUDE.md Context Files

Beyond JSON settings, CLAUDE.md files provide natural-language context:

```
Loading order:
  1. ~/.claude/CLAUDE.md            (global)
  2. {parent directories}/CLAUDE.md (inherited)
  3. ./CLAUDE.md                    (project root)
  4. ./.claude/CLAUDE.md            (project config dir)

Features:
  ├── Supports nested @include directives
  ├── Loaded into system prompt context
  ├── Prefetched during initialization
  └── Hot-reloaded on change
```

## Keybindings

```
~/.claude/keybindings.json

Default keybindings:
  ├── Enter: Submit message
  ├── Ctrl+C: Cancel/interrupt
  ├── Ctrl+D: Exit
  ├── Up/Down: History navigation
  ├── Tab: Autocomplete
  └── ... (customizable)
```

## Database Configuration

For server mode, database is configured via:

```typescript
// drizzle.config.ts
const databaseUrl = process.env.DATABASE_URL ?? ''

if (isPostgres(databaseUrl)) {
  // PostgreSQL configuration
  dialect: 'postgresql'
  schema: './src/server/db/schema/postgres.ts'
} else {
  // SQLite fallback
  dialect: 'sqlite'
  schema: './src/server/db/schema/sqlite.ts'
  url: process.env.SQLITE_PATH ?? '~/.claude/web/claude-code.db'
}
```
