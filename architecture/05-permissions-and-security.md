# Permission Model and Security Architecture

## Overview

Claude Code implements a defense-in-depth permission model. Every tool invocation passes through permission checking before execution. The system supports multiple permission modes, wildcard-based rules from multiple configuration sources, and enterprise policy enforcement.

## Permission Modes

```typescript
type PermissionMode =
  | 'default'              // Prompt user for non-allowed tools
  | 'plan'                 // Plan before executing, require approval
  | 'acceptEdits'          // Auto-accept file edits, prompt for others
  | 'bypassPermissions'    // Auto-approve all (requires explicit opt-in)
  | 'dontAsk'              // Suppress prompts (for non-interactive)
  | 'auto'                 // ML classifier decides (internal-only)
  | 'bubble'               // Surface permission request to parent agent
```

### Mode Selection

Permission mode is determined by (in order of precedence):
1. `--permission-mode` CLI flag
2. `--dangerously-skip-permissions` flag (sets `bypassPermissions`)
3. Enterprise policy override
4. Default: `default`

### Auto Mode

When enabled (`feature('TRANSCRIPT_CLASSIFIER')`), an ML classifier evaluates the conversation transcript to decide whether a tool call should be approved automatically. This is an internal-only feature with a killswitch mechanism.

## Permission Rules

### Rule Structure

```
Bash(git commit *)     → Allow bash tool with "git commit" prefix
Read(src/**/*.ts)      → Allow reading TypeScript files in src/
Write(/tmp/*)          → Allow writing to /tmp
*                      → Match all tools
```

### Rule Sources (5-level hierarchy)

```
Priority (lowest → highest):
  1. userSettings      ~/.claude/settings.json
  2. projectSettings   ./.claude/settings.json
  3. localSettings     ./.claude/settings-local.json
  4. flagSettings      --settings CLI flag
  5. policySettings    Enterprise managed settings
```

### Rule Actions

```typescript
type PermissionRuleAction = 'allow' | 'deny' | 'ask'

// In settings.json:
{
  "permissions": {
    "allow": [
      "Read",                    // All reads
      "Bash(git *)",             // Git commands
      "Write(src/**)"            // Write to src/
    ],
    "deny": [
      "Bash(rm -rf *)",          // Block destructive commands
      "Write(.env*)"             // Block writing env files
    ]
  }
}
```

### Rule Evaluation Order

```
1. Check deny rules (all sources, highest priority first)
   └── If any deny rule matches → DENIED

2. Check allow rules (all sources)
   └── If any allow rule matches → ALLOWED

3. Check ask rules
   └── If any ask rule matches → PROMPT USER

4. Default behavior based on permission mode
   ├── default → PROMPT USER
   ├── plan → REQUIRE PLAN APPROVAL
   ├── acceptEdits → AUTO-ALLOW edits, PROMPT others
   ├── bypassPermissions → AUTO-ALLOW
   ├── dontAsk → AUTO-ALLOW (silent)
   └── auto → CLASSIFY via ML
```

## Security Layers

### 1. Trust Dialog

Before any git operations, Claude Code requires workspace trust:

```
checkHasTrustDialogAccepted()
  ├── Git can execute arbitrary code via:
  │   ├── .git/hooks/ (pre-commit, post-checkout, etc.)
  │   ├── core.fsmonitor
  │   ├── diff.external
  │   └── Other gitconfig options
  ├── Non-interactive mode (--print) skips dialog
  └── Trust state persisted per-project
```

### 2. Dangerous Pattern Detection

The BashTool detects and flags dangerous command patterns:

```
Blocked patterns:
  ├── Git hook manipulation
  ├── core.fsmonitor injection
  ├── diff.external injection
  ├── Recursive deletion of important paths
  ├── Process injection attempts
  └── Environment variable manipulation
```

### 3. File System Boundaries

```
Working directory restriction:
  ├── Tools confined to project root by default
  ├── Additional directories added via --add-dir
  ├── Path traversal detection
  └── Symlink resolution before access checks
```

### 4. Sandbox Mode

Optional sandboxed execution for untrusted operations:

```
@anthropic-ai/sandbox-runtime
  ├── Isolated process execution
  ├── Restricted filesystem access
  ├── Network policy enforcement
  └── Resource limits (CPU, memory, time)
```

### 5. Enterprise Policy Enforcement

Organizations can enforce policies via managed settings:

```
policySettings (highest priority):
  ├── Cannot be overridden by users
  ├── Loaded from managed-settings.json or remote API
  ├── Can restrict:
  │   ├── Available tools
  │   ├── Allowed commands
  │   ├── MCP server allowlist/denylist
  │   ├── Permission modes
  │   ├── Model selection
  │   └── Feature access
  └── Validated via permissionValidation.ts
```

### 6. Classified Mode

For sensitive environments, transcript analysis ensures no information leakage:

```
Classified mode:
  ├── Transcript scanning for sensitive patterns
  ├── Redaction of classified content
  └── Audit logging
```

## Hook-Based Security

The hook system provides extensible security enforcement:

### Pre-Tool-Use Hooks

```typescript
// settings.json
{
  "hooks": {
    "pre_tool_use": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "validate-command.sh $ARGUMENTS",
            "timeout": 5000
          }
        ]
      }
    ]
  }
}
```

Hook return codes:
- `0`: Approve the tool call
- `1`: Block the tool call (with message)
- `2`: Block and rewake model (async hooks)

### Hook Types

| Type | Mechanism | Use Case |
|------|-----------|----------|
| `command` | Shell script | Custom validation logic |
| `prompt` | LLM evaluation | AI-based review |
| `http` | Webhook POST | External approval systems |
| `agent` | Sub-agent | Complex verification workflows |

### Hook Configuration

```typescript
type HookCommand = {
  type: 'command' | 'prompt' | 'http' | 'agent'
  if?: string                    // Permission rule syntax for conditional execution
  timeout?: number               // Max execution time (ms)
  once?: boolean                 // Only run once per session
  async?: boolean                // Non-blocking execution
  asyncRewake?: boolean          // Wake model on async failure
  statusMessage?: string         // Display message during execution
}
```

## Permission Context Threading

Permissions are initialized early and threaded through the entire execution chain:

```
main.tsx (startup)
  ├── initializeToolPermissionContext()
  ├── stripDangerousPermissionsForAutoMode()
  └── verifyAutoModeGateAccess()
      │
      ▼
QueryEngine receives canUseTool function
      │
      ▼
Tool execution checks canUseTool for each call
      │
      ▼
Sub-agents receive filtered permissions
  ├── bubble mode: request surfaces to parent
  └── inherited mode: child uses parent's rules
```

## Shadowed Rule Detection

The permission system detects when rules shadow each other:

```
Example:
  allow: ["Bash(git *)"]
  deny:  ["Bash(*)"]

The deny rule shadows the allow rule because "Bash(*)" is
broader than "Bash(git *)". The system warns about this
conflict during initialization.
```

## Bypass Killswitch

A safety mechanism to disable permission bypass remotely:

```
checkAndDisableBypassPermissions()
  ├── Checks remote killswitch endpoint
  ├── If activated: forces permission mode to 'default'
  └── Prevents bypassPermissions even if user requested it
```

This protects against scenarios where compromised automation attempts to run with elevated permissions.
