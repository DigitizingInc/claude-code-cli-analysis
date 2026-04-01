# Command and Skill System

## Commands Overview

Claude Code's command system provides slash-commands (e.g., `/commit`, `/config`, `/help`) that users invoke directly from the REPL. There are 89 commands organized into three types.

## Command Types

```typescript
// 1. PromptCommand - Invokes Claude with a crafted prompt
type PromptCommand = {
  type: 'prompt'
  progressMessage: string           // Status message during execution
  getPromptForCommand(args, context): Promise<ContentBlockParam[]>
  allowedTools?: string[]           // Tool whitelist for this command
  model?: string                    // Model override
  effort?: 'default' | 'high' | 'extreme'
  context?: 'inline' | 'fork'      // Execute in current or forked context
  hooks?: HookConfiguration         // Pre/post hooks
  paths?: string                    // File glob patterns
}

// 2. LocalCommand - Synchronous terminal operation
type LocalCommand = {
  type: 'local'
  call(args, context): Promise<LocalCommandResult>
  // Returns: text output, compact trigger, or skip
}

// 3. LocalJSXCommand - Interactive Ink UI component
type LocalJSXCommand = {
  type: 'local-jsx'
  load(): Promise<{
    default: (props) => React.ReactNode
  }>
  // Lazy-loaded to avoid pulling in heavy UI dependencies
}
```

### Why Three Types?

- **PromptCommands** need Claude's intelligence (e.g., `/commit` generates commit messages, `/review` analyzes code)
- **LocalCommands** are pure logic (e.g., `/cost` displays token usage, `/clear` resets conversation)
- **LocalJSXCommands** need interactive UI (e.g., `/config` opens a settings editor, `/mcp` shows server management)

## Command Loading Pipeline

Commands are assembled from multiple sources with memoization:

```
COMMANDS() [memoized static list]
    │
    ├── 59 built-in commands (hardcoded)
    │
    ├── Bundled skills → getBundledSkills()
    │   └── 12 skills from src/skills/bundled/
    │
    ├── Skill directory → getSkillDirCommands(cwd)
    │   ├── ~/.claude/skills/
    │   └── ./.claude/skills/
    │
    ├── Plugin skills → getPluginSkills()
    │
    ├── Plugin commands → getPluginCommands()
    │
    ├── Workflow commands → getWorkflowCommands(cwd)
    │
    ├── MCP commands → getMcpToolsCommandsAndResources()
    │   └── Prompt-type tools from MCP servers
    │
    └── Dynamic skills → getDynamicSkills()

Final: loadAllCommands(cwd)  [memoized per directory]
```

### Cache Invalidation

```
Configuration change → clearCommandsCache()
Skill file change   → clearCommandsCache()
Plugin install      → clearCommandsCache()
```

Both `settingsChangeDetector` and `skillChangeDetector` watch for filesystem changes and trigger reloads.

## Command Filtering

Not all commands are available in all contexts:

```
meetsAvailabilityRequirement()
  ├── Auth checks (claude-ai only, console only)
  └── Provider gating

isCommandEnabled()
  └── Feature flag check

Remote mode filtering:
  ├── REMOTE_SAFE_COMMANDS (17 whitelisted)
  └── BRIDGE_SAFE_COMMANDS (6 whitelisted for mobile)

Model-invocable filtering:
  └── getSkillToolCommands() for SkillTool access
```

## Key Built-In Commands

| Command | Type | Purpose |
|---------|------|---------|
| `/help` | local | Display help information |
| `/config` | local-jsx | Interactive settings editor |
| `/cost` | local | Show token usage and costs |
| `/clear` | local | Reset conversation |
| `/compact` | local | Trigger context compaction |
| `/commit` | prompt | Generate and create git commit |
| `/review` | prompt | Code review current changes |
| `/diff` | local | Show git diff |
| `/memory` | local-jsx | Manage persistent memories |
| `/mcp` | local-jsx | MCP server management |
| `/session` | local | Session management |
| `/tasks` | local-jsx | Task list management |
| `/doctor` | local-jsx | Environment diagnostics |
| `/init` | prompt | Initialize project configuration |
| `/agents` | local-jsx | Agent management |

## Skill System

Skills are reusable prompt workflows -- lighter weight than plugins, defined as markdown files with YAML frontmatter.

### Skill Discovery Hierarchy

```
1. Bundled skills (src/skills/bundled/)
   └── Compiled into the binary, always available

2. User skills (~/.claude/skills/)
   └── Global skills available in all projects

3. Project skills (./.claude/skills/)
   └── Project-specific, shared via git

4. Plugin skills (from installed plugins)
   └── Distributed via plugin marketplace

5. Dynamic skills (discovered at runtime)
   └── From file system scanning
```

### Skill Definition Format

```yaml
---
name: simplify
description: Review changed code for quality and efficiency
displayName: Simplify Code
whenToUse: After writing code, review for improvements
paths: "**/*.ts"          # Only trigger for TypeScript files
allowedTools:             # Tool whitelist
  - FileReadTool
  - FileEditTool
  - GrepTool
argumentHint: "<file>"
arguments:                # Typed arguments
  - name: file
    type: string
    default: "."
effort: default           # default | high | extreme
model: claude-sonnet-4-6  # Optional model override
context: fork             # inline (same context) | fork (isolated)
agent: Explore            # Agent type for forked execution
hooks:                    # Pre/post hooks
  pre_tool_use:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "echo 'About to run bash'"
disableModelInvocation: false  # Can model invoke this skill?
---

Your skill prompt content here. This is what Claude receives
when the skill is invoked. It can reference $ARGUMENTS for
user-provided arguments.
```

### Skill Execution

When a skill is invoked (via `/skill-name` or `SkillTool`):

```
1. Parse frontmatter for configuration
2. Resolve arguments ($ARGUMENTS substitution)
3. Determine execution context:
   ├── inline: Execute in current conversation
   └── fork: Spawn sub-agent with filtered tools
4. Build prompt from skill content
5. Apply model/effort overrides
6. Execute via QueryEngine
7. Return result to conversation
```

### Bundled Skills

| Skill | Purpose |
|-------|---------|
| `simplify` | Review code for quality and efficiency |
| `verify` | Verify code correctness |
| `debug` | Debugging utilities |
| `batch` | Batch operations on multiple files |
| `loop` | Recurring execution on interval |
| `schedule` | Cron-based scheduled execution |
| `remember` | Memory management |
| `skillify` | Create new skills from patterns |
| `update-config` | Configure settings |
| `keybindings` | Keyboard shortcut management |
| `claude-api` | Build apps with Claude API |

### Token Budgeting

Skills participate in token budget management:

```typescript
estimateSkillFrontmatterTokens(skill)
// Quick estimate from frontmatter without loading full content
// Used to decide whether to include skill in tool descriptions
```

### Deduplication

Skills loaded from multiple paths are deduplicated:

```typescript
getFileIdentity(path)
// Uses realpath() to detect duplicates via symlinks
// Avoids inode-based detection (unreliable on virtual filesystems)
```

## Workflow Scripts (Feature-Gated)

When `feature('WORKFLOW_SCRIPTS')` is enabled, additional workflow commands are loaded from `.claude/workflows/`. These are executable scripts that integrate with the command system.
