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

### Primary Key / Clustered Index
The column(s) uniquely identifying a row; most relational engines physically organize the table's B-tree around it. Prefer immutable generated IDs over mutable business fields, since changing the PK can mean relocating the row and updating every foreign key. See [Module 05](modules/05-relational-databases.md).

### Normalization (1NF/2NF/3NF) / Update Anomaly
Structuring tables so each fact is stored once, avoiding update anomalies (a duplicated fact going stale in some rows but not others). 3NF is the practical working target; normalize first, denormalize deliberately. See [Module 05](modules/05-relational-databases.md).

### Join Strategies (Nested Loop / Hash / Merge)
How a query planner physically combines rows from two tables. An indexed join column enables fast strategies (indexed nested loop, merge join); an unindexed one degrades toward near-full-table-scan cost. Check `EXPLAIN` before assuming a join is slow. See [Module 05](modules/05-relational-databases.md).

### Denormalization
Deliberate redundancy to avoid read-time joins, paid for with write-side sync complexity and reintroduced update-anomaly risk. Sync mechanisms: dual writes (risky), triggers, materialized views, async CDC. A targeted optimization for a measured hot read path, not a default. See [Module 05](modules/05-relational-databases.md).

### Training/Serving Skew
When a model's training-time feature pipeline and its real-time serving-time feature pipeline drift out of sync, so the model sees inference data that doesn't match what it learned from — the ML-specific consequence of denormalized-copy drift, silently degrading accuracy rather than raising an error. See [Module 05](modules/05-relational-databases.md), expanded in Module 24.

### ACID
Atomicity (all-or-nothing via WAL/undo logs), Consistency (constraints respected — the ACID-specific sense, distinct from CAP's or the everyday sense), Isolation (concurrent transactions act as if serial — a configurable spectrum), Durability (commit survives crash via fsync'd WAL). See [Module 06](modules/06-transactions.md).

### Isolation Levels (Read Uncommitted/Committed, Repeatable Read, Serializable)
A spectrum of how much concurrent-transaction interleaving is allowed, each level defined by which anomalies it still permits. Weakest to strongest: Read Uncommitted, Read Committed (common default, no dirty reads), Repeatable Read (MySQL default, no non-repeatable reads), Serializable (acts fully serial). See [Module 06](modules/06-transactions.md).

### Lost Update / Write Skew
Lost update: two transactions read-modify-write the same value, one overwrites the other's result — the database-layer version of Module 01's race condition. Write skew: each transaction's individual write is valid on its own, but the combination violates a cross-row invariant neither could see the other breaking. See [Module 06](modules/06-transactions.md).

### MVCC (Multi-Version Concurrency Control)
Keeps multiple versions of each row; each transaction reads from a consistent snapshot so readers never block writers and vice versa. Explains the Read Committed (fresh snapshot per statement) vs Repeatable Read (one snapshot per transaction) distinction. See [Module 06](modules/06-transactions.md).

### Pessimistic vs Optimistic Locking
Pessimistic: lock a row before touching it, others wait (safe, less concurrent, deadlock risk) — best under frequent contention. Optimistic: check a version at write time, reject and retry on conflict (no blocking) — best when conflicts are rare; wasteful under high contention. See [Module 06](modules/06-transactions.md).

### Key-Value Store
Exact-key lookups only (`get`/`put`), value treated as an opaque blob with no secondary query support. Best for caching, sessions, simple lookups (Redis, DynamoDB, etcd). See [Module 07](modules/07-nosql-landscape.md).

### Document Store (Embed vs Reference)
Stores semi-structured, field-queryable documents (MongoDB, Couchbase). Embed vs reference mirrors normalize/denormalize: embed when nested data is always read/written with its parent and not shared; reference when it's large, independently updated, or reused across parents. See [Module 07](modules/07-nosql-landscape.md).

### Wide-Column Store / Query-First Modeling
LSM-tree-based (Cassandra, HBase, Bigtable); schema is designed around specific queries via partition key + clustering columns, often creating multiple denormalized tables per query pattern rather than one normalized table plus joins. See [Module 07](modules/07-nosql-landscape.md).

### Graph Database / Index-Free Adjacency
Nodes store direct pointers to neighbors, so traversal is pointer-following rather than a join per hop — built for deep/unknown-depth relationship traversal (Neo4j, Neptune), not just "data has relationships." See [Module 07](modules/07-nosql-landscape.md).

### Time-Series Database
Optimized for high-volume timestamped writes and time-range aggregation reads via time-based partitioning, downsampling/rollups, and delta-encoded compression (InfluxDB, TimescaleDB, Prometheus). See [Module 07](modules/07-nosql-landscape.md).

### Vector Database / ANN Search
Stores embeddings and supports approximate nearest-neighbor search via specialized indexes (HNSW/IVF, Module 26) since brute-force O(n) search doesn't scale; the storage layer under RAG systems (Pinecone, Weaviate, Milvus, pgvector). See [Module 07](modules/07-nosql-landscape.md).

### Cache-Aside / Write-Through / Write-Behind / Refresh-Ahead
Cache-aside: app reads cache, falls back to DB on miss and populates cache (common default, staleness window). Write-through: writes hit cache+DB synchronously (always fresh, slower writes). Write-behind: writes hit cache first, DB async later (fast, durability risk). Refresh-ahead: proactively refresh hot keys before expiry. See [Module 08](modules/08-caching.md).

### Eviction Policy (LRU/LFU/FIFO) vs TTL
Eviction policy decides what to remove when a full cache needs space (LRU=recency, LFU=frequency, FIFO=insertion order); TTL bounds staleness independently. Used together — an unexpired item can still be evicted for capacity. See [Module 08](modules/08-caching.md).

### Cache Stampede (Thundering Herd)
Many concurrent requests miss the same expired/hot key simultaneously and all hit the database at once. Mitigated by request coalescing/single-flight (only one request recomputes, others wait and share), locking, probabilistic early expiration, or stale-while-revalidate. See [Module 08](modules/08-caching.md).

### Cache Invalidation
Keeping a cache in sync with its source of truth: TTL (simplest, bounded staleness), explicit invalidation on write (precise, easy to miss a path), event-driven via CDC (decoupled, adds delay). Centralizing cache-key construction avoids silent key-mismatch bugs across services. See [Module 08](modules/08-caching.md).

### Single-Leader / Multi-Leader / Leaderless Replication
Single-leader: all writes to one node, followers replay a streamed log (WAL shipping); simplest, no conflicts. Multi-leader: several write-accepting nodes (e.g. one per region) enable local writes but require conflict resolution (LWW, app merge, CRDTs). Leaderless: writes/reads go directly to multiple replicas, reconciled via quorums. See [Module 09](modules/09-replication.md).

### Replication Lag Anomalies
Effects of asynchronous replication: read-your-writes (can't see your own recent write on a lagging replica), monotonic reads (a later read shows older data than an earlier one from a different replica), consistent prefix (causally-ordered writes appear out of order). Each has a specific routing-based mitigation. See [Module 09](modules/09-replication.md).

### Quorum (W + R > N)
With N total replicas, W writes must ack a write and R replicas must be queried for a read; if W+R > N, the read and write sets are guaranteed to overlap (pigeonhole), so a read always sees the latest acknowledged write among contacted replicas. W/R/N are tunable per-operation to trade consistency vs availability vs latency. See [Module 09](modules/09-replication.md).

### Read Repair / Hinted Handoff
Read repair: a quorum read that finds stale replicas opportunistically writes the current version back to them, healing inconsistency as a byproduct of normal reads. Hinted handoff: a write meant for an unreachable replica is temporarily held elsewhere and delivered once that replica recovers. See [Module 09](modules/09-replication.md).

### Partitioning / Sharding
Splitting different subsets of data across machines (orthogonal to replication, which copies the same data) so storage and write throughput scale roughly linearly with node count. Usually combined with replication: each shard is itself replicated. See [Module 10](modules/10-partitioning-sharding.md).

### Range vs Hash Partitioning
Range: contiguous key ranges per partition, enables efficient range queries but risks hotspots under skewed/sequential keys (e.g. timestamps). Hash: `hash(key)` spreads keys near-uniformly, avoiding hotspots but destroying efficient range queries (scatter-gather instead). See [Module 10](modules/10-partitioning-sharding.md).

### Consistent Hashing / Virtual Nodes
Maps keys and nodes onto the same hash ring; adding/removing a node only moves the keys in its immediate ring neighborhood, not the whole dataset. Virtual nodes give each physical node many ring positions to smooth uneven spacing and rebalancing load. See [Module 10](modules/10-partitioning-sharding.md).

### Hot Key / Key Salting
A small number of keys receiving disproportionate traffic overwhelms their one partition despite spare cluster capacity (the partitioning-layer analog of a cache stampede). Key salting appends a random suffix to spread a hot key's writes across sub-keys/partitions, at the cost of needing to aggregate sub-keys on read. See [Module 10](modules/10-partitioning-sharding.md).

### Local vs Global Secondary Index (under partitioning)
Local/document-partitioned: index co-located per partition, fast writes but reads require scatter-gather across every partition. Global/term-partitioned: index partitioned by indexed value, fast targeted reads but writes may cross partitions, often async. See [Module 10](modules/10-partitioning-sharding.md).
