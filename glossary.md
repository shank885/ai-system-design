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
