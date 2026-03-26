---
name: realtime-patterns
description: "Covers realtime web communication patterns including WebSockets, Server-Sent Events (SSE), long polling, and WebTransport. Use when building live updates, chat, notifications, collaborative editing, streaming responses, or choosing between realtime transport mechanisms for bidirectional or server-push data flows."
---

# Realtime Patterns

## Overview

Realtime patterns enable the server to push data to connected clients without the client polling repeatedly. Choosing the right transport affects latency, scalability, complexity, and browser support.

This skill covers the four major transports, when to use each, connection lifecycle management, reconnection strategies, scaling considerations, and concrete implementation patterns — all framework-agnostic.

## Transport Decision Matrix

| Transport | Direction | Protocol | Browser Support | Best For |
|-----------|-----------|----------|-----------------|----------|
| WebSocket | Bidirectional | `ws://` / `wss://` | Universal | Chat, gaming, collaborative editing |
| Server-Sent Events (SSE) | Server → Client | HTTP/1.1+ | Universal (no IE) | Notifications, live feeds, LLM streaming |
| Long Polling | Server → Client | HTTP/1.1 | Universal | Legacy fallback, simple one-way updates |
| WebTransport | Bidirectional + Unreliable | HTTP/3 (QUIC) | Chrome, Edge | High-frequency data, media, gaming |

### Quick Selection Guide

1. **Need bidirectional messaging?** → WebSocket (or WebTransport if HTTP/3 available)
2. **Server-push only, ordered events?** → SSE
3. **Must support very old browsers or proxies that strip upgrades?** → Long Polling
4. **Need unreliable/unordered datagrams (gaming, video)?** → WebTransport
5. **Streaming LLM or AI responses?** → SSE (simplest) or WebSocket (if you also send user events)

## WebSockets

### How It Works

The client sends an HTTP Upgrade request. If the server accepts, the connection upgrades from HTTP to a persistent, full-duplex TCP connection. Both sides can send frames at any time.

```
Client                          Server
  |  GET / HTTP/1.1               |
  |  Upgrade: websocket           |
  |  Connection: Upgrade          |
  | ----------------------------→ |
  |  HTTP/1.1 101 Switching       |
  | ←---------------------------- |
  |  ← full-duplex frames →       |
```

### Client Implementation

```javascript
function createWebSocket(url, handlers) {
  const ws = new WebSocket(url);

  ws.addEventListener("open", () => {
    handlers.onOpen?.();
  });

  ws.addEventListener("message", (event) => {
    const data = JSON.parse(event.data);
    handlers.onMessage?.(data);
  });

  ws.addEventListener("close", (event) => {
    handlers.onClose?.(event.code, event.reason);
  });

  ws.addEventListener("error", () => {
    handlers.onError?.();
  });

  return {
    send: (data) => ws.send(JSON.stringify(data)),
    close: (code, reason) => ws.close(code, reason),
    get readyState() { return ws.readyState; },
  };
}
```

### Server Implementation (Node.js, no framework)

```javascript
import { WebSocketServer } from "ws";

const wss = new WebSocketServer({ port: 8080 });

wss.on("connection", (ws, req) => {
  console.log("Client connected from", req.socket.remoteAddress);

  ws.on("message", (raw) => {
    const msg = JSON.parse(raw);
    // Echo to all other clients
    for (const client of wss.clients) {
      if (client !== ws && client.readyState === 1) {
        client.send(JSON.stringify(msg));
      }
    }
  });

  ws.on("close", (code, reason) => {
    console.log("Disconnected", code, reason.toString());
  });
});
```

### When to Use WebSockets

- Chat and messaging applications
- Collaborative editing (multiplayer cursors, document sync)
- Live dashboards that also accept user input
- Gaming with low-latency bidirectional state

### When NOT to Use WebSockets

- Server-push only (SSE is simpler and uses standard HTTP)
- Infrequent updates (polling or SSE with retry is cheaper)
- Environments with aggressive proxies/firewalls that block Upgrade headers

## Server-Sent Events (SSE)

### How It Works

The client opens a standard HTTP GET request. The server holds it open and writes `text/event-stream` formatted chunks. The browser's `EventSource` API handles parsing, automatic reconnection, and last-event-ID tracking.

```
Client                          Server
  |  GET /events HTTP/1.1         |
  |  Accept: text/event-stream    |
  | ----------------------------→ |
  |  HTTP/1.1 200 OK              |
  |  Content-Type: text/event-stream
  | ←---------------------------- |
  |  data: {"price": 142.50}      |
  |  data: {"price": 143.10}      |
  |        ... (connection held)  |
```

### Client Implementation

```javascript
function createEventSource(url, handlers) {
  const source = new EventSource(url);

  source.addEventListener("open", () => {
    handlers.onOpen?.();
  });

  source.addEventListener("message", (event) => {
    const data = JSON.parse(event.data);
    handlers.onMessage?.(data);
  });

  // Named events
  source.addEventListener("notification", (event) => {
    handlers.onNotification?.(JSON.parse(event.data));
  });

  source.addEventListener("error", () => {
    // EventSource reconnects automatically
    handlers.onError?.(source.readyState);
  });

  return {
    close: () => source.close(),
    get readyState() { return source.readyState; },
  };
}
```

### Server Implementation (Node.js, no framework)

```javascript
import http from "node:http";

http.createServer((req, res) => {
  if (req.url === "/events") {
    res.writeHead(200, {
      "Content-Type": "text/event-stream",
      "Cache-Control": "no-cache",
      "Connection": "keep-alive",
    });

    // Send a comment to prevent proxy buffering
    res.write(": heartbeat\n\n");

    const interval = setInterval(() => {
      const payload = JSON.stringify({ time: Date.now() });
      res.write(`id: ${Date.now()}\n`);
      res.write(`data: ${payload}\n\n`);
    }, 1000);

    req.on("close", () => clearInterval(interval));
  }
}).listen(8080);
```

### SSE Event Format

```
id: 42
event: notification
retry: 5000
data: {"title": "New message", "body": "Hello"}

```

| Field | Purpose |
|-------|---------|
| `id:` | Sets `lastEventId`; sent back on reconnect via `Last-Event-ID` header |
| `event:` | Named event type (default is `message`) |
| `retry:` | Reconnection interval in ms (browser respects this) |
| `data:` | Payload (multiple `data:` lines are joined with newlines) |

### When to Use SSE

- Live notifications, news feeds, score updates
- LLM / AI streaming responses (token-by-token)
- Stock tickers, monitoring dashboards (read-only)
- Any scenario where the client only needs to receive

## Long Polling

### How It Works

The client sends a normal HTTP request. The server holds the response open until new data is available (or a timeout expires), then responds. The client immediately sends a new request, creating a loop.

```
Client                          Server
  |  GET /poll                    |
  | ----------------------------→ |
  |        ... (server waits) ... |
  |  200 OK {new data}           |
  | ←---------------------------- |
  |  GET /poll                    |  ← client immediately re-requests
  | ----------------------------→ |
```

### Implementation

```javascript
async function longPoll(url, onMessage) {
  let lastId = null;

  while (true) {
    try {
      const params = lastId ? `?after=${lastId}` : "";
      const res = await fetch(`${url}${params}`);

      if (!res.ok) {
        await delay(5000);
        continue;
      }

      const data = await res.json();
      lastId = data.id;
      onMessage(data);
    } catch {
      // Network error — back off then retry
      await delay(5000);
    }
  }
}

function delay(ms) {
  return new Promise((r) => setTimeout(r, ms));
}
```

### When to Use Long Polling

- Legacy browser support required (works everywhere HTTP works)
- Proxies/firewalls block WebSocket upgrades and SSE streaming
- Low update frequency where connection overhead is acceptable

### Drawbacks

- Higher latency than WebSocket or SSE (each message needs a new HTTP round-trip)
- More server resources per client (many open connections waiting)
- No multiplexing — each logical channel needs its own polling loop

## WebTransport

### How It Works

WebTransport runs over HTTP/3 (QUIC). It provides bidirectional streams (reliable, ordered) and datagrams (unreliable, unordered) through a single connection. It avoids head-of-line blocking that affects WebSocket over TCP.

### Client Implementation

```javascript
async function createWebTransport(url) {
  const transport = new WebTransport(url);
  await transport.ready;

  // Bidirectional stream (reliable)
  const stream = await transport.createBidirectionalStream();
  const writer = stream.writable.getWriter();
  const reader = stream.readable.getReader();

  await writer.write(new TextEncoder().encode("hello"));

  const { value } = await reader.read();
  console.log(new TextDecoder().decode(value));

  // Datagrams (unreliable, unordered — great for gaming)
  const dgWriter = transport.datagrams.writable.getWriter();
  await dgWriter.write(new Uint8Array([0x01, 0x02]));

  return transport;
}
```

### When to Use WebTransport

- High-frequency state sync (multiplayer gaming, VR/AR)
- Media streaming where dropped frames are acceptable
- Scenarios needing both reliable streams and unreliable datagrams
- When head-of-line blocking from TCP-based WebSocket is a bottleneck

### Current Limitations

- Requires HTTP/3 server support
- Browser support is limited (Chromium-based browsers; Firefox experimental)
- Server ecosystem is still maturing

## Connection Lifecycle Management

Every realtime connection moves through predictable states. Handling each state explicitly prevents silent failures and resource leaks.

### State Machine

```
[DISCONNECTED] → connect() → [CONNECTING]
                                  |
                         success /  \ failure
                               /    \
                    [CONNECTED]      [RECONNECTING]
                         |               |
                    close / error    backoff + retry
                         |               |
                   [DISCONNECTING]   → [CONNECTING]
                         |
                   [DISCONNECTED]
```

### Key Implementation Rules

1. **Track state explicitly** — maintain a `state` variable (`disconnected`, `connecting`, `connected`, `reconnecting`) and guard transitions
2. **Reset attempt counter on success** — when `onopen` fires, reset backoff state
3. **Clean up on disconnect** — remove event listeners, clear timers, close the underlying socket
4. **Emit lifecycle events** — let consumers react to `connected`, `disconnected`, `reconnecting`, `failed`
5. **Prevent concurrent connects** — guard `connect()` if already connecting or connected

## Reconnection Strategies

| Strategy | Formula | Use Case |
|----------|---------|----------|
| Fixed interval | `delay = 5000` | Simple, predictable; risk of thundering herd |
| Linear backoff | `delay = base * attempt` | Gradual ramp; still predictable |
| Exponential backoff | `delay = base * 2^attempt` | Standard for most realtime apps |
| Exponential + jitter | `delay = (base * 2^attempt) * random(0.5, 1)` | Best default — prevents synchronized reconnects |
| Fibonacci backoff | `delay = fib(attempt) * base` | Gentler ramp than exponential |

### Reconnection Checklist

1. **Cap the maximum delay** — do not let backoff grow past 30-60 seconds
2. **Cap the maximum attempts** — give up after N tries and surface the failure to the user
3. **Add jitter** — without it, all clients reconnect simultaneously after an outage (thundering herd)
4. **Respect server signals** — SSE `retry:` field, WebSocket close codes (4000+ for app errors)
5. **Re-authenticate on reconnect** — tokens may have expired while disconnected
6. **Replay missed events** — send last known event ID so the server can catch the client up
7. **Detect network state** — use `navigator.onLine` and `online`/`offline` events to pause/resume

## Heartbeat and Keep-Alive

Idle connections can be silently dropped by proxies, load balancers, or mobile OS network managers. Heartbeats detect dead connections early.

### Pattern

```javascript
function startHeartbeat(ws, intervalMs = 30000, timeoutMs = 5000) {
  let pongReceived = true;

  const timer = setInterval(() => {
    if (!pongReceived) {
      ws.close(4000, "heartbeat timeout");
      clearInterval(timer);
      return;
    }
    pongReceived = false;
    ws.send(JSON.stringify({ type: "ping" }));
  }, intervalMs);

  // Application-level pong handler
  ws.addEventListener("message", (event) => {
    const msg = JSON.parse(event.data);
    if (msg.type === "pong") pongReceived = true;
  });

  return () => clearInterval(timer);
}
```

Typical intervals: 15-30s for web apps, 10-15s for mobile (aggressive network management).

## Message Patterns

### Request-Response Over WebSocket

Assign each outbound message a unique `id`. Store a pending-promise map. When a response arrives with a matching `id`, resolve the promise. Always add a timeout to reject stale requests.

```javascript
const pending = new Map();
let nextId = 0;

function request(ws, method, params, timeoutMs = 10000) {
  const id = ++nextId;
  return new Promise((resolve, reject) => {
    pending.set(id, { resolve, reject });
    ws.send(JSON.stringify({ id, method, params }));
    setTimeout(() => {
      if (pending.delete(id)) reject(new Error(`Timeout: ${method}`));
    }, timeoutMs);
  });
}
```

### Pub-Sub Over WebSocket

Clients send `subscribe` / `unsubscribe` messages with a channel name. The server maintains a `Map<channel, Set<client>>` and fans out published messages only to subscribers of that channel.

## Scaling Considerations

### Horizontal Scaling Challenges

A single server can typically hold 10K-100K concurrent WebSocket connections (OS and memory dependent). Beyond that, you need multiple servers — which creates a message routing problem.

### Scaling Architecture

```
Clients → Load Balancer (sticky sessions or IP hash)
              ↓
         App Server 1  ←→  Pub/Sub Backbone  ←→  App Server 2
         (10K conns)       (Redis, NATS,         (10K conns)
                            Kafka, etc.)
```

### Key Scaling Decisions

| Concern | Approach |
|---------|----------|
| Load balancing | Sticky sessions (IP hash or cookie) for WebSocket; standard round-robin for SSE |
| Cross-server messaging | Pub/sub backbone: Redis Pub/Sub, NATS, Kafka |
| Connection limits | Monitor file descriptors (`ulimit`), tune OS TCP settings |
| Memory per connection | Minimize per-connection state; offload to shared store |
| Graceful shutdown | Drain connections on deploy: stop accepting new, close existing with reconnect signal |
| Auth on reconnect | Validate tokens on each new connection; do not trust prior session |

### Connection Draining on Deploy

```javascript
process.on("SIGTERM", () => {
  // Stop accepting new connections
  server.close();

  // Signal existing clients to reconnect (they'll hit a new instance)
  for (const client of wss.clients) {
    client.close(4001, "server restarting");
  }

  setTimeout(() => process.exit(0), 5000);
});
```

## Security Considerations

1. **Always use encrypted transports** — `wss://` for WebSocket, HTTPS for SSE
2. **Authenticate on connection** — pass tokens via query param (WebSocket) or cookie/header (SSE); validate server-side before accepting
3. **Authorize subscriptions** — just because a client is connected does not mean they can join any channel
4. **Rate-limit messages** — a malicious client can flood the server over an open WebSocket
5. **Validate payloads** — treat all incoming WebSocket messages as untrusted input
6. **Set origin checks** — verify the `Origin` header on WebSocket upgrade requests

## Related Skills

- **data-fetching-patterns** — client/server fetching strategies that complement realtime transports
- **caching-invalidation-patterns** — managing staleness when combining cache with realtime updates
- **error-handling-patterns** — structured error handling for connection failures and retries
- **performance-patterns** — optimizing throughput and latency in high-frequency update scenarios
- **state-management-patterns** — syncing realtime server state with client-side stores
