# Agent and Coordination Architecture

## Overview

Claude Code supports spawning sub-agents -- isolated instances of the query engine that can work on tasks independently. This ranges from simple read-only exploration agents to full multi-agent coordination with shared state.

## Agent Types

### Built-In Agents

| Agent Type | Tools | Model | Purpose |
|-----------|-------|-------|---------|
| `general-purpose` | `*` (all) | inherit | Deep research, complex multi-step tasks |
| `Explore` | Read-only subset | inherit | Fast codebase exploration |
| `Plan` | Read-only + plan tools | inherit | Architecture design, implementation planning |
| Custom | Configurable | Configurable | User-defined agent behaviors |

### Agent Definition

```typescript
type BuiltInAgentDefinition = {
  agentType: string              // Identifier
  whenToUse: string              // Description for model
  tools: string[]                // '*' = inherit all parent tools
  maxTurns: number               // Conversation turn limit
  model: 'inherit' | string      // Model selection
  permissionMode: 'bubble' | PermissionMode
  source: 'built-in'
  baseDir: string                // Working directory
  getSystemPrompt: () => string  // System prompt generator
}
```

### Custom Agents

Users define custom agents in `.claude/agents/` directories:

```
~/.claude/agents/         Global agent definitions
./.claude/agents/         Project-specific agents
```

## AgentTool Execution

When the model calls the `AgentTool`:

```
Parent QueryEngine
    │
    ├── AgentTool.call({
    │     prompt: "Search for all API endpoints",
    │     subagent_type: "Explore",
    │     description: "Find API endpoints",
    │     isolation: "worktree"  // optional
    │   })
    │
    ├── Resolve agent definition
    │   ├── Check built-in agents
    │   ├── Check custom agents (.claude/agents/)
    │   └── Apply model/tool overrides
    │
    ├── Build child QueryEngine
    │   ├── Filter tools per agent definition
    │   ├── Set system prompt (agent-specific)
    │   ├── Set model (inherit or override)
    │   ├── Set permission mode (bubble or specific)
    │   ├── Set max turns
    │   └── Set working directory
    │
    ├── Run child query engine
    │   ├── Child processes the prompt
    │   ├── Uses its filtered tool set
    │   ├── Permission requests bubble to parent
    │   └── Respects turn/budget limits
    │
    └── Return result to parent
        ├── Child's final response text
        ├── Token usage aggregated to parent
        └── File changes tracked
```

## Fork Sub-Agent (Feature-Gated)

When `feature('FORK_SUBAGENT')` is enabled, agents can be implicitly forked:

```
Fork characteristics:
  ├── Child inherits full parent context
  ├── Conversation history shared (cache-efficient)
  ├── Identical system prompt → prompt cache hits
  ├── Synthetic type: FORK_SUBAGENT_TYPE = 'fork'
  ├── Anti-recursion: FORK_BOILERPLATE_TAG prevents nested forks
  └── Tool results use identical placeholders for cache alignment
```

The fork optimization minimizes API costs by maximizing prompt cache reuse between parent and child conversations.

## Git Worktree Isolation

Agents can operate on isolated copies of the repository:

```
Parent repo (/project)
    │
    ├── AgentTool({isolation: "worktree"})
    │
    ├── git worktree add /tmp/worktree-abc123
    │
    ├── Child agent works in /tmp/worktree-abc123
    │   ├── Can modify files freely
    │   ├── Can create branches
    │   └── Changes don't affect parent
    │
    └── On completion:
        ├── Changes made?
        │   ├── Yes: Return worktree path + branch name
        │   │       (parent can merge or inspect)
        │   └── No: Auto-cleanup worktree
        └── Resource cleanup
```

## Multi-Agent Coordination

### Coordinator Mode

When `feature('COORDINATOR_MODE')` is enabled, Claude Code can orchestrate multiple agents:

```
Coordinator (src/coordinator/)
    │
    ├── Plan decomposition
    │   └── Break task into subtasks
    │
    ├── Agent assignment
    │   ├── Match subtask → agent type
    │   ├── Set dependencies between tasks
    │   └── Determine parallelism
    │
    ├── Parallel execution
    │   ├── Independent tasks run concurrently
    │   └── Dependent tasks wait for predecessors
    │
    ├── Result aggregation
    │   ├── Collect all agent outputs
    │   └── Synthesize final response
    │
    └── Progress tracking
        ├── CoordinatorAgentStatus component
        └── Real-time status updates
```

### Team/Swarm Mode

Multi-agent teams with different collaboration patterns:

```
CLI flags:
  --agent-teams          Enable multi-agent mode
  --teammate-mode        tmux | in-process | auto

Team structure:
  ├── Agent IDs for tracking
  ├── Agent names for display
  ├── Color coordination (agentColorManager.ts)
  ├── Parent session correlation
  └── Shared memory/context
```

#### Teammate Modes

| Mode | Mechanism | Use Case |
|------|-----------|----------|
| `tmux` | Separate tmux panes | Visible parallel work |
| `in-process` | Shared Node.js process | Lightweight, fast |
| `auto` | Runtime selection | Let system choose |

## Permission Bubbling

When a sub-agent encounters a permission check:

```
Child agent wants to run: Bash(npm install)
    │
    ├── permissionMode: 'bubble'
    │
    ├── Permission request surfaces to parent
    │   ├── Parent's UI shows the request
    │   ├── User approves/denies
    │   └── Result flows back to child
    │
    └── Child continues (or aborts if denied)
```

This ensures human oversight of all state-modifying operations, even when performed by deeply nested sub-agents.

## Agent Color System

Each agent gets a distinct terminal color for visual differentiation:

```typescript
// agentColorManager.ts
colors: ['red', 'blue', 'green', 'yellow', 'magenta', 'cyan']

// Status line shows:
// [Agent: Explore] in blue
// [Agent: general-purpose] in red
```

## Task System Integration

Agents can create and manage background tasks:

```
AgentTool spawns child
    │
    ├── TaskCreateTool: Create trackable task
    │   ├── Task ID assigned
    │   ├── Status: running
    │   └── Progress updates via TaskUpdateTool
    │
    ├── Parent can check: TaskListTool, TaskGetTool
    │
    ├── Output retrieval: TaskOutputTool
    │
    └── Cancellation: TaskStopTool
```

### Task Types

```
src/tasks/
  ├── shell-tasks/      Background shell commands
  ├── agent-tasks/      Background agent executions
  └── (orchestration)   Task lifecycle management
```

## SendMessage Tool

For agent-to-agent communication:

```
Agent A                    Agent B
   │                          │
   ├── SendMessage({          │
   │     to: "agent-b",      │
   │     message: "Found 3   │
   │     API endpoints"       │
   │   })                     │
   │         ───────────────► │
   │                          ├── Receives message
   │                          └── Incorporates into context
```

## Resource Management

### Abort Controllers

Each agent has its own `AbortController`:

```
Parent abort
    ├── Cancels all child agents
    └── Cleans up resources (worktrees, processes)
```

### Turn Limits

Agents enforce maximum turn counts to prevent runaway execution:

```
maxTurns: 10 (default for most agents)
  ├── Each API request/response cycle = 1 turn
  ├── Tool executions within a turn don't count
  └── Exceeding limit → agent returns with partial results
```

### Budget Limits

```
maxBudgetUsd: (optional)
  ├── Cumulative cost tracking per agent
  ├── Includes API calls + sub-agent costs
  └── Exceeding limit → graceful termination
```
