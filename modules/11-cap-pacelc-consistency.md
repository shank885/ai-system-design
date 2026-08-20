# Module 11 — CAP, PACELC & Consistency Models

> **TL;DR:** Modules 09 and 10 showed you the trade-offs concretely:
> replication lag anomalies, quorum tuning, multi-leader conflicts. This
> module gives them a single, formal name — the **CAP theorem** — and then
> immediately corrects the most common misunderstanding of it (it's not
> "pick 2 of 3"). It extends CAP with **PACELC**, which captures a
> trade-off CAP alone misses: the one that exists even when nothing is
> broken. Finally, it lays out the actual spectrum of consistency models
> so "consistency" stops being one vague word and becomes a specific,
> comparable set of guarantees you can choose between deliberately.

---

## 1. The CAP theorem, precisely stated

For a distributed data system, CAP names three properties:

- **Consistency (C)** — every read receives the most recent write (or an
  error). This is the strictest sense: specifically **linearizability**
  (Section 4) — every operation appears to happen instantaneously at some
  point between its start and end, and all nodes agree on a single, global
  order of operations. **This is a different, stricter "consistency" than
  ACID's C (Module 06)**, which is about respecting defined constraints,
  not about read/write recency across replicas — yet another instance of
  this overloaded word meaning something different depending on context.
- **Availability (A)** — every request to a non-failing node receives a
  (non-error) response, without guaranteeing it's the most recent data.
- **Partition tolerance (P)** — the system continues operating despite
  network partitions (messages between nodes being dropped or arbitrarily
  delayed).

**The theorem:** when a network partition occurs, a system must choose
between consistency and availability — it cannot provide both.

### 1.1 The critical correction: it's not "pick 2 of 3"

The common phrasing "CAP means you can only have 2 of 3" is misleading and
causes real confusion. Here's the precise version:

- **Partition tolerance is not optional** for any real distributed system
  — networks *will* partition (packets get dropped, links fail, this is
  physical reality, not a design choice you opt into or out of, echoing
  Module 02's "the network is unreliable" point). So "CA without P" isn't
  a real, available-to-choose option for a genuinely distributed system —
  it only makes sense to describe a single-node system, which trivially
  can't be partitioned from itself.
- **The actual choice CAP describes only applies *during* a partition.**
  When the network is healthy (no partition), a well-designed system can
  provide both strong consistency and availability simultaneously — CAP
  says nothing about the normal, non-partitioned case at all. The real
  question CAP forces you to answer is: **when a partition happens
  (not if — when), does this system prioritize returning a
  possibly-stale-but-immediate answer (favor A), or does it refuse to
  answer until it can guarantee correctness (favor C)?**

> **⚠️ Trap:** If you say "our system is CP" or "our system is AP" in an
> interview, be ready to immediately follow up with "...meaning, during a
> network partition, it does X" — stating the CAP letter alone without
> explaining the specific partition-time behavior is exactly the shallow
> version of this theorem that gets called out. The theorem is a
> statement about behavior *under partition*, full stop — not a general
> personality trait a database has at all times.

### 1.2 Concrete CP vs. AP behavior

- **CP example:** a leader-based system (Module 09) that, during a
  partition separating the leader from a majority of followers, refuses
  to accept writes (or a client can't reach any node that can confirm
  it has the latest data) rather than risk serving/accepting
  possibly-inconsistent data — it sacrifices availability to preserve
  consistency. Systems built on consensus protocols (Module 12 — e.g.,
  etcd, ZooKeeper) are classic CP choices, since consensus fundamentally
  requires a majority to agree, which is unavailable to the minority side
  of a partition.
- **AP example:** a leaderless, quorum-based system (Module 10's
  discussion of Cassandra/Dynamo-style databases) that continues accepting
  reads and writes on *both* sides of a partition, even though the two
  sides can't currently communicate — prioritizing "the system keeps
  responding" over "every response reflects the absolute latest write,"
  accepting that reconciliation (read repair, conflict resolution) happens
  once the partition heals.

> **🎤 Interview Angle:** "Design a system for X" answers frequently need
> an explicit CAP stance for at least one component. A strong answer picks
> based on the *actual* cost of being wrong: a payment/inventory system
> generally should lean CP (an unavailable checkout is bad, but
> double-spending money or overselling inventory is worse); a social media
> "like count" or "view count" can lean AP (briefly stale counts are
> harmless, but the feature being entirely down is a worse user
> experience than a number being slightly off). Naming *why* a specific
> component leans one way, tied to the real-world cost of inconsistency
> vs. unavailability for that specific data, is what a senior answer
> sounds like — not a blanket "we'll use a CP database" for the entire
> system.

---

## 2. PACELC: the extension CAP is missing

CAP only describes behavior **during a partition (P)**. But even when
there's **no partition at all**, there's still a trade-off: do you wait
for strong consistency (which costs latency, coordinating with other
replicas), or return a fast, possibly-stale answer? **PACELC** names this
explicitly:

> **If Partitioned (P): choose Availability (A) or Consistency (C) — this
> is CAP. Else (E, i.e., normal operation): choose Latency (L) or
> Consistency (C).**

This is the formal name for exactly what Module 09's synchronous-vs-
asynchronous replication trade-off (Section 2.2) and Module 10's quorum
`W`/`R` tuning were already demonstrating: even with a perfectly healthy
network, waiting for enough replicas to confirm a strongly consistent
read/write costs real latency (a network round trip, or several) —
choosing to *not* wait (accepting eventual consistency instead) is
strictly a latency optimization, entirely independent of partition
behavior.

| System | Partition behavior (CAP) | Normal-operation trade-off (PACELC) |
|---|---|---|
| A single-leader RDBMS with synchronous replication | CP (typically) | PC — waits for replica confirmation even when healthy, for durability |
| DynamoDB (default eventually-consistent reads) | AP | PL — returns fast, possibly-stale reads even when the network is fine |
| DynamoDB (strongly-consistent read option) | Still fundamentally AP under partition, but this specific read type behaves more like PC | PC for that specific read — trades latency for consistency, per-operation |

> **📐 Numbers:** This is precisely why Module 10's `W`/`R`/`N` quorum
> tuning is a *PACELC* knob, not strictly a CAP knob — choosing `W=1, R=1`
> versus `W=3, R=3` (with `N=5`) changes latency and consistency during
> completely normal, healthy-network operation, not just partition
> behavior. Recognizing that quorum tuning is fundamentally a PACELC
> decision (E-branch: Latency vs. Consistency), while replica-count and
> failover behavior are the CAP decision (P-branch: Availability vs.
> Consistency), is the kind of precise distinction that demonstrates real
> depth beyond memorizing "CAP theorem" as a single fact.

> **🤖 AI/ML Callout:** PACELC's "even without failure, there's still a
> latency-vs-consistency choice" framing maps directly onto a common RAG
> system design decision (Module 26): after a document is updated or newly
> ingested, should a search query wait for it to be fully indexed into the
> vector store before returning results (favoring consistency — the
> freshest possible retrieval, at the cost of ingestion-to-queryable
> latency), or should the system return results from a slightly-stale
> index immediately while indexing catches up asynchronously (favoring
> latency)? This is a PACELC-shaped decision made constantly in RAG
> pipeline design, with no partition or failure involved at all — purely
> a normal-operation freshness/speed trade-off.

---

## 3. The consistency model spectrum

"Consistency" isn't binary (strong vs. eventual) — it's a spectrum of
progressively weaker, cheaper-to-provide guarantees. Understanding where a
given guarantee sits on this spectrum lets you reason precisely about what
an application can and can't assume.

### 3.1 Linearizability (a.k.a. strict/strong consistency)

The strongest practical guarantee: the system behaves as if there's only
a single copy of the data, and every operation takes effect atomically at
some instant between when it was invoked and when it completed — all
clients see operations in the same, real-time-consistent order. This is
CAP's "C." Expensive to provide (requires coordination — consensus,
Module 12 — across replicas for every operation) but simplest to reason
about, since it matches naive single-machine intuition exactly.

### 3.2 Sequential consistency

Weaker than linearizability: all operations appear in *some* consistent
order that respects each individual client's own program order (if client
A does X then Y, every observer sees X before Y), but that global order
doesn't have to match real-world/wall-clock time across different
clients — two different clients' operations can be reordered relative to
real time, as long as internal consistency per-client is preserved.

### 3.3 Causal consistency

Weaker still: only operations that are **causally related** (one
happened-before the other, e.g., a reply to a comment) must be seen in
the same order by everyone; **concurrent, causally-unrelated** operations
can be seen in different orders by different observers without violating
the guarantee. This directly matches Module 09's "consistent prefix"
anomaly — causal consistency is precisely the guarantee that *prevents*
that anomaly, while still being considerably cheaper to provide than full
linearizability, since unrelated operations don't need global
coordination at all.

### 3.4 Eventual consistency

The weakest useful guarantee: if no new writes occur, all replicas will
*eventually* converge to the same value — but offers no guarantee about
*when*, and no ordering guarantee at all in the meantime. This is what
Module 09's async replication and Module 10's leaderless/AP systems
provide by default. Cheapest to provide (no coordination required at
write time at all), hardest to reason about at the application layer,
since literally any intermediate state might be observable during the
convergence window.

> **⚠️ Trap:** "Eventual consistency" is often treated as a single,
> uniform guarantee, but production systems frequently provide something
> stronger than *pure* eventual consistency without going all the way to
> linearizability — e.g., **read-your-writes** and **monotonic reads**
> (Module 09, Section 2.3) are themselves named, specific consistency
> guarantees stronger than plain eventual consistency but weaker than
> sequential/linearizable. Precisely naming which exact guarantee a system
> provides — not just "eventual" or "strong" as a binary — is what this
> whole spectrum is for.

### 3.5 The spectrum, ordered

```
Linearizable (strongest, most expensive)
   ↓
Sequential
   ↓
Causal  ← prevents "consistent prefix" violations
   ↓
Read-your-writes / Monotonic reads  ← session-level guarantees
   ↓
Eventual (weakest, cheapest)
```

Moving down this spectrum trades correctness guarantees for lower latency
and higher availability — this is PACELC's Latency-vs-Consistency axis,
made concrete as an actual menu of named options rather than an abstract
dial.

> **🎤 Interview Angle:** When a design calls for "eventual consistency,"
> a strong follow-up is specifying *which* weaker-than-linearizable
> guarantee is actually needed for correctness — often the real
> requirement is narrower than pure eventual consistency (e.g., "this
> specific user needs read-your-writes for their own actions, but
> cross-user visibility can be purely eventually consistent"), and
> providing exactly that (no more, no less) is usually both cheaper and
> more correct than reaching for either extreme by default.

---

## 4. Bringing it together: CAP/PACELC/consistency models are one coherent story

By this point in the curriculum, this module isn't introducing new
mechanisms — it's naming things you've already seen work:

- Module 09's **synchronous replication** = choosing consistency over
  availability/latency (CP-leaning, PC-leaning).
- Module 09's **asynchronous replication** = choosing availability/latency
  over consistency (AP-leaning, PL-leaning), and its replication-lag
  anomalies are literally instances of weaker consistency models
  (read-your-writes violations, monotonic-read violations, consistent-
  prefix violations) named formally in Section 3.
- Module 10's **quorum `W`/`R` tuning** = a PACELC E-branch (Latency vs.
  Consistency) knob, tunable per-operation, independent of partition
  behavior.
- Module 06's **isolation levels** are the *single-machine, single-database*
  analog of this entire discussion — Serializable isolation is
  linearizability's cousin within one database's transactions; Read
  Committed is a weaker guarantee accepted for performance, the same
  fundamental trade-off shape, just scoped to concurrent transactions on
  one system instead of concurrent operations across replicas.

> **🎤 Interview Angle:** Explicitly drawing this connection — "this
> replication lag anomaly is really just eventual consistency's ordering
> guarantee being violated, and Module 06's isolation levels are the same
> trade-off at the single-database-transaction scale" — is exactly the
> kind of cross-module synthesis that signals you've built a coherent
> mental model rather than memorized a list of disconnected facts, which
> is the entire point of this curriculum's structure.

---

## Cheat Sheet (for fast revision)

- CAP: Consistency (linearizability — strictest sense, different from
  ACID's C), Availability (every non-failing node responds), Partition
  tolerance (system keeps working despite dropped/delayed messages).
- The real theorem: **during a partition**, choose C or A — you cannot
  have both. Partition tolerance isn't optional for real distributed
  systems (networks will partition); CAP says nothing about behavior when
  there's no partition.
- CP systems (e.g., consensus-based like etcd/ZooKeeper) sacrifice
  availability during a partition to guarantee consistency. AP systems
  (e.g., leaderless/quorum-based like Cassandra/Dynamo) sacrifice strict
  consistency to keep responding on both sides of a partition.
- PACELC extends CAP: **if Partitioned, choose A or C (that's CAP) — Else
  (normal operation), choose Latency or Consistency.** This captures the
  trade-off that exists even with a perfectly healthy network — e.g.,
  Module 10's quorum W/R tuning is a PACELC (E-branch) knob, not a CAP
  knob.
- Consistency model spectrum, strongest to weakest: Linearizable →
  Sequential → Causal (prevents consistent-prefix violations) →
  Read-your-writes/Monotonic reads (session guarantees) → Eventual
  (weakest, cheapest, no ordering guarantee at all).
- This module doesn't introduce new mechanisms — it formally names what
  Modules 06 (isolation levels), 09 (replication anomalies), and 10
  (quorum tuning) were already demonstrating concretely.

---

## Quiz

1. A candidate says "our system is CP, so it's always consistent." Explain
   precisely what's wrong with this statement, in terms of what CAP
   actually describes.
2. Explain why "partition tolerance is optional — you can just choose CA"
   is not a meaningful option for a real, multi-node distributed system.
3. A DynamoDB table configured for eventually-consistent reads returns a
   fast, possibly-stale answer even when the network between all
   replicas is completely healthy, with zero partition occurring. Explain
   why this scenario is a PACELC decision, not a CAP decision, and name
   the specific axis PACELC adds that CAP doesn't cover.
4. Explain, precisely, the difference between causal consistency and
   sequential consistency, using a concrete example (a comment and a
   reply to it) to illustrate what causal consistency guarantees that a
   purely eventual-consistency system wouldn't.
5. Connect Module 09's "consistent prefix violation" anomaly to a specific
   named consistency model from this module's spectrum — which model,
   if correctly provided, would prevent that specific anomaly, and why?
6. A payment/checkout system and a "post view count" feature have
   different appropriate CAP leanings. Argue for which each should lean
   toward (CP or AP), grounded in the actual real-world cost of being
   wrong in each direction — not just "payments need consistency" as an
   unexamined assumption.
7. Explain how Module 06's isolation levels (Read Committed through
   Serializable) are structurally the "same trade-off" as this module's
   consistency model spectrum, despite operating at a different scale —
   what's the same about the underlying tension in both cases?

<details>
<summary><b>Answers — click to expand</b></summary>

1. This statement is imprecise in two ways. First, CAP's "consistency"
   only describes behavior **during a network partition** — "CP" means
   that when a partition occurs, the system chooses to preserve
   consistency at the cost of availability (e.g., refusing to serve
   requests on the minority side of the partition), not that the system
   is "always" providing some blanket consistency guarantee at all
   times independent of partition state. Second, even outside of
   partitions, CAP says nothing about the latency-vs-consistency
   trade-off during completely normal operation — that's PACELC's domain
   (Section 2), and a system correctly labeled "CP" could still make
   various latency/consistency trade-offs during healthy, non-partitioned
   operation that CAP alone doesn't describe or constrain at all. "CP, so
   always consistent" conflates a partition-specific behavioral choice
   with a permanent, unconditional property, which isn't what the theorem
   actually states.

2. Network partitions — dropped packets, failed links, arbitrary delays —
   are a physical reality of any network connecting multiple machines,
   not a failure mode a system can opt out of experiencing; this is the
   same "the network is unreliable" fact from Module 02 that underlies
   why retries, timeouts, and idempotency matter at all. "Choosing CA" would
   mean choosing to be simultaneously consistent and available *even when*
   a partition has occurred and nodes genuinely cannot communicate — but
   this is not something you can architect your way out of by
   "choosing" it, since a partition, once it happens, physically prevents
   the nodes on either side from coordinating at all; the system's
   observable behavior during that unavoidable event is necessarily
   either "keep responding, possibly inconsistently" (A) or "refuse to
   respond until consistency can be guaranteed" (C) — there is no third
   option available once communication is actually cut off, no matter
   what the system's designers intended or "chose" in the abstract. "CA"
   only makes coherent sense as a description of a single-node system,
   which trivially cannot experience an internal network partition since
   there's no internal network to partition.

3. This scenario is a **PACELC decision**, not a CAP decision, because it
   occurs during completely normal operation with **no partition present
   at all** — CAP's entire theorem is specifically scoped to behavior
   *during* a partition, and has nothing to say about a system's behavior
   when the network is healthy. What's actually happening here is the
   axis PACELC adds beyond CAP: the choice between **Latency and
   Consistency during normal operation (the "Else" branch)** — DynamoDB's
   eventually-consistent read option is choosing to skip waiting for full
   quorum/replica confirmation (which would guarantee the most current
   value but cost additional latency) in favor of returning a fast answer
   immediately, purely as a latency optimization that has nothing to do
   with partition tolerance or handling network failure.

4. **Sequential consistency** guarantees all operations appear in *some*
   single, consistent global order that every observer agrees on, and
   that order respects each individual client's own program order — but
   that agreed-upon order doesn't have to match real, wall-clock time
   across *different* clients' operations relative to each other.
   **Causal consistency** is weaker still: it only requires that
   operations which are **causally related** (one happened because of, or
   after seeing, the other) be observed in the same order by everyone;
   operations that are concurrent and causally unrelated can be seen in
   different orders by different observers without violating the
   guarantee at all. Concrete example: if user A posts a comment, and
   user B then writes a reply to that specific comment, causal
   consistency guarantees that **no observer will ever see B's reply
   before A's original comment** — because the reply is causally
   dependent on having seen the comment. A purely eventually-consistent
   system provides no such guarantee at all — it's entirely possible
   (during the convergence window) for some observer, reading from a
   replica that received the reply before the original comment due to
   arbitrary propagation timing, to see the reply appear with no visible
   parent comment yet, which is confusing and incorrect from the
   application's perspective, even though the system will "eventually"
   show both in some order.

5. This is exactly what **causal consistency** (Section 3.3) is designed
   to prevent, and it's directly named in this module as matching Module
   09's consistent-prefix anomaly. A consistent-prefix violation is
   precisely a case where causally related writes (e.g., a question, then
   its answer) are observed out of their causal order on some replica —
   causal consistency's specific guarantee ("causally related operations
   must be seen in the same order by everyone") is defined exactly to
   rule this out, while still being considerably cheaper to provide than
   full linearizability/sequential consistency, since causally
   *unrelated* operations (the vast majority of operations in most
   systems) don't require any cross-replica coordination at all under a
   causal consistency model — only genuinely causally dependent writes
   need their relative order preserved.

6. A **payment/checkout system** should lean **CP**: the real-world cost
   of being wrong in the "available but possibly inconsistent" direction
   is severe and often irreversible — accepting a payment or decrementing
   inventory based on stale data can lead to double-spending, overselling
   inventory that doesn't actually exist, or processing a duplicate
   charge, all of which are costly, hard-to-reverse, trust-damaging
   failures. The cost of being wrong in the "consistent but temporarily
   unavailable" direction — a checkout briefly failing or showing an
   error during a rare partition — is comparatively minor and recoverable
   (the user retries, or the system queues the request); an unavailable
   checkout is a bad but bounded, temporary inconvenience, while a
   double-charged customer or an oversold product is a lasting, costly
   correctness failure. A **"post view count" feature** should lean
   **AP**: the real-world cost of showing a slightly stale or
   under-counted number for a brief window is essentially nothing —
   nobody is harmed or makes an important decision based on whether a
   view count is exactly accurate to the second — whereas making the
   entire feature (or worse, the whole page) unavailable whenever any
   part of the counting infrastructure experiences a partition would be a
   real, user-visible degradation for a feature whose correctness
   tolerance is genuinely very loose. The reasoning in both cases is
   explicitly about the *actual, concrete cost* of being wrong in each
   direction for that specific piece of data — not a generic rule like
   "payments = consistency" applied without examining why.

7. Both are fundamentally the same tension: **guaranteeing that
   concurrent operations behave as if they happened one at a time, in
   some agreed order, costs coordination — and that coordination costs
   latency/availability/throughput.** Module 06's isolation levels
   describe this tension *within a single database*, among concurrent
   *transactions* touching shared rows — Serializable isolation is the
   strongest guarantee (transactions behave as if fully ordered/serial,
   expensive to provide), while Read Committed is a weaker, cheaper
   guarantee that permits specific named anomalies (non-repeatable reads)
   in exchange for better concurrency/throughput. This module's
   consistency model spectrum describes the *same shape of tension*, but
   scoped across *replicas of the same data on different machines* rather
   than concurrent transactions on one machine — linearizability is the
   strongest guarantee (all replicas behave as if there's a single
   consistent copy, expensive, requires coordination/consensus), while
   eventual consistency is a weaker, cheaper guarantee that permits
   specific named anomalies (staleness, out-of-order visibility) in
   exchange for availability/latency. In both cases, the underlying
   mechanism generating the trade-off is identical: **enforcing a single,
   globally-agreed-upon order of operations requires communication
   overhead among the concurrent parties involved (transactions, or
   replicas) — and the amount of that overhead you're willing to pay is
   exactly what determines where on the strength spectrum your actual
   guarantee lands**, whether the "concurrent parties" are transactions
   sharing a database or replicas sharing a dataset across a network.

</details>

---

## What's next

**Module 12 — Consensus, Coordination & Time: Raft, etcd, Distributed
Locks, Failure Detection, Vector Clocks** covers the actual *mechanism*
that lets CP systems achieve strong consistency across multiple machines
despite the network being unreliable — consensus algorithms — along with
the related problems of distributed locking, detecting whether a node has
actually failed (vs. just being slow or partitioned), and reasoning about
event ordering without a shared, trustworthy clock.
