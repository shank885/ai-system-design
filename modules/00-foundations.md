# Module 00 — Mental Models, Vocabulary & Back-of-the-Envelope Math

> **TL;DR:** System design is the discipline of deciding *where things live*,
> *how they communicate*, and *what breaks when scale or failure hits* —
> under real constraints (money, latency, consistency, team size). Before any
> concrete technology, you need: (1) a shared vocabulary, (2) a way to reason
> about scale numerically, and (3) intuition for the handful of trade-off axes
> that every design decision reduces to.

---

## 1. What is "a system," really?

A **system** is a set of components that communicate to satisfy requirements
that no single component could satisfy alone. In software, this almost always
means: some *clients* generate *requests*, some *servers* do *work*, some
*storage* remembers *state*, and a *network* connects them all — imperfectly.

The entire field of system design exists because of three uncomfortable facts:

1. **One machine is not enough.** Eventually you hit a wall on CPU, memory,
   disk, or network I/O for a single box. You must split work across many
   machines.
2. **Networks are unreliable.** The moment you have more than one machine,
   you have a network between them, and networks drop packets, add latency,
   partition, and reorder messages. This single fact is the root cause of
   almost everything hard in distributed systems (we'll formalize this in
   Module 11 as the CAP theorem).
3. **Failure is not an edge case, it's a certainty.** With enough machines
   running long enough, *something* is always failing — a disk, a process, a
   rack, a data center. Good design assumes failure as the default state, not
   an exception.

Everything in this curriculum is a response to these three facts.

### The four resources you are always allocating

Whatever the system, you are always trading off across four physical
resources:

| Resource | Bottleneck symptom | Classic fix |
|---|---|---|
| **CPU** (compute) | High latency under load, request queuing | More/faster machines, caching, async work |
| **Memory** (RAM) | OOM kills, GC pauses, cache misses | Bigger machines, eviction policies, sharding |
| **Disk** (storage + I/O) | Slow reads/writes, disk full | SSDs, better data structures (Module 04), replication |
| **Network** (bandwidth + latency) | Timeouts, slow cross-region calls | CDNs, compression, fewer round trips, locality |

Every "advanced" system design pattern you'll learn is ultimately a clever
way of moving load off one of these four resources — usually by spending
more of a *different* one. Caching trades memory for CPU/disk. Replication
trades storage/network for availability. Batching trades latency for
throughput. Keep asking "which resource is this saving, and which resource
is it spending instead?" — it will demystify almost every pattern you meet.

---

## 2. Vocabulary you need before anything else

These words get thrown around loosely in most explanations. Precise
definitions now save confusion for the next 40 modules.

- **Latency** — time for *one* request to complete, start to finish.
  Measured for a single operation. Usually reported as a percentile
  (p50, p95, p99) because averages hide the pain: a handful of very slow
  requests can dominate user experience while barely moving the mean.
- **Throughput** — number of requests/operations a system handles per unit
  time (e.g., requests/sec, or "RPS"). You can often increase throughput at
  the cost of latency (batching) and vice versa.
- **Availability** — the fraction of time a system successfully responds to
  requests. Expressed as "nines" (99.9% = "three nines" ≈ 8.7 hours of
  downtime/year). Not the same as correctness — a system can be "available"
  and return a wrong or stale answer.
- **Reliability** — the system does the *correct* thing, consistently, even
  under failure. A system can be highly available but unreliable (always
  responds, sometimes wrong).
- **Durability** — once data is acknowledged as written, it is not lost,
  even across crashes/power loss. Different from availability: data can be
  durable (safely on disk somewhere) but temporarily unavailable (that disk's
  server is down).
- **Fault tolerance** — the system keeps working correctly when some
  component fails. Achieved through redundancy, not through preventing
  failure (you can't prevent failure at scale).
- **Consistency** — informally, "do all readers see the same data at the
  same time?" This word is dangerously overloaded — it means something
  different in ACID (Module 06) vs. CAP (Module 11) vs. everyday English.
  We will keep flagging which sense is meant.
- **Scalability** — the ability to handle growth (more users, data, traffic)
  by adding resources, ideally without a redesign.
  - **Vertical scaling (scale up)** — bigger machine (more CPU/RAM/disk on
    one node). Simple, but has a hard ceiling and a single point of failure.
  - **Horizontal scaling (scale out)** — more machines. No hard ceiling, but
    introduces the network/coordination problem — this is why distributed
    systems exist as a field.
- **SPOF (Single Point of Failure)** — any component whose failure takes
  down the whole system. A primary design goal is finding and eliminating
  SPOFs, usually via redundancy.
- **Idempotency** — an operation that can be applied multiple times without
  changing the result beyond the first application (`SET x = 5` is
  idempotent; `x += 1` is not). Critical for safe retries in unreliable
  networks — you'll see this constantly from Module 13 onward.
- **State vs. stateless** — a *stateless* service keeps no client-specific
  data between requests (any instance can handle any request — easy to scale
  horizontally). A *stateful* service (e.g., a database) remembers data
  across requests, which is exactly why storage is the hard part of
  distributed systems.

> **⚠️ Trap:** People say "the system is down" when they mean "unavailable,"
> and "the system is consistent" without saying *which* consistency they
> mean. In an interview, naming the exact term you're using signals
> seniority. Vague words hide unexamined assumptions.

---

## 3. Back-of-the-envelope math: the interview and design superpower

You cannot pick between "one Postgres instance" and "a sharded, replicated
cluster" without first estimating *how much load you actually have*. This
skill — Fermi estimation for systems — is arguably the single highest-leverage
thing to get fluent at early.

### 3.1 The latency numbers everyone should memorize

These are approximate, but the *relative* magnitudes matter far more than
exact values, because they tell you what's "free" and what's "expensive."

| Operation | Approx. Latency |
|---|---|
| L1 cache reference | ~1 ns |
| Branch mispredict | ~5 ns |
| L2 cache reference | ~4 ns |
| Mutex lock/unlock | ~20 ns |
| Main memory (RAM) reference | ~100 ns |
| Compress 1KB with a fast codec | ~2,000 ns (2 μs) |
| Send 1KB over 1 Gbps network | ~10,000 ns (10 μs) |
| Read 1MB sequentially from RAM | ~10,000 ns (10 μs) |
| Round trip within same data center | ~500,000 ns (0.5 ms) |
| Read 1MB sequentially from SSD | ~1,000,000 ns (1 ms) |
| Disk seek (spinning disk, HDD) | ~10,000,000 ns (10 ms) |
| Read 1MB sequentially from HDD | ~30,000,000 ns (30 ms) |
| Round trip between US East and Europe | ~150,000,000 ns (150 ms) |

**The intuitions to internalize, not the exact digits:**

- Memory is ~100x faster than SSD, and SSD is ~10-100x faster than spinning
  disk for random access.
- A same-datacenter network round trip (~0.5 ms) is *slower* than reading
  1MB from RAM. Network calls are not free just because "it's just an API
  call" — this is the entire justification for caching and for minimizing
  chattiness between services (Module 03).
- A cross-continent round trip (~150 ms) is roughly the same order of
  magnitude as human reaction time. This is why global systems use regional
  deployments and CDNs (Module 02, Module 08) instead of one central server.

> **📐 Numbers:** If your API does 5 sequential cross-service calls at 0.5 ms
> each just for internal network hops, before touching a database, you've
> already spent 2.5 ms of latency budget on nothing but "calling other
> machines." At p99, where each hop's tail is worse, this compounds badly.
> This is why minimizing round trips is a recurring theme (Module 03, gRPC
> vs REST chattiness; Module 25, batching in inference).

### 3.2 Useful approximations for capacity math

Memorize these — you'll multiply by them constantly:

- 1 day ≈ **86,400 seconds** ≈ **~10^5 seconds** (round number for quick math)
- 1 million seconds ≈ 11.5 days
- 2^10 ≈ 1,000 (KB), 2^20 ≈ 1,000,000 (MB), 2^30 ≈ 10^9 (GB), 2^40 ≈ 10^12 (TB)
- A typical single modern server: ~16-64+ cores, ~64GB-1TB+ RAM,
  multi-TB SSD, ~10-25 Gbps NIC (varies a lot by cloud instance type — the
  point is to have *a* reference number, not the exact current best VM).
- A single well-tuned relational DB instance: commonly handles on the order
  of a few thousand to tens of thousands of simple queries/sec, highly
  dependent on query complexity, indexing, and hardware.

### 3.3 The estimation method (memorize this sequence)

When asked "design X," before writing a single box on a diagram, do this:

1. **Clarify scale inputs.** How many users (DAU/MAU)? How many
   requests/actions per user per day? What's the read:write ratio? What's
   the average payload size (bytes per record/message/image)?
2. **Convert to QPS (queries per second).**
   `QPS = (daily actions) / 86,400`. Remember traffic isn't flat — apply a
   peak factor (commonly 2-3x average) for peak QPS.
3. **Estimate storage.** `storage = records/day × size/record × retention
   period`, then consider replication factor (Module 09) multiplying that
   further.
4. **Estimate bandwidth.** `bandwidth = QPS × payload size`, separately for
   read and write paths, ingress and egress.
5. **Sanity check against known hardware limits.** Does this fit on one
   server, or do you clearly need to shard/replicate/cache? This step is
   *why* you did the math — it's what tells you whether you need Module 09
   (replication) and Module 10 (sharding) at all, or whether you're
   over-engineering a system that fits on a laptop.

**Worked example:** Design a system logging one event per user action for
an app with 50 million DAU, where each user performs ~20 actions/day, and
each event is ~500 bytes.

- Daily events = 50M × 20 = 1 billion events/day.
- Average write QPS = 1,000,000,000 / 86,400 ≈ **~11,600 writes/sec**.
- Peak QPS (3x factor) ≈ **~35,000 writes/sec**.
- Daily storage = 1B × 500 bytes = 500 GB/day (before replication/indexing
  overhead).
- At 3x replication (Module 09) and 90-day retention: 500GB × 3 × 90 ≈
  **~135 TB**.
- Conclusion: this instantly tells you a single machine and a single
  un-partitioned database are out of the question — you need horizontal
  partitioning (Module 10) and a storage engine built for high write
  throughput (Module 04, LSM-trees), not a single traditional B-tree RDBMS
  instance out of the box.

> **🎤 Interview Angle:** Interviewers are grading the *method*, not the
> final digit. Stating assumptions explicitly ("I'll assume 3x peak
> traffic, 3x replication") is what signals rigor — nobody expects you to
> know Twitter's exact DAU.

> **🤖 AI/ML Callout:** The same method applies directly to ML systems, just
> with different units. Instead of "requests/sec" you estimate
> "tokens/sec" or "inferences/sec"; instead of "bytes per record" you
> estimate "bytes per embedding vector" (e.g., a 1536-dim float32 embedding
> = 1536 × 4 bytes ≈ 6KB *per vector*, before any index overhead — suddenly
> "embed 1 billion documents" is a very concrete, very large storage number,
> which is exactly the kind of estimate that tells you whether you need a
> distributed vector index, covered in Module 26).

---

## 4. The trade-off axes: the real "language" of system design

Almost every design decision in this curriculum is a specific instance of
one of these general tensions. Learning to name *which* trade-off you're
making, out loud, is what separates senior design thinking from guessing.

1. **Latency vs. Throughput** — batching improves throughput but adds
   latency (you wait to fill the batch). Real-time systems favor low
   latency even at lower throughput.
2. **Consistency vs. Availability** — under a network partition, do you
   refuse to answer (favor consistency) or answer with possibly-stale data
   (favor availability)? Formalized in Module 11.
3. **Consistency vs. Latency** — even without a partition, keeping
   replicas in perfect sync costs round trips. Formalized as PACELC
   (Module 11).
4. **Read optimization vs. Write optimization** — B-trees favor reads,
   LSM-trees favor writes (Module 04). Denormalization favors reads at the
   cost of write complexity (Module 05).
5. **Simplicity vs. Scalability** — a monolith with one Postgres instance is
   easier to build, reason about, and operate than 20 microservices with 5
   datastores. Never distribute what doesn't need to be distributed
   (Module 22).
6. **Cost vs. Performance/Redundancy** — every extra replica, region, or
   cache node is a real dollar cost. "Design for Google scale" is often
   the *wrong* answer if the actual requirement is 10,000 users.
7. **Freshness vs. Efficiency** — caching, materialized views, and batch
   pipelines all trade "how up to date is this data" for "how cheap/fast is
   it to read" (Module 08, Module 16).

> **⚠️ Trap:** Beginners try to pick the objectively "best" architecture.
> There isn't one — there is only the best architecture *for a stated set of
> requirements and constraints*. The most common interview failure is
> jumping to a solution (e.g., "we'll use Kafka and shard by user ID")
> before establishing what problem is actually being solved and at what
> scale. Module 28 turns this into a repeatable framework.

---

## 5. Functional vs. Non-functional requirements

Every design starts by separating two categories, because they're gathered
and validated completely differently:

- **Functional requirements** — *what* the system does. "Users can post a
  message." "Users can search by keyword." These come from product/business
  needs.
- **Non-functional requirements (NFRs)** — *how well* it does it:
  scale (QPS, data volume), latency targets, availability targets,
  consistency requirements, security/compliance constraints, cost
  constraints. These come from *you*, the designer, asking clarifying
  questions — they are rarely handed to you up front, in an interview or in
  real work.

The entire rest of this curriculum (Modules 1-27) is really just "the toolbox
for satisfying non-functional requirements," while Part VIII (case studies)
is where you practice using the whole toolbox against realistic functional
requirements.

---

## Cheat Sheet (for fast revision)

- 3 root causes of distributed-systems difficulty: one machine isn't enough,
  networks are unreliable, failure is the default state.
- 4 resources you're always allocating: CPU, memory, disk, network.
- Latency ladder to memorize: RAM (~100ns) ≪ SSD (~ms) ≪ HDD seek (~10ms) ≪
  same-DC round trip (~0.5ms) ≪ cross-continent round trip (~150ms).
  (Note SSD/HDD sequential reads can beat a network round trip — network
  isn't free.)
- Estimation sequence: clarify scale → QPS → storage → bandwidth → sanity
  check against hardware.
- 7 trade-off axes: latency/throughput, consistency/availability,
  consistency/latency, read/write optimization, simplicity/scalability,
  cost/performance, freshness/efficiency.
- Always separate functional requirements (what) from non-functional
  requirements (how well) before designing anything.

---

## Quiz

Answer these before checking the solutions — write down your reasoning, not
just a final word.

1. A service is "available" 99.99% of the time but every response for the
   last hour has been silently returning yesterday's stock price instead of
   today's. Is this system available? Is it reliable? Is it consistent?
   Justify using the precise definitions above, not intuition.
2. You're told "make it durable." Does that mean the data must always be
   instantly readable? Give an example where data is durable but briefly
   unavailable.
3. Why is `x += 1` a dangerous operation to retry blindly over a flaky
   network, while `SET x = 5` is safe to retry? Name the property involved.
4. A system does 3 sequential internal network calls per user request,
   each averaging 0.5 ms same-datacenter round trip, before it even queries
   its database. Roughly how much latency has this added before any real
   work happens? What general principle does this illustrate, and which
   later module will formalize the fix?
5. Estimate: an app has 2 million DAU, each user sends 10 chat messages/day
   averaging 200 bytes each. Compute (a) average write QPS, (b) peak QPS
   assuming a 3x peak factor, (c) daily raw storage before replication,
   (d) storage after 5x replication over a 1-year retention period. Show
   your work.
6. Explain, in one or two sentences, why "scale up" (vertical scaling) has a
   hard ceiling while "scale out" (horizontal scaling) does not — and name
   the new class of problem that horizontal scaling introduces in exchange.
7. Give one example of the "simplicity vs. scalability" trade-off from your
   own past work (any system, AI-related or not) — what did you pick, and
   was it the right call in hindsight?

<details>
<summary><b>Answers — click to expand</b></summary>

1. **Available**: yes, technically — it answers 99.99% of requests without
   erroring or timing out. **Reliable**: no — it is not doing the *correct*
   thing; it's serving stale/wrong data as if it were current. **Consistent**
   (in the everyday sense of "up to date and matching reality"): no. This is
   the classic trap: availability only measures whether you got *a*
   response, not whether it was *right*. A system can max out every
   availability SLA while being functionally broken.

2. No — durability only guarantees that once a write is acknowledged, it
   will not be lost, even if the process/machine crashes right after.
   It says nothing about read availability. Example: you write a file to a
   replicated storage system and get an ack; the primary server hosting it
   then immediately crashes before serving any read, and a failover takes a
   few seconds. The data was durable (safely persisted, will survive and be
   recoverable) but was briefly unavailable during failover.

3. The property is **idempotency**. `SET x = 5` produces the same end state
   no matter how many times it's applied — safe to retry blindly if you're
   not sure whether the first attempt succeeded (e.g., you got a timeout
   but don't know if the server actually processed it). `x += 1` is not
   idempotent — if the original request actually succeeded server-side but
   the *response* was lost (a very common failure mode), retrying it will
   double-increment, corrupting the value. This is why idempotency keys and
   idempotent operation design matter so much in distributed systems
   (expanded in Module 13).

4. Roughly 1.5 ms (3 × 0.5 ms) added purely for internal network hops,
   before the database is even touched — and that's just the average; p99
   would be considerably worse since tail latencies stack when calls are
   sequential (the total tail is bounded below by the slowest single hop's
   tail, and typically worse due to compounding). The general principle:
   **network round trips are not free, and chatty sequential
   inter-service calls burn latency budget fast.** This is formalized in
   Module 03 (API design — batching/reducing round trips, gRPC vs REST
   chattiness) and revisited in Module 25 (batching in ML inference for the
   same underlying reason).

5. 
   - Daily messages = 2,000,000 users × 10 = 20,000,000 messages/day.
   - (a) Average write QPS = 20,000,000 / 86,400 ≈ **~231 writes/sec**.
   - (b) Peak QPS = 231 × 3 ≈ **~695 writes/sec** (round to ~700/sec).
   - (c) Daily raw storage = 20,000,000 × 200 bytes = 4,000,000,000 bytes =
     **~4 GB/day**.
   - (d) Over 1 year: 4 GB × 365 ≈ 1,460 GB (~1.46 TB) raw. With 5x
     replication: 1.46 TB × 5 ≈ **~7.3 TB**.
   - Takeaway: ~700 writes/sec and ~7TB/year is well within a single modern
     database's capability with proper indexing — this does *not* yet
     obviously require sharding. This is the point of the exercise: the
     math itself tells you whether you're over-engineering.

6. Vertical scaling has a hard ceiling because a single machine's CPU,
   RAM, and I/O are physically bounded — at some point no bigger machine
   exists (or it's prohibitively expensive), and it remains a single point
   of failure regardless of size. Horizontal scaling avoids that ceiling by
   adding more machines, but in exchange introduces **the distributed
   systems coordination problem** — now you must handle data
   partitioning, network unreliability between nodes, replication
   consistency, and partial failure, which a single machine never had to
   worry about.

7. Open-ended — there's no single correct answer. A strong answer names a
   concrete system, states the actual constraint that made "keep it simple"
   correct (or wrong) — e.g., low initial traffic not justifying
   microservices, or, conversely, a monolith that became the bottleneck
   once a specific NFR (scale, team size, deployment independence) was
   exceeded — and reflects honestly on whether the call held up over time.

</details>

---

## What's next

**Module 01 — Inside One Machine** will open up what's actually happening
on a single server (CPU scheduling, memory hierarchy, disk I/O,
concurrency models) — the foundation everything about "many machines" is
built on top of.
