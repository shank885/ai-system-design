# Glossary

Terms are added here as they're introduced in modules. Each entry links back
to the module where it was first taught.

<!-- Format:
### Term
One-line definition. See [Module 00](modules/00-foundations.md).
-->

### Latency
Time for one request/operation to complete, start to finish; usually reported as p50/p95/p99. See [Module 00](modules/00-foundations.md).

### Throughput
Number of requests/operations a system handles per unit time. Often traded against latency via batching. See [Module 00](modules/00-foundations.md).

### Availability
The fraction of time a system successfully responds to requests — says nothing about whether the response is correct. See [Module 00](modules/00-foundations.md).

### Reliability
The system does the correct thing consistently, even under failure — distinct from availability. See [Module 00](modules/00-foundations.md).

### Durability
Once a write is acknowledged, it is not lost, even across crashes — distinct from availability (durable data can be briefly unreadable). See [Module 00](modules/00-foundations.md).

### Fault Tolerance
The system keeps working correctly when components fail, achieved via redundancy rather than failure prevention. See [Module 00](modules/00-foundations.md).

### Consistency
Overloaded term — means something different in ACID vs. CAP vs. everyday English; always state which sense you mean. See [Module 00](modules/00-foundations.md), refined in [Module 11](modules/00-foundations.md) and [Module 06](modules/00-foundations.md).

### Scalability (Vertical vs Horizontal)
Ability to handle growth by adding resources. Vertical = bigger machine (hard ceiling, SPOF). Horizontal = more machines (no ceiling, but introduces distributed coordination problems). See [Module 00](modules/00-foundations.md).

### SPOF (Single Point of Failure)
Any component whose failure takes down the whole system; a primary design goal is eliminating these via redundancy. See [Module 00](modules/00-foundations.md).

### Idempotency
An operation that can be applied multiple times without changing the result beyond the first application — critical for safe retries over unreliable networks. See [Module 00](modules/00-foundations.md).

### Stateless vs Stateful
A stateless service keeps no client-specific data between requests (easy to scale horizontally); a stateful service (e.g. a database) remembers data across requests. See [Module 00](modules/00-foundations.md).

### Cache Hit / Cache Miss
A cache hit means requested data was found in a faster memory tier (e.g. CPU cache); a miss means the CPU must fall through to a slower tier. Sequential access patterns are prefetch-friendly and hit-heavy; random access causes misses. See [Module 01](modules/01-inside-one-machine.md).

### Context Switch
The OS saving one process/thread's state and loading another's to share a CPU core; costs time directly and indirectly via cache pollution. A key reason thread-per-connection designs don't scale to huge concurrency. See [Module 01](modules/01-inside-one-machine.md).

### Virtual Memory / Paging / Swapping
Each process sees a private address space translated to physical RAM via pages; when RAM is full, the OS can swap pages to disk, causing severe (orders-of-magnitude) latency spikes. See [Module 01](modules/01-inside-one-machine.md).

### Sequential vs Random I/O
Reading/writing contiguous disk locations (sequential) is far faster than scattered locations (random) on both HDD and SSD (gap larger on HDD). Foundational to WAL, LSM-tree, and log-structured queue design. See [Module 01](modules/01-inside-one-machine.md), expanded in [Module 04](modules/01-inside-one-machine.md).

### fsync / Durability Mechanism
Forcing the OS to flush buffered writes to persistent storage before returning; the concrete mechanism behind "durable" claims. A write acknowledged before fsync (or replication) can be lost on crash. See [Module 01](modules/01-inside-one-machine.md).

### Blocking vs Non-blocking vs Async I/O
Blocking I/O suspends a thread until data is ready (thread-per-connection doesn't scale); async I/O (epoll/kqueue/IOCP) lets one thread manage many connections via OS-level multiplexing, notified only when a socket is ready. See [Module 01](modules/01-inside-one-machine.md).

### Race Condition
When correctness depends on unpredictable timing/interleaving of concurrent operations on non-atomic operations (e.g. `x += 1` is read-modify-write, not atomic). Same root cause as retry-unsafety of non-idempotent operations across a network. See [Module 01](modules/01-inside-one-machine.md).

### Concurrency vs Parallelism
Concurrency: structuring a program to make progress on multiple tasks over overlapping time, without requiring simultaneous execution (e.g. a single-core event loop). Parallelism: literally executing multiple tasks at the same physical instant, requiring multiple cores/machines. See [Module 01](modules/01-inside-one-machine.md).

### End-to-End Principle
Keep the network core (routers, IP) simple/dumb, push reliability and intelligence to the endpoints. Explains why IP is best-effort and TCP (not the network) provides reliability. See [Module 02](modules/02-networking.md).

### TCP vs UDP
TCP: reliable, ordered, connection-oriented, congestion-controlled byte stream (handshake + retransmission cost). UDP: fire-and-forget datagrams, no ordering/reliability guarantees, low overhead. Choose based on whether late-but-complete or timely-but-imperfect matters more. See [Module 02](modules/02-networking.md).

### DNS TTL
How long a DNS record is cached before resolvers re-fetch it; trades freshness for lookup cost/speed, same trade-off as application caching (Module 08). Explains delayed propagation after DNS changes. See [Module 02](modules/02-networking.md).

### TLS Handshake
Negotiates encryption and verifies server identity via certificates on top of TCP; uses asymmetric crypto briefly to bootstrap a cheap symmetric session key. Costs additional round trips (fewer in TLS 1.3, near-zero with 0-RTT resumption). See [Module 02](modules/02-networking.md).

### Head-of-Line Blocking
When one stalled/lost unit of data blocks delivery of later, already-available data behind it. Occurs per-connection in HTTP/1.1, per-TCP-connection in HTTP/2 (despite app-level multiplexing), and is eliminated per-stream in HTTP/3 (QUIC/UDP). See [Module 02](modules/02-networking.md).

### HTTP/1.1 vs HTTP/2 vs HTTP/3
1.1: one in-flight request per connection. 2: multiplexed streams over one TCP connection (still TCP-level head-of-line blocking). 3: built on QUIC/UDP, per-stream reliability, integrated TLS 1.3, survives client IP changes. See [Module 02](modules/02-networking.md).

### Idempotency Key
A client-generated unique ID attached to a non-idempotent request (e.g. `POST`) so the server can recognize and dedupe retries, converting an unsafe-to-retry operation into one safe to retry. See [Module 03](modules/03-api-design.md).

### Cursor-Based Pagination
Paginating by anchoring to a stable data reference (e.g. last-seen ID/timestamp) rather than numeric offset; stays correct under concurrent writes and stays fast at depth, unlike offset/limit pagination. See [Module 03](modules/03-api-design.md).

### gRPC
RPC framework over HTTP/2 using protobuf (compact binary, schema'd) with native streaming modes; favored for internal service-to-service traffic over REST/JSON's broader compatibility and debuggability. See [Module 03](modules/03-api-design.md).

### N+1 Query Problem
Needing one call plus one additional call per item in a result set (e.g. fetching each post's author separately). Occurs in naive REST/ORM code and reappears inside GraphQL resolvers; standard fix is batching via a dataloader pattern. See [Module 03](modules/03-api-design.md).

### Polling vs Long Polling vs SSE vs WebSockets
Increasingly capable/costly real-time techniques: short polling (wasteful repeated asks), long polling (server holds request open), SSE (one-directional server→client stream over HTTP), WebSockets (full bidirectional persistent channel). Pick the cheapest one matching the actual directionality need. See [Module 03](modules/03-api-design.md).

### Batching vs Concurrency (Fan-out)
Batching reduces the number of round trips for many similar operations (trades latency for throughput). Concurrency/fan-out doesn't reduce call count but runs independent calls in parallel, cutting wall-clock time to ~the slowest call instead of the sum. Often needed together. See [Module 03](modules/03-api-design.md).

### API Gateway vs Load Balancer
A gateway routes by request meaning across different services and handles cross-cutting concerns (auth, rate limiting); a load balancer distributes traffic across interchangeable instances of one service. Commonly deployed together. See [Module 03](modules/03-api-design.md).

### B-Tree
Balanced, page-sized-node tree with high fanout; ~3-4 disk reads to look up any row even at huge scale. Reads are fast; writes update pages in place (random writes). Powers PostgreSQL, MySQL/InnoDB, SQLite. See [Module 04](modules/04-storage-engines.md).

### LSM-Tree (Log-Structured Merge-Tree)
Writes go to an in-memory memtable + WAL, flushed to disk as immutable sorted SSTables via sequential writes; reads may check multiple SSTables. Write-optimized; powers Cassandra, RocksDB, LevelDB, HBase. See [Module 04](modules/04-storage-engines.md).

### Write-Ahead Log (WAL)
Append intended changes to a sequential log before modifying data in place; enables crash recovery and makes durability cheap (sequential write) even when the protected update is a random write. See [Module 04](modules/04-storage-engines.md).

### Bloom Filter
Compact, probabilistic structure that definitively says "key not present" (no false negatives, some false positives), letting LSM-tree reads skip SSTables that can't contain a key without touching disk. See [Module 04](modules/04-storage-engines.md).

### Compaction
Background process merging multiple SSTables into fewer, larger ones, discarding stale/overwritten entries; tunes the read/write/space amplification trade-off in LSM-tree systems. See [Module 04](modules/04-storage-engines.md).

### Erasure Coding
Splits data into k fragments + m parity fragments, reconstructable from any k of k+m; much lower storage overhead than full replication (e.g. ~1.4x vs 3x+) at the cost of read/reconstruction latency and CPU. Favored for cold/large-volume data (object stores, archival). See [Module 04](modules/04-storage-engines.md).
