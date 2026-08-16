# Module 08 — Caching: Patterns, Eviction, Redis, CDNs, Stampedes

> **TL;DR:** Caching is the single highest-leverage technique for making
> a system feel fast and for taking load off a database — but it's also
> where "freshness vs. efficiency" (first flagged with DNS TTLs in Module
> 02) becomes a concrete, everyday engineering decision with real failure
> modes. This module covers the specific patterns for keeping a cache and
> its source of truth in sync, how caches decide what to evict, where in
> the stack caching happens (from the browser to a CDN to an in-process
> cache), and the stampede/invalidation problems that catch people out in
> production.

---

## 1. Why caching works: exploiting the same hierarchy from Module 01

A cache is any layer that stores a copy of data in a faster-to-access
place than its source of truth, to avoid repeating expensive work. This
is a direct, larger-scale application of Module 01's memory hierarchy
principle: CPU caches sit between the CPU and RAM for the same reason an
application cache (Redis) sits between an application and a database —
faster tiers are smaller/pricier, so you keep only the "hot" subset of
data there and fall back to the slower tier on a miss.

Caching works because most real workloads exhibit **locality**:

- **Temporal locality** — data accessed recently is likely to be accessed
  again soon (a trending post, a logged-in user's own profile).
- **Spatial/access-pattern locality** — related data is often accessed
  together (all the assets for one webpage).

If access patterns were perfectly random and uniform, caching would
provide almost no benefit — the value of a cache is directly proportional
to how skewed/non-uniform your actual access pattern is (a small
percentage of keys receiving a large percentage of requests — the
**Pareto/power-law distribution** commonly seen in real traffic, often
informally called the "80/20 rule" or **hot key** phenomenon).

---

## 2. Cache read/write patterns

These patterns answer: how does data get *into* the cache, and how do the
cache and the source of truth stay in sync?

### 2.1 Cache-Aside (Lazy Loading)

The application checks the cache first; on a **miss**, it reads from the
database, then populates the cache for next time.

```
Read:  App → Cache (miss) → DB → App writes result into Cache → return
Write: App → DB (cache is not updated directly; either invalidated or left to expire)
```

- **Pros:** Simple, resilient (a cache outage just means more DB load, not
  broken writes), only caches data that's actually been requested.
- **Cons:** First request for any key is always a slow miss (**cold
  cache**); a write to the DB can leave the cache stale until the entry
  expires or is explicitly invalidated.
- This is the most common pattern in practice (e.g., the default way most
  applications use Redis alongside a primary database).

### 2.2 Write-Through

Every write goes to the cache *and* the database together
(synchronously), as part of the same write operation, so the cache is
never stale relative to the database.

- **Pros:** Cache is always consistent with the DB (no staleness window).
- **Cons:** Every write pays the latency cost of writing to both stores;
  data that's *written* but never *read* still gets cached, wasting cache
  space.

### 2.3 Write-Behind (Write-Back)

Writes go to the cache immediately (fast) and are asynchronously flushed
to the database later, in the background.

- **Pros:** Very fast writes (client only waits on the cache write).
- **Cons:** A risky durability gap — if the cache crashes before the async
  flush completes, unflushed writes are lost. This is precisely Module
  01's "acknowledged vs. durable" gap and Module 04's WAL/fsync discussion,
  now at the cache layer: the write was acknowledged before it was safely
  persisted anywhere durable.

### 2.4 Refresh-Ahead

The cache proactively refreshes a value *before* it expires (typically
for known-hot keys, based on access frequency), aiming to avoid ever
serving a stale value or forcing a request to wait on a cold miss.

- **Pros:** Reduces both staleness and cold-miss latency for hot keys.
- **Cons:** Wastes work refreshing keys that might not actually be
  requested again soon; adds complexity to predict which keys are worth
  proactively refreshing.

> **🎤 Interview Angle:** "Which caching pattern should this system use?"
> almost always reduces to the same question Module 00 keeps raising:
> how much staleness can this specific data tolerate, and how
> read-heavy vs. write-heavy is the access pattern? Cache-aside is the
> sensible default answer unless the interviewer's requirements
> specifically demand zero staleness (favoring write-through) or extreme
> write throughput (favoring write-behind, with an explicit acknowledgment
> of its durability risk).

---

## 3. Eviction policies: what to remove when the cache is full

A cache has finite capacity; when it's full and a new item needs to be
added, something must be evicted. The eviction policy determines *what*
gets removed, and it's a direct bet on which data is least likely to be
needed again soon.

- **LRU (Least Recently Used):** evict the item that hasn't been accessed
  for the longest time. Exploits temporal locality directly — the most
  common general-purpose default (used by Redis's default eviction, many
  CPU caches, and countless application-level caches).
- **LFU (Least Frequently Used):** evict the item with the fewest total
  accesses. Better than LRU when access frequency matters more than
  recency (e.g., a rarely-but-consistently-popular item shouldn't be
  evicted just because it wasn't accessed in the last minute) — but more
  expensive to track (needs a counter per item, not just a timestamp) and
  can be slow to adapt when popularity shifts (an old, previously-popular
  item can linger despite no longer being relevant).
- **FIFO (First In, First Out):** evict the oldest-inserted item,
  regardless of access pattern. Simple, cheap, but ignores actual usage —
  rarely the best choice unless access patterns are genuinely
  time-ordered with no reuse (e.g., certain streaming/log-like workloads).
- **TTL (Time To Live)-based expiration:** items are evicted (or marked
  stale) after a fixed duration regardless of access pattern — this is
  the exact same mechanism as DNS TTL from Module 02, now applied to any
  cached value: an explicit, engineer-chosen bound on how stale data is
  allowed to become.

> **⚠️ Trap:** Eviction policy and TTL solve *different* problems and are
> usually used *together*, not as alternatives — TTL bounds staleness
> (correctness), eviction policy manages capacity (what to remove *when
> full*, independent of whether an item happens to still be "fresh" by
> TTL). A cache can evict a perfectly fresh, well-within-TTL item under
> memory pressure if the eviction policy decides it's the least valuable
> item to keep.

---

## 4. Where caching happens: the layers, from client to origin

Caching isn't one layer — production systems typically cache at multiple
levels simultaneously, each catching what the layer above it missed:

```
Browser cache → CDN (edge cache) → Load Balancer / Reverse Proxy cache
   → Application-level cache (Redis/Memcached) → Database (its own
   internal buffer/page cache) → Disk
```

- **Browser cache:** controlled by HTTP caching headers
  (`Cache-Control`, `ETag`) — avoids the network round trip entirely for
  repeat requests from the same client.
- **CDN (Content Delivery Network):** caches content at edge locations
  geographically close to users, primarily for static assets (images, JS,
  CSS) but increasingly for dynamic content too. This directly addresses
  Module 00's cross-continent round-trip latency problem — instead of
  every request crossing an ocean to the origin server, most requests are
  served from a nearby edge node.
- **Reverse proxy cache** (e.g., Nginx, Varnish): sits in front of
  application servers, caching full HTTP responses for repeat identical
  requests before they even reach application code.
- **Application-level cache** (Redis, Memcached): the layer Sections 1-3
  primarily describe — caching query results, computed values, session
  data, etc., under explicit application control.
- **Database's own internal cache** (e.g., PostgreSQL's shared buffers,
  or the OS page cache from Module 01, Section 3.3): even "hitting the
  database" often means hitting RAM, not disk, for frequently accessed
  pages.

> **🤖 AI/ML Callout:** LLM inference systems have their own analogous
> multi-layer caching story. **Prompt/response caching** (caching the
> full output for an identical or near-identical prompt) is the
> Redis-equivalent layer. More specific to LLMs: **KV-cache** (covered in
> depth in Module 25) caches the intermediate attention key/value tensors
> computed for tokens already processed in a generation, so continuing a
> conversation or generating the next token doesn't require
> recomputing attention over the entire prior context from scratch — it's
> the same "avoid repeating expensive work you've already done" principle
> as every cache in this module, just operating on transformer internals
> instead of database query results.

---

## 5. Cache invalidation: "the hard problem," properly named

The famous quip "there are only two hard things in computer science:
cache invalidation and naming things" points at something real: **knowing
when a cached value has become wrong, and correctly removing/updating
it, is genuinely difficult** because it requires either the writer or the
cache to correctly track every place a piece of data might be cached.

Standard strategies:

- **TTL-based expiration** (Section 3) — the simplest strategy: don't try
  to *know* when data changes, just bound how long you're willing to serve
  potentially-stale data. Easy to implement, but you're always trading off
  staleness window vs. cache hit rate (a shorter TTL means fresher data
  but more cache misses hitting the database).
- **Explicit invalidation on write** — when the source of truth changes,
  the application (or a trigger/event) explicitly deletes or updates the
  corresponding cache entry. More precise (no staleness window at all if
  done correctly), but requires the write path to know about every cache
  entry that might need invalidating — easy to miss one, especially as a
  system grows and multiple write paths can affect the same cached data.
- **Event-driven invalidation via CDC** (Module 16) — instead of the
  application manually invalidating caches inline with every write, a
  change data capture stream watches the source database and publishes
  invalidation events consumed by cache-invalidating subscribers,
  decoupling "who writes the data" from "who's responsible for
  invalidating which caches" — trading some propagation delay for much
  better separation of concerns as a system grows.

> **⚠️ Trap:** A very common, easy-to-miss bug: caching a value keyed one
> way, but invalidating (or writing) it a different, inconsistent way
> across different code paths — e.g., caching a user's profile under
> `user:42` in one service, but a different service that updates the
> user's email invalidates `profile:42` instead, silently leaving stale
> data behind indefinitely (or until TTL saves you, if one is set). This
> is why centralizing cache-key construction logic (a single shared
> function/convention, not ad-hoc string formatting scattered across
> services) is a genuinely valuable, low-effort discipline.

---

## 6. Cache stampedes (thundering herd) and mitigations

A **cache stampede** (also called **thundering herd**) happens when a
popular, cached key expires (or the cache goes down entirely), and a
large burst of concurrent requests for that same key all miss the cache
**simultaneously**, and all of them hit the database at once trying to
recompute/refetch the same value — potentially overwhelming the database
with a spike far larger than its normal load, even though the actual
number of *distinct* pieces of data being requested is just one.

### 6.1 Mitigations

- **Request coalescing (a.k.a. "single-flight"):** when multiple
  concurrent requests miss the cache for the *same* key, only let the
  *first* one actually query the database; make the others wait for that
  first request's result and share it, rather than each independently
  hitting the database. This directly caps the "fan-out" from N concurrent
  misses down to 1 actual database query, regardless of N.
- **Locking / mutex on cache-population:** a specific implementation of
  request coalescing — acquire a lock (often itself stored in the cache,
  e.g., Redis's `SETNX`) before recomputing a value, so concurrent
  requests for the same expired key see the lock and wait rather than all
  recomputing independently.
- **Probabilistic early expiration:** instead of every request treating a
  key as either "valid" or "expired" with a hard cutoff, requests
  probabilistically decide to refresh a soon-to-expire key *slightly
  early*, with the probability increasing as the actual expiration time
  approaches — spreading out refresh attempts across many requests over
  a window, instead of all requests colliding at the exact expiration
  instant.
- **Never-expire + background refresh:** don't let popular keys expire on
  the read path at all — instead, refresh them proactively in the
  background (this is Section 2.4's refresh-ahead pattern) or simply keep
  serving a stale value while a background process asynchronously
  refreshes it (**stale-while-revalidate**, a real HTTP `Cache-Control`
  directive and a general pattern), so no request-path miss ever triggers
  a synchronous, blocking recompute at all.

> **📐 Numbers:** Imagine a cached homepage summary computed via an
> expensive aggregate query taking 2 seconds, requested at 5,000 requests/
> sec, with a TTL causing it to expire simultaneously for all clients.
> Without coalescing, the moment it expires, potentially thousands of
> concurrent requests each independently trigger that same 2-second query
> against the database — a spike of thousands of simultaneous
> expensive queries the database was never sized for, likely causing
> cascading slowdowns or an outage. With request coalescing, that same
> moment triggers **exactly one** 2-second query, with every other
> concurrent request simply waiting ~2 seconds for that single result to
> become available and be shared. The difference between these two
> outcomes is entirely due to whether stampede mitigation is in place —
> this is one of the most concrete, high-stakes illustrations in this
> curriculum of why a seemingly small implementation detail (how cache
> misses are handled under concurrency) can be the difference between
> "the system degrades a bit" and "the system falls over."

> **🎤 Interview Angle:** Mentioning cache stampede risk unprompted, when
> discussing a hot cached key with a TTL, is a strong signal — it shows
> you're thinking about the *failure mode* of your own caching design, not
> just its happy path. Following up with a specific mitigation (request
> coalescing or stale-while-revalidate are the two most commonly cited)
> completes the answer.

---

## Cheat Sheet (for fast revision)

- Caching works because of locality (temporal/spatial) — its value is
  proportional to how skewed your access pattern is; near-zero benefit
  under perfectly uniform random access.
- Read/write patterns: cache-aside (lazy, most common, staleness window),
  write-through (always consistent, slower writes), write-behind (fast
  writes, durability risk — same acknowledged-vs-durable gap as Module
  04's WAL), refresh-ahead (proactive refresh of hot keys before expiry).
- Eviction policies (what to remove when full): LRU (recency, common
  default), LFU (frequency, costlier to track), FIFO (simple, usage-blind).
  TTL bounds staleness; eviction manages capacity — used together, not as
  alternatives.
- Caching happens at multiple layers simultaneously: browser → CDN → 
  reverse proxy → application cache (Redis) → DB's own internal cache →
  disk. CDNs directly attack cross-continent latency from Module 00.
- Cache invalidation strategies: TTL (simplest, staleness-bounded),
  explicit invalidation on write (precise, easy to miss a path),
  event-driven via CDC (decoupled, adds propagation delay). Centralize
  cache-key construction to avoid silent invalidation-key mismatches.
- Cache stampede (thundering herd): many concurrent requests miss the
  same expired/hot key simultaneously, all hammering the DB at once.
  Mitigations: request coalescing/single-flight (only 1 request actually
  recomputes), locking, probabilistic early expiration, stale-while-
  revalidate/never-expire-with-background-refresh.
- LLM systems have an analogous caching story: prompt/response caching
  (Redis-equivalent) and KV-cache (Module 25) — avoiding recomputation of
  already-processed attention state, the same "don't redo expensive work"
  principle applied to transformer internals.

---

## Quiz

1. Explain why caching provides little to no benefit under a perfectly
   uniform random access pattern, connecting your answer to the concept
   of "locality."
2. Contrast write-through and write-behind caching in terms of what
   happens if the cache crashes immediately after a write is
   acknowledged to the client. Which Module 04 concept does write-behind's
   risk directly mirror?
3. A cache is full, and a new key needs to be added. The least-recently-
   used item happens to still be well within its TTL (i.e., not stale by
   the TTL rule). Will it still be evicted? Explain why TTL and eviction
   policy are solving different problems.
4. Two different services cache and invalidate data for the same user
   under different key naming schemes (`user:42` vs. `profile:42`).
   Describe the specific bug this causes and the general engineering
   practice that prevents it.
5. Walk through, step by step, what happens to a database when a
   heavily-requested cached key expires simultaneously for thousands of
   concurrent clients, with no stampede mitigation in place. Then explain
   how request coalescing changes that outcome.
6. A CDN caches static assets at edge locations close to users. Explain
   specifically which Module 00 problem this solves, and why it wouldn't
   help as much for highly personalized, per-user dynamic content.
7. Connect this module's KV-cache AI/ML callout to the general caching
   principle the whole module is built on — what expensive work is being
   avoided, and what would happen without it?

<details>
<summary><b>Answers — click to expand</b></summary>

1. Caching's value comes from **temporal locality** (data accessed
   recently is likely to be accessed again soon) and **spatial/access-
   pattern locality** (related data tends to be accessed together) — a
   cache only helps because it can keep a relatively small, "hot" subset
   of data readily available and expect a high hit rate against that
   subset. Under a perfectly uniform random access pattern, every piece of
   data is equally likely to be requested at any time, so there's no
   "hot" subset to identify and keep cached — any fixed-size cache would
   have a hit rate roughly proportional to (cache size / total data size)
   regardless of *which* items it holds, since past access gives no useful
   signal about future access. With no locality to exploit, a cache
   provides no advantage over just picking a random subset of data to
   keep readily available, which is not meaningfully useful.

2. With **write-through**, the write goes to both the cache and the
   database synchronously as part of the same operation — so by the time
   the client receives an acknowledgment, the data is already safely in
   the database (durable, per Module 04's fsync/WAL discussion) as well
   as in the cache; if the cache crashes right after, no data is lost,
   since the database already has it. With **write-behind**, the write
   is acknowledged to the client after only reaching the cache, with the
   database update happening asynchronously later — if the cache crashes
   before that async flush completes, the write is **lost entirely**,
   even though the client was already told it succeeded. This is exactly
   the **acknowledged-vs-durable gap** from Module 04 (and originally
   Module 01's fsync discussion) — an operation that returns "success"
   before the data is actually safely persisted anywhere that survives a
   crash is not truly durable at the moment of acknowledgment, regardless
   of how fast or convenient that early acknowledgment feels.

3. Yes, it can still be evicted — **TTL and eviction policy solve
   different, independent problems**. TTL answers "is this data still
   considered fresh/correct enough to serve?" (a correctness/staleness
   concern), while eviction policy answers "which item should be removed
   *when the cache is full and something must go*?" (a capacity-management
   concern). An item can be perfectly fresh by its TTL and still be the
   eviction policy's chosen victim simply because it hasn't been accessed
   recently (under LRU) or accessed often (under LFU) — freshness and
   "worth keeping in a limited-capacity cache" are simply not the same
   question, and a cache implementation applies both rules independently:
   TTL as a hard correctness bound, eviction policy as the tiebreaker for
   what to discard under memory pressure among items that haven't
   necessarily expired yet.

4. The specific bug: when the user's underlying data changes (say, their
   email is updated), the service responsible for the update invalidates
   `profile:42`, but the *other* service's cached copy is stored under
   `user:42` — a completely different key from the invalidator's
   perspective — so that other service's cache entry is never touched and
   continues silently serving the old, stale data indefinitely (or until
   its own independent TTL, if any, eventually expires it, which may be a
   long time or never). The general engineering practice that prevents
   this: **centralize cache-key construction** — a single shared
   function/convention/library used by every service and every code path
   that reads or writes a given piece of cached data, rather than each
   service independently formatting cache key strings by hand, so there's
   exactly one place that determines "what key does user 42's profile
   live under," eliminating the possibility of two code paths silently
   disagreeing.

5. Without mitigation: the moment the key expires, every one of the
   thousands of concurrent clients requesting it independently
   experiences a cache miss at roughly the same time, and each one
   independently issues the same expensive query/recomputation against
   the database to repopulate the cache — so instead of one query running
   once, the database is hit with potentially thousands of *identical*,
   simultaneous, expensive queries, a load spike the database was likely
   never provisioned to handle, which can cause severe slowdown or a full
   outage (and can even prevent the cache from ever successfully
   repopulating, since the database itself may become too overloaded to
   respond to any of them in reasonable time — a self-reinforcing
   collapse). **Request coalescing** changes this outcome by ensuring only
   the *first* request that misses the cache for that specific key is
   allowed to actually query the database; every other concurrent request
   for the same key is made to wait for that first request's in-flight
   result and then shares it once available, rather than independently
   triggering its own database query — collapsing what would have been
   thousands of simultaneous database queries down to exactly one,
   regardless of how many clients were waiting.

6. A CDN directly solves **Module 00's cross-continent round-trip latency
   problem** (the same one behind Module 02's discussion of TLS
   termination at the edge): instead of every request traveling all the
   way to a potentially distant origin server (e.g., ~150ms round trip
   cross-continent), a CDN serves the (cached) content from an edge
   location physically close to the user, cutting that latency down to
   roughly a local/regional round trip. It wouldn't help as much for
   highly personalized, per-user dynamic content because CDN caching
   fundamentally relies on the same **locality** principle from Question
   1 — it works well when many different users request the *same* content
   (a static image, a shared JS bundle), so caching it once at each edge
   location serves many subsequent requests. Highly personalized content
   (a specific user's private dashboard, computed uniquely for them) has
   effectively no reuse across different users/requests — there's no "hot,
   shared" content to cache at the edge, so each request would still need
   to reach the origin (or a personalized cache closer to application
   logic) regardless of CDN presence.

7. The general caching principle: avoid repeating expensive work whose
   result hasn't changed. In the KV-cache case, the "expensive work" is
   the attention computation a transformer performs over all prior tokens
   in a sequence to produce the key and value tensors used at each
   generation step — this computation is deterministic given the same
   prior tokens, so recomputing it from scratch for every new token
   generated (rather than reusing what was already computed for the
   tokens processed so far) would be pure wasted, repeated work — the
   exact same category of waste a database query cache avoids by not
   re-running an unchanged query. Without a KV-cache, generating each new
   token in a sequence would require re-running the full attention
   computation over the *entire* growing context from scratch every single
   time, making generation cost grow substantially with sequence length
   for reasons that have nothing to do with genuinely new work being
   done — Module 25 covers exactly how large this cost actually is and how
   KV-cache management becomes its own scaling/memory-management problem
   at serving time.

</details>

---

## What's next

**Module 09 — Replication: Leader/Follower, Multi-Leader, Leaderless,
Quorums** opens **Part III — Distribution**, moving from "how does one
machine (or one cache) store and serve data efficiently" into "how do
multiple machines cooperate to store the *same* data reliably" — the
foundation for everything about availability, durability at scale, and
the CAP theorem that follows in Module 11.
