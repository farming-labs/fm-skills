---
name: developer-experience
description: "Covers dev server architecture, Hot Module Replacement (HMR), error overlays, file watching, and transform pipelines. Use when setting up dev servers, debugging HMR issues, building developer tooling, configuring Vite or similar tools, or optimizing development workflow feedback loops."
---

# Developer Experience (DX) Patterns

## Overview

Developer Experience encompasses the tools, workflows, and feedback loops that make building applications fast and enjoyable. This skill covers dev server architecture, HMR internals, error overlays, file watching, and transform pipelines.

## Prerequisites

- Understanding of [build-pipelines-bundling](../build-pipelines-bundling/SKILL.md)
- Familiarity with [meta-frameworks-overview](../meta-frameworks-overview/SKILL.md)
- Basic knowledge of module systems (ESM, CommonJS)

---

## The Evolution of Dev Servers

```
TRADITIONAL BUNDLER-BASED DEV (Webpack, 2015-2020):

Source Files ──► Full Bundle ──► Dev Server ──► Browser
     │              │                              │
     │         (slow!)                             │
     └── Change ───►│◄──── Rebuild entire bundle ──┘
              (seconds to minutes)


MODERN UNBUNDLED DEV (Vite, 2020+):

Source Files ──► Dev Server ──► Browser (native ESM)
     │              │                │
     │         (instant!)            │
     └── Change ───►│◄─── Only transform changed file
              (milliseconds)
```

**Why traditional bundling was slow:** Cold start required scanning all files, building the complete dependency graph, transforming everything, and bundling into one or few files. For 10,000+ modules: 30s-2min startup, 5-30s per save.

**The unbundled insight:** Browsers support ES Modules natively. No bundling step needed during development — on-demand transformation, native browser caching, and true per-module HMR.

---

## Hot Module Replacement (HMR)

### HMR vs Full Reload

| Aspect | Full Reload | HMR |
|--------|-------------|-----|
| State | Lost (forms, scroll, modals) | Preserved |
| Speed | Rebuild + reload | Transform single file |
| Mechanism | Browser navigation | WebSocket + module patch |

### HMR Protocol

```javascript
// 1. SERVER → CLIENT: File changed
{ type: 'update', updates: [{
    type: 'js-update',
    path: '/src/components/Button.tsx',
    acceptedPath: '/src/components/Button.tsx',
    timestamp: 1699123456789
}]}

// 2. CLIENT: Fetches new module
GET /src/components/Button.tsx?t=1699123456789

// 3. CLIENT: Replaces old module via accept callback
import.meta.hot.accept(() => { /* re-render */ });

// 4. If no boundary found → full reload
{ type: 'full-reload', path: '/src/main.tsx' }
```

### HMR Boundaries

A boundary is a module that "accepts" hot updates. Updates bubble up the import chain to the nearest boundary. If none is found, a full page reload occurs.

```javascript
// ✓ HMR BOUNDARY - accepts updates (auto-injected by framework plugins)
if (import.meta.hot) {
  import.meta.hot.accept();
}

// Update propagation:
//     main.tsx (boundary)
//         │
//    ┌────┴────┐
//  App.tsx   utils.ts ◄── Change here
//    │                     │
//    └────────────────────►│
//  Update bubbles to nearest boundary (main.tsx)
```

### React Fast Refresh

| Preserved | Triggers Full Remount |
|-----------|-----------------------|
| `useState` values | Changed hooks order |
| `useRef` values | Changed hook dependencies |
| Component local state | Class components |
| | Non-component exports mixed with components |

**Best practice:** Keep component files pure — only export React components. Move constants and utilities to separate files to avoid breaking Fast Refresh.

---

## Vite Architecture

### Core Components

```
┌─────────────────────────────────────────────────┐
│                      VITE                        │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐│
│  │  Connect   │  │  Module    │  │  Plugin    ││
│  │  Server    │  │  Graph     │  │  Container ││
│  │ HTTP/WS    │  │ Deps/HMR   │  │ Transform  ││
│  └────────────┘  └────────────┘  └────────────┘│
│         └──────────────┼──────────────┘          │
│                  ┌─────┴─────┐                   │
│                  │  esbuild  │                   │
│                  │ Pre-bundle │                   │
│                  │ + TS/JSX  │                   │
│                  └───────────┘                   │
└─────────────────────────────────────────────────┘
```

### Dependency Pre-Bundling

node_modules often are not ESM-ready (CommonJS, hundreds of small files). Vite pre-bundles with esbuild on first run:

1. Scans code for bare imports
2. Bundles each dependency into a single ESM file (`node_modules/.vite/deps/`)
3. Rewrites imports to pre-bundled versions with content-hash query params
4. Cached until dependency changes

### Transform Pipeline (Request Lifecycle)

```javascript
// 1. Browser requests: GET /src/components/Button.tsx
// 2. Resolve: determine file path
// 3. Load: read file contents
// 4. Transform: compile TS/JSX, rewrite imports
// 5. Serve with headers:
//    - Dependencies: max-age=31536000 (long cache, hash-versioned)
//    - Source files: no-cache (timestamp-busted on change)
```

### Import Rewriting

```javascript
// Your code:
import React from 'react';
import Button from './Button';
import styles from './App.module.css';

// After Vite transforms:
import React from '/node_modules/.vite/deps/react.js?v=abc123';
import Button from '/src/Button.tsx?t=1699123456789';
import styles from '/src/App.module.css?t=1699123456789';

// ?v=hash → content hash for deps (long cache)
// ?t=timestamp → cache bust for source files on change
```

---

## Error Overlays

### Error Types and Handling

| Error Type | When | Example | Overlay Shows |
|-----------|------|---------|---------------|
| Compile-time | During transform | Syntax error, TS error | Code frame + location |
| Runtime | In browser | `TypeError: cannot read 'map' of undefined` | Stack trace mapped to source |
| HMR | During hot update | Failed module execution | Error + suggestion |

**Source maps** are critical: they map compiled positions back to original source files, making error locations actionable (e.g., `src/Button.tsx:42:15` instead of `bundle.js:15847:23`).

---

## File Watching

### Key Concepts

- **Chokidar** is the standard file watcher used by most dev servers
- Always debounce rapid changes (IDEs often save multiple times quickly, ~50ms debounce)
- Ignore `node_modules/` and `.git/`

### Platform Differences

| Platform | Mechanism | Notes |
|----------|-----------|-------|
| macOS | FSEvents | Fast, kernel-level, handles large dirs |
| Linux | inotify | Good, but has watch limit (`max_user_watches`) |
| Windows | ReadDirectoryChangesW | Can miss rapid changes; WSL2 cross-fs issues |

**Polling fallback:** When native watching fails, use `usePolling: true` with configurable interval.

---

## Module Graph

The module graph tracks import relationships and HMR boundaries.

```javascript
class ModuleNode {
  url;                    // Module URL
  file;                   // File path
  importers;              // Set: who imports this module
  importedModules;        // Set: what this module imports
  acceptedHmrDeps;        // Set: HMR boundaries
  transformResult;        // Cached transform (invalidated on change)
}
```

**On file change:**
1. Invalidate module's transform cache
2. Walk importers to find nearest HMR boundary
3. If boundary found → send WebSocket update for that boundary
4. If no boundary → send full-reload

---

## CSS Hot Update

**Plain CSS:** Update `<link>` href with cache-busting param. Browser fetches new stylesheet and swaps atomically — no flash, no JS re-execution.

**CSS Modules:** Trickier — must update both the stylesheet (visual) and the JS module (class name mapping). Vite wraps CSS modules in JS with `import.meta.hot.accept()`.

---

## TypeScript in Development

### Transform-Only Strategy

Full `tsc` is slow (type-checks entire project). Dev servers use **esbuild** or **SWC** for transform-only (strip types, convert JSX) at <1ms per file.

| Tool | Language | Transform Speed (10K files) |
|------|----------|-----------------------------|
| Babel | JS | ~45s |
| esbuild | Go | ~0.5s |
| SWC | Rust | ~0.3s |

**Type checking** runs separately via `tsc --noEmit` in background (e.g., `vite-plugin-checker`).

---

## Proxy and API Mocking

### Dev Proxy

```javascript
// vite.config.ts
export default {
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:8080',
        changeOrigin: true,
      },
      '/ws': {
        target: 'ws://localhost:8080',
        ws: true,
      },
    },
  },
};
// Same-origin from browser's view → no CORS issues
```

### Mock Service Worker (MSW)

Intercepts requests at the network level via Service Worker. Define handlers for `GET`, `POST`, etc. with simulated responses, errors, and latency. Starts conditionally in development only.

---

## Zero-Config Defaults

Modern dev servers provide sensible defaults with escape hatches:

| Default | Override |
|---------|----------|
| Entry: `index.html` | `root` option |
| Output: `dist/` | `build.outDir` |
| Port: 5173 | `server.port` |
| TypeScript: works OOTB | `tsconfig.json` |
| CSS: just import | Preprocessor config |
| Path aliases | `resolve.alias` |

---

## Tool Comparison

```
┌────────────┬──────────┬────────────┬─────────────┬─────────┐
│            │ Vite     │ Turbopack  │ Bun         │ Parcel  │
├────────────┼──────────┼────────────┼─────────────┼─────────┤
│ Language   │ TS/JS    │ Rust       │ Zig         │ Rust/JS │
│ Approach   │ Unbundled│ Incremental│ All-in-one  │ Zero-cfg│
│ Dev Mode   │ ESM      │ Bundled    │ ESM         │ ESM     │
│ Framework  │ Agnostic │ Next.js    │ Agnostic    │ Agnostic│
│ Plugins    │ Rollup   │ Webpack*   │ Limited     │ Own     │
│ Maturity   │ Stable   │ Beta       │ Stable      │ Stable  │
└────────────┴──────────┴────────────┴─────────────┴─────────┘
```

---

## Building a Minimal Dev Server

> **For framework authors.** This section shows the core patterns for building DX tooling. Adapt based on your framework's bundler integration and target audience.

### Core Architecture Pattern

```javascript
class DevServer {
  constructor(root) {
    this.root = root;
    this.moduleGraph = new Map();
    this.clients = new Set();       // WebSocket connections
  }

  async start(port = 3000) {
    // HTTP server for transformed files
    this.httpServer = createServer(this.handleRequest.bind(this));

    // WebSocket for HMR notifications
    this.wss = new WebSocketServer({ server: this.httpServer });
    this.wss.on('connection', (ws) => {
      this.clients.add(ws);
      ws.on('close', () => this.clients.delete(ws));
    });

    // File watcher triggers HMR
    this.watcher = chokidar.watch(this.root, {
      ignored: /node_modules/,
      ignoreInitial: true,
    });
    this.watcher.on('change', this.handleFileChange.bind(this));
  }

  async handleRequest(req, res) {
    // HTML: inject HMR client script
    // TS/JSX: transform with esbuild, rewrite imports, serve as JS
    // CSS: serve directly (or wrap CSS Modules in JS)
  }

  handleFileChange(file) {
    this.moduleGraph.delete(file);  // invalidate cache
    this.broadcast({                // notify connected browsers
      type: 'update',
      path: file.replace(this.root, ''),
      timestamp: Date.now(),
    });
  }

  broadcast(message) {
    const data = JSON.stringify(message);
    for (const client of this.clients) {
      client.send(data);
    }
  }
}
```

### Transform Pipeline Pattern

```javascript
class TransformPipeline {
  transformers = [];  // sorted by enforce: pre → normal → post
  cache = new Map();

  use(transformer) { /* add and sort */ }

  async transform(code, id) {
    // Check cache → run matching transformers in order → cache result
    // Each transformer: { name, enforce, filter(id), transform(code, id) }
  }

  invalidate(id) { /* clear cached transforms for file */ }
}

// Typical transformer chain:
// 1. esbuild (pre): TS/JSX → JS
// 2. react-refresh (post): inject Fast Refresh wrappers
// 3. import-rewriter (post): rewrite bare imports to pre-bundled deps
```

### HMR Client Runtime Pattern

```javascript
// Injected into browser, connects via WebSocket
const socket = new WebSocket('ws://' + location.host + '/__hmr');

socket.onmessage = async (event) => {
  const data = JSON.parse(event.data);
  switch (data.type) {
    case 'update':
      // Fetch new module with cache-busting timestamp
      await import(data.path + '?t=' + data.timestamp);
      break;
    case 'full-reload':
      location.reload();
      break;
    case 'error':
      showErrorOverlay(data.error);
      break;
  }
};
```

---

## Related Skills

- See [build-pipelines-bundling](../build-pipelines-bundling/SKILL.md) for bundler architecture
- See [meta-frameworks-overview](../meta-frameworks-overview/SKILL.md) for framework integration
- See [universal-javascript-runtimes](../universal-javascript-runtimes/SKILL.md) for server architecture
