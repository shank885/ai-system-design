# Module 02 — Networking for System Designers: IP, TCP/UDP, DNS, TLS, HTTP/1-2-3

> **TL;DR:** Module 01 explained what happens when I/O crosses process
> boundaries on one machine. This module opens up what happens when I/O
> crosses the wire to a *different* machine. Every distributed systems
> problem in this curriculum — replication lag, timeouts, retries,
> partitions — traces back to the fact that the network sits between your
> components and it is slow, unreliable, and outside your control. You
> need a working model of IP, TCP/UDP, DNS, TLS, and HTTP to reason about
> *why* those failures happen and *where* the levers are to mitigate them.

---

## 1. The layered model, pragmatically

You don't need the full 7-layer OSI model memorized. You need the four
layers that actually show up in system design conversations, bottom to
top:

| Layer | Job | Examples |
|---|---|---|
| **Network (IP)** | Get a packet from one machine to another, across networks, best-effort | IP, routing |
| **Transport (TCP/UDP)** | End-to-end delivery semantics between two processes | TCP, UDP |
| **Session/Security (TLS)** | Encrypt and authenticate the connection | TLS/SSL |
| **Application** | Meaningful message formats applications actually speak | HTTP, gRPC, DNS, WebSocket |

Each layer solves a specific problem the layer below it deliberately
doesn't solve. IP doesn't guarantee delivery — TCP adds that. TCP doesn't
guarantee privacy or authenticity — TLS adds that. Neither defines what a
"request" or "response" means — HTTP (or gRPC, etc.) adds that. Keeping
these separate is what lets you reason about *which layer* a given failure
or design decision belongs to.

---

## 2. IP: addressing and the best-effort promise

**IP (Internet Protocol)** is responsible for getting a packet from a
source address to a destination address, across potentially many
intermediate networks (**routers** hop packets closer to the destination
one link at a time). Crucially, IP makes **no reliability guarantees**:
packets can be dropped, duplicated, or arrive out of order. This is not a
flaw — it's a deliberate design choice that keeps routers simple and fast,
pushing reliability to the endpoints (a principle called the **end-to-end
principle**, worth knowing by name: put smarts at the edges, keep the
network core simple).

- **IPv4** addresses are 32-bit (~4.3 billion addresses, exhausted for
  direct global allocation years ago — the reason **NAT** exists: it lets
  many private devices share one public IP by having a router rewrite
  addresses/ports as traffic passes through).
- **IPv6** uses 128-bit addresses, a large enough space that NAT is no
  longer strictly necessary, though NAT remains common due to legacy
  infrastructure and habit.

> **⚠️ Trap:** "The network is reliable" is #1 on the classic list of
> *Fallacies of Distributed Computing*. Every retry mechanism, timeout
> policy, and idempotency requirement you'll learn in Modules 13 and 18
> exists specifically because IP (and the internet built on it) makes zero
> reliability promises — reliability, where it exists, was engineered on
> top, not given for free.

---

## 3. TCP vs. UDP: the fundamental transport trade-off

This is one of the highest-leverage distinctions in all of system design —
almost every "why does X protocol use TCP but Y uses UDP" question reduces
to this trade-off.

### 3.1 TCP (Transmission Control Protocol)

TCP turns IP's unreliable, unordered packet delivery into a **reliable,
ordered, connection-oriented byte stream**. It does this via:

- **The three-way handshake** (`SYN` → `SYN-ACK` → `ACK`) to establish a
  connection before any data flows — this costs a full round trip *before*
  useful work starts.
- **Sequence numbers** so the receiver can reorder packets and detect
  gaps/duplicates.
- **Acknowledgments and retransmission** — the sender resends anything not
  acknowledged in time.
- **Flow control** — the receiver tells the sender how much it can accept
  right now (its buffer isn't infinite).
- **Congestion control** — the sender deliberately throttles itself when it
  detects network congestion (packet loss is interpreted as a congestion
  signal), to avoid making a congested network worse. This is a
  cooperative, self-limiting behavior baked into TCP itself.

All of this reliability machinery costs latency and overhead — which is
exactly what you're paying for when you choose TCP.

### 3.2 UDP (User Datagram Protocol)

UDP is a thin layer over IP: send a packet (**datagram**), no handshake, no
ordering guarantee, no retransmission, no congestion control. It is
"fire and forget."

### 3.3 When to choose which

| Use TCP when... | Use UDP when... |
|---|---|
| Correctness of every byte matters (file transfer, API calls, DB replication) | Losing occasional data is acceptable, and low latency matters more than completeness (live video/audio, gaming state updates) |
| Ordering matters | You'll build your own lightweight reliability/ordering on top if truly needed (e.g., QUIC, DNS, some real-time protocols) |
| You want built-in congestion control | You need tighter control over retransmission behavior yourself |

> **📐 Numbers:** The TCP handshake alone costs one full network round trip
> before any application data is sent — recall from Module 00 that a
> same-datacenter round trip is ~0.5ms, but a cross-continent one is
> ~150ms. For a client making a fresh TCP connection across a continent
> before even starting a TLS handshake (Section 5) or sending an HTTP
> request, you're already several round trips deep before "your" logic
> runs at all. This is the direct motivation for **connection reuse
> (keep-alive)**, **connection pooling**, and protocol-level improvements
> like HTTP/2 and HTTP/3 (Section 6).

> **🤖 AI/ML Callout:** Real-time voice/video AI assistants (live speech-to-
> speech models) often use UDP-based transports (e.g., WebRTC, which is
> built on UDP) for the same reason video calls do: a dropped audio frame
> that would arrive late is worse than one that's simply skipped — TCP's
> insistence on retransmitting and preserving order would introduce
> exactly the kind of stall that makes real-time interaction feel broken.
> Meanwhile, standard LLM API calls (batch or streaming text) use TCP
> (via HTTP), because losing or reordering a token silently would corrupt
> the output — correctness beats latency there.

---

## 4. DNS: turning names into addresses

**DNS (Domain Name System)** resolves human-readable names
(`api.example.com`) into IP addresses. It's a distributed, hierarchical,
heavily-cached lookup system — and it's a great early example of several
patterns you'll see formalized later in this curriculum.

### 4.1 The resolution path

1. Your client asks a **recursive resolver** (often run by your ISP or a
   public service like `8.8.8.8`) to resolve the name.
2. If not cached, the resolver asks a **root server** ("who handles `.com`?"),
   then a **TLD server** ("who handles `example.com`?"), then the
   **authoritative name server** for `example.com`, which returns the
   actual IP.
3. The result is cached at multiple levels (browser, OS, resolver) for a
   duration controlled by **TTL (time to live)**, set by the domain owner.

### 4.2 Why this matters for system design

- **DNS is a caching system**, and it exhibits the exact same trade-off
  you'll meet formally in Module 08 (Caching): low TTL = fresher data but
  more lookup load and slower propagation of changes; high TTL = cheaper
  and faster lookups but stale data lingers longer (e.g., after you migrate
  a service to a new IP, clients with a cached old record won't notice
  until TTL expires).
- **DNS is also a load-balancing and failover mechanism** in its own right:
  returning multiple IPs for one name (round-robin DNS), or using
  **GeoDNS** to return the IP of the nearest regional data center based on
  the resolver's location — a simple, coarse-grained precursor to the load
  balancing covered in depth in Module 14.

> **⚠️ Trap:** DNS propagation is not instant — "I updated the DNS record"
> does not mean "all clients see the new value immediately." Some clients
> will keep hitting the old IP until their cached TTL expires. This is a
> classic operational gotcha during migrations/failovers and a good thing
> to proactively mention in an interview when discussing disaster recovery
> (Module 20).

---

## 5. TLS: encrypting and authenticating the connection

**TLS (Transport Layer Security)**, the successor to SSL, sits on top of
TCP and provides three things: **confidentiality** (encrypted so
eavesdroppers can't read it), **integrity** (tampering is detectable), and
**authentication** (you can cryptographically verify you're talking to who
you think you are, via certificates).

### 5.1 The TLS handshake, at a level useful for system design

1. Client and server negotiate a TLS version and cipher suite.
2. The server presents a **certificate** (issued by a trusted **Certificate
   Authority**) proving its identity; the client verifies it.
3. Both sides derive a shared **symmetric session key** using asymmetric
   cryptography during the handshake (asymmetric crypto is computationally
   expensive, so it's used briefly to bootstrap trust, then a cheaper
   symmetric key encrypts the actual traffic — this hybrid approach is a
   pattern worth recognizing, it recurs in Module 21's security module).
4. From then on, traffic is encrypted with the fast symmetric key.

This handshake costs **additional round trips on top of the TCP
handshake** — historically 2 round trips for TLS 1.2, reduced to
effectively 1 round trip for TLS 1.3 (and **0-RTT** resumption is possible
for repeat connections to a previously-visited server, at some security
trade-off). This stacking of round trips (TCP handshake, then TLS
handshake, then finally the actual HTTP request) is precisely why
**connection reuse** matters so much in practice — paying these costs once
per connection and then reusing it for many requests, rather than paying
them per request.

> **🎤 Interview Angle:** If discussing global-latency-sensitive systems
> (e.g., "design a CDN" or "design a system for users worldwide"), being
> able to say "a fresh HTTPS connection from a distant client costs a TCP
> handshake plus a TLS handshake before the first byte of actual response —
> that's why we terminate TLS at edge nodes close to the user and reuse
> keep-alive connections" demonstrates you understand where the latency
> actually comes from, not just that "CDNs make things faster."

---

## 6. HTTP: from 1.1 to 2 to 3

HTTP is the application-layer protocol nearly all of this curriculum's
APIs (Module 03) sit on top of. Its evolution is a case study in
progressively removing the inefficiencies discussed above.

### 6.1 HTTP/1.1

- Text-based request/response protocol over a single TCP connection.
- Supports **keep-alive** (reuse one TCP connection for multiple sequential
  requests, avoiding repeated handshakes), but requests on one connection
  are processed **one at a time, in order** — a slow response blocks
  everything queued behind it on that connection (**head-of-line
  blocking**).
- Browsers historically worked around this by opening multiple parallel
  TCP connections per host (commonly 6), trading connection overhead for
  parallelism.

### 6.2 HTTP/2

- Introduces **multiplexing**: multiple requests/responses interleaved
  over a *single* TCP connection, each tagged with a stream ID, eliminating
  application-layer head-of-line blocking from Section 6.1.
- Adds **header compression** (HPACK) and **server push** (largely
  unused/deprecated in practice).
- **⚠️ Residual problem:** because it's still built on TCP, if a single TCP
  packet is lost, TCP's in-order delivery guarantee blocks *all* multiplexed
  streams on that connection until the lost packet is retransmitted —
  head-of-line blocking reappears, just moved down a layer, from the
  application to the transport.

### 6.3 HTTP/3 (built on QUIC)

- Replaces TCP with **QUIC**, which runs over UDP but reimplements
  reliability, ordering, and congestion control itself — but critically,
  **per-stream**, not per-connection. A lost packet only blocks the one
  stream it belonged to, not every other in-flight request — solving
  Section 6.2's residual problem at the root.
- Integrates TLS 1.3 directly into the handshake (rather than layering it
  separately on top), reducing to essentially one combined round trip to
  establish a fully secure, multiplexed connection — and supports 0-RTT
  resumption for repeat connections.
- Because it's UDP-based, it also handles network changes better (e.g., a
  phone switching from Wi-Fi to cellular) without dropping the connection,
  using a **connection ID** that survives the client's IP address changing.

> **📐 Numbers:** Going from a cold HTTP/1.1-over-TLS-1.2 connection to a
> warmed-up HTTP/3 connection can remove 2-3 full round trips before the
> first useful byte of response — at ~150ms cross-continent per round
> trip, that's not a rounding error, it's several hundred milliseconds of
> pure protocol overhead eliminated. This is exactly the kind of number an
> interviewer wants to see you reason through, not memorize.

> **🤖 AI/ML Callout:** Streaming LLM responses (tokens arriving
> incrementally as they're generated) are typically delivered over HTTP
> using **chunked transfer encoding** or **Server-Sent Events (SSE)** on
> top of a single long-lived HTTP connection — conceptually similar to how
> a video stream keeps one connection open and pushes data as it becomes
> available, rather than the client polling repeatedly. Module 03 covers
> SSE vs. WebSockets vs. polling in depth; the point to internalize now is
> that "streaming" at the application layer is just a particular usage
> pattern of the same HTTP/TCP machinery covered in this module, not a
> fundamentally different protocol.

---

## Cheat Sheet (for fast revision)

- Layer separation: IP (best-effort delivery) → TCP/UDP (transport
  semantics) → TLS (security) → HTTP/etc. (application meaning). Each layer
  deliberately doesn't solve what the layer above handles.
- IP makes zero reliability guarantees — this is *the* root cause behind
  needing retries, timeouts, and idempotency later in the curriculum.
- TCP = reliable, ordered, connection-oriented, congestion-controlled, at
  the cost of handshake latency and per-packet in-order delivery. UDP =
  fire-and-forget, low overhead, no ordering/reliability guarantees.
- Choose UDP when timely-but-imperfect beats late-but-complete (live
  audio/video); choose TCP when every byte and its order matters (most
  APIs, file transfer, DB traffic).
- DNS is a distributed, cached name→IP lookup system; TTL trades freshness
  for lookup cost, exactly like caching (Module 08). DNS also does
  coarse-grained load balancing/failover via multiple records or GeoDNS.
- TLS adds confidentiality + integrity + authentication on top of TCP,
  using asymmetric crypto briefly to bootstrap a cheap symmetric session
  key. TLS 1.3 costs ~1 round trip; 0-RTT resumption is possible for repeat
  connections.
- HTTP/1.1: one request in flight at a time per connection (head-of-line
  blocking); mitigated historically via multiple parallel connections.
- HTTP/2: multiplexes streams over one TCP connection, but TCP's in-order
  delivery still causes head-of-line blocking on packet loss.
- HTTP/3 (QUIC/UDP): per-stream reliability, no cross-stream head-of-line
  blocking, integrated TLS 1.3, survives client IP changes.
- Total round trips before the first useful response byte stack up: TCP
  handshake + TLS handshake + first request — this is why connection
  reuse, edge TLS termination, and protocol upgrades (HTTP/2 → 3) all
  matter for latency-sensitive, geographically distributed systems.

---

## Quiz

1. Explain the "end-to-end principle" using IP and TCP as the concrete
   example, and name one later-module topic that exists *because* of this
   design choice.
2. A live multiplayer game sends frequent position updates to a server.
   Why might the designers choose UDP over TCP for this traffic, even
   though UDP can lose or reorder packets? What would go wrong if TCP were
   used instead, specifically due to TCP's guarantees?
3. You migrate a service to a new server with a new IP and update the DNS
   record immediately. A user reports still reaching the old server 20
   minutes later. Explain why, using a concept from this module, and name
   the module/concept later in the curriculum this same trade-off
   reappears under.
4. Why does establishing a fresh HTTPS connection from a client on another
   continent cost more than one round trip before any application data is
   exchanged? Break down which round trips are being spent on what.
5. HTTP/2 multiplexes many requests over a single TCP connection to solve
   HTTP/1.1's head-of-line blocking — yet HTTP/2 connections can still
   stall entirely when one packet is lost. Explain why, precisely, and
   explain in one sentence how HTTP/3 fixes it at the root.
6. Give one concrete reason a real-time voice AI assistant would prefer a
   UDP-based transport (e.g., WebRTC) over a standard HTTPS/TCP connection
   for the audio stream itself, connecting it to the TCP-vs-UDP trade-off
   from Section 3.

<details>
<summary><b>Answers — click to expand</b></summary>

1. The end-to-end principle says: keep the network core (routers, IP)
   simple and dumb, and push complexity/intelligence to the endpoints. IP
   deliberately provides only best-effort delivery — no acknowledgment, no
   ordering, no retransmission — and TCP, running only on the two
   communicating endpoints, is entirely responsible for building
   reliability on top. This design choice is *why* Module 13
   (Distributed Transactions / idempotency) and Module 18 (Resiliency:
   timeouts, retries, circuit breakers) exist as major topics at all — if
   the network guaranteed delivery itself, applications wouldn't need to
   defensively design for partial failure, retries, and duplicate
   delivery.

2. UDP is preferred because in a live game, a *stale* position update is
   useless or actively harmful (you want the *latest* position, not a
   perfectly complete history of every position ever sent), and low
   latency matters more than perfect completeness. If TCP were used
   instead: TCP guarantees in-order delivery, so if an early packet is
   lost, TCP will hold back and refuse to deliver *any* later, already-
   arrived packets to the application until the lost one is retransmitted
   and arrives — introducing exactly the kind of stall/lag a real-time
   game cannot tolerate, even though the "stale" data being blocked on is
   worthless anyway by the time it's needed.

3. DNS results are cached at multiple levels (client/OS/resolver) for a
   duration set by the TTL on the DNS record. Updating the authoritative
   record doesn't retroactively invalidate caches that already hold the
   old IP — those caches keep serving the stale answer until their TTL
   expires. This is the exact same freshness-vs-efficiency trade-off
   formalized generally for application caching in **Module 08**, and it's
   also directly relevant to disaster recovery / failover planning in
   **Module 20**, where DNS TTL is a real operational lever (and
   limitation) during a region failover.

4. Two separate handshakes must complete sequentially before any HTTP data
   flows: (1) the **TCP three-way handshake** (SYN, SYN-ACK, ACK) —
   roughly one round trip — to establish the underlying reliable
   connection, and (2) the **TLS handshake** on top of that (TLS 1.3 costs
   roughly one more round trip; older TLS 1.2 costs more) to negotiate
   encryption and verify the server's certificate. Only after both
   complete can the actual HTTP request be sent — so at minimum ~2 round
   trips are spent purely on connection setup before any application logic
   runs, and at cross-continent latencies (~150ms/round trip from
   Module 00), that's 300ms+ of pure overhead before "real work" starts.

5. HTTP/2's multiplexing happens at the *application* layer (multiple
   logical streams share one TCP connection), but the underlying TCP
   connection still enforces strict in-order byte delivery to the
   application. If a single TCP packet is lost, TCP will not deliver any
   *later*-arrived data — even data belonging to a completely unrelated
   HTTP/2 stream — until the lost packet is retransmitted and arrives,
   because TCP doesn't know or care about HTTP/2's stream boundaries; it
   just sees one ordered byte stream. So one lost packet stalls every
   multiplexed stream on that connection. HTTP/3 fixes this at the root by
   replacing TCP with QUIC (over UDP), which implements reliability and
   ordering *per stream* rather than per connection, so a lost packet only
   blocks the one stream it belonged to.

6. A dropped or late-arriving audio frame in a real-time voice conversation
   is far worse if TCP tries to guarantee its eventual, in-order delivery —
   the retransmission and in-order delivery requirement would force the
   receiver to wait for that one late frame, introducing an audible stall
   or lag spike, by which point the frame is stale and useless for a live
   conversation anyway. A UDP-based transport instead simply drops the
   occasional frame and moves on to the next one, prioritizing low,
   consistent latency over perfect completeness — exactly mirroring the
   live-multiplayer-game reasoning in Question 2: timely-but-imperfect
   beats late-but-complete for real-time interactive media.

</details>

---

## What's next

**Module 03 — API Design & Communication Styles** builds directly on this
module's HTTP foundation: REST, RPC, gRPC (which runs over HTTP/2), GraphQL,
WebSockets, and the polling/SSE/WebSocket trade-off for real-time-feeling
communication — plus how these choices interact with the round-trip and
head-of-line-blocking concerns just covered.
