# Entry Points and Bootstrap Sequence

## Entry Points

Claude Code has multiple entry points for different execution contexts:

```
src/entrypoints/
  cli.tsx       Primary CLI entry (bin: "claude")
  init.ts       Shared initialization logic
  mcp.ts        MCP server mode entry
  sdk/          Agent SDK entry points
```

### CLI Entry (`src/entrypoints/cli.tsx`)

The main entry point handles early-exit flags before loading the full application:

```
cli.tsx execution order:
  1. Process --version, --dump-system-prompt (exit immediately)
  2. Load runtime shims (macro.ts, bun-bundle.ts)
  3. Parse early CLI flags via eagerParseCliFlag()
     - --settings, --setting-sources processed BEFORE init()
     - This affects which config files load
  4. Call init() from init.ts
  5. Hand off to main.tsx
```

**Why early flag parsing matters:** Settings flags must be processed before `init()` because configuration loading happens during initialization. The `eagerParseCliFlag()` utility extracts specific flags from `process.argv` without loading Commander.js, avoiding the full CLI framework overhead for flags that affect bootstrapping.

### MCP Server Entry (`src/entrypoints/mcp.ts`)

When Claude Code runs as an MCP server (providing tools to other clients), it uses a separate entry point that skips the REPL/TUI setup entirely.

### SDK Entry (`src/entrypoints/sdk/`)

For programmatic usage via the Agent SDK, providing a headless API without terminal UI.

## Main Application (`src/main.tsx`)

At ~4,700 lines, `main.tsx` is the largest single file and acts as the application's spine:

```
main.tsx responsibilities:
  1. Commander.js CLI parser setup (lines 585-900+)
     - All flags: -p, -m, -d, -c, --settings, --permission-mode, etc.
  2. Pre-action hooks (line 907)
     - Initialize state before command execution
  3. Trust dialog (lines 354-380)
     - Workspace trust must be established before git operations
     - Git can execute arbitrary code via hooks/config
     - Non-interactive mode (--print) skips trust dialog
  4. Deferred prefetches (line 388)
     - System context, user context, tips, model capabilities
     - Loaded async after first render
     - --bare mode skips all prefetches
  5. React/Ink renderer initialization
     - Custom Ink instance creation
     - Root component mounting
  6. REPL loop
     - Interactive prompt → QueryEngine → display results
  7. Session management
     - History persistence, resume, session IDs
```

### CLI Flags

Major flags parsed by Commander.js:

| Flag | Purpose |
|------|---------|
| `-p, --print` | Non-interactive mode (pipe-friendly) |
| `-m, --model` | Model selection override |
| `-d, --debug` | Debug mode with optional category filter |
| `-c, --continue` | Resume previous session |
| `--settings` | Path to settings JSON or inline JSON string |
| `--setting-sources` | Filter which setting sources load (user,project,local) |
| `--permission-mode` | Permission handling mode |
| `--dangerously-skip-permissions` | Skip all permission prompts |
| `--worktree` | Create git worktree for isolated session |
| `--advisor` | Enable server-side advisor tool |
| `--proactive` | Start in autonomous mode |
| `--brief` | Enable SendUserMessage tool |
| `--assistant` | Force assistant mode (Agent SDK) |
| `--remote` | Create remote session |
| `--sdk-url` | Remote WebSocket endpoint for SDK I/O |

## Initialization Sequence

```
1. cli.tsx
   ├── Early flag parsing (--version, --settings)
   ├── Runtime shim loading
   └── init()

2. init.ts
   ├── Load configuration from all sources
   ├── Initialize telemetry (OpenTelemetry, Sentry)
   ├── Set up API client
   ├── Check for updates
   └── Return to main.tsx

3. main.tsx (pre-action)
   ├── checkAndDisableBypassPermissions()
   ├── initializeToolPermissionContext()
   ├── stripDangerousPermissionsForAutoMode()
   ├── verifyAutoModeGateAccess()
   ├── checkHasTrustDialogAccepted()
   └── startDeferredPrefetches()

4. main.tsx (render)
   ├── Create Ink renderer instance
   ├── Mount <App /> component tree
   ├── Initialize REPL bridge (for IDE mode)
   ├── Start settings/skill change detectors
   └── Enter REPL loop or run --print pipeline
```

## Deferred Prefetching

After the first render frame, Claude Code kicks off background work to warm caches:

```typescript
startDeferredPrefetches()
  ├── prefetchSystemContextIfSafe()   // Git info, project structure
  ├── prefetchUserContext()            // CLAUDE.md files, memory
  ├── prefetchTips()                   // Usage tips
  └── prefetchModelCapabilities()     // Available models, limits
```

This pattern ensures the UI is responsive immediately while expensive I/O happens in the background. The `--bare` flag disables all prefetches for maximum startup speed.

## Session Lifecycle

```
Start
  │
  ├── New session → Generate session ID, create history file
  │
  └── Resume session (--continue) → Load history, restore state
       │
       ├── REPL loop
       │   ├── Read user input
       │   ├── Run QueryEngine
       │   ├── Display results
       │   ├── Persist to history
       │   └── Loop
       │
       ├── Context compaction (when approaching token limit)
       │
       └── Exit
            ├── Save final state
            ├── Report telemetry
            └── Clean up resources (MCP servers, worktrees)
```
