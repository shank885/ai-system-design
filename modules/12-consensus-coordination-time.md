# Module 12 — Consensus, Coordination & Time: Raft, etcd, Distributed Locks, Failure Detection, Vector Clocks

> **TL;DR:** Module 11 said CP systems "sacrifice availability to preserve
> consistency during a partition" — this module is *how*. Consensus
> algorithms are the actual mechanism that lets a group of unreliable
> machines agree on a single value (like "who's the leader") despite
> failures and an unreliable network. This module covers how consensus
> works (Raft, as the accessible modern standard), the coordination
> services built on top of it (etcd/ZooKeeper), why distributed locks are
> deceptively hard, how to tell a genuinely dead node from a merely slow
> one, and why you can't trust wall-clock time across machines at all —
> the last piece needed before Module 13's distributed transactions.

---

## 1. The consensus problem, precisely

**Consensus**: get a group of nodes to agree on a single value, such that:

- **Agreement:** every non-faulty node that decides, decides the same
  value.
- **Validity:** the decided value was actually proposed by some node (no
  making up an arbitrary answer).
- **Termination:** every non-faulty node eventually decides *something*
  (the algorithm doesn't stall forever).

This sounds simple but is genuinely hard because of exactly the two facts
Module 02 and Module 09 have been building toward: the network can drop,
delay, or reorder messages, and nodes themselves can crash — and a node
receiving no response from another node **cannot tell whether that other
node crashed, is just slow, or is network-partitioned** (this ambiguity
is formalized as the **FLP impossibility result**: in an asynchronous
network where you can't bound message delay, no algorithm can guarantee
consensus in bounded time if even one node might fail — real systems work
around this practically using timeouts, accepting a small risk of
incorrectly treating a slow node as failed, rather than solving the
theoretical impossibility outright).

> **⚠️ Trap:** "Just have every node vote and go with the majority" sounds
> like it trivially solves consensus, but the hard part isn't the voting
> arithmetic — it's handling votes that never arrive, nodes that crash
> mid-vote, and messages that arrive out of order or duplicated, all while
> still guaranteeing agreement. This is precisely why consensus algorithms
> (Paxos, Raft) are notoriously difficult to get right from a rough
> description alone, and why using a battle-tested implementation (etcd,
> ZooKeeper) rather than hand-rolling your own is the overwhelming default
> practical advice.

---

## 2. Raft: consensus you can actually reason about

**Raft** was explicitly designed to be more understandable than its
predecessor, Paxos, while providing equivalent guarantees — it's the
consensus algorithm underlying etcd, Consul, and many modern systems.
Raft breaks consensus into three separable sub-problems:

### 2.1 Leader election

At any time, each node is in one of three states: **leader**, **follower**,
or **candidate**. Only one leader exists per **term** (a monotonically
increasing counter — think of it as an "epoch" or election cycle).

- Followers expect periodic **heartbeats** from the leader. If a follower
  doesn't hear from a leader within a randomized timeout, it becomes a
  **candidate**, increments the term, and requests votes from other nodes.
- A node votes for at most one candidate per term (first-come,
  first-served) — this prevents two candidates from both winning the same
  term.
- A candidate becomes leader only if it receives votes from a **majority**
  of all nodes (not just a majority of responders — a true majority of the
  total cluster size).
- **Randomized timeouts** are a deliberately simple trick to avoid
  **split votes** (multiple candidates starting elections simultaneously,
  each getting some votes but no one reaching a majority) — since each
  node's timeout is randomized, one candidate usually starts its election
  slightly before others and wins before a competing election even begins.

### 2.2 Log replication

Once elected, the leader accepts all writes (client requests), appends
them to its own log (Module 04's WAL, again — the same mechanism,
reused at the cluster-coordination layer this time, not just the
single-database-replication layer from Module 09), and replicates each
log entry to followers. **An entry is considered "committed" (safe,
durable, agreed-upon) only once a majority of nodes have it in their
log** — this is a direct instance of Module 10's quorum reasoning: a
majority-write guarantee ensures that any future leader election (which
also requires a majority of votes) is guaranteed to include at least one
node that has every previously committed entry, because two majorities
out of the same total node count must overlap (the exact pigeonhole
argument from Module 09's `W+R>N` quorum math).

### 2.3 Why majority quorums specifically prevent split-brain

This is the precise mechanism behind Module 09's "split-brain" concern:
because becoming leader requires a majority vote, and a network partition
can put at most *one* side of a partition in possession of a true majority
of the total cluster (it's mathematically impossible for both sides of a
partition to simultaneously contain a majority of the same fixed total —
more than 50% can't exist on two different sides at once), **at most one
leader can exist at any given term, even during a partition.** The
minority side simply cannot elect a leader at all (it can't gather enough
votes), so it correctly becomes unavailable for writes rather than
incorrectly electing a second, conflicting leader — this is precisely the
CP choice (Module 11) made concrete: the minority partition sacrifices
availability specifically to prevent the consistency violation split-brain
would cause.

> **📐 Numbers:** A 5-node Raft cluster requires 3 votes (a majority of 5)
> to elect a leader or commit a log entry — it can tolerate up to 2 node
> failures and continue operating normally (3 remaining nodes still form
> a majority), but if 3 or more nodes fail or are partitioned away, the
> remaining nodes cannot reach a majority and the cluster becomes
> unavailable for writes (correctly, per Section 2.3) until enough nodes
> recover. This "tolerate `f` failures with `2f+1` total nodes" formula is
> the standard sizing rule for consensus-backed clusters — it's why these
> clusters are typically deployed with odd numbers of nodes (3, 5, 7):
> even numbers waste a node's worth of resources without improving fault
> tolerance (a 4-node cluster tolerates only 1 failure, the same as a
> 3-node cluster, since you still need 3 for a majority either way).

---

## 3. Coordination services: etcd and ZooKeeper

Raft (or ZooKeeper's own consensus protocol, ZAB) isn't usually
implemented fresh for every application that needs it — it's packaged
into general-purpose **coordination services** (etcd, ZooKeeper, Consul)
that applications use as a dependency for specific coordination needs:

- **Configuration storage:** a small amount of critical, frequently-read,
  rarely-written configuration data that many services need to agree on
  consistently (e.g., Kubernetes stores its entire cluster state in etcd).
- **Leader election:** application services use the coordination service's
  consensus-backed primitives to elect one instance of themselves as a
  leader/coordinator for some task, without implementing Raft themselves.
- **Service discovery:** a registry of "which instances of service X are
  currently alive and where," kept consistent via the underlying consensus
  mechanism (expanded fully in Module 14).
- **Distributed locks:** covered next, in Section 4.

> **⚠️ Trap:** Coordination services like etcd/ZooKeeper are explicitly
> **not** meant to store your application's primary, high-volume data —
> they're optimized for consensus-backed correctness on a *small* amount
> of critical metadata (kilobytes to low megabytes, not your product
> database), and consensus's coordination overhead (Section 2.2's
> majority-write requirement on every change) makes them comparatively
> low-throughput by design. Using etcd/ZooKeeper as a general-purpose
> database is a recognizable anti-pattern.

---

## 4. Distributed locks: harder than they look

A **distributed lock** lets multiple processes across different machines
coordinate exclusive access to a shared resource — conceptually the
cluster-wide version of Module 01's mutex/Module 06's row lock, but now
with the network's unreliability in the mix.

### 4.1 The problem: a lock holder can be wrong about still holding the lock

A process acquires a distributed lock, then experiences a **long pause**
(a GC pause — Module 01 — a slow disk I/O, or simply being descheduled by
its OS for an extended period) that lasts longer than the lock's timeout.
The lock service, having received no renewal/heartbeat, correctly
concludes the process is gone and gives the lock to someone else. But the
original process **eventually resumes**, still believing it holds the
lock, and proceeds to act as if it has exclusive access — while another
process now *also* believes it holds the same lock. Two processes now
believe they have exclusive access simultaneously, which is exactly the
correctness violation the lock was supposed to prevent.

> **⚠️ Trap:** This isn't a hypothetical edge case — it's a well-known,
> genuinely tricky failure mode (sometimes discussed under the term
> "the pause of death"). "Just use a distributed lock" is not, by itself,
> a complete safety guarantee — the lock mechanism itself needs to be
> paired with the fencing mechanism below to be actually safe against
> this specific failure.

### 4.2 Fencing tokens: the actual fix

Rather than trusting a lock holder's own belief that it still holds the
lock, the lock service issues a **monotonically increasing fencing
token** with each lock grant. Every operation the lock holder performs on
the protected resource must include its fencing token, and the resource
itself (or whatever it's protecting) **rejects any operation whose token
is lower than the highest token it has already seen.** If the original
process resumes after its lock was reassigned, its old (now-stale) token
is lower than the token issued to the new holder, so its writes are
correctly rejected — the resource itself enforces exclusivity, rather
than trusting each client's own (potentially wrong) belief about whether
it still holds the lock.

> **🎤 Interview Angle:** If a design uses a distributed lock (e.g., "only
> one worker should process this job at a time"), proactively raising the
> "what if the lock holder pauses and resumes after its lock expired?"
> failure mode, and naming fencing tokens as the fix, is a strong, fairly
> advanced signal — many candidates stop at "we'll use a distributed
> lock" without considering this specific, real failure mode at all.

---

## 5. Failure detection: telling dead from slow

Every mechanism in this module (leader election, lock timeouts) depends
on deciding "is this node still alive?" — and per Section 1's FLP point,
this **cannot be known with certainty** over an asynchronous network; it
can only be estimated, with some risk of being wrong.

- **Heartbeats + fixed timeout:** the simplest approach — if no heartbeat
  is received within a fixed window, declare the node dead. Simple, but a
  poor fit for real networks: too short a timeout causes **false
  positives** (a merely slow/congested node gets incorrectly declared dead,
  potentially triggering unnecessary, disruptive failover); too long a
  timeout means **slow detection** of genuine failures, extending an
  outage.
- **Phi Accrual Failure Detector:** instead of a hard yes/no threshold,
  continuously computes a **suspicion level** (a numeric confidence that
  the node is dead) based on the statistical distribution of past
  heartbeat inter-arrival times — a node with historically very regular
  heartbeats missing just one is treated as more suspicious than a node
  whose heartbeats have always been irregular. This lets the *application*
  choose its own risk threshold, rather than baking a single fixed timeout
  into the detection mechanism itself, and adapts to each node/network's
  actual observed behavior rather than assuming uniform conditions
  cluster-wide.
- **Gossip protocols:** rather than every node individually monitoring
  every other node (which doesn't scale — O(n²) heartbeat traffic), nodes
  periodically exchange failure-detection state with a few random peers,
  and that state propagates epidemically through the cluster over a few
  rounds — reaching eventual, scalable, cluster-wide agreement on which
  nodes are suspected failed without requiring all-to-all communication.
  Used by Cassandra and other large leaderless clusters (Module 09) for
  exactly this scalability reason.

> **🤖 AI/ML Callout:** Failure detection's core tension — false positives
> (wrongly declaring something dead, triggering disruptive action) vs.
> slow detection (a real failure going unnoticed too long) — reappears
> directly in ML model monitoring (Module 19): a model-drift or
> degraded-accuracy detector faces the identical trade-off between
> reacting too aggressively to normal statistical noise (false alarms,
> unnecessary retraining/rollback) and being too conservative and missing
> a genuine, costly degradation for too long. The phi-accrual approach's
> core idea — use the historical distribution of normal behavior to
> calibrate a continuous suspicion score, rather than a single fixed
> threshold — is a design pattern directly transferable to that
> monitoring problem too.

---

## 6. Time: why you can't trust the clock

Every machine has a **wall clock**, but wall clocks on different machines
drift relative to each other (crystal oscillators aren't perfectly
synchronized) and are periodically corrected by protocols like **NTP** —
which itself introduces jumps, not smooth adjustment, and network delay
in reaching an NTP server adds its own uncertainty. **The practical
consequence: you cannot assume two different machines' timestamps are
directly comparable with sub-second (or sometimes even multi-second)
precision, and definitely cannot assume "the write with the later
timestamp genuinely happened later in real, physical time."**

This is precisely why Module 09 flagged **Last Write Wins (LWW)** conflict
resolution as dangerous — LWW's entire correctness depends on trusting
wall-clock timestamps across different machines to determine true
ordering, which this section shows is not a safe assumption in general.

### 6.1 Logical clocks: ordering without trusting wall-clock time

Instead of relying on physical clocks, **logical clocks** derive an
ordering purely from the actual causal relationships between events —
which is exactly what actually matters for correctness (Module 11's causal
consistency), not physical simultaneity.

- **Lamport clocks:** each node keeps a simple counter, incremented on
  every local event, and included in every outgoing message; a receiving
  node sets its own counter to `max(local counter, received counter) + 1`.
  This guarantees that if event A causally happened-before event B (A's
  message was received before B occurred), then A's Lamport timestamp is
  guaranteed to be smaller than B's — but the reverse isn't guaranteed
  (two causally-unrelated/concurrent events can end up with either
  ordering of Lamport timestamps, since Lamport clocks provide a partial,
  not total, correctness guarantee about causality specifically, even
  though the numbers themselves impose *some* total order for
  tie-breaking purposes).
- **Vector clocks:** extend the idea to let the system explicitly detect
  **concurrency** (as opposed to just approximating an order) — each node
  maintains a full vector of counters, one per node in the system,
  incrementing its own entry on each local event and merging (taking the
  element-wise max) whenever it receives a message. Comparing two vector
  clocks tells you definitively whether one event causally preceded the
  other, followed it, **or neither (they were truly concurrent)** — this
  three-way distinction (before / after / concurrent) is something a
  simple Lamport clock or a wall-clock timestamp cannot reliably provide,
  and it's exactly the information needed to correctly detect *and*
  resolve write conflicts in multi-leader/leaderless replication (Module
  09) without falsely assuming a real ordering existed when the writes
  were actually concurrent.

> **📐 Numbers:** This is the actual, principled alternative to LWW that
> Module 09 gestured toward: instead of picking a winner based on
> untrustworthy wall-clock timestamps, a system using vector clocks can
> correctly identify "these two writes were truly concurrent (neither
> caused the other)" and hand *both* versions to the application (or a
> CRDT merge function) to resolve meaningfully — rather than silently and
> arbitrarily discarding one based on a timestamp comparison that might
> not even reflect true causal or real-world ordering at all.

---

## Cheat Sheet (for fast revision)

- Consensus: get nodes to agree on a value (agreement, validity,
  termination) despite failures and network unreliability. FLP
  impossibility: can't guarantee bounded-time consensus with even one
  possible failure in an async network — real systems use timeouts as a
  practical (imperfect) workaround.
- Raft: leader election (randomized timeouts avoid split votes, majority
  vote wins) + log replication (WAL, but cluster-wide; entry "committed"
  once on a majority of nodes). Majority-based election is precisely why
  at most one leader can exist per term, even during a partition — the
  mechanism behind Module 11's CP choice.
- `2f+1` nodes tolerate `f` failures — why consensus clusters use odd
  sizes (3, 5, 7); more nodes than needed for a given fault tolerance
  wastes resources without adding safety.
- etcd/ZooKeeper: coordination services packaging consensus for config
  storage, leader election, service discovery, distributed locks — not
  meant for high-volume primary application data.
- Distributed locks: a lock holder can wrongly believe it still holds the
  lock after a long pause exceeds its timeout ("pause of death") — fixed
  by **fencing tokens**: monotonically increasing tokens the protected
  resource itself validates, rejecting stale-token operations regardless
  of what the client believes.
- Failure detection can never be certain (FLP) — fixed timeouts trade off
  false positives (too short) vs. slow detection (too long); phi-accrual
  uses historical heartbeat distributions for a continuous, tunable
  suspicion score; gossip protocols scale failure detection beyond
  all-to-all heartbeating (O(n²)) via epidemic propagation.
- Wall clocks drift and aren't safely comparable across machines — this
  is exactly why LWW (Module 09) is risky. Lamport clocks give a
  causally-consistent partial ordering cheaply; vector clocks additionally
  detect true concurrency (before/after/concurrent), enabling correct
  conflict detection instead of silently discarding data via timestamp
  guessing.

---

## Quiz

1. Explain precisely why "have every node vote and take the majority" is
   not, by itself, a complete solution to consensus — what specific
   failure scenarios does a full consensus algorithm need to handle that
   simple majority voting doesn't address on its own?
2. A 7-node Raft cluster experiences a network partition splitting it
   into a group of 4 and a group of 3. Explain what happens on each side,
   and connect your answer explicitly to Module 11's CP framing.
3. Why do consensus clusters typically use odd numbers of nodes (3, 5, 7)
   rather than even numbers (4, 6)? Use the `2f+1` formula in your
   explanation.
4. Walk through the exact failure scenario ("pause of death") that makes
   a naive distributed lock (without fencing tokens) unsafe, and explain
   precisely how fencing tokens fix it — specifically, where the safety
   check actually happens.
5. Explain why a fixed heartbeat timeout for failure detection forces a
   trade-off between false positives and slow detection, and how the
   phi-accrual failure detector's approach avoids committing to a single
   fixed threshold.
6. Explain precisely why Last Write Wins (LWW) conflict resolution
   (Module 09) is unsafe to rely on for correctness, using this module's
   explanation of wall-clock behavior across machines.
7. Two events, A and B, occur on different nodes with no message passed
   between them either way. Using vector clocks, would you expect their
   comparison to show "A before B," "B before A," or "concurrent"? Explain
   why this distinction matters for conflict resolution compared to what
   a wall-clock timestamp comparison would tell you.

<details>
<summary><b>Answers — click to expand</b></summary>

1. Simple majority voting handles the case where all nodes are
   reachable and respond promptly, but real consensus algorithms must
   also correctly handle: votes that never arrive at all (a node crashed,
   or the message was lost — the voter can't distinguish these), multiple
   candidates simultaneously starting elections and splitting the vote so
   no one reaches a majority (requiring a mechanism like randomized
   timeouts to make this rare and self-resolving), a candidate winning an
   election but then crashing before informing all nodes of the result,
   and messages arriving out of order or being duplicated across an
   unreliable network. "Majority voting" describes the *decision rule*,
   but a complete consensus algorithm (like Raft) also has to specify the
   full protocol for detecting when an election is needed, preventing
   conflicting simultaneous elections, safely handling partial/lost
   communication, and ensuring that even a leader that wins but then fails
   partway through doesn't leave the cluster in an inconsistent state —
   none of which "take the majority" addresses by itself.

2. The **4-node side** contains a true majority of the total 7-node
   cluster (4 out of 7 is more than half) — it can successfully hold a
   leader election (or retain its existing leader, if the leader happens
   to be on this side) and continue accepting and committing writes
   normally, since it can gather the majority votes/acknowledgments Raft
   requires. The **3-node side** does *not* have a majority (3 out of 7 is
   less than half) — it cannot elect a new leader if it doesn't have one,
   and cannot commit new writes even if it does still have its old leader
   present (a leader that discovers it can no longer reach a majority of
   the cluster should step down or refuse to commit new entries, to avoid
   exactly the split-brain scenario Section 2.3 describes) — so this side
   becomes unavailable for writes. This is Module 11's **CP** behavior
   made concrete: during this partition, the system as a whole chooses to
   sacrifice availability specifically on the minority side, in order to
   guarantee that at most one side is ever accepting writes, preserving
   consistency rather than risking two independently-diverging leaders.

3. The `2f+1` formula says a cluster of `2f+1` total nodes can tolerate
   up to `f` node failures while still being able to gather a majority
   (`f+1` nodes) from the remaining nodes. A 3-node cluster (`f=1`)
   tolerates 1 failure, needing 2 votes for a majority out of the
   remaining 2 nodes. A 4-node cluster still only tolerates 1 failure
   (`f=1` as well, since you need 3 votes for a majority out of 4, and
   losing 2 nodes leaves only 2, which isn't enough) — the 4th node adds
   no additional fault tolerance over 3 nodes; it only adds cost
   (an extra machine to run, and one more node that must participate in
   every majority-write for log replication) without improving how many
   failures the cluster can survive. A 5-node cluster (`f=2`) genuinely
   does tolerate 2 failures with 3 needed for a majority — this is why odd
   sizes are chosen: each additional *odd* increment (3→5, 5→7) buys
   genuinely additional fault tolerance, while each even increment (3→4,
   5→6) buys none, making even-sized clusters strictly wasteful compared
   to the next lower odd size.

4. The "pause of death" scenario: a process acquires a distributed lock
   with some timeout, then experiences an unusually long pause (a GC
   pause, disk I/O stall, being descheduled by the OS, or similar) that
   exceeds the lock's timeout duration — since the process isn't sending
   heartbeats/renewals during this pause, the lock service correctly
   concludes it's gone and grants the lock to a different process. When
   the original, paused process eventually resumes execution, it has no
   way of knowing time has passed or that its lock was reassigned — it
   still believes it holds the lock and proceeds to act on the protected
   resource, now concurrently with the new legitimate lock holder, which
   also believes (correctly, this time) that it holds the lock — two
   processes now act with what they each believe is exclusive access
   simultaneously. **Fencing tokens** fix this not by trying to prevent
   the pause or improve the client's own awareness (which is
   fundamentally unreliable, since you can't guarantee a paused process
   knows it was paused), but by moving the safety check to **the resource
   being protected itself**: every lock grant includes a monotonically
   increasing token, every operation on the protected resource must
   include the caller's token, and the resource rejects any operation
   whose token is lower than the highest token it has already
   accepted — so when the original process resumes and tries to act with
   its now-stale (lower) token, the resource itself correctly rejects the
   operation, regardless of what the client mistakenly believes about
   still holding the lock.

5. A fixed timeout forces a binary choice with costs on both sides: set
   the timeout **too short**, and normal, temporary slowness (network
   congestion, brief GC pause, momentary load spike) gets misclassified
   as a failure, triggering unnecessary and potentially disruptive
   failover/recovery action against a node that was actually fine (a
   **false positive**); set the timeout **too long**, and a genuine
   failure takes longer to detect and react to, extending the actual
   outage/unavailability window (**slow detection**) — there's no single
   fixed value that avoids both costs simultaneously, since the "right"
   timeout depends on real, variable network/system conditions that a
   single fixed number can't adapt to. The **phi-accrual failure
   detector** avoids committing to one fixed threshold by instead
   continuously tracking the *statistical distribution* of past heartbeat
   inter-arrival times for each node specifically, and computing a
   continuous suspicion score based on how anomalous the current gap is
   relative to that node's own historical pattern — a node with a history
   of very regular heartbeats missing just one becomes highly suspicious
   quickly, while a node with historically irregular/bursty heartbeat
   timing requires a longer gap before becoming similarly suspicious —
   letting the detector adapt per-node to actual observed behavior, and
   letting the *application* choose its own risk tolerance (how high a
   suspicion score to act on) rather than baking one universal cutoff
   into the detection mechanism itself.

6. LWW's correctness fundamentally depends on being able to trust that
   "the write with the later timestamp genuinely happened later, in real
   physical time, relative to the other write" — but this module explains
   that wall clocks on different machines **drift relative to each other**
   and are only periodically, imperfectly corrected (e.g., via NTP, which
   itself introduces jumps and has its own network-delay-induced
   uncertainty, not smooth continuous correction) — meaning two different
   machines' timestamps, especially ones close together in time, cannot
   be safely assumed to be directly comparable with the precision LWW's
   correctness actually requires. If Machine A's clock is running even
   slightly ahead of Machine B's, a write that genuinely happened
   *earlier* in real time on Machine B could be assigned a *later*
   timestamp than a write that happened afterward on Machine A, causing
   LWW to keep the wrong write (from B) and silently discard the
   genuinely more recent one (from A) — the entire mechanism's correctness
   rests on an assumption (trustworthy, precisely comparable wall-clock
   time across independent machines) that this section explicitly shows
   doesn't hold in real distributed systems.

7. With no message passed between A and B in either direction, vector
   clocks would correctly show them as **concurrent** — neither event's
   vector clock would show evidence of having observed or been influenced
   by the other (neither vector "dominates" the other when compared
   element-wise), which is exactly the correct, honest answer: without any
   causal link (a message, or any chain of messages) connecting them,
   there genuinely is no meaningful "before" or "after" relationship
   between A and B from the system's perspective — they are causally
   independent. This distinction matters enormously for conflict
   resolution because it tells the application the *truth*: these are two
   independent, potentially-conflicting writes that both deserve
   consideration (e.g., merged via a CRDT, or surfaced to the user/
   application to resolve meaningfully), rather than one having any
   legitimate claim to being "more correct" or "more recent" than the
   other. A wall-clock timestamp comparison, by contrast, would still
   confidently report *some* answer ("A's timestamp is earlier/later than
   B's") even though that answer is essentially meaningless here — since,
   per Question 6, wall-clock timestamps across machines aren't reliably
   comparable in the first place, and even if they were, "which happened
   first physically" isn't actually the same question as "which one
   should logically take precedence," especially for genuinely
   independent, concurrent operations — vector clocks give you the
   correct, actionable information (concurrency, not a false sense of
   ordering) that a wall clock cannot.

</details>

---

## What's next

**Module 13 — Distributed Transactions: 2PC, Sagas, Outbox, Idempotency**
uses this module's foundation (consensus, quorums, fencing, causality) to
tackle the next hard problem: making a single logical operation that spans
*multiple* independent services/databases either fully happen or fully
not happen — the distributed version of Module 06's ACID transactions,
without a single database's WAL to rely on.
