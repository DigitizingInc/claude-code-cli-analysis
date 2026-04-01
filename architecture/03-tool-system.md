# Tool System Architecture

## Overview

The tool system is the foundational abstraction in Claude Code. Every capability the model can invoke -- from reading files to spawning sub-agents -- is represented as a `Tool` with a uniform interface. This enables consistent permission checking, telemetry, concurrency control, and error handling across all tools.

## Tool Interface (`src/Tool.ts`)

```typescript
type Tool<Input extends AnyObject, Output, Progress extends ToolProgressData> = {
  // Identity
  name: string
  aliases?: string[]
  searchHint?: string                  // For ToolSearch discovery

  // Execution
  call(
    args: Input,
    context: ToolUseContext,
    canUseTool: CanUseToolFn,
    parentMessage: AssistantMessage,
    onProgress: (progress: Progress) => void
  ): Promise<ToolResult<Output>>

  // Schema
  inputSchema: Input                   // Zod schema for validation
  inputJSONSchema?: ToolInputJSONSchema // JSON Schema for MCP/API

  // Description (dynamic based on input)
  description(input: Input, options: DescriptionOptions): Promise<string>

  // Classification
  isConcurrencySafe(input: Input): boolean  // Can run in parallel?
  isEnabled(): boolean                       // Currently available?
  isReadOnly(input: Input): boolean          // No side effects?
  isDestructive?(input: Input): boolean      // Could cause harm?
  interruptBehavior?(): 'cancel' | 'block'
  isSearchOrReadCommand?(input: Input): { isSearch, isRead, isList }

  // Loading
  isMcp?: boolean              // From MCP server?
  shouldDefer?: boolean        // Defer loading for ToolSearch?
  alwaysLoad?: boolean         // Never defer?
}
```

### Key Design Decisions

**Dynamic descriptions.** Tool descriptions are functions, not static strings. This allows the description to change based on the current input, providing the model with contextually relevant guidance.

**Concurrency classification.** Each tool declares whether it's safe to run concurrently (`isConcurrencySafe`) and whether it's read-only (`isReadOnly`). This drives the batch partitioning in the execution pipeline -- read-only tools run in parallel, state-modifying tools run serially.

**Deferred loading.** Tools with `shouldDefer: true` are not included in the API request by default. Instead, their schemas are loaded on-demand via `ToolSearchTool`. This keeps the initial tool list manageable while supporting 44+ built-in tools plus MCP tools.

## Tool Use Context

Every tool call receives a `ToolUseContext` that provides access to the full application state:

```typescript
type ToolUseContext = {
  options: {
    commands: Command[]
    tools: Tools
    mcpClients: MCPServerConnection[]
    mcpResources: Record<string, ServerResource[]>
    agentDefinitions: AgentDefinitionsResult
    // ... more
  }
  abortController: AbortController
  readFileState: FileStateCache
  getAppState(): AppState
  setAppState(updater): void
  setInProgressToolUseIDs: (f) => void
  updateFileHistoryState: (f) => void
  updateAttributionState: (f) => void
  requestPrompt?: (sourceName, toolInputSummary) => (request) => Promise<PromptResponse>
  // ... 20+ more context fields
}
```

This context threading pattern avoids global state while giving tools access to everything they need: permissions, MCP connections, file caches, abort signals, and UI callbacks.

## Built-In Tools (44)

### Core File Operations

| Tool | Purpose | Read-Only |
|------|---------|-----------|
| `FileReadTool` | Read file contents with line ranges | Yes |
| `FileWriteTool` | Write/create files (atomic operations) | No |
| `FileEditTool` | String replacement edits with diff tracking | No |
| `GlobTool` | Pattern-based file search | Yes |
| `GrepTool` | Content search with regex (uses ripgrep) | Yes |
| `NotebookEditTool` | Edit Jupyter notebook cells | No |

### Execution

| Tool | Purpose | Read-Only |
|------|---------|-----------|
| `BashTool` | Execute shell commands (160 KB implementation) | No |
| `PowerShellTool` | PowerShell execution (Windows) | No |
| `REPLTool` | Interactive Python REPL | No |

### Agent & Coordination

| Tool | Purpose | Read-Only |
|------|---------|-----------|
| `AgentTool` | Spawn sub-agents (233 KB, most complex tool) | No |
| `SendMessageTool` | Send messages between agents | No |
| `TaskCreateTool` | Create background tasks | No |
| `TaskListTool` | List active tasks | Yes |
| `TaskUpdateTool` | Update task status | No |
| `TaskGetTool` | Get task details | Yes |
| `TaskStopTool` | Stop a running task | No |
| `TaskOutputTool` | Get task output | Yes |

### Planning & Workflow

| Tool | Purpose | Read-Only |
|------|---------|-----------|
| `EnterPlanModeTool` | Enter plan approval workflow | No |
| `ExitPlanModeTool` | Leave plan mode | No |
| `EnterWorktreeTool` | Create git worktree for isolation | No |
| `ExitWorktreeTool` | Return from worktree | No |
| `SkillTool` | Execute custom skills in forked context | No |

### External Integration

| Tool | Purpose | Read-Only |
|------|---------|-----------|
| `MCPTool` | Execute tools from MCP servers | Varies |
| `WebSearchTool` | Search the web | Yes |
| `WebFetchTool` | Fetch and process URLs | Yes |
| `LSPTool` | Language Server Protocol operations | Yes |
| `RemoteTriggerTool` | Trigger remote agent execution | No |
| `ScheduleCronTool` | Schedule cron-based triggers | No |

### User Interaction

| Tool | Purpose | Read-Only |
|------|---------|-----------|
| `AskUserQuestionTool` | Prompt user for input | Yes |
| `SleepTool` | Wait for specified duration | Yes |
| `ToolSearchTool` | Discover deferred tools by keyword | Yes |

## Tool Registration (`src/tools.ts`)

```typescript
function getAllBaseTools(): Tools {
  const tools = [
    // Always loaded (alwaysLoad: true)
    BashTool,
    FileReadTool,
    FileWriteTool,
    FileEditTool,
    GlobTool,
    GrepTool,
    AgentTool,
    AskUserQuestionTool,
    ToolSearchTool,

    // Conditionally loaded
    ...(feature('VOICE_MODE') ? [VoiceTool] : []),
    ...(feature('KAIROS') ? [KairosTool] : []),

    // Deferred (loaded via ToolSearch)
    NotebookEditTool,     // shouldDefer: true
    WebSearchTool,        // shouldDefer: true
    WebFetchTool,         // shouldDefer: true
    TaskCreateTool,       // shouldDefer: true
    EnterWorktreeTool,    // shouldDefer: true
    // ... more deferred tools
  ]

  // Filter by USER_TYPE (external build excludes internal-only tools)
  return tools.filter(t => t.isEnabled())
}
```

### Tool Loading Pipeline

```
Built-in tools (getAllBaseTools)
      +
MCP tools (from connected MCP servers)
      +
Plugin tools (from installed plugins)
      +
Skill tools (from loaded skills)
      =
Final tool set (passed to QueryEngine)
```

## BashTool Deep Dive

The BashTool (`src/tools/BashTool/`, 160 KB) is the most security-sensitive tool:

### Security Layers

1. **Permission rules** - Wildcard matching on command patterns (`Bash(git *)`, `Bash(npm install)`)
2. **Dangerous pattern detection** - Blocks commands that could execute arbitrary code (git hooks, core.fsmonitor)
3. **Sandbox mode** - Optional sandboxed execution environment
4. **Working directory restriction** - Commands restricted to project directory
5. **Timeout enforcement** - Configurable command timeout (default 120s, max 600s)
6. **Background execution** - Long-running commands can run in background with notification

### Command Execution Flow

```
User/Model requests: Bash(command: "git status")
    │
    ├── Parse command string
    ├── Check permission rules
    ├── Detect dangerous patterns
    ├── Resolve working directory
    ├── Set up environment variables
    ├── Spawn child process (execa)
    ├── Stream stdout/stderr
    ├── Enforce timeout
    ├── Collect exit code
    └── Return ToolResult with output
```

## AgentTool Deep Dive

The AgentTool (`src/tools/AgentTool/`, 233 KB) is the most architecturally complex tool, enabling recursive agent spawning:

### Agent Types

```
Built-in agents (defined in AgentTool):
  ├── general-purpose    Full tool access, deep research
  ├── Explore           Fast codebase exploration (read-only tools)
  ├── Plan              Architecture/planning (read-only + plan tools)
  └── Custom agents     Defined in .claude/agents/ directories

Fork agents (feature-gated):
  └── fork              Implicit fork inheriting parent context
```

### Spawning Model

```
Parent QueryEngine
    │
    ├── AgentTool.call({prompt, subagent_type})
    │   ├── Resolve agent definition
    │   ├── Build child QueryEngine with:
    │   │   ├── Filtered tool set (per agent definition)
    │   │   ├── Custom system prompt
    │   │   ├── Model override (or inherit)
    │   │   ├── Permission mode (bubble to parent)
    │   │   └── Max turns limit
    │   ├── Run child query
    │   └── Return child's final response
    │
    └── Child QueryEngine
        ├── Can spawn its own agents (nested)
        ├── Permission requests bubble up
        └── Isolated state (except shared AppState)
```

### Git Worktree Isolation

When `isolation: "worktree"` is specified:

```
1. Create temporary git worktree
2. Set child's cwd to worktree path
3. Child works on isolated copy
4. On completion:
   ├── If changes made: return worktree path + branch name
   └── If no changes: auto-cleanup worktree
```

## ToolSearch and Deferred Loading

To keep the initial API request manageable, less-frequently-used tools are deferred:

```
API Request (initial):
  Tools: [Bash, Read, Write, Edit, Glob, Grep, Agent, AskUser, ToolSearch]
  + Deferred tool summary: "Available via ToolSearch: WebSearch, WebFetch,
    NotebookEdit, TaskCreate, TaskList, EnterWorktree, ..."

Model calls: ToolSearch({query: "web search"})
  → Returns full schema for WebSearchTool
  → Model can now use WebSearchTool in subsequent calls
```

This pattern keeps the tool list in the system prompt compact while maintaining access to the full tool ecosystem.
