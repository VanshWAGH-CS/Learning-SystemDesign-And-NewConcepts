# 🌐 API Types – Complete Study Notes
> Source: *"7 API Types That Power the Internet"* – Bitemunk  
> Use: Future reference, interview prep, system design

---

## 📋 Table of Contents
1. [Quick Comparison Table](#quick-comparison-table)
2. [REST API](#1-rest-api)
3. [SOAP](#2-soap)
4. [gRPC](#3-grpc)
5. [GraphQL](#4-graphql)
6. [Webhooks](#5-webhooks)
7. [WebSockets](#6-websockets)
8. [WebRTC](#7-webrtc)
9. [Decision Flowchart](#decision-flowchart)
10. [When to Use What](#when-to-use-what--cheat-sheet)

---

## Quick Comparison Table

| API Type | Protocol | Data Format | Direction | Best For | Real-World Example |
|----------|----------|-------------|-----------|----------|--------------------|
| **REST** | HTTP | JSON/XML | Client → Server | CRUD apps, public APIs | Most web apps |
| **SOAP** | HTTP/SMTP | XML only | Client → Server | Enterprise, compliance | Banking, Healthcare |
| **gRPC** | HTTP/2 | Binary (Protobuf) | Both ways | Microservices, streaming | Netflix internals |
| **GraphQL** | HTTP | JSON | Client → Server | Flexible data fetching | GitHub, Shopify |
| **Webhooks** | HTTP | JSON/XML | Server → Client | Event notifications | Stripe, GitHub |
| **WebSockets** | WS/WSS | Any | Both ways | Real-time bidirectional | Chat apps, games |
| **WebRTC** | P2P | Binary/Media | Peer ↔ Peer | Video/audio, file sharing | Zoom, Discord |

---

## 1. REST API

> **R**epresentational **S**tate **T**ransfer — The workhorse of the internet (~90% of APIs)

### How it Works

```
CLIENT                          SERVER
  |                                |
  |--- GET /videos/123 ----------> |   ← Fetch a video
  |<-- 200 OK { id, title... } --- |
  |                                |
  |--- POST /likes { videoId } --> |   ← Hit Like button
  |<-- 201 Created { likeCount }-- |
  |                                |
  |--- PUT /comments/5 { text }--> |   ← Edit comment
  |<-- 200 OK { updated } -------- |
  |                                |
  |--- DELETE /subscriptions/7 --> |   ← Unsubscribe
  |<-- 204 No Content ------------ |
```

### HTTP Methods

| Method | Action | Example |
|--------|--------|---------|
| `GET` | Read/Fetch data | Get video details |
| `POST` | Create new data | Create a like |
| `PUT` | Update existing | Edit a comment |
| `DELETE` | Remove data | Unsubscribe |

### Key Characteristics

- **Stateless** – Server remembers nothing between requests. Every call is independent.
- **Horizontally scalable** – Add more servers easily (no shared session state)
- **Universal** – Every developer knows REST

### ✅ Use When
- Building standard CRUD applications
- Public APIs for other developers
- Mobile apps talking to a backend
- Need simplicity + universal compatibility

### ❌ Avoid When
- You need real-time bidirectional communication
- Strict compliance/audit trail required
- Internal microservices needing max performance

---

## 2. SOAP

> **S**imple **O**bject **A**ccess **P**rotocol — *Anything but simple*

### Message Structure

```
┌─────────────────────────────────────┐
│            SOAP ENVELOPE            │
│  ┌───────────────────────────────┐  │
│  │          SOAP HEADER          │  │  ← Auth, security tokens
│  │   (security, auth, routing)   │  │
│  └───────────────────────────────┘  │
│  ┌───────────────────────────────┐  │
│  │           SOAP BODY           │  │  ← Actual payload (XML)
│  │      (actual payload)         │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
```

### What a SOAP Message Looks Like (simplified)

```xml
<soap:Envelope>
  <soap:Header>
    <Security><!-- auth token --></Security>
  </soap:Header>
  <soap:Body>
    <TransferFunds>
      <from>Account123</from>
      <to>Account456</to>
      <amount>1000.00</amount>
    </TransferFunds>
  </soap:Body>
</soap:Envelope>
```

### Key Characteristics

- **Strict XML schema** – Every message validated
- **Built-in security** – WS-Security standards
- **Guaranteed delivery** – Message receipts, retries
- **Audit trail** – Full compliance logging
- **Heavy & verbose** – Not dev-friendly

### ✅ Use When
- Bank-to-bank transfers
- Healthcare patient record exchange (HIPAA)
- Government systems
- Legal compliance + paper trail required
- Enterprise-grade security is mandatory

### ❌ Avoid When
- Speed matters
- Building a modern consumer-facing API
- Developer experience is a priority

---

## 3. gRPC

> Google's answer to: *"What if APIs were really fast?"*

### REST vs gRPC Data Format

```
REST (JSON - text, human readable):
{"userId":1,"name":"Alice","email":"alice@example.com"}   ← ~50 bytes

gRPC (Protocol Buffers - binary):
[0x08 0x01 0x12 0x05 0x41 ...]                           ← ~12 bytes
                                               ↑ Up to 10x smaller & faster
```

### 4 Communication Patterns

```
1. UNARY (like REST)
   Client ──[request]──► Server
   Client ◄──[response]── Server

2. SERVER STREAMING
   Client ──[request]──► Server
   Client ◄──[data]────── Server  (continuous)
   Client ◄──[data]────── Server
   Client ◄──[data]────── Server
   Example: Uber driver location updates

3. CLIENT STREAMING
   Client ──[data]──► Server  (continuous)
   Client ──[data]──► Server
   Client ──[data]──► Server
   Client ◄──[response]── Server
   Example: GPS uploading your movement

4. BIDIRECTIONAL STREAMING
   Client ──[data]──► Server
   Client ◄──[data]── Server
   Client ──[data]──► Server  (simultaneous)
   Client ◄──[data]── Server
   Example: Real-time chat
```

### HTTP/2 Multiplexing (vs HTTP/1.1)

```
HTTP/1.1:           HTTP/2 (gRPC):
[Request 1]         [Req1][Req2][Req3]  ← All at once
    ↓                   ↓   ↓   ↓
[Response 1]        [Res1][Res2][Res3]  ← Single connection
[Request 2]
    ↓
[Response 2]         Much faster!
```

### ✅ Use When
- Microservices communicating internally
- Real-time data streaming needed
- Performance is critical
- Multiple programming languages in your stack (gRPC is polyglot)

### ❌ Avoid When
- Public-facing API (browser support limited)
- Team unfamiliar with Protocol Buffers
- Simple CRUD is all you need

---

## 4. GraphQL

> Ask for exactly what you need. Nothing more, nothing less.

### The Over/Under-fetching Problem (REST)

```
REST Problem:                        GraphQL Solution:
                                     
GET /users/1                         query {
{                                      user(id: 1) {
  id, name, email,                       name
  address, phone,          →→→           posts {
  createdAt, updatedAt,                    title
  preferences, ...                       }
}  ← Too much! (overfetch)           }
                                     }
Need posts? Another call:            ↑ One request, exactly this
GET /users/1/posts        (underfetch)
```

### GraphQL Architecture

```
                    ┌─────────────────┐
                    │   GraphQL API   │
                    │  Single /graphql│
                    │   endpoint      │
                    └────────┬────────┘
                             │
           ┌─────────────────┼─────────────────┐
           ▼                 ▼                 ▼
    ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
    │  Users DB   │  │  Posts DB   │  │ Comments DB │
    └─────────────┘  └─────────────┘  └─────────────┘
```

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Schema** | Defines all types & operations. Self-documenting. |
| **Query** | Read data (like GET) |
| **Mutation** | Write/change data (like POST/PUT/DELETE) |
| **Subscription** | Real-time updates (like WebSockets) |
| **Resolver** | Function that fetches data for each field |

### ✅ Use When
- Frontend needs flexible/custom data shapes
- Multiple clients (web, mobile) need different data
- Rapid iteration without versioning APIs
- Large interconnected data graphs

### ❌ Avoid When
- Simple, stable endpoints
- Team new to it (learning curve)
- Caching is critical (harder than REST)

---

## 5. Webhooks

> *"Don't call us, we'll call you."*  — Also known as **Reverse APIs**

### Polling vs Webhooks

```
POLLING (inefficient):                WEBHOOKS (efficient):

App: Any updates?      every          API: Hey! Payment succeeded!
API: Nope              30 secs              ↓
App: Any updates?        ↕           App: [processes event]
API: Nope              wasted
App: Any updates?      resources
API: Nope
App: Any updates?
API: YES! Payment done!
```

### How Webhooks Work

```
Step 1: Register
App ──[POST your callback URL]──► Stripe API
       "notify me at: yourapp.com/webhooks"

Step 2: Event Happens
Customer pays → Stripe processes it

Step 3: Notification
Stripe ──[POST /webhooks]──► Your App
        { event: "payment.succeeded",
          amount: 5000, currency: "usd" }

Step 4: React
Your App sends receipt email ✓
```

### Common Webhook Use Cases

| Service | Event | Webhook Payload |
|---------|-------|-----------------|
| **Stripe** | Payment succeeded | `{ amount, currency, customer }` |
| **GitHub** | Code pushed | `{ repo, commits, author }` |
| **Shopify** | Order placed | `{ order_id, items, customer }` |
| **Twilio** | SMS received | `{ from, to, body }` |

### ✅ Use When
- Automating workflows between services
- Need instant event notifications
- Syncing data across systems in real time
- Avoiding constant polling

### ❌ Avoid When
- You need immediate request/response
- Recipient server may be unreliable (need retry logic)

---

## 6. WebSockets

> A permanent, open phone line between client and server.

### HTTP vs WebSockets

```
HTTP (REST):                      WebSockets:

Client: Hello? (connect)          Client: Hello? (handshake)
Server: Hi! Here's data. (close)  Server: Hi! Connected.
                                          (line stays OPEN)
Client: Hello? (reconnect)        Client: Send message
Server: Hi! Here's data. (close)  Server: Responds instantly
                                  Server: Pushes update (anytime!)
Client: Hello? (reconnect)        Client: Another message
...repeated for every update...   ...persistent connection...
```

### WebSocket Lifecycle

```
1. HANDSHAKE (HTTP Upgrade)
   Client ──[GET /chat, Upgrade: websocket]──► Server
   Server ──[101 Switching Protocols]────────► Client

2. OPEN CONNECTION
   Client ◄══════════════════════════════════► Server
              (bidirectional, persistent)

3. DATA EXCHANGE (anytime, either side)
   Client ──[message]──► Server
   Server ──[message]──► Client
   Server ──[push]─────► Client  ← Server can push without request!

4. CLOSE
   Either side sends close frame
```

### ✅ Use When
- Chat applications
- Live sports scores / stock tickers
- Multiplayer games
- Collaborative editing (Google Docs-style)
- Real-time dashboards

### ❌ Avoid When
- Simple request-response is enough
- Stateless operations (use REST)
- Server resources are limited (connections are persistent)

---

## 7. WebRTC

> **Web** **R**eal-**T**ime **C**ommunication — Peer to peer, no server in the middle.

### Architecture: Traditional vs WebRTC

```
TRADITIONAL (e.g., old video call):
You ──► Central Server ──► Friend
         (bottleneck,
          high latency)

WEBRTC (e.g., Zoom, Google Meet):
You ◄══════════════════► Friend
     (direct peer-to-peer,
      sub-500ms latency)
```

### The Connection Problem (& Solution)

```
The Problem:
Your Laptop              Friend's Laptop
192.168.1.5              192.168.1.10
    │                          │
[Your Router]            [Their Router]
  NAT/Firewall             NAT/Firewall
    │                          │
  [Internet]─────────────[Internet]
  
Neither knows each other's real IP!
```

```
WebRTC 3-Step Signaling Solution:

Step 1: STUN Server discovers your public IPs
  You ──[What's my public IP?]──► STUN Server
  You ◄──[Your IP is 203.x.x.x]── STUN Server

Step 2: Signaling (exchange connection info via server)
  You ──[Here's my connection info]──► Signal Server ──► Friend
  You ◄──[Here's my connection info]── Signal Server ◄── Friend

Step 3: Direct P2P Connection established
  You ◄══════════════════════════════════════════════► Friend
         (TURN server used as fallback if direct fails)
```

### What WebRTC Handles Automatically

| Feature | What it does |
|---------|-------------|
| **Adaptive Bitrate** | Adjusts video quality to your network speed |
| **Codec Negotiation** | Agrees on audio/video format with peer |
| **Jitter Buffering** | Smooths out packet arrival inconsistencies |
| **Encryption** | All streams encrypted by default (DTLS/SRTP) |

### ✅ Use When
- Video/audio calling (Zoom, Google Meet)
- Real-time multiplayer games (frame-critical)
- Browser-to-browser file sharing
- Live streaming with ultra-low latency

### ❌ Avoid When
- Simple data transfer (use REST)
- Need a server-side record of communication
- Targeting older browsers

---

## Decision Flowchart

```
START: What kind of API do I need?
              │
              ▼
   ┌─────────────────────┐
   │ Need real-time P2P  │──YES──► WebRTC
   │ video/audio/files?  │         (Zoom, Discord)
   └──────────┬──────────┘
              │ NO
              ▼
   ┌─────────────────────┐
   │ Need direct server  │──YES──► WebSockets
   │ push / persistent   │         (Chat, games,
   │ connection?         │          live scores)
   └──────────┬──────────┘
              │ NO
              ▼
   ┌─────────────────────┐
   │ Need to react to    │──YES──► Webhooks
   │ external events     │         (Stripe, GitHub)
   │ (event-driven)?     │
   └──────────┬──────────┘
              │ NO
              ▼
   ┌─────────────────────┐
   │ Enterprise-level    │──YES──► SOAP
   │ compliance, strict  │         (Banking, Health,
   │ security needed?    │          Government)
   └──────────┬──────────┘
              │ NO
              ▼
   ┌─────────────────────┐
   │ Internal service    │──YES──► gRPC
   │ communication OR    │         (Microservices,
   │ performance-        │          Netflix internals)
   │ critical?           │
   └──────────┬──────────┘
              │ NO
              ▼
   ┌─────────────────────┐
   │ Frontend needs      │──YES──► GraphQL
   │ flexible queries /  │         (GitHub, Shopify)
   │ many data sources?  │
   └──────────┬──────────┘
              │ NO
              ▼
            REST
     (Default for ~90% of APIs)
```

---

## When to Use What — Cheat Sheet

```
┌──────────────────────────────────────────────────────────────────┐
│                    API SELECTION CHEAT SHEET                     │
├─────────────┬────────────────────────────────────────────────────┤
│ REST        │ Default. CRUD apps. Public APIs. Mobile backends.  │
├─────────────┼────────────────────────────────────────────────────┤
│ SOAP        │ Banks. Healthcare. Government. Need audit trail.   │
├─────────────┼────────────────────────────────────────────────────┤
│ gRPC        │ Microservices. High throughput. Internal systems.  │
├─────────────┼────────────────────────────────────────────────────┤
│ GraphQL     │ Flexible frontend queries. Many data sources.      │
├─────────────┼────────────────────────────────────────────────────┤
│ Webhooks    │ Event notifications. Payment alerts. CI/CD hooks.  │
├─────────────┼────────────────────────────────────────────────────┤
│ WebSockets  │ Chat. Live games. Stock tickers. Collab tools.     │
├─────────────┼────────────────────────────────────────────────────┤
│ WebRTC      │ Video calls. P2P file share. Low-latency streams.  │
└─────────────┴────────────────────────────────────────────────────┘
```

---

## Performance Spectrum

```
SPEED / EFFICIENCY
Low ◄────────────────────────────────────────► High

SOAP ──────── REST ──────── GraphQL ──────── gRPC
(XML)         (JSON)         (JSON)         (Binary)
  │              │               │               │
Verbose      Industry       Flexible       Fastest
Secure       Standard       Queries        Internal
Enterprise   90% of web     Self-docs      Streaming
```

```
REAL-TIME CAPABILITY
None ◄─────────────────────────────────────► Full

REST ── Webhooks ── GraphQL Subs ── WebSockets ── WebRTC
         (push        (push via        (full         (P2P,
         events)      WS)             duplex)      no server)
```

---

## Key Terms Glossary

| Term | Meaning |
|------|---------|
| **Stateless** | Server doesn't remember between requests (REST) |
| **Horizontal Scaling** | Add more servers to handle load |
| **Protocol Buffers** | Binary serialization format used by gRPC |
| **HTTP/2** | Multiplexed HTTP, used by gRPC for multiple simultaneous streams |
| **Schema** | Typed definition of an API's data structure (GraphQL, SOAP) |
| **Polling** | Client repeatedly asks "anything new?" (inefficient) |
| **Callback URL** | Your endpoint that receives webhook POSTs |
| **Handshake** | Initial negotiation to upgrade HTTP → WebSocket |
| **STUN Server** | Helps WebRTC peers discover their public IP |
| **TURN Server** | Relay fallback if direct P2P fails (WebRTC) |
| **NAT** | Network Address Translation — hides private IPs behind router |
| **Peer-to-Peer (P2P)** | Direct device-to-device, no central server |
| **Bidirectional** | Both client and server can send messages anytime |
| **Over-fetching** | Getting more data than needed (REST problem) |
| **Under-fetching** | Needing multiple requests to get all data (REST problem) |

---

## Real-World Examples Map

```
                    WHO USES WHAT
                    
Netflix:    REST (user-facing) + gRPC (internal) + GraphQL (70+ services)
GitHub:     GraphQL (public API) + Webhooks (CI/CD)
Stripe:     REST (payments API) + Webhooks (payment events)
Zoom:       WebRTC (video calls) + WebSockets (chat/signaling)
Uber:       REST + gRPC (streaming location) + WebSockets
Shopify:    GraphQL (storefront API) + Webhooks (order events)
Discord:    WebSockets (messages) + WebRTC (voice/video)
Banks:      SOAP (inter-bank transfers, regulatory compliance)
```

---

> 📝 **Notes to Self**  
> - WebRTC deep dive pending (signaling, STUN/TURN, build video chat app)  
> - gRPC: learn Protocol Buffers `.proto` file syntax  
> - GraphQL: practice writing schemas and resolvers  
> - REST: already solid — review HTTP status codes if needed
