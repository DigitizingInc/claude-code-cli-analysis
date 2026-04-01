# State Management Architecture

## Overview

Claude Code manages state at multiple levels: global application state (AppState), session-level bootstrap state, conversation history, file operation tracking, and persistent configuration. The architecture avoids global mutable singletons in favor of context-threaded state with controlled update paths.

## AppState (`src/state/`)

### Structure

```typescript
type AppState = {
  // Working directory
  cwd: string
  originalCwd: string
  projectRoot: string

  // Cost tracking
  totalCostUSD: number
  modelUsage: { [modelName: string]: ModelUsage }

  // Session
  sessionId: SessionId
  parentSessionId?: SessionId
  kairosActive: boolean

  // MCP
  mcp: {
    servers: MCPServerConnection[]
    commands: Command[]
    tools: Tool[]
    resources: Record<string, ServerResource[]>
  }

  // Plugins
  plugins: {
    enabled: LoadedPlugin[]
    installed: InstalledPlugin[]
  }

  // Teams & agents
  teams: Team[]
  agents: AgentDefinition[]

  // Telemetry
  meter: Meter | null
  sessionCounter: AttributedCounter | null

  // ... 50+ more fields
}
```

### State Store (`AppStateStore.ts`)

The store provides controlled update mechanics:

```typescript
// Read state
const state = getAppState()

// Update state (immutable update pattern)
setAppState(prev => ({
  ...prev,
  totalCostUSD: prev.totalCostUSD + 0.01
}))
```

### State Watchers (`onChangeAppState.ts`)

Listeners can subscribe to state changes:

```
State change
    │
    ├── UI re-render (React context)
    ├── Status line update
    ├── Cost display update
    ├── MCP server list refresh
    └── Telemetry emission
```

## Bootstrap State (`src/bootstrap/state.ts`)

Low-level global state for the session lifecycle. This is the earliest state initialized, before React rendering begins:

```
Bootstrap state includes:
  ├── Session ID generation
  ├── API configuration
  ├── Authentication tokens
  ├── Feature flag evaluation
  ├── Telemetry initialization
  └── Cost accumulator seed
```

At 56 KB, this file handles the complex initialization that must happen before the React tree exists.

## Session Persistence

### History (`src/history.ts`)

Conversation history is persisted to disk for session resume:

```
Session storage location:
  ~/.claude/sessions/{sessionId}/

Contents:
  ├── messages.json      Full conversation history
  ├── metadata.json      Session metadata (model, cost, etc.)
  └── attachments/       Uploaded files/images
```

### Session Storage (`src/utils/sessionStorage.ts`, 56 KB)

Handles the full persistence lifecycle:

```
Save flow:
  1. Serialize messages (strip non-persistent fields)
  2. Write to temp file
  3. Atomic rename to final path
  4. Update session index

Resume flow:
  1. Read session index
  2. Load messages from disk
  3. Reconstruct AppState
  4. Rebuild tool/MCP state
  5. Continue conversation
```

### Attribution Tracking

Every file change is attributed to the tool/turn that made it:

```typescript
updateAttributionState(prev => ({
  ...prev,
  fileChanges: [
    ...prev.fileChanges,
    {
      file: '/path/to/file.ts',
      tool: 'FileEditTool',
      turnId: 'turn-123',
      timestamp: Date.now()
    }
  ]
}))
```

This enables features like undo, change review, and git blame attribution.

## File State Cache

The `readFileState` cache tracks file contents across tool calls:

```
FileStateCache
  ├── Cache file contents on first read
  ├── Invalidate on write/edit
  ├── Track file versions (before/after)
  ├── Used by:
  │   ├── FileReadTool (populate)
  │   ├── FileWriteTool (invalidate + snapshot)
  │   ├── FileEditTool (invalidate + diff)
  │   └── GrepTool (populate)
  └── Shared across tool calls within a turn
```

## Configuration State

### Settings Hierarchy

```
Settings merge order (lowest → highest priority):
  1. Defaults (hardcoded)
  2. User settings (~/.claude/settings.json)
  3. Project settings (./.claude/settings.json)
  4. Local settings (./.claude/settings-local.json)
  5. CLI flag settings (--settings)
  6. Enterprise policy settings
```

### Configuration Watching

```
settingsChangeDetector
  ├── Watch ~/.claude/settings.json
  ├── Watch ./.claude/settings.json
  ├── Watch ./.claude/settings-local.json
  ├── On change:
  │   ├── Re-parse settings
  │   ├── Clear command cache
  │   ├── Update permission rules
  │   └── Trigger UI refresh
  └── Uses chokidar for filesystem watching

skillChangeDetector
  ├── Watch ~/.claude/skills/
  ├── Watch ./.claude/skills/
  ├── On change:
  │   ├── Reload skills
  │   └── Clear command cache
  └── Debounced to avoid rapid reloads
```

## React Context Architecture

State flows through React context providers:

```
<AppStateProvider value={appState}>
  <TerminalSizeContext.Provider>
    <StdinContext.Provider>
      <TerminalFocusContext.Provider>
        <ClockContext.Provider>
          <NotificationContext.Provider>
            <App />
          </NotificationContext.Provider>
        </ClockContext.Provider>
      </TerminalFocusContext.Provider>
    </StdinContext.Provider>
  </TerminalSizeContext.Provider>
</AppStateProvider>
```

### Key Context Providers

| Provider | Purpose |
|----------|---------|
| `AppStateProvider` | Global application state |
| `TerminalSizeContext` | Terminal dimensions (reactive on resize) |
| `StdinContext` | Raw stdin stream access |
| `TerminalFocusContext` | Window focus tracking |
| `ClockContext` | Animation frame timing |
| `NotificationContext` | Toast/notification system |
| `FPSContext` | Frame rate monitoring |

## Memory System (`src/memdir/`)

Persistent memory across sessions:

```
Memory storage:
  ~/.claude/projects/{project-hash}/memory/
    ├── MEMORY.md          Index file (always loaded)
    ├── user_role.md       User memory
    ├── feedback_*.md      Feedback memories
    ├── project_*.md       Project memories
    └── reference_*.md     Reference memories

Loading:
  1. MEMORY.md loaded into system context
  2. Individual memories loaded on demand
  3. Prefetched during initialization

Extraction:
  src/services/extractMemories/
  └── Auto-extract memories from conversation
```

## Database (Server Mode)

When running as a server (`src/server/`), state is persisted to a database:

```
Database support:
  ├── PostgreSQL (production)
  │   └── src/server/db/schema/postgres.ts
  ├── SQLite (development/local)
  │   └── src/server/db/schema/sqlite.ts
  └── Drizzle ORM for both

Migrations:
  └── src/server/db/migrations/

Default SQLite path:
  ~/.claude/web/claude-code.db
```

## State Flow Diagram

```
User Input
    │
    ▼
REPL Loop (main.tsx)
    │
    ├──► AppState.setAppState()
    │       │
    │       ├── UI re-renders
    │       └── Watchers fire
    │
    ├──► QueryEngine
    │       │
    │       ├── API call → cost tracking → AppState
    │       ├── Tool call → file state → FileStateCache
    │       ├── Tool call → attribution → AttributionState
    │       └── Result → messages → History
    │
    ├──► Session persistence
    │       └── Periodic save to disk
    │
    └──► Memory extraction
            └── Persist insights to memory files
```
