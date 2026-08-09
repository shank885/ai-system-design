# Module 03 — API Design & Communication Styles: REST, RPC, gRPC, GraphQL, WebSockets

> **TL;DR:** Module 02 covered the transport plumbing (TCP/UDP, HTTP/1-2-3).
> This module covers what you actually build on top of it: how clients and
> servers agree on the *shape* of requests and responses, and how to choose
> a communication style — request/response, streaming, or push — for a
> given problem. Every style here is a different trade-off among the same
> handful of forces from Module 00: latency, chattiness, coupling, and
> how "live" the data needs to be.

---

## 1. REST: resource-oriented HTTP

**REST (Representational State Transfer)** models a system as a set of
**resources** (nouns — `/users/42`, `/orders/17`), manipulated via standard
HTTP methods (verbs):

| Method | Meaning | Idempotent? | Safe (no side effects)? |
|---|---|---|---|
| `GET` | Read a resource | Yes | Yes |
| `POST` | Create a resource / non-idempotent action | No | No |
| `PUT` | Replace a resource entirely | Yes | No |
| `PATCH` | Partially update a resource | No (usually) | No |
| `DELETE` | Remove a resource | Yes | No |

Recall **idempotency** from Module 00: `PUT /users/42 {name: "Bob"}` sets
the same end state no matter how many times it's retried — safe to retry
blindly after a timeout. `POST /orders` (create a new order) is *not*
idempotent — retrying it blindly after a timeout can create duplicate
orders, which is exactly why **idempotency keys** (a client-generated
unique ID attached to the request, so the server can recognize and dedupe
a retried `POST`) are a standard production pattern, covered again in
Module 13.

> **⚠️ Trap:** Many "REST" APIs marketed as RESTful are really just
> "HTTP + JSON," ignoring HATEOAS (hypermedia-driven navigation) and
> proper resource modeling. This is fine in practice — pragmatic REST is
> the industry norm — but know that "REST" as an interview term usually
> means "resource-oriented HTTP API using standard verbs and status
> codes," not the full academic definition, and be ready to use HTTP
> status codes correctly (`404` not found, `409` conflict, `429` rate
> limited, `503` unavailable) since that precision signals experience.

### 1.1 Pagination

Returning a huge collection in one response is both slow and memory-heavy
on both ends. Two dominant patterns:

- **Offset/limit pagination** (`?offset=100&limit=20`) — simple, but
  breaks under concurrent writes (items can shift between pages, causing
  skips/duplicates) and gets slower for large offsets on many databases
  (the DB still has to scan/skip the offset rows).
- **Cursor-based pagination** (`?cursor=<opaque-token>&limit=20`) — the
  cursor encodes a stable position (e.g., "last seen ID + timestamp"),
  avoiding the shifting-page problem and typically staying fast regardless
  of how deep you paginate. This is what almost every large-scale feed
  (Module 31) uses in practice.

---

## 2. RPC and gRPC: calling a function, not fetching a resource

**RPC (Remote Procedure Call)** frames communication as "call a function on
another machine," rather than "manipulate a resource." This fits
action-oriented operations (`ProcessPayment(orderId)`) more naturally than
forcing everything into REST's noun-based model.

**gRPC** is the dominant modern RPC framework, and it's worth understanding
*why* it's fast, tying directly back to Module 02:

- Runs over **HTTP/2** by default, getting multiplexing and header
  compression for free.
- Uses **Protocol Buffers (protobuf)** — a compact **binary** serialization
  format with a strict schema (`.proto` file) — instead of text-based JSON.
  This means smaller payloads and faster (de)serialization, at the cost of
  losing JSON's human-readability and requiring a shared schema/codegen
  step on both client and server.
- Supports **streaming** natively: unary (one request, one response,
  like REST), server-streaming, client-streaming, and bidirectional
  streaming — all as first-class API contract types, not bolted on.

> **📐 Numbers:** Protobuf payloads are commonly 3-10x smaller than the
> equivalent JSON, and (de)serialization is meaningfully faster because
> there's no text parsing — this matters most for high-QPS internal
> service-to-service traffic, less so for a public API where
> human-debuggability and broad client compatibility often outweigh the
> efficiency gain.

### 2.1 REST vs. gRPC — when to choose which

| Choose REST/JSON when... | Choose gRPC when... |
|---|---|
| Public-facing API, broad client compatibility (browsers, many languages) needed | Internal service-to-service traffic, both ends you control |
| Human-readability/debuggability with `curl` matters | Performance/payload size matters at high QPS |
| Simple CRUD-shaped resources | Action-oriented calls, or need native streaming |

> **🎤 Interview Angle:** "Why does this microservices architecture use
> gRPC internally but expose a REST API externally?" is one of the most
> common architecture-review questions — the answer is precisely the table
> above: internal traffic you control end-to-end benefits from gRPC's
> efficiency and schema strictness; external traffic needs broad client
> compatibility and easy debugging, which REST/JSON provides.

---

## 3. GraphQL: letting the client shape the response

REST and gRPC both have the server decide the shape of each endpoint's
response. **GraphQL** flips this: the client sends a query describing
*exactly* which fields it wants, potentially across multiple related
resources in one request, and the server returns exactly that shape —
nothing more, nothing less.

This solves two classic REST pain points:

- **Over-fetching** — a REST endpoint returns a fixed shape (e.g., a full
  user object) even if the client only needed the username.
- **Under-fetching → N+1 round trips** — needing data from multiple
  resources (a user, their orders, and each order's items) might mean
  multiple sequential REST calls; GraphQL can express this as one query,
  resolved server-side.

The cost: a single GraphQL query can trigger arbitrarily complex,
expensive server-side resolution (a poorly-designed query can silently
turn into hundreds of downstream calls — the **N+1 query problem** just
moves from the client's round trips to the server's internal resolution),
so production GraphQL servers need query complexity limits, batching
(e.g., the **Dataloader** pattern), and careful caching — caching is
notably harder than REST's, since GraphQL typically uses a single endpoint
and POST requests, defeating simple HTTP-level (URL-keyed) caching.

> **⚠️ Trap:** GraphQL doesn't eliminate the N+1 problem — it just moves
> where it lives, from "the client makes many round trips" to "the
> server's resolvers make many downstream calls per single client
> request." If asked to design a GraphQL API at scale, mentioning
> **batching/dataloaders** to collapse those downstream calls demonstrates
> you understand this, rather than treating GraphQL as a free lunch.

---

## 4. Real-time-feeling communication: polling, long polling, SSE, WebSockets

Sometimes a client needs to know about server-side changes without asking
"is there anything new?" repeatedly and wastefully. There's a spectrum of
solutions, each with a different chattiness/complexity/directionality
trade-off — directly built on the HTTP/TCP mechanics from Module 02.

| Technique | How it works | Directionality | Notes |
|---|---|---|---|
| **Short polling** | Client repeatedly asks "anything new?" on a fixed interval | Client→Server only | Simple, but wastes requests when nothing changed, and adds latency up to one polling interval |
| **Long polling** | Client asks, server *holds the request open* until there's something to say (or a timeout), then responds; client immediately re-asks | Client→Server only (server can only "answer," not initiate) | Reduces wasted requests vs. short polling, but each open request still consumes server resources; each response requires a new connection/request cycle |
| **SSE (Server-Sent Events)** | Client opens one HTTP connection; server pushes a stream of text events over it indefinitely | Server→Client only | Built on plain HTTP (works through most infra, auto-reconnect built into the browser API), but strictly one-directional |
| **WebSockets** | A single handshake upgrades an HTTP connection to a persistent, full-duplex TCP-like channel | Bidirectional | Most powerful/flexible, but stateful (harder to load-balance — Module 14 — since a client must keep talking to the same server instance holding its connection) and more operationally complex |

> **🤖 AI/ML Callout:** Streaming an LLM's response token-by-token to a
> chat UI is a textbook SSE use case — it's server→client only (the model
> is generating a one-directional stream of tokens), works over plain
> HTTP/1.1 or HTTP/2, and gets auto-reconnect for free in browsers. Full
> WebSockets are typically reserved for genuinely bidirectional AI use
> cases — e.g., a live voice assistant needing to *both* stream audio out
> and accept interruption/barge-in input from the user simultaneously.
> Choosing SSE vs. WebSockets for an AI product is exactly this module's
> "directionality" question, not a separate AI-specific decision.

> **🎤 Interview Angle:** "Design a live notification system" or "design a
> live chat app" almost always requires you to explicitly justify polling
> vs. long polling vs. SSE vs. WebSockets — a strong answer states the
> actual directionality and latency requirement first ("does the client
> ever need to push data mid-stream, or only receive?"), then picks the
> cheapest technique that satisfies it, rather than jumping straight to
> WebSockets by default (which is the most common overcorrection —
> WebSockets solve a bidirectional need, and are unnecessary operational
> overhead if you only need server→client push).

---

## 5. Minimizing chattiness: the recurring theme

Module 00 flagged that sequential network round trips burn latency budget.
Module 02 showed TCP/TLS handshakes stack up per connection. This module's
techniques are largely about the *application-level* version of the same
problem:

- REST's N+1 problem (needing multiple sequential calls to assemble one
  view) → solved by GraphQL's single expressive query, or by REST's own
  convention of **composite/aggregate endpoints** (`GET
  /users/42/dashboard` returning everything a dashboard needs in one call,
  intentionally denormalized for the client's convenience).
- gRPC's streaming modes → avoid re-establishing a connection and
  re-sending headers for every single message in a rapid sequence.
- **Batching** — combining multiple logical operations into one network
  call (e.g., a bulk `POST /orders/batch` instead of N individual `POST
  /orders` calls) — trades a bit of latency (waiting to assemble a batch)
  for dramatically higher throughput and fewer round trips, the exact
  latency-vs-throughput trade-off from Module 00's trade-off list.

> **📐 Numbers:** If an operation needs data from 5 different services and
> you call them sequentially at ~1ms each internally, that's ~5ms just in
> round trips before any real computation — call them **concurrently**
> (fan out, then join the results) instead of sequentially, and the cost
> drops to roughly the slowest single call (~1ms), not the sum. This is a
> distinct lever from batching: batching reduces the *number* of round
> trips for many similar operations, concurrency reduces the *wall-clock
> cost* of round trips you must make anyway because they're independent of
> each other.

---

## 6. API Gateways, versioning, and contracts

### 6.1 API Gateway

In a system with many backend services, an **API Gateway** is a single
entry point that sits in front of them, handling cross-cutting concerns
once instead of duplicating them in every service: authentication, rate
limiting (Module 17), request routing, request/response transformation,
and sometimes aggregating multiple backend calls into one client-facing
response (mitigating Section 5's chattiness problem at the edge).

> **⚠️ Trap:** An API Gateway is not the same thing as a **Load Balancer**
> (Module 14), even though both sit "in front of" backend services. A load
> balancer distributes traffic across *interchangeable* instances of the
> *same* service based on network/transport-level concerns; a gateway
> routes to *different* services based on the request's meaning (path,
> method, payload) and handles application-level concerns. In practice
> they're often deployed together (gateway routes to the correct service,
> then a load balancer picks which instance of that service handles it),
> and the distinction is a common interview clarifying point.

### 6.2 Versioning and backward compatibility

APIs change, but clients (especially external ones, or mobile apps that
can't be force-upgraded instantly) can't always update in lockstep with
the server. Common strategies:

- **URL versioning** (`/v1/orders`, `/v2/orders`) — explicit and simple,
  but can lead to maintaining many parallel versions.
- **Header/content-negotiation versioning** — cleaner URLs, less visible,
  slightly more complex to implement and debug.
- **Additive-only evolution** — the practice of only ever *adding*
  optional fields, never removing or repurposing existing ones, so old
  clients keep working unmodified against a newer server (this is the
  default philosophy behind protobuf's field-numbering design in gRPC,
  and behind GraphQL's schema evolution model — both are built assuming
  fields get added over time, and deprecated rather than deleted).

> **🎤 Interview Angle:** Mentioning backward compatibility unprompted when
> designing any public-facing API is a strong signal — it shows you're
> thinking about the system's *lifecycle*, not just its initial shape.

---

## Cheat Sheet (for fast revision)

- REST: resource-oriented, standard HTTP verbs, know which are idempotent
  (`GET`/`PUT`/`DELETE` yes, `POST`/usually `PATCH` no) — this is why
  idempotency keys exist for `POST`.
- Cursor-based pagination beats offset/limit at scale: stable under
  concurrent writes, doesn't slow down for deep pages.
- gRPC = HTTP/2 + protobuf (binary, schema'd, compact) + native streaming
  modes; best for internal service-to-service traffic you control on both
  ends. REST/JSON wins for public APIs needing broad compatibility and
  debuggability.
- GraphQL lets the client shape the response, solving over-fetching and
  client-side N+1 round trips — but moves N+1 to the server's resolvers;
  batching/dataloaders are the standard mitigation.
- Real-time spectrum, cheapest-to-most-powerful: short polling → long
  polling → SSE (server→client stream over plain HTTP) → WebSockets (full
  bidirectional). Pick the cheapest one that satisfies the actual
  directionality requirement — don't default to WebSockets.
- Chattiness mitigations: batching (fewer round trips for many similar
  ops, trades latency for throughput) vs. concurrency/fan-out (same round
  trips, but parallel instead of sequential, cutting wall-clock cost to
  ~the slowest single call).
- API Gateway ≠ Load Balancer: gateway routes by request meaning across
  different services + handles cross-cutting concerns; load balancer
  distributes traffic across interchangeable instances of one service.
- Version APIs defensively: URL or header versioning, and prefer
  additive-only schema evolution so old clients keep working against a
  newer server.

---

## Quiz

1. A client calls `POST /orders` to place an order, but the response times
   out. The client doesn't know if the order was actually created. What
   REST/HTTP-method property makes blindly retrying this dangerous, and
   what production mechanism fixes it? Connect this to a concept from
   Module 00.
2. Explain why cursor-based pagination stays fast and correct under
   concurrent writes/deletes while offset/limit pagination can skip or
   duplicate items, and can slow down for deep pages.
3. Two teams debate: one wants gRPC for a new public-facing mobile API,
   the other wants REST/JSON. Lay out the actual trade-off and state which
   you'd recommend for a *public* API and why.
4. GraphQL is sometimes pitched as "solves the N+1 problem." Explain
   precisely what's misleading about that claim, and name the standard
   mitigation.
5. You're designing a live sports score notification feature (server
   pushes score updates to subscribed clients; client never needs to send
   data back once subscribed) and a collaborative whiteboard (both sides
   need to send drawing events to each other in real time). Which
   real-time technique fits each, and why would picking WebSockets for
   both be a design smell?
6. An endpoint needs to call 4 independent internal services to assemble
   one response. Contrast the effect of (a) batching these into fewer
   calls vs. (b) calling all 4 concurrently instead of sequentially — what
   problem does each actually solve, and could you need both at once?
7. Explain the difference between an API Gateway and a Load Balancer in
   one or two sentences each, and describe a system where both are used
   together.

<details>
<summary><b>Answers — click to expand</b></summary>

1. `POST` is **not idempotent** — retrying it can create a second,
   duplicate order, because the server has no way to know the retry is
   "the same request" rather than a new one. The production fix is an
   **idempotency key**: the client generates a unique ID once per logical
   operation and sends it with the request (and any retries of it); the
   server records which idempotency keys it has already processed and
   returns the original result for a repeat, instead of creating a second
   order. This directly extends Module 00's point about `x += 1` vs.
   `SET x = 5` — `POST /orders` behaves like the dangerous non-idempotent
   case, and the idempotency key is what converts it into something safe
   to retry, by making the *effective* operation idempotent even though
   the underlying action (creating a new row) inherently isn't on its own.

2. Offset/limit pagination identifies a page purely by position ("skip
   100, take 20"). If an item is inserted or deleted before that position
   between page requests, every subsequent item shifts, causing the next
   page to skip an item (something shifted past the boundary unseen) or
   repeat one (something shifted back into view again) — the page
   boundary is defined relative to a list that isn't stable. It can also
   get slower for large offsets because many databases still must
   scan/count through the skipped rows to find the offset, even though it
   discards them. Cursor-based pagination instead anchors each page to a
   stable reference point from the actual data (e.g., "items after ID X,
   inserted after timestamp Y") — that anchor doesn't move even if items
   elsewhere in the list are added or removed, so no items are skipped or
   duplicated, and lookups from a given cursor are typically an efficient
   indexed range scan regardless of how deep into the collection you are.

3. gRPC's benefits — compact binary payloads, fast (de)serialization,
   native streaming — assume both client and server share a generated
   schema/codegen pipeline and that you control both ends. A public mobile
   API faces a very different reality: many client versions in the wild
   you can't force-upgrade, potential need for browser/web clients (gRPC's
   HTTP/2-only, streaming-heavy design has historically been awkward from
   browsers without a translation layer like grpc-web), and a strong need
   for easy debugging (`curl`, browser dev tools) by both your own team and
   possibly third-party integrators. **REST/JSON is the better fit for a
   public API** for these reasons — broad compatibility and
   debuggability generally outweigh gRPC's raw efficiency gains for
   external-facing traffic, whereas gRPC's advantages shine specifically
   for internal service-to-service calls where both ends are controlled
   and upgraded together.

4. The misleading part: GraphQL doesn't eliminate the N+1 problem, it
   **relocates** it. Without mitigation, a single GraphQL query fetching,
   say, a list of posts and each post's author can trigger one query for
   the post list plus one additional resolver call *per post* to fetch its
   author — the same N+1 pattern that plagues naive REST/ORM code, except
   now it happens inside the server's resolver execution for a single
   client request, instead of being visible as N+1 separate client-to-server
   round trips. The standard mitigation is **batching via a dataloader
   pattern**: collect all the individual "fetch author for post X" calls
   requested during one query's resolution and issue them as a single
   batched downstream call (e.g., "fetch authors WHERE id IN (...)")
   instead of N separate ones.

5. The live sports score feature is **strictly server→client** (client
   subscribes once, then only receives) — a great fit for **SSE**: it runs
   over plain HTTP, gives you automatic reconnection in the browser, and
   matches the actual directionality needed with the least operational
   complexity. The collaborative whiteboard needs **genuine bidirectional,
   low-latency communication** (either party can send a drawing event at
   any time) — this is exactly what **WebSockets** are for. Picking
   WebSockets for the sports score feature would be a design smell because
   it adds real operational cost (stateful connections that complicate
   load balancing, more complex reconnection handling, more server
   resources held open per client) to satisfy a directionality requirement
   the feature doesn't actually have — you'd be paying for bidirectional
   capability you never use.

6. **(a) Batching** reduces the *number* of round trips when you have many
   similar/repeated operations to perform (e.g., turning 100 individual
   `POST /orders` calls into one `POST /orders/batch` call) — it trades a
   small amount of added latency (time spent assembling the batch) for
   dramatically higher throughput and far fewer total round trips.
   **(b) Concurrency/fan-out** doesn't reduce the number of calls at all —
   you still make all 4 calls — but it reduces the *wall-clock time* those
   calls take by issuing them in parallel instead of one after another, so
   total latency approaches the slowest single call rather than the sum of
   all 4. These solve different problems and are frequently needed
   **together**: e.g., batch similar requests to reduce call count, *and*
   fire the resulting (fewer) distinct batched calls to different services
   concurrently rather than sequentially, to minimize both the number of
   round trips and the wall-clock cost of the ones that remain.

7. An **API Gateway** is a single entry point that routes requests to
   different backend *services* based on the request's meaning (path,
   method, headers, payload), and centrally handles cross-cutting concerns
   like authentication, rate limiting, and request aggregation. A **Load
   Balancer** distributes traffic across multiple *interchangeable
   instances of the same service*, based on transport/network-level
   signals, to spread load and provide failover — it has no concept of
   "which service" beyond what it's configured to route to, and doesn't
   care about request semantics the way a gateway does. A typical system
   uses both together: a client request first hits the API Gateway, which
   determines *which backend service* the request is for (e.g., the
   `orders` service vs. the `payments` service) based on the path, and then
   that service sits behind its *own* load balancer, which picks *which
   instance* of the orders service actually handles this particular
   request.

</details>

---

## What's next

**Module 04 — Storage Engine Internals** moves from "how machines talk to
each other" (Modules 01-03) into "how machines remember things" — B-trees,
LSM-trees, write-ahead logs, and indexes, building directly on Module 01's
sequential-vs-random-I/O foundation to explain why different databases are
built the way they are.

