# Web Frontend Architecture

## Overview

Claude Code includes a web interface built with Next.js 14 (App Router). The frontend provides a chat UI, file browser, collaboration features, and a browser-based terminal emulator. It connects to the backend via API routes and WebSocket.

## Dual-Stack Architecture

The web layer has two distinct rendering paths:

```
1. Next.js Web App (web/)
   ├── Chat interface
   ├── File browser
   ├── Settings UI
   └── Collaboration features

2. Browser Terminal (src/server/web/)
   ├── xterm.js terminal emulator
   ├── WebSocket → PTY bridge
   └── Session persistence
```

## Directory Structure

```
web/
  ├── app/                    Next.js App Router pages
  │   ├── page.tsx            Main chat interface
  │   ├── layout.tsx          Root layout
  │   ├── accessibility/      Accessibility page
  │   └── api/                API routes (backend proxy)
  │       ├── chat/           Chat message streaming
  │       ├── exec/           Command execution
  │       ├── fs/             File system operations
  │       ├── files/          File content management
  │       ├── env/            Environment variables
  │       ├── cwd/            Working directory
  │       ├── share/          Session sharing
  │       ├── export/         Conversation export
  │       └── analytics/      Analytics events
  │
  ├── components/             React components
  │   ├── chat/               Message bubbles, input, markdown, code blocks
  │   ├── layout/             Sidebar, header, file explorer, theme
  │   ├── command-palette/    Slash command menu
  │   ├── file-viewer/        File preview
  │   ├── collaboration/      Real-time collaboration
  │   ├── adapted/            Radix UI adapted components
  │   └── ui/                 Generic primitives
  │
  ├── lib/                    Utility libraries
  │   ├── api/                Backend API client
  │   ├── collaboration/      WebSocket presence, permissions
  │   ├── workers/            Web Workers (markdown, syntax, search)
  │   ├── export/             Export formats (JSON, HTML, MD, text)
  │   ├── ink-compat/         Ink.js compatibility layer
  │   ├── platform/           Platform abstraction (Node vs Browser)
  │   └── shims/              Browser polyfills for Node.js APIs
  │
  └── e2e/                    Playwright E2E tests
      ├── chat.spec.ts
      ├── shortcuts.spec.ts
      ├── mobile.spec.ts
      └── visual.spec.ts
```

## Technology Stack

| Layer | Technology |
|-------|------------|
| Framework | Next.js 14 (App Router) |
| UI Components | Radix UI primitives |
| State | zustand + SWR |
| Animation | framer-motion |
| Code Editing | CodeMirror 6 |
| Syntax Highlighting | shiki |
| Markdown | react-markdown + remark-gfm |
| Terminal | xterm.js 5.5 |
| Icons | lucide-react |
| Testing | Vitest + Playwright |

## API Routes

The Next.js API routes proxy requests to the Claude Code backend:

| Route | Method | Purpose |
|-------|--------|---------|
| `/api/chat` | POST | Stream chat messages |
| `/api/exec` | POST | Execute commands |
| `/api/fs/stat` | GET | File stat |
| `/api/fs/list` | GET | Directory listing |
| `/api/fs/read` | GET | Read file content |
| `/api/fs/write` | POST | Write file |
| `/api/fs/mkdir` | POST | Create directory |
| `/api/fs/rm` | DELETE | Remove file/directory |
| `/api/files/*` | Various | File content management |
| `/api/env` | GET | Environment variables |
| `/api/cwd` | GET/POST | Working directory |
| `/api/share` | POST | Share session |
| `/api/export` | GET | Export conversation |
| `/api/analytics/*` | POST | Analytics events |

## Web Workers

Heavy computation is offloaded to Web Workers:

```
lib/workers/
  ├── markdown-worker.ts     Markdown parsing and rendering
  ├── highlight-worker.ts    Syntax highlighting (shiki)
  └── search-worker.ts       Full-text search
```

This keeps the main thread responsive during large markdown renders or code highlighting.

## Browser Terminal

The browser-based terminal (`src/server/web/terminal.ts`) provides a full PTY experience:

### WebSocket Protocol

```
Server → Client:
  { type: "session", token: string }   Session created
  { type: "resumed", token: string }   Session resumed
  { type: "pong" }                     Keepalive response
  { type: "error", message: string }   Error
  { type: "exit", code: number }       Process exit

Client → Server:
  { type: "resize", cols, rows }       Terminal resize
  { type: "ping" }                     Keepalive
  raw bytes                            Terminal input
```

### Session Persistence

```
1. Server assigns session token
2. Client stores token in localStorage
3. On reconnect: send token to resume
4. Server replays scrollback buffer
5. PTY continues where it left off
```

### xterm.js Addons

| Addon | Purpose |
|-------|---------|
| FitAddon | Auto-fit terminal to container size |
| WebLinksAddon | Clickable URLs |
| SearchAddon | Ctrl+Shift+F search |
| Unicode11Addon | Full Unicode support |
| WebglAddon | GPU-accelerated rendering |

## Ink Compatibility Layer

`lib/ink-compat/` bridges Claude Code's Ink terminal components to the browser:

```
Ink components (terminal)
        ↓
ink-compat adapter
        ↓
HTML/CSS rendering (browser)
```

This allows sharing component logic between the terminal and web interfaces.

## Collaboration Features

```
lib/collaboration/
  ├── socket.ts        WebSocket connection management
  ├── presence.ts      User presence (who's online)
  └── permissions.ts   Collaboration permissions

Components:
  └── collaboration/   Real-time collaboration UI
```

## Deployment

### Vercel

```json
// vercel.json
{
  "framework": "nextjs",
  "routes": [
    { "src": "/api/(.*)", "dest": "/web/api/$1" },
    { "src": "/health", "dest": "/web/api/health" },
    { "src": "/(.*)", "dest": "/web/$1" }
  ]
}
```

### Security Headers

```
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
Referrer-Policy: strict-origin-when-cross-origin
Cache-Control: public, max-age=31536000, immutable (static assets)
```

### Performance

Lighthouse CI enforces:
- Performance: >= 0.9
- Accessibility: >= 0.9
- Best Practices: >= 0.9
- SEO: >= 0.8

## Export Formats

Conversations can be exported in multiple formats:

```
lib/export/
  ├── json.ts       Raw JSON (full fidelity)
  ├── html.ts       Styled HTML document
  ├── markdown.ts   GitHub-flavored markdown
  └── plaintext.ts  Plain text
```
