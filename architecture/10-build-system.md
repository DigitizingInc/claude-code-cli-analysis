# Build System and Feature Flags

## Overview

Claude Code compiles from ~1,900 TypeScript source files into a single ESM bundle (`dist/cli.mjs`) using esbuild. The build system handles source resolution, internal package stubbing, compile-time constant replacement, and feature flag dead-code elimination.

## Build Pipeline

### CLI Bundle (`scripts/build-bundle.ts`)

```
Entry: src/entrypoints/cli.tsx
                │
                ▼
        esbuild (v0.25.0)
                │
    ┌───────────┼───────────┐
    │           │           │
  Plugins    Externals    Defines
    │           │           │
    ▼           ▼           ▼
  Single ESM bundle: dist/cli.mjs
    + dist/meta.json (analysis)
    + dist/package.json ({"type": "module"})
    + Source maps (dev: inline, prod: external)
```

### Build Modes

```bash
npm run build          # Development (inline sourcemaps)
npm run build:watch    # Watch mode (live reload)
npm run build:prod     # Production (minified, external sourcemaps)
```

### esbuild Configuration

```typescript
{
  entryPoints: ['src/entrypoints/cli.tsx'],
  bundle: true,
  format: 'esm',
  platform: 'node',
  target: 'node20',
  outfile: 'dist/cli.mjs',
  treeShaking: true,
  metafile: true,        // Bundle analysis
  banner: { js: '#!/usr/bin/env node' },
  // ... plugins, externals, defines
}
```

## Custom esbuild Plugins

### 1. `src-resolver`

Resolves bare `src/...` imports to filesystem TypeScript files:

```
Import: "src/utils/config"
    ↓
Resolves to: /absolute/path/src/utils/config.ts
```

This mirrors the `baseUrl` + `paths` from `tsconfig.json`, allowing imports like `import { foo } from 'src/utils/config'` without relative paths.

### 2. `stub-missing`

Handles missing Anthropic-internal packages:

```
Import: "@ant/some-internal-package"
    ↓
Stub: src/shims/ant-stub.ts (exports empty object)

Import: "./some-file.js"
    ↓
Tries: ./some-file.ts, ./some-file.tsx
```

This is critical for the leaked source build: internal `@ant/*` packages don't exist in the public build, so they're stubbed to empty objects. The feature flag system then ensures none of their code paths execute.

## Externals

These packages are **not bundled** (expected at runtime):

```
Node built-ins:
  fs, path, os, crypto, child_process, http, https, net, tls,
  url, util, stream, events, buffer, readline, zlib,
  worker_threads, v8, inspector

Native addons:
  fsevents, sharp, node-pty, image-processor-napi,
  audio-capture-napi, modifiers-napi

Cloud SDKs:
  @aws-sdk/*, @azure/identity, google-auth-library

Anthropic internal:
  @anthropic-ai/sandbox-runtime, @anthropic-ai/claude-agent-sdk,
  @anthropic-ai/mcpb

Optional dependencies:
  asciichart, bidi-js, env-paths, fflate, https-proxy-agent,
  lru-cache, shell-quote, turndown, vscode-jsonrpc, xss
```

## Compile-Time Constants (MACRO System)

### Definition

```typescript
// src/types/macro.d.ts
declare const MACRO: {
  VERSION: string           // Package version
  PACKAGE_URL: string       // "@anthropic-ai/claude-code"
  ISSUES_EXPLAINER: string  // GitHub issues URL
}
```

### Build-Time Replacement

```typescript
// In build-bundle.ts:
define: {
  'MACRO.VERSION': JSON.stringify(pkg.version),
  'MACRO.PACKAGE_URL': JSON.stringify("@anthropic-ai/claude-code"),
  'MACRO.ISSUES_EXPLAINER': JSON.stringify("report issues at ..."),
  'process.env.USER_TYPE': JSON.stringify("external"),
  'process.env.NODE_ENV': JSON.stringify(mode),
}
```

These become string literals in the bundle, enabling dead-code elimination when used in conditionals.

### Runtime Shim

For development (without esbuild), `src/shims/macro.ts` provides runtime values:

```typescript
// Read from package.json at startup
globalThis.MACRO = {
  VERSION: pkg.version ?? "0.0.0-dev",
  PACKAGE_URL: "@anthropic-ai/claude-code",
  ISSUES_EXPLAINER: "report issues at ..."
}
```

## Feature Flag System

### `bun:bundle` Shim (`src/shims/bun-bundle.ts`)

```typescript
export function feature(name: string): boolean
```

In Bun's native runtime, `bun:bundle` provides compile-time feature evaluation. For the esbuild build, this is shimmed to a runtime function that checks environment variables.

### Feature Flags

| Flag | Env Variable | Purpose |
|------|-------------|---------|
| `PROACTIVE` | `CLAUDE_CODE_PROACTIVE` | Autonomous agent mode |
| `KAIROS` | `CLAUDE_CODE_KAIROS` | Assistant mode |
| `KAIROS_BRIEF` | `CLAUDE_CODE_KAIROS_BRIEF` | Brief assistant variant |
| `BRIDGE_MODE` | `CLAUDE_CODE_BRIDGE_MODE` | IDE bridge mode |
| `DAEMON` | `CLAUDE_CODE_DAEMON` | Background daemon |
| `VOICE_MODE` | `CLAUDE_CODE_VOICE_MODE` | Voice input/output |
| `COORDINATOR_MODE` | `CLAUDE_CODE_COORDINATOR_MODE` | Multi-agent orchestration |
| `WORKFLOW_SCRIPTS` | `CLAUDE_CODE_WORKFLOW_SCRIPTS` | Workflow automation |
| `FORK_SUBAGENT` | `CLAUDE_CODE_FORK_SUBAGENT` | Implicit fork spawning |
| `HISTORY_SNIP` | `CLAUDE_CODE_HISTORY_SNIP` | Aggressive history trimming |
| `ULTRAPLAN` | `CLAUDE_CODE_ULTRAPLAN` | Enhanced planning |
| `TORCH` | `CLAUDE_CODE_TORCH` | Advanced reasoning mode |
| `BUDDY` | `CLAUDE_CODE_BUDDY` | Companion sprite |
| `MCP_SKILLS` | `CLAUDE_CODE_MCP_SKILLS` | MCP prompts as skills |
| `REACTIVE_COMPACT` | `CLAUDE_CODE_REACTIVE_COMPACT` | On-the-fly compression |
| `UDS_INBOX` | `CLAUDE_CODE_UDS_INBOX` | Unified dispatcher inbox |
| +10 more | ... | ... |

### Usage Pattern

```typescript
import { feature } from 'bun:bundle'

if (feature('KAIROS')) {
  // This entire block is eliminated in external builds
  // because KAIROS defaults to false
  import('./kairos/KairosMode')
}

// Conditional tool loading
const tools = [
  BashTool,
  FileReadTool,
  ...(feature('VOICE_MODE') ? [VoiceTool] : []),
]
```

### Dead-Code Elimination

In production builds:
1. `process.env.USER_TYPE` is replaced with `"external"`
2. Feature flags evaluate to `false` for internal features
3. esbuild's tree shaking removes unreachable code branches
4. Internal-only tools, commands, and UI components are stripped

## Web Build (`scripts/build-web.ts`)

```
Entry: src/server/web/terminal.ts
                │
                ▼
        esbuild (v0.25.0)
                │
Output: src/server/web/public/
  ├── terminal.js (ESM)
  └── terminal.css

Platform: browser (es2020)
Target: chrome90, firefox90, safari14
```

The web build is separate from the CLI build, targeting browsers for the xterm.js-based terminal viewer.

## Post-Build Processing

After esbuild completes:

```
1. Fix typeof checks
   - Convert `typeof module` and `typeof exports` checks
   - Prevents Bun from misclassifying the ESM bundle as CJS

2. Add shebang
   - #!/usr/bin/env node (already via banner)

3. Create package.json
   - {"type": "module"} in dist/

4. Write metafile
   - dist/meta.json for bundle analysis

5. Report size
   - Print gzipped bundle size
```

## Development Mode

### Bun Plugin Shim (`scripts/bun-plugin-shims.ts`)

```toml
# bunfig.toml
preload = ["./scripts/bun-plugin-shims.ts"]
```

In development, Bun's preload mechanism intercepts `bun:bundle` imports and redirects them to `src/shims/bun-bundle.ts`. This lets the source code run directly under Bun without bundling.

### Vitest Configuration

```typescript
// vitest.config.ts
plugins: [resolveJsToTs],  // .js → .ts resolution
resolve: {
  alias: [
    { find: /^src\//, replacement: resolve(__dirname, 'src') + '/' },
    { find: 'bun:bundle', replacement: resolve(__dirname, 'src/shims/bun-bundle.ts') },
  ],
}
```

Tests use the same alias resolution as the build, ensuring consistent behavior.

## CI/CD Pipeline

```
GitHub Actions (ci.yml):
  ├── Lint (Biome)
  ├── Type check (tsc --noEmit)
  ├── Security audit (bun audit)
  ├── Build web (esbuild)
  │   └── Upload .next/ artifact
  └── Bundle size gate
      └── 150 KB gzipped limit for JS chunks

GitHub Actions (test.yml):
  ├── Unit tests (vitest)
  ├── Component tests (vitest + jsdom)
  ├── E2E tests (Playwright + chromium)
  └── Visual regression (Playwright snapshots)
```

## Docker Build

```dockerfile
# Multi-stage build
FROM oven/bun:1-alpine AS builder
  COPY . .
  RUN bun run build:prod

FROM oven/bun:1-alpine
  RUN apk add --no-cache git ripgrep
  COPY --from=builder /app/dist/cli.mjs /app/cli.mjs
  ENTRYPOINT ["bun", "/app/cli.mjs"]
```

The Docker image includes only the single bundle file plus git and ripgrep (required by BashTool and GrepTool).
