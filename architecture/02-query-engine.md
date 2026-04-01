# Query Engine and Execution Pipeline

## Overview

The query engine is the heart of Claude Code, orchestrating the cycle of sending messages to the Claude API, executing tools, and managing conversation state. It spans two main files:

- **`src/QueryEngine.ts`** (46.6 KB) - Core LLM API caller and tool orchestrator
- **`src/query.ts`** (68.7 KB) - Top-level query coordinator with streaming, budget, and retry

## Query Flow

```
User Input
    │
    ▼
QueryEngine.query()
    │
    ├── 1. Build system prompt
    │      └── Merge base prompt + CLAUDE.md + context attachments
    │
    ├── 2. Assemble API request
    │      ├── Convert messages to API format
    │      ├── Serialize tool schemas
    │      ├── Select model + parameters
    │      └── Apply token budget
    │
    ├── 3. Stream from Claude API
    │      ├── Process content blocks as they arrive
    │      ├── Handle text, tool_use, thinking blocks
    │      └── Track token usage
    │
    ├── 4. Execute tool calls
    │      ├── Partition into concurrent/serial batches
    │      ├── Check permissions for each tool
    │      ├── Run pre-tool-use hooks
    │      ├── Execute tool logic
    │      ├── Run post-tool-use hooks
    │      └── Collect results
    │
    ├── 5. Feed results back to API
    │      └── Loop to step 2 if model wants more tool calls
    │
    └── 6. Return final response
           ├── Update cost tracker
           ├── Persist to history
           └── Trigger compaction if needed
```

## QueryEngine Configuration

```typescript
type QueryEngineConfig = {
  cwd: string                          // Working directory
  tools: Tools                         // Available tool set
  commands: Command[]                  // Available commands
  mcpClients: MCPServerConnection[]    // MCP server connections
  agents: AgentDefinition[]            // Agent definitions
  canUseTool: CanUseToolFn             // Permission checker
  getAppState: () => AppState          // State accessor
  setAppState: (f) => void             // State mutator
  initialMessages?: Message[]          // Pre-loaded conversation
  readFileCache: FileStateCache        // File content cache
  customSystemPrompt?: string          // Override system prompt
  appendSystemPrompt?: string          // Append to system prompt
  userSpecifiedModel?: string          // Model override
  maxTurns?: number                    // Turn limit
  maxBudgetUsd?: number                // Cost limit
  jsonSchema?: Record<string, unknown> // Structured output schema
}
```

## Message Type System

All conversation state flows through a discriminated union of message types (`src/types/message.ts`):

```
Message
  ├── UserMessage          User input text + attachments
  ├── AssistantMessage     Model response (text + tool calls)
  ├── SystemMessage        Internal state changes
  │   ├── SystemInformationalMessage
  │   ├── SystemAPIErrorMessage
  │   ├── SystemLocalCommandMessage
  │   ├── SystemThinkingMessage
  │   ├── SystemMemorySavedMessage
  │   └── SystemTurnDurationMessage
  ├── AttachmentMessage    File/image attachments
  ├── ProgressMessage      Streaming tool progress
  ├── TombstoneMessage     Deleted/compacted messages
  └── ToolUseSummaryMessage  Aggregated tool results
```

This type system ensures exhaustive handling of all message variants throughout the UI and persistence layers.

## Tool Execution Pipeline

### Concurrency Model

The tool execution system (`src/services/tools/`) partitions tool calls into batches:

```
Incoming tool calls: [Read A, Read B, Write C, Read D]
                            │
                     Partition by safety
                            │
              ┌─────────────┼──────────────┐
              │             │              │
        Batch 1 (parallel) │   Batch 2 (serial)  │  Batch 3 (parallel)
        [Read A, Read B]   │   [Write C]          │  [Read D]
              │             │              │
              ▼             ▼              ▼
        Run concurrently  Run exclusively  Run concurrently
        (max 10 parallel)                  (max 10 parallel)
```

**Rules:**
- Read-only tools (`isReadOnly(input) === true`) can run in parallel (up to 10)
- State-modifying tools run serially, one at a time
- A single non-read-only tool forces a batch boundary

### Execution Stages

For each individual tool call:

```
1. Permission check (canUseTool)
   ├── Evaluate allow/deny/ask rules
   ├── Check policy limits
   └── Prompt user if needed

2. Pre-tool-use hooks (toolHooks.ts)
   ├── Run matching command/prompt/http/agent hooks
   ├── Hook can block, modify, or approve
   └── Timeout handling

3. Tool execution (tool.call())
   ├── Validate input against Zod schema
   ├── Execute tool logic
   ├── Stream progress via onProgress callback
   └── Return ToolResult

4. Post-tool-use hooks
   ├── Run matching hooks with result
   └── Can trigger additional actions

5. Result processing
   ├── Update file history state
   ├── Update attribution state
   ├── Report telemetry
   └── Feed result back to conversation
```

### Streaming Tool Executor

`StreamingToolExecutor.ts` manages tool execution during an active API stream:

- Tools can begin executing before the full API response is received
- As soon as a `tool_use` block's input is complete, execution starts
- Results are queued and sent back to the API in order
- Abort controllers enable cancellation if the user interrupts

## Token Budget and Compaction

The query pipeline enforces token budgets at multiple levels:

```
Budget Enforcement Points:
  1. Pre-request: Check if conversation fits in context window
  2. Mid-stream: Monitor token usage during response
  3. Post-response: Track cumulative cost
  4. Auto-compact: Trigger compaction when approaching limits

Compaction Strategy (src/services/compact/):
  1. Summarize older messages
  2. Replace detailed tool results with summaries
  3. Preserve recent context and system instructions
  4. Feature-gated: HISTORY_SNIP for aggressive snipping
  5. REACTIVE_COMPACT for on-the-fly compression
```

## API Integration

### Provider Abstraction

The API layer (`src/services/api/claude.ts`, 125 KB) supports multiple backends:

```
Claude API Request
       │
       ├── Direct API (api.anthropic.com)
       │
       ├── AWS Bedrock (CLAUDE_CODE_USE_BEDROCK=1)
       │   └── @aws-sdk/client-bedrock-runtime
       │
       ├── Google Vertex AI (CLAUDE_CODE_USE_VERTEX=1)
       │   └── google-auth-library
       │
       └── Azure Foundry (CLAUDE_CODE_USE_FOUNDRY=1)
```

### Retry Strategy

`withRetry.ts` (26 KB) implements sophisticated retry logic:

- Exponential backoff with jitter
- Distinguishes transient vs permanent errors
- Quota/rate limit detection with appropriate wait times
- Overloaded server handling
- Maximum retry count per error class

### Request Structure

```typescript
type QueryRequest = {
  messages: BetaMessageParam[]
  model: string
  system: SystemPrompt          // With cache scope markers
  tools: BetaToolUnion[]        // Serialized tool schemas
  tool_choice: BetaToolChoice
  max_tokens: number
  betas?: string[]              // Beta feature flags
  budget_tokens?: {             // Token budget constraints
    input_tokens: number
    output_tokens?: number
  }
}
```

### System Prompt Assembly

The system prompt is assembled from multiple sources:

```
Base system prompt (constants/prompts.ts)
  + Environment context (OS, shell, git status)
  + CLAUDE.md files (project, user, parent directories)
  + Tool descriptions (deferred tools summarized)
  + Permission context
  + Custom system prompt (--system-prompt flag)
  + Appended system prompt (--append-system-prompt flag)
```

Cache scope markers are inserted to maximize prompt cache hits across turns.

## Cost Tracking

`src/cost-tracker.ts` tracks usage by model:

```typescript
type ModelUsage = {
  inputTokens: number
  outputTokens: number
  cacheCreationInputTokens: number
  cacheReadInputTokens: number
  costUSD: number
}
```

The cost tracker maintains running totals across the session and reports them via the status line and `/cost` command.
