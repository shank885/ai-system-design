# Module 01 — Inside One Machine: CPU, Memory, Disk, OS, Concurrency

> **TL;DR:** Every distributed system is, underneath, just many copies of
> *one machine* talking over a network. If you don't have a correct mental
> model of what one machine can and can't do cheaply — how its CPU, memory,
> disk, and OS actually behave — you will misjudge every trade-off built on
> top of it: why caching works, why sequential disk I/O is prized, why
> "just add a thread" doesn't always help, why context switches show up in
> your p99 latency. This module builds that ground floor.

---

## 1. The CPU: cores, cache, and context switches

### 1.1 Cores and clock speed

A CPU core executes instructions one at a time (per pipeline stage,
roughly). Modern servers have many cores (commonly 16-128+), which is why
"more throughput" on one machine usually means **parallelism across
cores**, not a faster single core — clock speeds have been roughly flat
for 15+ years because of power/heat limits (this is why "just get a faster
CPU" stopped being the default scaling strategy industry-wide, and
horizontal scaling became the norm).

### 1.2 The memory hierarchy and why cache misses matter

Recall the latency ladder from Module 00. The reason it exists is physical:
faster memory is smaller and more expensive per byte, so hardware is built
as a hierarchy — small and fast at the top, large and slow at the bottom.

```
Registers    (few bytes, ~sub-ns)
   ↓
L1 Cache     (~32-64 KB per core, ~1 ns)
   ↓
L2 Cache     (~256KB-1MB per core, ~4 ns)
   ↓
L3 Cache     (shared across cores, several MB, ~10-20 ns)
   ↓
Main Memory / RAM   (GBs, ~100 ns)
   ↓
SSD / Disk   (TBs, ~10 μs - 10 ms)
```

Your code doesn't control this hierarchy directly, but data layout and
access patterns determine how often the CPU has a **cache hit** (fast) vs.
a **cache miss** (must wait on a slower level). Sequential memory access
(iterating an array) is fast because it lets the CPU **prefetch** the next
chunk into cache before you ask for it. Random access (chasing pointers
around a linked list or hash map scattered in memory) causes cache misses
constantly.

> **⚠️ Trap:** "Big O" complexity analysis (an algorithms-class habit)
> assumes uniform-cost memory access. In reality, an O(n) array scan can
> beat an O(log n) tree lookup for small-to-medium n, purely because the
> array is cache-friendly and the tree is not. This is precisely why
> LSM-trees and B-trees (Module 04) are designed around minimizing random
> I/O, not just minimizing comparison counts.

### 1.3 Context switching

The OS gives the illusion that many programs run "at once" by rapidly
switching the CPU between them (**time-slicing**). Each switch is a
**context switch**: the OS saves one process/thread's registers and state,
loads another's. This costs time (roughly microseconds) and — more
importantly for system design — **it usually flushes CPU caches**, meaning
the next few operations for the newly-scheduled task are cache misses even
if they wouldn't otherwise be.

> **📐 Numbers:** A context switch itself costs on the order of 1-10
> microseconds directly, but the *indirect* cost from cache pollution can
> be several times larger. This is one reason why systems handling huge
> numbers of concurrent connections (Module 02/03) prefer event loops or
> async I/O over "one OS thread per connection" — thousands of threads
> means thousands of potential context switches competing for the same
> cores.

> **🤖 AI/ML Callout:** GPUs exist precisely because ML workloads (matrix
> multiplication) are embarrassingly parallel and cache/memory-bandwidth
> bound in a very regular, predictable pattern — a GPU trades a CPU's
> complex per-core control logic (branch prediction, out-of-order
> execution) for thousands of simple cores plus massive memory bandwidth.
> When you learn GPU scheduling and batching in Module 25, it's the same
> memory-hierarchy story as here, just with different hardware built to
> exploit the *same* physical trade-off (bandwidth vs. latency, hit vs.
> miss) at a different point on the curve.

---

## 2. Memory: RAM, virtual memory, and garbage collection

### 2.1 Virtual memory

Every process sees its own private, contiguous address space (**virtual
memory**), even though physical RAM is shared and fragmented across all
running processes. The OS + CPU's **MMU (Memory Management Unit)**
translate virtual addresses to physical ones via **page tables**, in fixed-size
chunks called **pages** (commonly 4KB).

This gives you two critical guarantees for free: process isolation (one
process can't accidentally read another's memory) and the illusion of more
memory than physically exists, via **swapping** pages to disk when RAM is
full.

> **⚠️ Trap:** Swapping is not a "slower but fine" fallback — it's often
> catastrophic. Once a server starts swapping RAM to disk (or SSD) under
> memory pressure, latency can jump by 3-4 orders of magnitude for the
> affected process, because a "memory access" silently became a disk
> access. In production, you generally want to **provision for zero
> swapping** for latency-sensitive services, not rely on swap as a safety
> net.

### 2.2 Garbage collection (why it matters for system design, not just language choice)

Languages like Java, Go, Python, and JS manage memory automatically via
**garbage collection (GC)**: the runtime periodically finds objects no
longer reachable and frees them. This is convenient, but GC pauses can
introduce latency spikes ("stop-the-world" pauses in older/simpler
collectors) that are invisible in average latency but show up sharply at
p99/p999.

> **🎤 Interview Angle:** If you're asked to design a low-latency system
> (e.g., a stock trading matching engine, Module 34-adjacent territory)
> and you propose a GC'd language without addressing GC pause risk, a
> strong interviewer will probe this. You don't need to rule out GC'd
> languages — you need to *acknowledge the trade-off* (tuned GC settings,
> off-heap memory, or a non-GC'd language for the hottest path) rather
> than being unaware of it.

---

## 3. Disk: HDD vs. SSD, and sequential vs. random I/O

### 3.1 The physical difference

- **HDD (spinning disk):** data sits on physical platters; reading requires
  physically moving a read head (**seek**) and waiting for the disk to spin
  to the right position (**rotational latency**). A random-access seek
  costs ~10ms — an eternity in CPU terms.
- **SSD (solid state):** no moving parts, data read electronically via
  flash memory. Random access is vastly faster than HDD (microseconds,
  not milliseconds), though still much slower than RAM, and writes have
  their own quirks (SSDs write in blocks and need "garbage collection" of
  their own via wear-leveling — not the same GC as in Section 2, but a
  useful naming coincidence to notice).

### 3.2 Sequential vs. random I/O — the single most important disk fact for system design

Whether on HDD or SSD, **sequential reads/writes are dramatically faster
than random ones**, though the *gap* is far larger on HDD (100x+) than on
SSD (often much smaller, but still real due to internal flash page/block
structure).

This single fact explains an enormous amount of database and storage
system design that will follow in Part II:

- **Write-Ahead Logs (WAL)** append sequentially instead of updating
  records in place, converting slow random writes into fast sequential
  ones (Module 04).
- **LSM-trees** (used by Cassandra, RocksDB, LevelDB) are built entirely
  around turning random writes into sequential ones, accepting more
  complex reads in exchange (Module 04).
- **Log-structured message queues** like Kafka get their famous throughput
  substantially from sequential disk access patterns, not from avoiding
  disk entirely (Module 15).

> **📐 Numbers:** A spinning HDD might do ~100-200 random IOPS
> (I/O operations per second) but many hundreds of MB/s for sequential
> access. A modern NVMe SSD might do hundreds of thousands of random IOPS
> and multiple GB/s sequential — the absolute numbers moved a lot with
> SSDs, but the *shape* of the trade-off (sequential ≥ random) persists,
> which is why storage engine design principles from the HDD era are still
> relevant today, just less extreme.

### 3.3 Filesystems, buffered I/O, and `fsync`

Most application writes don't go straight to the physical disk — they go
into the OS's **page cache** in RAM first, and the OS flushes them to disk
later on its own schedule. This makes writes look fast, but means a crash
before the flush can lose "written" data.

- **`fsync`** (or equivalent) forces the OS to actually flush data to
  persistent storage before returning, trading latency for a durability
  guarantee.
- This is the physical mechanism behind the durability guarantees discussed
  abstractly in Module 00 — "durable" in practice usually means "fsync'd,"
  and every database's durability claims ultimately reduce to exactly when
  and how often it calls fsync (or an equivalent, e.g., write to multiple
  replicas before ack, covered in Module 09).

> **⚠️ Trap:** "The write returned successfully" does not automatically
> mean "the data is durable." If a system acknowledges a write the moment
> it hits an in-memory buffer or OS page cache — without an fsync or
> without replicating to another machine — a crash right after can lose
> that "successful" write. This gap between *acknowledged* and *durable*
> is one of the most common sources of subtle data-loss bugs and a classic
> interview probing point ("what happens if the machine crashes right
> after this write is acknowledged?").

---

## 4. The OS: processes, threads, and I/O models

### 4.1 Processes vs. threads

- A **process** has its own memory address space; processes are isolated
  from each other by default (safe, but communication between them —
  **IPC**, inter-process communication — requires explicit mechanisms:
  pipes, sockets, shared memory).
- A **thread** is a unit of execution *within* a process; all threads in a
  process share the same memory address space. This makes communication
  between threads cheap (just shared variables) but dangerous (**race
  conditions** — see Section 5) since nothing stops two threads from
  reading/writing the same memory simultaneously unless you explicitly
  synchronize.

### 4.2 Blocking, non-blocking, and async I/O

This is the concept that determines how a server handles thousands of
simultaneous connections/requests, and it's foundational for Module 02-03.

- **Blocking I/O:** a thread that calls "read from network/disk" is
  suspended (blocked) until the data arrives. Simple to program, but if you
  want to serve N concurrent clients, naively you need N threads — and
  threads are not free (memory for each thread's stack, plus context-switch
  overhead as discussed in Section 1.3).
- **Non-blocking I/O:** a call that would block instead returns immediately
  with "not ready yet," and the application must poll or be notified later.
- **Event loop / async I/O (e.g., epoll on Linux, kqueue on BSD/macOS,
  IOCP on Windows):** the OS lets a single thread register interest in many
  file descriptors/sockets at once and get notified when *any* of them is
  ready, without needing a thread per connection. This is the mechanism
  behind Node.js's single-threaded event loop, Nginx's high-concurrency
  design, and most modern high-throughput servers.

> **🤖 AI/ML Callout:** LLM inference servers face an analogous fan-out
> problem: many concurrent requests need to share limited, expensive GPU
> resources. The solution — continuous/dynamic batching of many requests
> into shared GPU compute (Module 25) — is conceptually the same move as
> an event loop multiplexing many connections onto one thread: don't give
> each request its own dedicated expensive resource; interleave many
> requests over a shared, efficient resource instead.

> **🎤 Interview Angle:** "Why can Nginx handle tens of thousands of
> concurrent connections on a few threads while a naive thread-per-request
> server falls over at a few thousand?" is a classic probe of whether you
> understand this section, not just whether you've memorized "Nginx is
> fast."

---

## 5. Concurrency: race conditions, locks, and why "just add threads" isn't free

### 5.1 Race conditions

A **race condition** occurs when the correctness of a result depends on the
unpredictable timing/interleaving of concurrent operations. Classic
example: two threads both read a counter's value (say, 5), both increment
their local copy to 6, both write back 6 — one increment is silently lost,
even though `x += 1` looks like a single instruction in source code (it
almost never is at the machine level: it's a read, a modify, and a write,
and another thread can interleave between any of those steps).

This is precisely why Module 00 flagged `x += 1` as non-idempotent and
dangerous to retry — the same underlying issue (an operation that is not a
single atomic step) causes both the retry-safety problem across a network
*and* the race-condition problem across threads on one machine. It's the
same root cause — **lack of atomicity** — showing up at two different
scales.

### 5.2 Locks, mutexes, and the cost of correctness

A **mutex (mutual exclusion lock)** ensures only one thread executes a
critical section at a time, preventing races — at the cost of throughput
(other threads must wait) and the risk of **deadlock** (two threads each
waiting on a lock the other holds).

> **⚠️ Trap:** Adding more threads to a lock-heavy piece of code often
> makes it *slower*, not faster, past a certain point — threads spend more
> time contending for the lock and getting context-switched than doing
> useful work. This is directly analogous to (and the single-machine
> ancestor of) the coordination overhead you'll see formalized for
> multi-machine systems in Module 12 (consensus/coordination) — "adding
> more workers" is never free once they need to agree on shared state,
> whether those workers are threads on one box or nodes on a network.

### 5.3 Concurrency vs. parallelism (a distinction worth being precise about)

- **Concurrency**: structuring a program to *deal with* multiple things at
  once (may or may not literally execute simultaneously — e.g., a
  single-core event loop is concurrent but not parallel).
- **Parallelism**: literally executing multiple things *at the same
  physical instant* (requires multiple cores/machines).

You can have concurrency without parallelism (single core, interleaved via
context switching or an event loop) and parallelism without much explicit
concurrency management (e.g., simple data-parallel batch jobs). Conflating
the two leads to imprecise design conversations.

---

## Cheat Sheet (for fast revision)

- Memory hierarchy (fast→slow, small→large): registers → L1 → L2 → L3 →
  RAM → SSD/disk. Sequential access is prefetch-friendly; random access
  causes cache misses.
- Context switches cost time directly (~μs) and indirectly (cache
  pollution) — this is why thread-per-connection doesn't scale to huge
  concurrency.
- Virtual memory gives isolation + the illusion of more RAM via paging;
  swapping to disk under memory pressure can be catastrophic for latency.
- GC pauses are invisible in averages but hit p99/p999 — always a
  trade-off to name explicitly for latency-sensitive designs.
- Sequential I/O ≫ random I/O on both HDD and SSD (gap far larger on HDD).
  This single fact motivates WALs, LSM-trees, and log-structured queues.
- "Write succeeded" ≠ "write is durable" unless it was fsync'd (or
  replicated) — durability is a concrete mechanism, not a vague promise.
- Blocking I/O = 1 thread per connection (doesn't scale past thousands);
  event loops/async I/O (epoll etc.) let one thread manage many connections.
- Race conditions come from non-atomic operations shared across
  threads/machines — same root cause as network retry-safety issues from
  Module 00.
- Locks prevent races but cost throughput and risk deadlock; more threads
  contending for a lock can make things slower, not faster.
- Concurrency (dealing with many things) ≠ Parallelism (doing many things
  at the exact same instant).

---

## Quiz

1. Explain why an O(n) sequential array scan can outperform an O(log n)
   tree lookup for small/medium data sizes, using concepts from this
   module (not just asymptotic notation).
2. A server starts swapping memory to disk under load. Why is this often
   far worse than "just a bit slower," and what latency-sensitive
   guarantee should you provision to avoid entirely?
3. Your database acknowledges a write instantly, before calling fsync or
   replicating it anywhere. The machine crashes one second later. What
   happened to that "successful" write, and what's the general lesson for
   evaluating any system's durability claims?
4. Why can Nginx (or Node.js) serve tens of thousands of concurrent
   connections on a handful of OS threads, while a naive "one thread per
   connection" design collapses under similar load? Name the underlying
   OS mechanism.
5. Two threads increment a shared counter without a lock;
   `finalCount < expectedCount` after both finish. Explain the exact
   sequence of events that causes this, and connect it to a concept from
   Module 00.
6. True or false, with justification: "Adding more threads to a
   lock-heavy critical section always improves throughput." What
   multi-machine concept from a later module does this single-machine
   fact foreshadow?
7. Define concurrency and parallelism precisely enough that you could
   classify a single-core event loop and a 32-core batch data pipeline
   correctly using your definitions.

<details>
<summary><b>Answers — click to expand</b></summary>

1. Big-O analysis counts comparisons/operations assuming uniform-cost
   memory access, but real memory access cost varies enormously by access
   pattern (Section 1.2). A sequential array scan reads contiguous memory,
   which the CPU can prefetch into cache ahead of time, making nearly every
   access a fast cache hit. A tree traversal (e.g., a linked structure of
   nodes scattered across the heap) jumps between non-contiguous memory
   locations, causing a cache miss on nearly every node visited. For small
   to medium n, the tree's fewer "steps" (log n vs n) can be outweighed by
   each of its steps being far more expensive (cache miss) than each of the
   array's steps (cache hit) — so real wall-clock time doesn't track
   asymptotic step-count alone.

2. Swapping silently turns what the application believes is a fast memory
   access into a slow disk (or SSD) access — a latency jump of 3-4 orders
   of magnitude for the affected operations, and it tends to cascade (more
   swapping causes more contention, causing more swapping) rather than
   degrade gracefully. The guarantee to provision for is **zero swapping**
   for latency-sensitive services — size memory so the working set always
   fits in RAM, rather than relying on swap as a "slower but survivable"
   fallback.

3. The write is lost. It only existed in memory (application buffer and/or
   OS page cache) and was never forced to persistent storage (fsync) nor
   copied to another machine (replication) before the crash — both of
   which are the actual mechanisms durability is built from. The general
   lesson: never accept "the write succeeded" as proof of durability at
   face value — always ask *what exactly* happens between "acknowledged"
   and "safely on non-volatile storage in at least one place that survives
   this specific failure," because that gap is where data loss lives.

4. The mechanism is **event-driven / async I/O via OS-level multiplexing**
   (epoll on Linux, kqueue on BSD/macOS, IOCP on Windows). Instead of
   dedicating one OS thread (with its own stack memory and context-switch
   overhead) to each connection and blocking that thread while waiting for
   I/O, a small number of threads register interest in many sockets at
   once and get notified only when a socket actually has data ready. This
   avoids both the memory cost of thousands of thread stacks and the
   scheduling/context-switch overhead of the OS juggling thousands of
   mostly-idle threads.

5. Both threads read the counter's current value into a local copy before
   either writes back the incremented result — e.g., both read 5, both
   compute 6 locally, both write 6 back, so one increment is silently lost
   even though two increments were performed. This is a **race condition**
   caused by `+=` not being a single atomic operation (it's read-modify-
   write under the hood) — the same root cause Module 00 used to explain
   why `x += 1` is unsafe to retry blindly over a network: an operation
   that isn't atomic can be corrupted by *any* uncontrolled interleaving,
   whether that interleaving comes from concurrent threads on one machine
   or concurrent/retried requests across a network.

6. **False.** Past a certain point, adding more threads to a lock-heavy
   critical section increases throughput loss to contention — threads
   spend more time waiting for the lock and being context-switched than
   doing useful work, and lock contention itself has overhead. This
   foreshadows the **consensus/coordination overhead** covered in
   Module 12: just as threads contending for a shared lock don't scale
   linearly, machines that must coordinate/agree on shared state (via
   consensus protocols, distributed locks, etc.) hit the same
   diminishing-and-eventually-negative returns as you add more of them.

7. **Concurrency** is a program structure that deals with multiple
   logically-independent tasks making progress over overlapping time
   periods, without requiring simultaneous physical execution.
   **Parallelism** is multiple tasks executing at the literal same
   physical instant, which requires multiple execution units (cores/
   machines). A single-core event loop is **concurrent but not parallel**
   — it interleaves many tasks' progress via non-blocking I/O and
   scheduling, but only one instruction executes at any given nanosecond.
   A 32-core batch data pipeline processing independent partitions
   simultaneously is **parallel** (and, if it also structures/interleaves
   many logical tasks per core, concurrent as well) — genuinely
   simultaneous execution across cores.

</details>

---

## What's next

**Module 02 — Networking for System Designers** builds directly on Section
4's I/O model discussion: now that one machine's I/O is understood, we
open up what happens when that I/O crosses the wire to another machine —
IP, TCP/UDP, DNS, TLS, and the evolution from HTTP/1 to HTTP/3.
