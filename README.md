# OpenClaw-documentation



# OpenClaw Architecture Pattern

## Pattern: Event-Driven Architecture (EDA) with Hub-and-Spoke Topology

OpenClaw follows an **Event-Driven Architecture (EDA)** pattern, structured around a **Hub-and-Spoke** topology. The Gateway acts as the central event broker. All system participants — clients, nodes, messaging platforms, agents, and tools — connect exclusively to the Gateway. No spoke communicates directly with another spoke.

The transport layer is a **persistent WebSocket connection** using JSON text frames. This distinguishes OpenClaw from a conventional HTTP request-response system: the connection persists across exchanges, and the Gateway pushes events to connected parties without being polled.

---

## Hub-and-Spoke Topology

```
                    ┌─────────────────────────────┐
                    │                             │
  [Messaging        │                             │        [Agent]
   Platforms] ──────┤                             ├──────
  WhatsApp,         │       Gateway (Hub)         │        Session + reasoning
  Telegram…         │                             │
                    │   Routes events over        │        [Tools / Models]
  [Clients] ────────┤   persistent WebSocket      ├──────
  CLI, Web UI,      │                             │        LLM, APIs, actions
  App               │                             │
                    │                             │        [Canvas / WebChat]
  [Nodes] ──────────┤                             ├──────
  macOS, iOS,       │                             │        UI host
  Android           │                             │
                    └─────────────────────────────┘
```

**Key rule:** All communication flows through the Gateway. No spoke talks directly to another spoke.

---

## WebSocket Event Lifecycle

The runtime behaviour of OpenClaw follows a five-phase event lifecycle over a single persistent WebSocket connection.

```
Client                      Gateway                    Agent / Tool
  │                            │                            │
  │── req:connect ────────────>│                            │
  │<─ res (ok) ────────────────│                            │
  │                            │                            │
  │<─ event:presence ──────────│  (async, ongoing)          │
  │<─ event:heartbeat ─────────│  (async, ongoing)          │
  │                            │                            │
  │── req:agent ──────────────>│                            │
  │<─ res (accepted) ──────────│  ← ack ≠ answer            │
  │                            │── invoke agent + tools ───>│
  │                            │<─ chunks / tool results ───│
  │                            │                            │
  │<╌ event:agent (chunk 1) ───│  (streamed)                │
  │<╌ event:agent (chunk 2) ───│  (streamed)                │
  │<╌ event:agent (chunk N) ───│  (streamed)                │
  │                            │                            │
  │<─ final completion ────────│                            │
  │                            │                            │
```

**Legend:**
- `──>` Client-initiated (synchronous call)
- `<──` Gateway push (synchronous response)
- `<╌╌` Async / streamed event

---

## Key Architectural Properties

### 1. Event-driven, not request-response

OpenClaw does not follow a naive HTTP request-response model. The Gateway emits events independently of client requests. Clients must remain connected and listen for incoming events such as `event:agent`, `event:presence`, `event:heartbeat`, and `event:cron`.

### 2. Acknowledgment is not an answer

When a client sends `req:agent`, the Gateway immediately returns an accepted acknowledgment (`res: accepted`). The actual response arrives later, streamed as a sequence of `event:agent` frames. This non-blocking design allows the system to handle long-running agent tasks without blocking the connection.

### 3. Streaming over chunked events

Agent responses are delivered as multiple `event:agent` frames rather than a single payload. This enables progressive rendering on the client and supports long-form outputs from the underlying language model.

### 4. Persistent WebSocket transport

All communication uses a single, persistent WebSocket connection per participant. JSON text frames are used throughout. Nodes connecting to the Gateway must include `role: "node"` in their connect frame to distinguish themselves from regular clients.

### 5. Gateway as control plane

The Gateway is the control plane of the entire system. It is responsible for:

- Maintaining provider connections to messaging platforms (WhatsApp, Telegram, Discord, etc.)
- Exposing a typed WebSocket API
- Validating inbound frames using JSON Schema
- Routing sessions between clients, agents, and nodes
- Emitting system events: `agent`, `chat`, `presence`, `health`, `heartbeat`, `cron`

---

## Event Types

| Event | Direction | Description |
|---|---|---|
| `req:connect` | Client → Gateway | Initiate connection and register participant |
| `res (ok)` | Gateway → Client | Connection accepted |
| `event:presence` | Gateway → Client | Async presence broadcast |
| `event:heartbeat` | Gateway → Client | Keep-alive tick |
| `req:agent` | Client → Gateway | Submit a user message for agent processing |
| `res (accepted)` | Gateway → Client | Immediate acknowledgment of agent request |
| `event:agent` | Gateway → Client | Streamed response chunk from the agent |
| Final completion | Gateway → Client | Metadata marking end of agent response |

---

## Node Role

Nodes are device-hosted participants (macOS, iOS, Android, or headless) that connect to the same WebSocket server as regular clients but declare `role: "node"` during the connect handshake. Nodes expose device-level commands including:

- `canvas.*` — Canvas and UI operations
- `camera.*` — Camera access
- `screen.record` — Screen recording
- `location.get` — Device location

---

## Summary

OpenClaw's architecture can be characterised as follows:

- **Pattern:** Event-Driven Architecture (EDA)
- **Topology:** Hub-and-Spoke (Gateway as central broker)
- **Transport:** Persistent WebSocket, JSON text frames
- **Interaction model:** Asynchronous, streaming — not synchronous request-response
- **Key insight:** The Gateway acknowledges requests immediately and delivers responses as a stream of events. This decouples message receipt from message processing and supports long-running agent tasks natively.

# OpenClaw Runtime Behavior

## The four actors

A user sends a message on WhatsApp (or Telegram, or web). It hits the **Gateway**, which is the central hub. The Gateway talks to the **Agent** (the AI brain), and the Agent can call out to **Tools or Nodes** (like a camera, canvas, or screen recorder) to do things.

```
[User / Messaging Platform] ──> [Gateway] ──> [Agent] ──> [Tools / Nodes]
```

---

## The key insight: this is not HTTP

The naive assumption is that OpenClaw works like a normal HTTP API — you send a request, you wait, you get one response back. It does not work that way.

**HTTP (conventional):**
```
Client  ──── req ────>  Server
Client  <─── res ────   Server
        (one request, one response, connection closes)
```

**OpenClaw (WebSocket + EDA):**
```
Client  ──── req:agent ──────────────>  Gateway
Client  <─── ack (accepted) ───────────  Gateway
Client  <╌╌╌ event:agent (chunk 1) ───  Gateway
Client  <╌╌╌ event:agent (chunk 2) ───  Gateway
Client  <╌╌╌ event:agent (chunk N) ───  Gateway
Client  <─── completion metadata ──────  Gateway
        (one request, many events, connection stays open)
```

The Gateway keeps the WebSocket connection open and streams multiple events back to the client. The AI's response arrives in chunks — like watching someone type in real time. It is not "ask and hang up," it is "stay on the line."

---

## The full connection lifecycle

This is the complete sequence from the moment a client connects until the conversation is done.

```
Client                      Gateway                    Agent / Tool
  │                            │                            │
  │  ── Phase 1: Connect ──────────────────────────────     │
  │                            │                            │
  │── req:connect ────────────>│                            │
  │   (role: "node")           │                            │
  │<─ res(ok) ─────────────────│                            │
  │                            │                            │
  │  ── Phase 2: Alive ────────────────────────────────     │
  │                            │                            │
  │<─ event:presence ──────────│  (who else is online)      │
  │<─ event:tick ──────────────│  (heartbeat, ongoing)      │
  │                            │                            │
  │  ── Phase 3: Request ──────────────────────────────     │
  │                            │                            │
  │── req:agent ──────────────>│                            │
  │<─ ack (accepted) ──────────│  ← not the answer          │
  │                            │── invoke agent ───────────>│
  │                            │   (+ tool/node calls)      │
  │                            │<─ result ──────────────────│
  │                            │                            │
  │  ── Phase 4: Stream ───────────────────────────────     │
  │                            │                            │
  │<╌╌ event:agent (chunk 1) ──│                            │
  │<╌╌ event:agent (chunk 2) ──│                            │
  │<╌╌ event:agent (chunk N) ──│                            │
  │                            │                            │
  │  ── Phase 5: Done ─────────────────────────────────     │
  │                            │                            │
  │<─ completion metadata ─────│  (token count, timing…)    │
  │                            │                            │
  │        [WebSocket stays open for next message]          │
```

### Phase 1 — Connect

Before anything happens, the client opens a WebSocket connection to the Gateway. It sends `req:connect` and includes `role: "node"` to identify itself. The Gateway validates this and responds with `res(ok)`. The connection is now established and stays open.

### Phase 2 — Alive

Once connected, the Gateway starts sending background events without being asked. `event:presence` tells the client who else is online. `event:tick` is a heartbeat — a "still here?" pulse that keeps the connection alive. This happens continuously in the background, not in response to any user action.

### Phase 3 — Request

The user sends a message. The client packages it as `req:agent` and sends it over the WebSocket. The Gateway immediately replies with an `ack` saying "accepted" — this is not the answer, it is confirmation that the request was received. The Gateway then forwards it to the Agent for processing. If the Agent needs external information, it calls a Tool or Node and waits for the result.

### Phase 4 — Stream

The Agent's response does not come back as one large payload. The Gateway streams it as multiple `event:agent` messages — chunk by chunk. The user sees the response building in real time, word by word.

### Phase 5 — Done

When the Agent finishes, the Gateway sends a final completion message with metadata (token count, timing, etc.). The WebSocket connection stays open for the next message.

---

## Transport

The transport layer is **WebSocket with JSON text frames** throughout. Every arrow in the sequence diagram above is a JSON object flowing over a single persistent connection. There is no HTTP polling, no connection teardown between messages, no single-response model.
