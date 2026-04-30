# OpenClaw-documentation

## What is OpenClaw?
OpenClaw is essentially a connector or middleware that facilitates communication between different platforms, tools, and applications. Think of it as a translator or intermediary that ensures data from one system is understood and processed by another. It’s built with flexibility in mind, making it suitable for a wide range of industries, from e-commerce to cloud computing.

## What are the Key Features?
•	Data Integration: OpenClaw allows systems with different protocols, data formats, or platforms to communicate. For instance, it might enable a CRM system to talk to an email marketing tool, or a POS system to sync data with an inventory management platform. 

•	Automation: With its real-time data syncing, OpenClaw can automate workflows. This is especially useful in environments like marketing or e-commerce, where processes like order processing, customer updates, and inventory management need to happen automatically. 

•	Open Source: Being open-source means anyone can inspect, modify, or contribute to the project. This flexibility has made it popular for developers looking for cost-effective integration solutions. 

•	Customization: OpenClaw allows users to customize integrations to fit specific needs, making it adaptable across a wide variety of industries and use cases.







## Who Does OpenClaw Connect To?
•	Cloud platforms (like AWS, Google Cloud, Azure) 
•	Web services (such as APIs for social media, CRMs, and analytics) 
•	E-commerce platforms (like Shopify, WooCommerce, Magento) 
•	Payment gateways (PayPal, Stripe, etc.) 
•	Third-party apps and tools (email marketing platforms, customer service tools) 
By linking these systems together, OpenClaw enables cross-platform data flow and ensures that the relevant information from one system is pushed to another where it’s needed, without human intervention.

## Why Does OpenClaw Exist?
•	OpenClaw exists to simplify complex integrations. Traditionally, connecting systems and automating workflows would require a significant amount of custom coding, time, and expertise. OpenClaw eliminates the need for this, providing an easy-to-use platform for system integration. It reduces manual work, increases efficiency, and minimizes errors that come with manual data transfers.

•	In conclusion, OpenClaw is a powerful tool for businesses and developers who need to make their systems talk to each other without building everything from scratch. Its role in connecting various applications and platforms is crucial in today’s data-driven world, where automation and seamless data exchange are key to maintaining competitiveness and operational efficiency.



## **OpenClaw Architecture Pattern**

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
![Sequence diagram](images/OpenClaw%20Runtime%20Behavior%20Sequence%20diagram.png)
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

## **Deployment, Trust Boundaries, and Security**
This deployment diagram outlines a highly secure, multi-tier microservices architecture for the OpenClaw application. It is designed around the principle of "Defense in Depth" and a Zero Trust model, separating different components into distinct network zones (Trust Boundaries) to minimize the attack surface.
```
+-----------------------+                    +---------------------------+
| UserDevice [Zone=Internet]|                    | PublicGateway [Zone=DMZ]  |
|                       |                    |                           |
|  +----------------+   | <<HTTPS/TLS 1.3>>   | +-----------------------+ |
|  | Web Browser    | - - - - - - - - - - - - > | API Gateway/              |
|  +----------------+   |                    | Load Balancer           | |
|                       |                    | +-----------------------+ |
|  +----------------+   |                    |                           |
|  | Mobile App     |   |                    +---------------------------+
|  +----------------+   |                                  |
|                       |                                  | <<Authenticated internal RPC>>
+-----------------------+                                  V
                                                +-------------------------------+
                                                | MicroservicesCluster [Zone=SecureOpenClawVNET] |
                                                |                               |
                                                | +---------------------------+ |
                                                | | MicroservicesRuntime      | |
                                                | |                           | |
                                                | |  +---------+ +---------+  | |
                                                | |  | Auth    | | Secrets  |  | |
                                                | |  | Service | | Mgr      |  | |
                                                | |  +---------+ +---------+  | |
                                                | |      ^           ^        | |
                                                | |      |<<mTLS>>    |<<mTLS>> | |
                                                | |      v           v        | |
                                                | |  +-----------------------+ | |
                                                | |  | Core App Backend      | | |
                                                | |  +-----------------------+ | |
                                                | +---------------------------+ |
                                                +-------------------------------+
                                                                  |
                                                                  | <<IAM Auth/TDE>>
                                                                  V
                                                      +---------------------------+
                                                      | DataServer [Zone=DataTierSubnet] |
                                                      |                           |
                                                      |  +---------------------+  |
                                                      |  | OpenClaw DB [Isolated]|  |
                                                      |  +---------------------+  |
                                                      |                           |
                                                      +---------------------------+
```
---
## 1.Deployment Overview

| Trust Boundary | Example Threat | Risk | Security Control |
| --- | --- | --- | --- |
| UserDevice → PublicGateway | MITM, token theft, malicious requests | Unauthorized access, data interception | TLS 1.3, HSTS, input validation, WAF |
| PublicGateway → MicroservicesCluster | Forged internal calls, replay attacks | Service impersonation | mTLS, short-lived service identity, signed requests |
| Microservices → Database | Over-privileged queries, injection | Data leakage or corruption | IAM DB auth, parameterized queries, least-privilege DB roles |
| Admin → Management Plane | Privilege abuse, stolen admin credentials | Full system compromise | MFA, RBAC, audit logging, IP allowlisting |
| Service → Secrets Manager | Secret exfiltration | System-wide compromise | Access policies, rotation, audit trail, short-lived credentials |
---
## 2. Network Segmentation & Trust Zones

The architecture is divided into four distinct tiers, each with increasing levels of security:

- **Internet Zone (Untrusted):** Represented by the **UserDevice**. This is where the client-side applications (Web Browser and Mobile App) reside. Since this environment is outside the organization's control, it is treated as entirely untrusted.
- **DMZ (Semi-Trusted):** The **PublicGateway** acts as a buffer between the public internet and the internal network. It houses the API Gateway/Load Balancer, which is the only entry point for external traffic.
- **Secure OpenClaw VNET (Trusted):** This is a private network containing the **MicroservicesCluster**. It is shielded from the internet and only accepts traffic that has been validated by the gateway.
- **Data Tier Subnet (Highly Trusted):** The most isolated layer. The **DataServer** is placed here to ensure that the database is not directly reachable by anything other than the application backend.

## 3. Secure Communication Protocols

The arrows in your diagram define the specific "language" and "security handshake" used between devices:

- **HTTPS/TLS 1.3:** External traffic from the user's device to the gateway is encrypted using the latest TLS standard to prevent eavesdropping or man-in-the-middle attacks.
- **Authenticated Internal RPC:** Once inside the perimeter, the gateway communicates with the microservices using a private, authenticated Remote Procedure Call (RPC) mechanism.
- **mTLS (Mutual TLS):** Within the **MicroservicesRuntime**, the Auth Service, Secrets Manager, and Core App Backend use two-way encryption. This ensures that every service must prove its identity to the others before exchanging data.
- **IAM Auth & TDE:** Connection to the **OpenClaw DB** is secured via Identity and Access Management (IAM) roles rather than simple passwords. Furthermore, Transparent Data Encryption (TDE) ensures the data is encrypted "at rest" on the disk.

## 4. Authentication and Authorization

### 4.1 Authentication

Authentication in OpenClaw verifies the identity of users, services, and privileged operators before access is granted. End users authenticate through the Auth Service, which validates submitted credentials and issues a session or access token for subsequent requests. These tokens are then validated by the PublicGateway before protected requests are forwarded internally. For privileged administrative access, stronger controls such as multi-factor authentication should be applied due to the higher risk associated with administrative accounts. OWASP recommends strong authentication controls and tighter protection for sensitive accounts, while NIST zero-trust guidance emphasizes verifying identity for each access request rather than relying on network location alone.

Within the private environment, service-to-service authentication is enforced using mutual TLS (mTLS). This ensures that each microservice proves its identity before it can communicate with another internal component. As a result, internal traffic is not implicitly trusted simply because it originates from inside the OpenClaw VNET. Backend access to the database is also tied to approved service identities through IAM-based authentication, reducing reliance on static shared credentials.

### 4.2 Authorization

Authorization determines what an authenticated identity is allowed to do after it has been verified. OpenClaw should follow a deny-by-default model, where access is only granted when an explicit rule permits it. Permissions should also follow the principle of least privilege, meaning that each user, service, or administrator receives only the minimum level of access required for its role. OWASP’s authorization guidance recommends validating permissions on every request and avoiding broad, implicit trust relationships.

For end users, authorization should ensure that a user can access only their own records and permitted application functions. This helps prevent broken object-level authorization, where a user might attempt to access another user’s data by modifying request parameters. Administrative operations should be separately protected through function-level authorization so that only approved privileged roles can invoke sensitive endpoints. Internal services should also be authorized individually, ensuring that even a valid internal service can only call the APIs, secrets, and database resources specifically assigned to it.

---
## OpenClaw Access Control Policy Table
| Identity / Role | Authentication Method | Allowed Access | Restrictions |
| --- | --- | --- | --- |
| Guest User | None / public access | Public content and unauthenticated endpoints only | No access to protected APIs |
| Authenticated User | Token issued by Auth Service | Own profile, own application data, normal user functions | Cannot access admin endpoints or other users’ records |
| Admin | Strong authentication with MFA | Administrative dashboard, audit views, privileged operations | Actions are logged and subject to tighter monitoring |
| Core App Backend | mTLS and service identity | Internal business logic, approved database operations | No direct public access; least-privilege DB permissions only |
| Auth Service | mTLS and service identity | Credential validation, token issuance, token verification | Cannot perform unrelated business operations |
| Secrets Manager | Internal identity controls | Provides secrets to approved services only | Secret access is scoped, logged, and not exposed publicly |
---
## 5. Internal Logic & Instance Specification

This section covers the "brains" and the "vault" of the system, along with the specific UML grammar that brings them to life.

- **Microservices Runtime (`<<execution environment>>`):** This serves as the operational brain of OpenClaw. By housing the **Core App Backend**, **Auth Service**, and **Secrets Mgr** within a single logical runtime, the system can enforce strict **mTLS** communication internally. This ensures that sensitive tasks—like retrieving credentials or verifying a user's identity—are decoupled from the main app logic and never "leak" to the outer, less-trusted layers.
- **The Instance Marker (`:`):** You’ll notice the labels start with a colon (e.g., `:DataServer` or `:UserDevice`). In UML, this colon is the "Instance" marker. It signifies that the diagram isn't just a generic blueprint of a *type* of server; it represents a **specific, live instance** of that device currently running in the OpenClaw environment. It’s the difference between looking at a car's manual and looking at the specific car parked in your driveway.
- **Isolated Database Instance:** The `{Isolated}` property marks this as the vault. It sits within the **Data Tier Subnet**, meaning it has no public IP address and is physically and logically separated from the application tier. This fulfills the requirement for "stricter access" by ensuring the data can only be reached by authenticated requests from the backend.

# 6. Performance and Scalability

## 6.1 Why Performance Matters for OpenClaw

One thing worth pointing out early is that OpenClaw's architecture was designed with security as the top priority — and rightly so. But a consequence of layering multiple trust boundaries, encryption protocols, and authentication checks is that every single request has to pass through several security gates before it even reaches the business logic. If we are not careful about how we handle this, the user experience will suffer noticeably.

In practice, a typical API request in OpenClaw travels through four distinct zones: it starts at the user's device, hits the PublicGateway in the DMZ, gets routed into the MicroservicesCluster over authenticated RPC, and finally reaches the isolated database in the Data Tier Subnet. Each hop involves some form of authentication or encryption, and each one adds time. The challenge is keeping total response times low enough that users do not notice this overhead.

## 6.2 Performance Targets

Based on what is generally expected for interactive web applications and considering our multi-tier setup, the following targets seem reasonable for OpenClaw:

---
| Metric | Target | Why this number |
| --- | --- | --- |
| API response time (P95) | Under 300ms | Most users start to notice delays beyond this point |
| Auth token validation | Under 100ms | This happens on every single authenticated request |
| Database query time (P95) | Under 50ms | Assuming properly indexed tables and parameterized queries |
| Full page load | Under 2 seconds | Standard usability threshold for web applications |
| Sustained throughput | At least 500 requests/sec | Enough for moderate concurrent usage without degradation |
---

These are not arbitrary — the 300ms API target, for instance, has to account for roughly 30 to 80ms of pure security overhead that gets added before any actual work happens. That leaves around 220ms for the application logic and database queries, which is tight but achievable with proper optimization.

## 6.3 Where the Time Goes — Latency Across Trust Boundaries

To understand the performance picture, it helps to break down where time is actually spent during a request. Each security boundary in our architecture adds its own cost:

**TLS 1.3 at the edge** — When a user connects for the first time, the TLS handshake with the PublicGateway takes about 20 to 50ms depending on network conditions. This is unavoidable for new connections but can be reduced significantly on repeat visits through TLS session resumption and 0-RTT, which cuts it down to roughly 5ms.

**Gateway processing** — The API Gateway does not just route traffic. It also runs WAF rules, validates input, applies rate limiting, and checks basic request structure. All of this adds around 5 to 15ms. The WAF rules in particular need to be reviewed periodically because overly complex rule sets can slow things down without adding proportional security value.

**mTLS inside the cluster** — Once the request enters the Secure OpenClaw VNET, the gateway needs to prove its identity to the backend services via mutual TLS. This two-way certificate exchange adds about 2 to 10ms. Since these are internal connections between services that talk to each other constantly, connection pooling helps a lot here — we can keep mTLS sessions alive rather than renegotiating every time.

**Database access** — The final hop to the DataServer involves IAM-based authentication (no static passwords) and Transparent Data Encryption. Together, these add around 3 to 8ms on top of the actual query execution time. TDE specifically adds CPU overhead because every read from disk needs to be decrypted on the fly.

The total security overhead per request sits somewhere between 30ms and 80ms. That might not sound like much, but it is a significant chunk of our 300ms budget, and it applies to every single request — not just the first one.

## 6.4 How to Keep Things Fast Despite the Security Overhead

The good news is that there are well-established techniques to offset most of this overhead. The key idea is to avoid doing unnecessary work on requests that do not need it.

**Token caching** is probably the single most impactful optimization. The Auth Service validates tokens on every authenticated request, and if it has to do a full cryptographic verification each time, that adds up quickly. By caching validated tokens in something like Redis with a short TTL, we can turn most token checks into a fast cache lookup instead. The TTL needs to be short enough that revoked tokens do not stay valid for too long — maybe 5 minutes — but long enough to make a real difference.

**Gateway-level caching** can handle frequently requested public endpoints. If the same public content is being fetched hundreds of times per second, there is no reason for each request to travel all the way to the backend. The gateway can serve cached responses directly, bypassing the mTLS handshake and database access entirely.

**Application-level caching** in the Core App Backend reduces load on the database. This is especially important for OpenClaw because every database query pays the TDE decryption cost, so avoiding unnecessary queries has a compounding benefit — it saves both query time and decryption overhead.

**Connection pooling** should be implemented at two levels: between the gateway and the microservices (to reuse mTLS sessions) and between the backend and the database (to reuse IAM-authenticated connections). Establishing a new authenticated connection to the database is expensive; reusing an existing one is nearly free.

**CDN for static assets** moves JavaScript, CSS, and images to edge servers close to the user, so those requests never even reach the PublicGateway. This is straightforward to set up and has a big impact on perceived page load time.

## 6.5 Database Performance Considerations

The OpenClaw DB sits in the most isolated part of the architecture — the Data Tier Subnet with no public IP. This is exactly right from a security standpoint, but it means the database is physically and logically further from the application than it would be in a simpler setup. A few things need attention to keep query performance acceptable:

Proper **indexing** on frequently queried columns is essential. This is true for any database, but it matters more here because TDE means every unnecessary row read also triggers an unnecessary decryption operation. A full table scan on an encrypted database is significantly more expensive than on an unencrypted one.

**Read replicas** could be introduced if the workload becomes read-heavy. Each replica would need to maintain the same IAM controls and encryption standards as the primary, so there is an operational cost, but it would allow us to distribute read traffic and reduce contention on the primary instance.

**Query optimization** goes hand in hand with the parameterized queries we already use for SQL injection prevention. The same practices that make queries safe — using bind parameters, avoiding dynamic SQL — also tend to make them faster because the database can cache execution plans.

## 6.6 Performance vs Security Trade-Offs

It is worth being honest about the fact that some of our security controls have a measurable performance cost. The table below summarizes the main trade-offs and how we plan to manage them:

---
| Security control | What it costs | How we offset it |
| --- | --- | --- |
| TLS 1.3 (user to gateway) | ~20–50ms on new connections | Session resumption, 0-RTT on repeat visits |
| WAF and input validation | ~5–15ms per request | Keep rule sets lean, log asynchronously |
| mTLS (internal services) | ~2–10ms per service call | Connection pooling, persistent mTLS sessions |
| IAM database auth | ~3–5ms per query | Connection pooling, long-lived sessions |
| TDE (encryption at rest) | ~1–3ms CPU per read | Hardware-accelerated AES-NI, proper indexing |
| Parameterized queries | Essentially nothing | Also prevents SQL injection, so it is a win-win |
---

None of these trade-offs are deal-breakers, but ignoring them would be a mistake. The cumulative effect of "just a few milliseconds" at every layer adds up quickly when you have four trust boundaries in the request path.

# 7. Scalability

## 7.1 Why Scalability Needs Its Own Discussion

Performance and scalability are related but they are not the same thing. Performance is about how fast the system responds to a single request under normal conditions. Scalability is about whether the system can maintain that performance level as demand grows — more users, more data, more concurrent requests. A system can be fast for ten users and completely unusable for ten thousand.

OpenClaw's microservices architecture is actually well-suited for scaling, but it does not happen automatically. The fact that we have separate services for authentication, business logic, and secrets management means we can scale each one independently based on where the bottleneck actually is. But we need a concrete strategy for doing so.

## 7.2 Horizontal Scaling of Microservices

The most straightforward way to handle increased load is horizontal scaling — running more instances of a service rather than trying to make a single instance faster. For this to work well with OpenClaw, a few things need to be true:

**Stateless service design** is a prerequisite. The Core App Backend, Auth Service, and any other service in the MicroservicesRuntime must not store session data locally. If a user's second request lands on a different instance than their first one, the system should not break. Session state belongs in the token cache (Redis or similar), not in the service's memory.

**Container orchestration** through something like Kubernetes is the practical mechanism for horizontal scaling. It lets us define auto-scaling policies — for example, spin up additional Auth Service instances when CPU utilization exceeds 70%, or scale the Core App Backend when request queue depth crosses a certain threshold. The key is choosing the right metric to trigger scaling; CPU is common but request latency or queue depth often gives better results.

**Load balancing** at the API Gateway already exists in our architecture, but it needs to be configured to distribute traffic intelligently across scaled instances. Simple round-robin works for stateless services. If we ever introduce any affinity (which we should try to avoid), we would need session-aware routing.

One important point: every new instance of a service needs its own mTLS certificate and must be registered with the Auth Service and Secrets Manager. This means our scaling process cannot just be "start another container" — it needs to include identity provisioning as part of the startup sequence. If this takes too long, it slows down how quickly we can react to traffic spikes.

## 7.3 Database Scalability

The database is almost always the hardest component to scale, and OpenClaw is no exception. The isolated placement in the Data Tier Subnet makes it even more important to plan ahead because we cannot just throw more database instances at the problem without considering the security implications.

**Vertical scaling** (bigger machine, more RAM, faster storage) is the simplest first step and often buys a lot of runway. For a single-instance database with TDE enabled, having more RAM means more data can be cached in its decrypted form in the buffer pool, which reduces the number of disk reads that trigger the decryption overhead.

**Read replicas** make sense once read traffic dominates. Each replica must sit within the same Data Tier Subnet and enforce the same IAM authentication and TDE encryption — we cannot trade security for scalability. The Core App Backend would need to be aware of read replicas and route read queries accordingly, while writes continue to go to the primary instance.

**Connection pooling** becomes even more critical at scale. Without it, each new backend instance opens its own set of database connections, and the database quickly runs out of connection slots. A connection pooler (like PgBouncer for PostgreSQL) sitting between the backend and the database can multiplex hundreds of backend connections over a smaller number of actual database connections.

**Sharding** is an option for the long term if data volume grows significantly, but it adds considerable complexity and should be treated as a last resort. It also complicates things like cross-shard queries and transactional consistency.

## 7.4 Scaling the Security Layer

This is an aspect of scalability that often gets overlooked. As OpenClaw scales, the security components need to scale with it — otherwise they become the bottleneck.

The **Auth Service** will see its load grow linearly with overall traffic, since every authenticated request needs token validation. If we are caching tokens, the cache infrastructure (Redis cluster) needs to scale as well. A single Redis instance might handle a few thousand lookups per second, but beyond that we need Redis Sentinel or a cluster setup.

The **Secrets Manager** is less frequently accessed (services typically fetch secrets at startup or on rotation), but if we are scaling to dozens or hundreds of service instances, the startup load on the Secrets Manager could spike during a scaling event or a rolling deployment. Rate limiting on the secrets side and pre-fetching secrets during provisioning can help.

**mTLS certificate management** gets more complex at scale. Every service instance needs a valid certificate, and certificates need to be rotated periodically. An automated certificate authority (like SPIFFE/SPIRE or Vault's PKI backend) is practically necessary once you move beyond a handful of instances.

## 7.5 Auto-Scaling Strategy

Rather than scaling everything uniformly, OpenClaw should scale each component based on its specific bottleneck indicator:

---
| Component | Scaling metric | Scale-out trigger | Scale-in trigger |
| --- | --- | --- | --- |
| Core App Backend | Request latency P95 | P95 > 200ms for 3 minutes | P95 < 80ms for 10 minutes |
| Auth Service | Token validation queue depth | Queue > 50 for 2 minutes | Queue < 10 for 10 minutes |
| API Gateway | Active connections | Connections > 80% capacity | Connections < 30% capacity |
| Token cache (Redis) | Memory utilization | Memory > 75% | Memory < 40% |
| Database read replicas | Replication lag | Lag > 500ms sustained | Manual scale-in only |
---

The scale-in timers are deliberately longer than scale-out timers. It is better to have a few extra instances running for a few minutes than to scale down too aggressively and immediately need to scale back up — especially given the overhead of mTLS certificate provisioning during startup.

## 7.6 What Could Go Wrong — Scalability Risks

There are a few scenarios worth planning for:

**Thundering herd after a cache failure** — If the Redis token cache goes down, every request suddenly needs full token validation from the Auth Service. This could overwhelm the Auth Service and cascade into gateway timeouts. The mitigation is to run Redis in a clustered or replicated configuration so that a single node failure does not take out the cache entirely.

**Database connection exhaustion** — As we scale out backend instances, each one wants its own database connections. Without a connection pooler, we could hit the database's maximum connection limit, causing new queries to queue or fail. This is particularly dangerous because the symptom (slow or failed queries) looks like a database performance problem when it is actually a connection management problem.

**Certificate provisioning delays** — If the system needs to scale quickly in response to a traffic spike, but certificate provisioning takes 30 seconds per instance, we could have a window where new instances are running but cannot communicate over mTLS. Pre-provisioning certificates or using short-lived tokens as a bootstrap mechanism can close this gap.

**Uneven scaling** — If we scale the backend but not the Auth Service, token validation becomes the bottleneck. If we scale everything but not the database, the database becomes the bottleneck. The auto-scaling strategy needs to be monitored as a system, not component by component.

---

# Early Design Decisions

This section documents the earliest architectural decisions made for OpenClaw. These three decisions were made before any code was written and they constrain everything that came after. Each decision is presented with the architectural pattern it implements, the quality attribute it is intended to address, and an individual pros-and-cons analysis tied back to those quality attributes.

The structure follows Bass et al. — quality attributes are not afterthoughts, they are the prediction we make about the system based on the architectural decisions we choose.

---

## Decision D1 — Hub-and-Spoke Topology with the Gateway as the Hub

### The decision

The Gateway is the single point through which every participant in the system communicates. Clients, messaging platforms, nodes, agents, and tools all connect to the Gateway and only to the Gateway. No spoke ever talks directly to another spoke.

### Pattern implemented

**Hub-and-Spoke topology**, layered on top of an **Event-Driven Architecture (EDA)**. The Gateway is the central event broker; spokes are event producers and consumers.

### Quality attribute addressed

**Modifiability** (primary), **Integrability** (secondary).

By forcing all communication through one hub, the cost of adding, removing, or replacing any spoke is local to that spoke and the Gateway — not spread across the system. A new messaging platform (Discord, Slack) can be added by writing one new connector to the Gateway, without touching any client, agent, or node. This is the modifiability tactic of *limiting dependencies* applied at the topology level.

### Individual pros (tied to quality attributes)

- **Modifiability** — Adding a new spoke is a local change. No other spoke needs to know it exists.
- **Integrability** — The Gateway is the single integration surface. Third parties have one contract to learn, not N contracts.
- **Observability** — Every message in the system passes through one place. Logging, tracing, and rate limiting can be enforced uniformly.
- **Security** — Authentication, schema validation, and authorization are enforced at one boundary instead of being re-implemented per spoke.

### Individual cons (tied to quality attributes)

- **Availability** — The Gateway is a single point of failure. If the Gateway is down, the entire system is down. This is the direct cost of centralization.
- **Performance** — Every message takes at least one extra hop through the Gateway, even when two spokes conceptually need to exchange information. There is no shortcut path.
- **Scalability** — The Gateway is the first component that will saturate under load. Horizontal scaling of the Gateway requires sticky sessions or session-state externalization, which adds complexity.

---

## Decision D2 — Persistent WebSocket with JSON Frames as the Transport

### The decision

Every spoke maintains a single, long-lived WebSocket connection to the Gateway. All communication uses JSON text frames over that connection. There is no HTTP polling, no per-request connection setup, and no synchronous request-response model. Acknowledgments are returned immediately; actual results stream back as separate `event:agent` frames over the same connection.

### Pattern implemented

**Asynchronous messaging over a persistent duplex channel** — the transport-level realization of EDA. The connection itself is the channel; events are the messages.

### Quality attribute addressed

**Performance** (primary, specifically perceived latency), **Usability** (secondary).

Streaming responses chunk-by-chunk lets the user see output as it is generated rather than waiting for the full response. For an AI assistant where a single response can take 10+ seconds to fully generate, this is the difference between a usable system and an unusable one. The persistent connection also eliminates per-request TLS and connection setup overhead, which matters because the system is conversational and chatty.

### Individual pros (tied to quality attributes)

- **Performance (perceived latency)** — Streamed chunks mean the user sees the first token within ~100ms, not after the full response is generated.
- **Performance (throughput)** — No per-request connection overhead. One TLS handshake, then thousands of frames.
- **Usability** — Real-time, conversational feel. Matches user expectations from modern chat applications.
- **Functionality** — The Gateway can push unsolicited events (`event:presence`, `event:cron`, `event:health`) to clients without being polled, enabling features that would be awkward over HTTP.

### Individual cons (tied to quality attributes)

- **Availability** — Persistent connections are stateful. A Gateway restart drops every active connection at once, and clients must reconnect and re-authenticate. Rolling deployments are harder than for stateless HTTP services.
- **Scalability** — Each connected spoke consumes one open file descriptor and some memory on the Gateway, indefinitely. A single Gateway instance has a hard ceiling on concurrent connections, regardless of how busy each one is.
- **Integrability** — Many existing tools, services, and SDKs only speak HTTP. They cannot integrate directly; they need a WebSocket-aware adapter. This is the most-felt cost in practice.
- **Testability** — WebSocket flows are harder to test than HTTP. Standard tools (curl, Postman, most load testers) don't natively support persistent bidirectional streams. CI tests need a WebSocket client harness.
- **Debuggability** — A single connection carries many logical exchanges interleaved. Tracing one request end-to-end requires correlation IDs in every frame; a network capture alone is not enough.

---

## Decision D3 — Four-Part Functional Decomposition (Client, Gateway, Agent, Tools/Nodes)

### The decision

The system is decomposed into four functional roles, each with a single responsibility:

- **Client** — presents the UI and forwards user input. Owns nothing else.
- **Gateway** — routes, validates, and enforces protocol. Owns no business logic.
- **Agent** — performs reasoning, holds session state, decides what to do. Owns no I/O to the outside world.
- **Tools / Nodes** — execute actions on devices or external APIs. Own no reasoning or state.

Each role can be replaced independently as long as it continues to honour the Gateway protocol.

### Pattern implemented

**Separation of concerns** with **information hiding** (Bass et al., Chapter 1, "Structural Rules of Thumb"). Each part hides what it does behind its protocol contract; the others depend on the contract, not the implementation.

### Quality attribute addressed

**Modifiability** (primary), **Reusability** (secondary), **Testability** (secondary).

Because each role has one responsibility and a defined contract, the implementation behind that contract can be swapped without ripple effects. The LLM provider can change without touching the Client. The CLI can be replaced with a web app without touching the Agent. A new device-level tool can be added without touching anything except the Gateway's tool registry.

### Individual pros (tied to quality attributes)

- **Modifiability** — Swap the LLM, the UI, or any single tool independently. Each change is local.
- **Reusability** — The Agent is reusable across every Client; the Tools are reusable across every Agent. There is no UI logic embedded in the Agent and no reasoning embedded in the Tools.
- **Testability** — Each role can be tested in isolation against a mocked version of the protocol. The Agent can be tested without a real Client; the Gateway can be tested with a stub Agent.
- **Team scalability** — Different team members can own different roles without stepping on each other, because the Gateway protocol is the contract between them.

### Individual cons (tied to quality attributes)

- **Performance** — Strict role separation means more hops per user action. A "take a screenshot and summarize it" request crosses Client → Gateway → Agent → Gateway → Node → Gateway → Agent → Gateway → Client. Every boundary adds serialization, validation, and routing cost.
- **Complexity** — Four roles plus a protocol is more moving parts than a monolith. The conceptual overhead is real, especially for new contributors.
- **Discipline cost** — The decomposition only delivers its modifiability benefit if every contributor respects the boundaries. If reasoning starts leaking into Tools or UI logic into the Agent, the benefit is lost and the complexity remains.

---

# The Protocol Rules

Decision D2 commits the system to a specific transport contract. The teacher's comment correctly notes that this contract has consequences — both positive (every spoke speaks the same language) and negative (every spoke *must* speak that language). This section specifies the rules explicitly and is honest about what they cost.

## Rule 1 — Frame envelope

Every message exchanged over the WebSocket is a JSON object that conforms to one of three shapes:

```
Request        { "kind": "req:<name>",   "id": "<uuid>", "payload": { ... } }
Response       { "kind": "res",          "id": "<uuid>", "status": "ok|accepted|error", "payload": { ... } }
Event          { "kind": "event:<name>", "id": "<uuid>", "payload": { ... } }
```

The Gateway validates every inbound frame against a JSON Schema before routing it. Frames that fail validation are rejected with an error response and never reach the Agent or any Tool.

**Why this rule exists:** schema validation at the boundary is a *security* and *modifiability* tactic. Spokes cannot send malformed input that breaks downstream components, and the schema itself is the documentation of the contract.

## Rule 2 — Acknowledgment is not an answer

A `res` with `status: "accepted"` means the Gateway received the request and queued it. It does not mean the request has been processed and it does not contain the answer. The answer arrives later as one or more `event:agent` frames, correlated by the original request `id`.

**Why this rule exists:** this is what makes the system non-blocking. Long-running agent operations (LLM calls, tool execution) cannot be allowed to hold up the connection. Decoupling acknowledgment from completion is a *performance* tactic for the Gateway under load.

## Rule 3 — Spokes never talk to each other

If the Client needs the result of a Tool, the Client does not call the Tool. The Client sends a request to the Gateway; the Gateway routes it to the Agent; the Agent invokes the Tool through the Gateway; the result returns through the Gateway. There is no spoke-to-spoke shortcut, ever.

**Why this rule exists:** this is the *integrity* of D1. Allowing exceptions would erase every modifiability benefit of the hub-and-spoke topology, because spokes would now have direct dependencies on each other.

## Rule 4 — Agent-to-agent communication routes through the Gateway

Agents do not hold direct references to other agents. If Agent A needs Agent B to do something, Agent A sends a request to the Gateway addressed to Agent B, exactly as a Client would. From the Gateway's perspective there is no special case for inter-agent traffic.

**Why this rule exists:** treating agents as just another type of spoke means the same protocol, the same validation, the same logging, and the same security controls apply. Adding a new agent does not require changes to any other agent. This preserves *modifiability* and *observability* as the agent population grows.

## Rule 5 — Third-party tool integration contract

A third-party tool that wants to integrate with OpenClaw must:

1. Open and maintain a WebSocket connection to the Gateway.
2. Identify itself in the connect frame with `role: "tool"` and a unique tool name.
3. Register the operations it supports (e.g. `weather.get`, `database.query`) so the Gateway and Agent know how to dispatch to it.
4. Respond to incoming `req:<operation>` frames with a `res` envelope.
5. Validate against the JSON Schema for every operation it claims to support.

There is no second integration path. A tool that cannot or will not speak this protocol cannot integrate directly.

**Why this rule exists:** one integration contract means one place to enforce security, schema, rate limiting, and observability — the Gateway. The cost of this rule is the subject of the next section.

---

# The Cost of the Protocol — and How Future Decisions Could Address It

The protocol rules above are good for the system but they are not free. The teacher's comment specifically asks for an honest description of the negatives and what future architectural decisions could address them. The negatives below are the ones that matter most in practice.

## Negative 1 — Many existing tools and services only speak HTTP

The vast majority of public APIs, SDKs, and existing internal services use HTTP. They cannot integrate with OpenClaw directly under Rule 5. Every such integration today requires a custom WebSocket-aware wrapper, written and maintained by the OpenClaw team or by the third party.

This is the largest practical cost of D2 + Rule 5 combined. It hurts *integrability* — the very quality attribute D1 was supposed to maximize.

**Future architectural decision that could address this:** introduce an **HTTP-to-WebSocket adapter layer** as a first-class component. The adapter would be a long-lived WebSocket spoke (`role: "tool"`) that exposes an HTTP listener; for each registered HTTP service, it translates incoming HTTP responses into Gateway events and outbound Gateway requests into HTTP calls. This preserves the integrity of the protocol (Rule 3, Rule 5 still hold internally) while letting the system absorb the HTTP world without per-tool wrapper work.

## Negative 2 — The Gateway is a single point of failure

D1 centralizes all communication. If the Gateway process dies, every spoke loses its connection at the same moment, and the system is fully unavailable until the Gateway returns and every spoke reconnects.

This is a direct *availability* cost.

**Future architectural decision that could address this:** introduce **Gateway high availability** through an active-active cluster of Gateway instances behind a load balancer, with shared session state externalized to Redis (or equivalent). Spokes would still see one logical Gateway, but the failure of any single Gateway instance would only drop the connections held by that instance, not the entire fleet. This adds complexity (state externalization, sticky reconnection) but converts a system-wide outage into a partial, transient one.

## Negative 3 — Per-request hop count is high

Decision D3's strict role separation means every user action crosses multiple Gateway hops. For a simple "take a screenshot" request the path is Client → Gateway → Agent → Gateway → Node → Gateway → Agent → Gateway → Client — eight boundary crossings. Each crossing adds serialization, schema validation, and routing time.

This is a *performance* cost that compounds as roles get more granular.

**Future architectural decision that could address this:** introduce a **fast path for tightly coupled operations** — specifically, allow the Agent to declare a small set of "trusted tool calls" that the Gateway can dispatch synchronously, returning the tool result inline in the agent's processing context rather than as a separate event roundtrip. This is a controlled exception to Rule 3, scoped to operations the Agent owns end-to-end, and would need to be implemented carefully to avoid eroding D1's modifiability benefit.

## Negative 4 — Stateful WebSocket connections complicate deployment

Persistent connections from D2 mean that a Gateway deployment cannot just drain HTTP requests and exit cleanly. Every connection has to be migrated, drained, or dropped. Standard rolling-deployment tools (Kubernetes rolling updates, blue-green deployments) need extra care.

This is an *operability* and *availability* cost.

**Future architectural decision that could address this:** combine the Gateway HA cluster (above) with a **graceful connection migration protocol** — when a Gateway instance is about to be retired, it sends a `event:reconnect` frame to its connected spokes pointing them at a sibling instance, and only exits once they have moved. This makes deployments invisible to users.

---

# Summary — Decisions, Patterns, and Quality Attributes

| Decision | Pattern | Primary QA Addressed | Largest QA Cost | Future Decision to Mitigate |
|----------|---------|----------------------|-----------------|-----------------------------|
| **D1** Hub-and-spoke with Gateway | Hub-and-Spoke + EDA | Modifiability, Integrability | Availability (Gateway is SPOF) | Gateway HA cluster |
| **D2** Persistent WebSocket + JSON | Async messaging on duplex channel | Performance (perceived latency) | Integrability (HTTP tools can't connect directly) | HTTP-to-WebSocket adapter layer |
| **D3** Four-part decomposition | Separation of concerns + information hiding | Modifiability, Reusability, Testability | Performance (high hop count) | Trusted-tool fast path for Agent-owned operations |

These three decisions form the architectural skeleton of OpenClaw. Every later decision in the system either follows from them or is constrained by them, which is exactly what an early design decision is supposed to do (Bass et al., Chapter 2 — "The architecture is the carrier of the earliest, most fundamental, hardest-to-change design decisions").

## Project Logs
https://docs.google.com/spreadsheets/d/19eP-f9D0icTfOgCbX4RY3jcTt0ONy1vSKLz8U42olSo/edit?gid=0#gid=0
