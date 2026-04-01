# Terminal Renderer Architecture

## Overview

Rather than depending on the `ink` npm package, Claude Code ships its own complete React-based terminal renderer. This custom implementation (`src/ink/`) provides a full flexbox layout engine, ANSI protocol handling, object pooling for performance, and frame-level diff optimization.

## Why a Custom Renderer?

1. **Performance control.** Terminal rendering is latency-sensitive. Custom pooling (StylePool, CharPool, HyperlinkPool) and diff algorithms minimize allocations and screen updates.
2. **Terminal protocol depth.** Full CSI/OSC/DEC/SGR handling enables features like hyperlinks (ANSI 8), iTerm2 progress bars, mouse tracking, and alternate screen mode.
3. **Layout precision.** The flexbox engine (ported from yoga-layout) gives exact control over how content flows in the terminal, including bidirectional text support.
4. **Tight integration.** The renderer integrates directly with Claude Code's event system, focus management, and selection/search features.

## Architecture

```
src/ink/
  ├── ink.tsx                    Main Ink class (lifecycle, frame management)
  ├── reconciler.ts              React Reconciler bridge
  ├── dom.ts                     Virtual DOM (element, text, comment nodes)
  ├── renderer.ts                Layout + rendering orchestration
  │
  ├── layout/                    Flexbox layout engine
  │   ├── engine.ts              Single-pass flexbox calculation
  │   ├── yoga.ts                yoga-layout API wrapper
  │   ├── geometry.ts            Layout data structures
  │   └── node.ts                Layout node abstraction
  │
  ├── termio/                    Terminal protocol handling
  │   ├── parser.ts              ANSI escape sequence parser
  │   ├── csi.ts                 Control Sequence Introducer
  │   ├── osc.ts                 Operating System Commands
  │   ├── dec.ts                 DEC private sequences
  │   ├── esc.ts                 Single-char escapes
  │   ├── sgr.ts                 Select Graphic Rendition (colors/styles)
  │   ├── ansi.ts                ANSI color codes
  │   └── tokenize.ts            Text tokenization
  │
  ├── screen.ts                  Screen buffer (2D cell grid)
  ├── render-node-to-output.ts   Element → text/style conversion
  ├── render-to-screen.ts        Output → screen state
  ├── output.ts                  Output stream abstraction
  │
  ├── components/                Built-in components
  │   ├── App.tsx                Root wrapper
  │   ├── Box.tsx                Flex container
  │   ├── Text.tsx               Text node
  │   ├── ScrollBox.tsx          Scrolling container
  │   ├── Button.tsx             Interactive button
  │   ├── Link.tsx               Hyperlink (ANSI 8)
  │   ├── RawAnsi.tsx            Raw ANSI passthrough
  │   ├── AlternateScreen.tsx    Alt screen toggle
  │   ├── Spacer.tsx             Flex spacer
  │   ├── Newline.tsx            Line break
  │   └── ErrorOverview.tsx      Error boundary
  │
  ├── events/                    Event system
  │   ├── keyboard-event.ts      Key events with modifiers
  │   ├── click-event.ts         Mouse click handling
  │   ├── focus-event.ts         Focus management
  │   └── dispatcher.ts          Global event dispatch
  │
  ├── hooks/                     React hooks
  │   ├── use-input.ts           Keyboard input
  │   └── use-stdin.ts           Raw stdin access
  │
  ├── parse-keypress.ts          Raw stdin → key events
  ├── selection.ts               Text selection state
  ├── searchHighlight.ts         Search highlighting
  ├── hit-test.ts                Click → DOM element mapping
  ├── focus.ts                   Focus management
  ├── wrap-text.ts               Word-wrap algorithm
  ├── stringWidth.ts             Emoji-aware width calculation
  ├── wrapAnsi.ts                ANSI-aware text wrapping
  ├── bidi.ts                    Bidirectional text support
  ├── colorize.ts                Color application
  ├── render-border.ts           Border rendering
  ├── clearTerminal.ts           Screen clearing
  └── log-update.ts              In-place console update
```

## Rendering Pipeline

```
React Component Tree
        │
        ▼
React Reconciler (reconciler.ts)
  Bridges React's fiber tree to the virtual DOM.
  Handles createElement, appendChild, removeChild,
  updateProps, commitUpdate, etc.
        │
        ▼
Virtual DOM (dom.ts)
  Tree of DOMElement and DOMText nodes.
  Each element has: tag, props, children, style.
        │
        ▼
Layout Engine (layout/engine.ts)
  Single-pass flexbox calculation:
  - flex-direction (row/column + reverse)
  - flex-grow / flex-shrink / flex-basis
  - align-items / align-self / justify-content
  - margin / padding / border / gap
  - width / height / min / max constraints
  - position: relative / absolute
  - display: flex / none
  - Measure functions for text nodes
        │
        ▼
Render to Output (render-node-to-output.ts)
  Converts positioned DOM nodes to styled text spans.
  Resolves colors, borders, padding.
        │
        ▼
Screen Buffer (screen.ts)
  2D grid of cells. Each cell has:
  - Character (from CharPool)
  - Style (from StylePool)
  - Hyperlink (from HyperlinkPool)
  Object pools minimize GC pressure.
        │
        ▼
Diff Algorithm
  Compares current frame to previous frame.
  Only emits ANSI sequences for changed cells.
        │
        ▼
Terminal Output
  CSI sequences for cursor positioning, colors, etc.
  Written to stdout (or WebSocket for web terminal).
```

## Performance Optimizations

### Object Pooling

```typescript
// screen.ts pools
StylePool    // Reuse identical style objects across cells
CharPool     // Reuse identical character strings
HyperlinkPool // Reuse hyperlink definitions

// Result: A 200x80 terminal screen doesn't allocate
// 16,000 style objects per frame. Most cells share
// the same few pool entries.
```

### Frame Buffering

```
Front buffer: What's currently displayed
Back buffer:  What's being rendered

1. Render new frame to back buffer
2. Diff against front buffer
3. Emit only changed cells
4. Swap buffers
```

This prevents flicker during complex re-renders.

### Line Width Caching

```typescript
// line-width-cache.ts
// String width calculation (handling emoji, CJK, etc.)
// is expensive. Cache results for repeated strings.
lineWidthCache.get("Hello 👋")  // → 8 (cached after first calc)
```

### Debounced Resize

Terminal resize events trigger layout recalculation, but are debounced to avoid redundant work during window drag-resizing.

### Lazy Measurement

Text measurement functions are only called when the layout engine actually needs dimensions, not eagerly on every render.

## Component System

### Core Components

**Box** - Flex container (the `<div>` of terminal UI):
```tsx
<Box flexDirection="column" padding={1} borderStyle="round">
  <Text>Hello</Text>
  <Text color="green">World</Text>
</Box>
```

**Text** - Styled text node:
```tsx
<Text bold color="blue" dimColor>Status: OK</Text>
```

**ScrollBox** - Scrolling container with viewport management:
```tsx
<ScrollBox height={20}>
  {/* Content taller than 20 lines scrolls */}
</ScrollBox>
```

**Link** - Terminal hyperlink (ANSI 8 protocol):
```tsx
<Link url="https://example.com">Click here</Link>
// Renders as clickable link in supporting terminals
```

### Context Providers

```
AppContext          Application-level state
StdinContext        Raw stdin stream access
TerminalSizeContext Terminal dimensions (reactive)
TerminalFocusContext Window focus state
CursorDeclarationContext Cursor positioning
ClockContext        Animation timing
```

## Input Handling

### Keyboard Events

```
Raw stdin bytes
      │
      ▼
parse-keypress.ts
  Decodes escape sequences into structured key events:
  - Character keys (a-z, 0-9, symbols)
  - Modifier keys (ctrl, alt, shift, meta)
  - Special keys (enter, tab, escape, arrows, etc.)
  - Function keys (F1-F12)
  - Paste detection (bracketed paste mode)
      │
      ▼
Event Dispatcher (dispatcher.ts)
  Routes to focused component
      │
      ▼
useInput() hook
  React hook consumed by components
```

### Mouse Events

```
DEC mouse tracking mode enabled
      │
      ▼
Click coordinates decoded
      │
      ▼
hit-test.ts
  Maps (x, y) → DOM element in the layout tree
      │
      ▼
click-event.ts
  Dispatched to target component
```

### Text Selection

```
selection.ts
  ├── Track selection start/end coordinates
  ├── Map to text content via hit-test
  ├── Handle multi-line selection
  └── Copy to clipboard via OSC 52
```

## Terminal Protocol Support

The `termio/` directory implements comprehensive terminal protocol handling:

| Protocol | File | Purpose |
|----------|------|---------|
| CSI | csi.ts | Cursor movement, scrolling, erase, colors (256 + truecolor) |
| SGR | sgr.ts | Bold, italic, underline, strikethrough, reverse, dim, etc. |
| OSC | osc.ts | Hyperlinks, iTerm2 progress, tab titles, clipboard |
| DEC | dec.ts | Alt screen, cursor visibility, mouse tracking, focus events |
| ESC | esc.ts | Single-character escape sequences |

## Web Terminal Integration

For the web interface, a separate xterm.js-based terminal connects via WebSocket:

```
Browser (xterm.js)
      │
   WebSocket
      │
Server (terminal.ts)
      │
   node-pty
      │
Shell (Claude Code CLI)
```

The web terminal supports:
- Session persistence (localStorage tokens)
- Auto-reconnect with exponential backoff
- Scrollback buffer replay on resume
- GPU rendering via WebGL addon
- Search (Ctrl+Shift+F)
- Clickable links
- Theme from CSS custom properties
