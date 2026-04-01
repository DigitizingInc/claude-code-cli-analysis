# Services and External Integrations

## Overview

Claude Code integrates with numerous external systems through a modular service layer (`src/services/`). Each service is self-contained with its own types, configuration, and error handling.

## Service Directory

```
src/services/
  ├── api/                 Anthropic API client
  ├── mcp/                 Model Context Protocol
  ├── oauth/               OAuth 2.0 flows
  ├── lsp/                 Language Server Protocol
  ├── analytics/           Feature flags (GrowthBook)
  ├── plugins/             Plugin loader
  ├── compact/             Context compression
  ├── extractMemories/     Auto memory extraction
  ├── teamMemorySync/      Team memory synchronization
  ├── policyLimits/        Enterprise policy enforcement
  ├── remoteManagedSettings/ Enterprise settings
  ├── tools/               Tool execution orchestration
  └── (more)
```

## API Client (`src/services/api/`)

### Architecture

```
src/services/api/
  ├── claude.ts       Main SDK integration (125 KB)
  ├── errors.ts       Error classification (41 KB)
  ├── withRetry.ts    Retry logic (26 KB)
  ├── bootstrap.ts    Initial API calls
  ├── filesApi.ts     File upload/download
  ├── logging.ts      Session metrics (24 KB)
  └── sessionIngress.ts  Auth integration
```

### Multi-Cloud Provider Support

```
Provider selection (checked in order):
  1. CLAUDE_CODE_USE_BEDROCK=1 → AWS Bedrock
     └── Uses @aws-sdk/client-bedrock-runtime
     └── Region: AWS_REGION or us-east-1

  2. CLAUDE_CODE_USE_VERTEX=1 → Google Vertex AI
     └── Uses google-auth-library
     └── Region: CLOUD_ML_REGION or us-central1

  3. CLAUDE_CODE_USE_FOUNDRY=1 → Azure Foundry
     └── Uses @azure/identity

  4. Default → Direct Anthropic API
     └── Uses @anthropic-ai/sdk
     └── Base URL: ANTHROPIC_BASE_URL or api.anthropic.com
```

### Error Handling Strategy

```
Error classification (errors.ts):
  ├── Transient errors → Retry with backoff
  │   ├── Network timeouts
  │   ├── 5xx server errors
  │   └── Rate limit (429) with Retry-After
  │
  ├── Quota errors → Surface to user
  │   ├── Insufficient credits
  │   └── Plan limits exceeded
  │
  ├── Authentication errors → Re-auth flow
  │   ├── Invalid API key
  │   └── Expired token
  │
  └── Permanent errors → Fail immediately
      ├── Invalid request
      ├── Model not found
      └── Content policy violation
```

### Retry Logic (`withRetry.ts`)

```
Retry strategy:
  ├── Exponential backoff: 1s, 2s, 4s, 8s, 16s
  ├── Jitter: ±25% randomization
  ├── Max retries: 5 (default, configurable per error class)
  ├── Rate limit: Wait for Retry-After header
  ├── Overloaded: Wait 30-60s
  └── Circuit breaker: Stop after consecutive failures
```

## OAuth Service (`src/services/oauth/`)

Handles browser-based authentication flows:

```
OAuth 2.0 PKCE Flow:
  1. Generate code_verifier (random 128 bytes)
  2. Compute code_challenge (SHA-256 + base64url)
  3. Open browser to authorization endpoint
  4. Local HTTP server listens for callback
  5. Exchange authorization code for tokens
  6. Store refresh token securely
  7. Auto-refresh on expiration
```

## Language Server Protocol (`src/services/lsp/`)

Integration with language servers for code intelligence:

```
LSP capabilities:
  ├── Hover information
  ├── Go-to-definition
  ├── Find references
  ├── Diagnostics (errors/warnings)
  ├── Code completion
  └── Symbol search
```

The LSPTool exposes these capabilities to the model, enabling it to use IDE-quality code navigation.

## Analytics Service (`src/services/analytics/`)

### GrowthBook Feature Flags

```
GrowthBook integration:
  ├── Feature flag evaluation
  ├── A/B testing assignment
  ├── Lazy initialization (don't block startup)
  └── Offline fallback (default values)
```

### Telemetry (OpenTelemetry)

```
OpenTelemetry integration:
  ├── @opentelemetry/api          Core API
  ├── @opentelemetry/api-logs     Log export
  ├── @opentelemetry/core         Utilities
  ├── @opentelemetry/resources    Resource detection
  ├── @opentelemetry/sdk-logs     Log SDK
  ├── @opentelemetry/sdk-metrics  Metrics SDK
  └── @opentelemetry/sdk-trace-base  Tracing SDK

Metrics tracked:
  ├── API call latency
  ├── Tool execution duration
  ├── Token usage (input/output/cache)
  ├── Error rates by type
  ├── Session duration
  └── Feature flag evaluations
```

### Sentry Error Reporting

```
@sentry/node integration:
  ├── Unhandled exception capture
  ├── Performance monitoring
  ├── Breadcrumb trail
  └── User context (anonymized)
```

## Plugin System (`src/services/plugins/`)

Third-party extensibility:

```
Plugin lifecycle:
  1. Discovery
     ├── ~/.claude/plugins/ directory
     └── Project .claude/plugins/

  2. Loading
     ├── Parse plugin manifest
     ├── Validate against schema
     └── Initialize plugin module

  3. Registration
     ├── Plugin tools → merged into tool set
     ├── Plugin commands → merged into command list
     ├── Plugin skills → merged into skill list
     └── Plugin hooks → merged into hook list

  4. Execution
     ├── Tools called via standard Tool interface
     ├── Commands via standard Command interface
     └── Hooks via standard Hook interface
```

## Context Compression (`src/services/compact/`)

When conversations approach the context window limit:

```
Compaction strategies:
  1. Message summarization
     └── Older messages condensed to key points

  2. Tool result trimming
     └── Large tool outputs replaced with summaries

  3. History snipping (feature-gated: HISTORY_SNIP)
     └── Remove entire old message blocks

  4. Reactive compaction (feature-gated: REACTIVE_COMPACT)
     └── Compress on-the-fly during streaming

Trigger conditions:
  ├── Context utilization > 80%
  ├── Manual: /compact command
  └── API returns context_length_exceeded error
```

## Memory Extraction (`src/services/extractMemories/`)

Automatically extracts persistent insights from conversations:

```
Extraction flow:
  1. Analyze conversation for memorable information
  2. Classify: user preference, project fact, feedback
  3. Check for duplicates against existing memories
  4. Write to ~/.claude/projects/{hash}/memory/
  5. Update MEMORY.md index
```

## Team Memory Sync (`src/services/teamMemorySync/`)

Synchronizes memories across team members:

```
Sync flow:
  ├── Push local memories to remote
  ├── Pull remote memories from teammates
  ├── Merge with conflict resolution
  └── Deduplicate across sources
```

## Policy Limits (`src/services/policyLimits/`)

Enterprise policy enforcement:

```
Policy checks:
  ├── Model restrictions (allowed model list)
  ├── Tool restrictions (allowed/denied tools)
  ├── MCP server restrictions
  ├── Permission mode restrictions
  ├── Token/cost limits
  └── Feature access controls

Enforcement:
  ├── Checked at startup
  ├── Re-checked on configuration change
  └── Cannot be overridden by user settings
```

## Voice Service (`src/voice/`)

Voice input/output (feature-gated: VOICE_MODE):

```
src/services/voiceStreamSTT.ts    Speech-to-text streaming
src/services/voiceKeyterms.ts     Key term detection

Voice pipeline:
  Audio input → STT → Text → QueryEngine → Text → (optional TTS)
```

## Bridge Service (`src/bridge/`)

IDE integration layer:

```
src/bridge/
  ├── bridgeMain.ts              Main bridge logic
  ├── bridgeMessaging.ts         Message protocol
  ├── bridgePermissionCallbacks.ts  Permission UI in IDE
  ├── replBridge.ts              REPL integration
  ├── jwtUtils.ts                Token management
  └── sessionRunner.ts           Session lifecycle

Supported IDEs:
  ├── VS Code (via extension)
  └── JetBrains (via plugin)

Communication:
  ├── WebSocket (primary)
  ├── SSE (fallback)
  └── stdio (for embedded mode)
```

## Remote Sessions (`src/remote/`)

Remote execution support:

```
Remote modes:
  ├── --remote         Create remote session
  ├── --teleport       Resume teleported session
  └── --sdk-url        WebSocket endpoint

Features:
  ├── Session migration between machines
  ├── Remote permission handling
  ├── Bandwidth-efficient protocol
  └── Connection resilience
```

## Monitoring (Grafana)

```
grafana/
  ├── dashboards/       Pre-built dashboards
  └── provisioning/     Dashboard provisioning config

Monitored metrics:
  ├── API latency and error rates
  ├── Token usage and costs
  ├── Session statistics
  ├── Tool execution metrics
  └── MCP server health
```

## Infrastructure (Helm)

```
helm/
  └── Kubernetes Helm charts for deployment

Deployment targets:
  ├── Individual developer machines (CLI)
  ├── Shared server (daemon mode)
  └── Kubernetes cluster (enterprise)
```
