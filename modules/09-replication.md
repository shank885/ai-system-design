# Module 09 — Replication: Leader/Follower, Multi-Leader, Leaderless, Quorums

> **TL;DR:** Replication means keeping copies of the same data on multiple
> machines. It's the foundation of every distributed system's durability
> and availability story — but the moment you have multiple copies, you
> have to answer a hard question: what happens when they disagree, even
> briefly? This module covers the three replication topologies
> (single-leader, multi-leader, leaderless), the specific consistency
> anomalies replication introduces, and quorums — the mathematical tool
> for reasoning about how many replicas must participate in a read or
> write to guarantee correctness. This module is the direct on-ramp to
> Module 11's CAP theorem: everything here is CAP in disguise, before it
> gets the formal name.

---

## 1. Why replicate at all?

Three distinct motivations, worth naming separately because they lead to
different design choices:

- **Durability** — if data lives on only one machine, a single disk
  failure loses it permanently. Multiple copies on independent machines
  make data loss require multiple independent, simultaneous failures —
  directly related to Module 04's erasure coding discussion (replication
  and erasure coding are two different mechanisms toward the same
  durability goal, with the storage-vs-latency trade-off from that
  module).
- **Availability** — if the one machine holding your data goes down
  (crash, network partition, maintenance), the system as a whole is down
  unless another machine has a copy and can take over.
- **Read scalability / locality** — spreading read traffic across
  multiple replicas increases total read throughput beyond what one
  machine can serve, and placing replicas geographically closer to users
  reduces read latency (the same cross-continent-latency motivation
  behind CDNs in Module 08).

> **⚠️ Trap:** Replication does *not* automatically help write throughput
> — in most topologies (Section 2), every write still has to eventually
> reach every replica, meaning total write volume the system can absorb
> is often bounded by a single node's capacity (the leader) or by
> coordination overhead across replicas, not multiplied by replica count.
> Confusing "replication helps reads scale" with "replication helps
> writes scale" is a common interview misstep — write scaling is what
> Module 10 (partitioning/sharding) actually solves.

---

## 2. Single-Leader (Leader/Follower) Replication

The most common topology: one replica (the **leader**, or **primary**)
accepts all writes; the other replicas (**followers**, or **replicas**)
receive a continuous stream of changes from the leader and apply them,
serving reads.

### 2.1 How changes propagate: it's the WAL again

Recall Module 04's Write-Ahead Log: every change to the leader's data is
first recorded as a sequential log entry. Leader/follower replication
typically works by **streaming that same log** to followers, who replay
it to reconstruct the identical sequence of changes — this is why it's
often literally called "WAL shipping" or "log replication" in database
documentation. The same mechanism built for crash recovery on one machine
(Module 04) is reused, essentially unchanged, as the replication
mechanism across machines — a good example of how foundational mechanisms
compound across this curriculum.

### 2.2 Synchronous vs. asynchronous replication

- **Synchronous:** the leader waits for at least one follower to confirm
  it has received (and possibly applied) the change before acknowledging
  the write to the client. Guarantees the write is durable on more than
  one machine before the client is told it succeeded — but adds latency
  (waiting on a network round trip to the follower) and reduces
  availability (if that follower is unreachable, writes stall).
- **Asynchronous:** the leader acknowledges the write immediately, without
  waiting for any follower — followers catch up "eventually." Faster and
  more available, but reintroduces exactly the acknowledged-vs-durable gap
  flagged repeatedly in this curriculum (Modules 01, 04, 08): if the
  leader crashes before a follower catches up, an acknowledged write can
  be lost.
- **Semi-synchronous** (a common middle ground): wait for confirmation
  from at least *one* follower (not all), balancing some durability
  improvement against not paying the full latency/availability cost of
  waiting for every replica.

### 2.3 Replication lag and its anomalies

Asynchronous replication means followers can be **behind** the leader by
some amount of time (**replication lag**) — this isn't a bug, it's the
accepted cost of the async trade-off, but it produces specific, nameable
anomalies applications must handle:

- **Read-your-writes inconsistency:** a user writes data (to the leader),
  then immediately reads it back — if that read is served by a lagging
  follower, the user might not see their own just-made change. Common
  mitigation: route a user's own reads to the leader for some window
  after they write, or read from a replica only known to have caught up
  past that write.
- **Monotonic reads violation:** a user reads a value, then reads again
  later and sees an *older* value than before — happens if the two reads
  are served by different followers with different lag amounts. Mitigated
  by routing a given user's reads consistently to the same replica.
- **Consistent prefix violation:** if writes happen in a causal order
  (e.g., a question is written, then its answer), a lagging replica might
  show the answer before the question is visible, if it received the
  writes out of the "logical" order relative to what a causally-related
  observer expects — mitigated by ensuring causally related writes are
  applied in the same order on every replica.

> **🎤 Interview Angle:** "We use async replication with read replicas —
> what could go wrong?" is a very common follow-up in system design
> interviews. Naming these three specific anomalies (read-your-writes,
> monotonic reads, consistent prefix) — and their mitigations — precisely
> is what separates "I know replication has trade-offs" from actually
> demonstrating you understand the mechanism.

### 2.4 Failover

If the leader fails, the system must promote a follower to become the
new leader (**failover**) — either automatically (a monitoring system
detects the failure and triggers promotion) or manually. Failover is
trickier than it sounds:

- **Data loss risk:** with async replication, the newly promoted leader
  might be missing the most recent writes the old leader had acknowledged
  but never replicated.
- **Split-brain:** if the old leader isn't actually dead (e.g., it's just
  network-partitioned from the monitoring system, not truly down), you
  can end up with *two* leaders simultaneously accepting writes — a
  serious correctness hazard requiring careful handling (fencing tokens,
  quorum-based leader election — Module 12 covers this formally via
  consensus algorithms).

---

## 3. Multi-Leader Replication

Multiple nodes can each accept writes (each is a "leader" for its own
writes) and asynchronously replicate to each other. Most common in
**multi-datacenter** deployments — each data center has its own local
leader (so local writes don't pay cross-region latency), with changes
propagating between data centers afterward.

### 3.1 The core new problem: write conflicts

Because more than one node can accept writes to potentially the *same*
piece of data independently and concurrently, **conflicting writes** can
happen — two data centers each independently accept a different update to
the same record before either has heard about the other's change. Someone
(or something) has to decide how to resolve this.

- **Last Write Wins (LWW):** attach a timestamp to each write, keep
  whichever has the latest timestamp, discard the other. Simple, but
  **silently loses data** (the discarded write simply vanishes) and is
  vulnerable to clock skew across machines (Module 12 covers why trusting
  wall-clock timestamps across distributed machines is genuinely
  dangerous).
- **Application-level / custom merge logic:** the application defines how
  to combine two conflicting writes (e.g., a shopping cart conflict is
  resolved by taking the *union* of items from both versions, rather than
  picking one arbitrarily) — more correct, but requires the application
  to know how to meaningfully merge its own data.
- **CRDTs (Conflict-free Replicated Data Types):** data structures
  specifically designed so that concurrent updates can always be merged
  automatically and deterministically without conflict or data loss
  (e.g., a counter that only increments, or a set with well-defined
  merge semantics). Covered in depth in Module 37 (Collaborative Editing),
  since this is the mechanism behind many real-time collaborative
  document editors' offline/concurrent-edit handling.

> **⚠️ Trap:** Multi-leader replication is a genuinely harder problem than
> single-leader — it's often reached for prematurely because "multiple
> regions accepting local writes" sounds appealing, without fully
> reckoning with conflict resolution complexity. The honest default
> guidance: prefer single-leader with regional *read* replicas (writes
> still go to one leader, potentially cross-region for non-local writers,
> but reads are fast everywhere) unless you have a specific, validated
> need for multi-region *write* availability that justifies conflict
> resolution complexity.

---

## 4. Leaderless Replication

No designated leader at all — a client's write goes directly to multiple
replicas (often via a coordinator), and a read similarly queries multiple
replicas and reconciles the results. Popularized by Amazon's Dynamo paper
and used by Cassandra, Riak, and DynamoDB.

### 4.1 Quorums: the mathematical core of leaderless replication

Given `N` total replicas for a piece of data, define:

- `W` = the number of replicas that must acknowledge a **write** before
  it's considered successful.
- `R` = the number of replicas a **read** must query (and reconcile
  answers from) before returning a result.

**The quorum guarantee:** if `W + R > N`, every read is guaranteed to
overlap with at least one replica that has the most recent write — because
you can't pick `W` write-replicas and `R` read-replicas out of `N` total
without at least one replica appearing in both sets (a pigeonhole
argument). This means a correctly-implemented quorum read will always be
able to see the latest acknowledged write, *among the replicas it
contacts*, even without a leader coordinating anything.

- Common configuration: `N=3, W=2, R=2` (satisfies `W+R=4 > N=3`) —
  tolerates one replica being down for either a read or a write to still
  succeed, while still guaranteeing overlap.
- If `W + R ≤ N`, you lose this guarantee — reads can return stale data
  even when there's no failure at all, because it's mathematically
  possible for the read set and write set to not overlap.

> **📐 Numbers:** With `N=5`, choosing `W=1, R=1` maximizes availability
> and minimizes latency (only one replica needs to respond for either
> operation) but provides **no** consistency guarantee (`1+1=2 ≤ 5`) —
> reads can trivially miss the latest write. Choosing `W=5, R=1` (or
> `W=1, R=5`) guarantees strong read consistency but sacrifices
> availability/latency on whichever side requires all N replicas —
> if any single replica is unreachable, that operation fails entirely.
> `W=3, R=3` with `N=5` (`3+3=6 > 5`) is a typical balanced choice,
> tolerating up to 2 replica failures for either operation while
> maintaining the quorum overlap guarantee — this `W`/`R`/`N` tuning is a
> direct, tunable knob exposed by databases like Cassandra, letting
> different applications (or even different queries) choose their own
> point on the consistency/availability/latency trade-off triangle.

### 4.2 What quorums don't guarantee

Quorum overlap (`W + R > N`) guarantees a read *can* see the latest write
among the replicas it contacts — it does **not** by itself guarantee full
linearizability (Module 11's strictest consistency model) in the presence
of concurrent writes, clock skew, or certain failure/retry edge cases;
real systems layer additional mechanisms (version vectors, read repair,
below) on top of the basic quorum math to handle these subtleties
correctly.

### 4.3 Read repair and hinted handoff

- **Read repair:** when a quorum read discovers that different replicas
  returned different (stale vs. current) versions of the data, the
  coordinator can proactively write the most current version back to the
  stale replicas — opportunistically healing inconsistency as a
  side-effect of normal read traffic, rather than requiring a separate
  repair process.
- **Hinted handoff:** if a replica that should receive a write is
  temporarily unreachable, another node can temporarily store a "hint"
  (the write, plus a note about which replica it was really meant for)
  and deliver it once the target replica comes back — trading a temporary
  availability/durability workaround for eventually restoring full
  replication, without blocking the write in the meantime.

> **🤖 AI/ML Callout:** The quorum trade-off triangle (consistency vs.
> availability vs. latency, tunable per-operation via W/R/N) reappears
> directly in distributed vector database deployments (Module 26) —
> a RAG system ingesting a continuous stream of new document embeddings
> might tolerate a brief window of read inconsistency (a newly-indexed
> document not immediately appearing in similarity search results across
> every replica) in exchange for high write availability during bulk
> ingestion, then tighten consistency requirements for the read path once
> ingestion volume is lower — the exact same knob, applied to a
> vector-search-shaped read/write workload instead of Dynamo's original
> key-value one.

---

## 5. Comparing the three topologies

| | Single-Leader | Multi-Leader | Leaderless |
|---|---|---|---|
| **Write path** | All writes to one leader | Writes to any of several leaders | Writes to multiple replicas directly |
| **Conflict handling** | None needed (one writer) | Required (LWW, app merge, CRDTs) | Required (versioning, read repair) |
| **Write availability during a partition** | Writers cut off from the leader can't write | Each partition's local leader can still accept writes | Depends on quorum reachability |
| **Complexity** | Lowest | Highest (conflict resolution) | High (quorum tuning, repair mechanisms) |
| **Typical use** | Most relational databases, most systems by default | Multi-datacenter active-active systems | Cassandra, DynamoDB, Riak — high write availability at scale |

> **🎤 Interview Angle:** "Design a system available across multiple
> regions" is one of the most common prompts that pulls directly from
> this module. A strong answer explicitly walks through this comparison
> table's trade-offs rather than jumping straight to "we'll use
> multi-leader" or "we'll use Cassandra" — naming *why* a given topology
> fits the stated availability/consistency requirements (Module 11 will
> give you the final formal vocabulary — CAP/PACELC — to make this
> argument fully rigorous).

---

## Cheat Sheet (for fast revision)

- Three reasons to replicate: durability (survive disk/machine loss),
  availability (survive a node going down), read scalability/locality
  (spread read load, reduce latency via geographic placement).
  Replication does NOT inherently scale writes — that's Module 10.
- Single-leader: all writes to one leader, followers replay a streamed
  log (literally the WAL from Module 04, shipped across machines).
  Sync replication = safer, slower, less available; async = faster, more
  available, risks lost acknowledged writes on leader failure.
- Replication lag anomalies (async replication): read-your-writes
  (can't see your own recent write), monotonic reads (a later read shows
  older data than an earlier one), consistent prefix (causally-ordered
  writes appear out of order). Each has a specific named mitigation.
- Failover risks: data loss (unreplicated writes lost with old leader),
  split-brain (two leaders accepting writes simultaneously) — properly
  solved via consensus, Module 12.
- Multi-leader: multiple write-accepting nodes, needed for local-write
  multi-region setups, but introduces write conflicts requiring
  resolution (LWW — simple but loses data silently; app-level merge;
  CRDTs — automatic, deterministic merging). Prefer single-leader +
  regional read replicas unless multi-region write availability is a
  validated hard requirement.
- Leaderless: no leader, writes/reads go to multiple replicas directly.
  Quorum rule: `W + R > N` guarantees read/write set overlap (a read is
  guaranteed to see the latest write among contacted replicas). W/R/N are
  tunable per-operation to trade off consistency/availability/latency.
  Read repair and hinted handoff are the mechanisms that heal
  inconsistency and handle temporarily-unreachable replicas.

---

## Quiz

1. Explain precisely why replication improves read scalability but not
   write scalability in most topologies, and name the module that
   actually addresses write scaling.
2. A leader acknowledges a write to the client via asynchronous
   replication, then crashes 100ms later before any follower received
   the change. What happened to that write, and which prior module's
   concept does this directly instantiate?
3. A user updates their profile picture, and refreshing the page
   immediately afterward shows the old picture. Which specific
   replication-lag anomaly is this, and name one concrete mitigation.
4. Explain why "Last Write Wins" conflict resolution in multi-leader
   replication is described as "simple but silently loses data" — walk
   through a concrete scenario where it discards a write a user
   reasonably expected to have been saved.
5. With `N=5` replicas, explain precisely why `W=2, R=2` fails to
   guarantee a read will see the latest write, while `W=3, R=3` succeeds
   — use the actual quorum math/reasoning, not just the formula.
6. What does read repair actually do, and why is it described as
   "opportunistic" rather than a separate scheduled process?
7. A system needs multi-region write availability (users in Europe and
   Asia both need to write locally, with acceptable eventual
   synchronization). Walk through why single-leader replication doesn't
   satisfy this requirement, and name the specific new problem the
   alternative topology introduces that single-leader never had to solve.

<details>
<summary><b>Answers — click to expand</b></summary>

1. Replication improves read scalability because *read* traffic can be
   spread across multiple replicas — any follower/replica holding a copy
   of the data can independently serve a read request, so total read
   capacity scales roughly with the number of replicas added. Write
   scalability isn't inherently improved because, in most topologies
   (single-leader especially), every write still ultimately needs to
   reach and be applied by *every* replica eventually — the total volume
   of writes the system can absorb is bounded by how fast a single leader
   can accept and propagate writes (or, in leaderless/multi-leader
   setups, by coordination overhead), not multiplied by adding more
   replicas that all still need to receive every single write. **Module
   10 (Partitioning & Sharding)** is what actually addresses write
   scaling, by splitting data (and thus write load) across multiple
   independent shards, rather than replicating the same full dataset
   everywhere.

2. The write is **lost** — the client was told it succeeded, but since
   asynchronous replication means the leader acknowledges before any
   follower confirms receipt, and the leader crashed before any follower
   received the change, there is no surviving copy of that write anywhere
   in the system. This is a direct instantiation of the
   **acknowledged-vs-durable gap** discussed repeatedly across this
   curriculum (Module 01's fsync discussion, Module 04's WAL/durability
   mechanism, Module 08's write-behind cache risk) — an operation
   returning "success" before the data is safely persisted somewhere that
   survives the specific failure that actually occurs is not truly
   durable at the moment of acknowledgment, regardless of the layer
   (disk, cache, or now, database replica) at which the gap occurs.

3. This is a **read-your-writes inconsistency**: the user's write went to
   the leader, but their subsequent read was served by a follower that
   hadn't yet caught up (replication lag) — so the user doesn't see their
   own just-made change. A concrete mitigation: after a user writes data,
   route that specific user's reads to the leader (or to a replica
   confirmed to have caught up past their specific write) for some
   window of time afterward, rather than routing all reads to
   potentially-lagging replicas indiscriminately.

4. LWW keeps whichever conflicting write has the later timestamp and
   **discards the other entirely**, with no merging or recovery of the
   discarded write's content. Concrete scenario: a user in Europe adds an
   item to their shopping cart via the European data center's local
   leader at time T; almost simultaneously, the same user (perhaps via a
   VPN, or a mobile app that failed over to a different region) adds a
   *different* item to their cart via the Asian data center's local
   leader at time T+1ms, before either data center has heard about the
   other's write. Under LWW, when these two writes are eventually
   compared, the Asian write (later timestamp) wins and is kept as "the
   cart," and the European write — the user's first item addition — is
   silently discarded as if it never happened, even though from the
   user's perspective they successfully added two items to their cart and
   would reasonably expect both to be there. Nothing in this process
   surfaces an error or conflict to the user; the data is just quietly
   gone.

5. With `N=5`: `W=2, R=2` gives `W+R=4`, which is **not greater than**
   `N=5` (`4 ≤ 5`) — this means it's mathematically possible to choose 2
   replicas for a write and 2 different replicas for a read such that
   these two sets of replicas **don't overlap at all** (e.g., write
   replicas {1,2}, read replicas {3,4} — replica 5 exists but neither set
   needed to include it, and {1,2} and {3,4} share no common replica) —
   so a read could query only replicas that never received the latest
   write, returning stale data even with zero failures involved. With
   `W=3, R=3`: `W+R=6 > N=5` — by the pigeonhole principle, any 3
   replicas chosen for a write and any 3 replicas chosen for a read, out
   of only 5 total, **must** share at least one common replica (you
   cannot pick two disjoint sets of size 3 each from a pool of only 5
   items, since 3+3=6 exceeds the total pool of 5) — guaranteeing the
   read set always includes at least one replica that received the
   latest write, so the read is guaranteed to be able to see it.

6. Read repair happens **as a byproduct of a normal quorum read**: when a
   coordinator queries multiple replicas for a read and notices that some
   of them returned an older/stale version of the data compared to
   others, it proactively writes the most current version back to
   whichever replicas returned the stale one, right then, during that
   same read operation. It's described as "opportunistic" rather than a
   separate scheduled process because it doesn't run on its own
   independent schedule scanning for inconsistencies — it only triggers
   and only touches the specific replicas and specific data that happened
   to be involved in a read that a client was already performing for its
   own purposes; the repair "rides along" with organic read traffic
   instead of requiring dedicated background infrastructure to sweep the
   whole dataset looking for staleness (though many systems also run
   separate, scheduled anti-entropy repair processes in addition to this,
   for data that might rarely or never be read organically).

7. Single-leader replication requires **all writes to go to the single
   leader**, regardless of where the leader is physically located — so a
   user in whichever region does *not* host the leader must send every
   write across a (likely slow, cross-continent, per Module 00's ~150ms
   round-trip figure) network link to reach it, meaning that region
   cannot actually achieve fast local writes no matter how many
   read-replicas are placed near it. This directly fails the stated
   requirement that both Europe and Asia need to write *locally*. The
   alternative — **multi-leader replication**, giving each region its own
   local leader that can accept writes without waiting on the other
   region — satisfies the local-write requirement, but introduces the
   **write conflict problem** single-leader never had to solve: because
   more than one node can now accept writes to potentially the same data
   independently and concurrently, the system needs an explicit conflict
   resolution strategy (LWW, application-level merge, or CRDTs, Section
   3.1) to decide what happens when Europe and Asia each accept a
   different, conflicting update to the same record before either has
   heard about the other's change — a problem that simply cannot occur
   under single-leader replication, since there's only ever one node
   accepting writes in the first place.

</details>

---

## What's next

**Module 10 — Partitioning & Sharding: Consistent Hashing, Rebalancing,
Hotspots** picks up exactly where this module's Question 1 pointed:
replication solves durability/availability/read-scaling, but *write*
scaling and handling datasets too large for one machine requires
splitting data across independent shards — the actual mechanism, and its
own set of hard problems (rebalancing, hot keys), covered next.

---
