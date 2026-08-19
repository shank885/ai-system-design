# Module 10 — Partitioning & Sharding: Consistent Hashing, Rebalancing, Hotspots

> **TL;DR:** Module 09 closed with a question its topic couldn't answer:
> replication scales reads and durability, but not writes or total data
> volume — every replica still holds the *entire* dataset. **Partitioning**
> (a.k.a. **sharding**) is the answer: split data across independent
> machines so each one holds only a *subset*, letting both storage
> capacity and write throughput scale roughly linearly with the number of
> machines. This module covers how to decide which data goes where, the
> elegant trick (consistent hashing) that makes adding/removing machines
> not require reshuffling everything, and the specific new failure mode
> partitioning introduces: hotspots.

---

## 1. Why partition: the problem replication can't solve

A dataset (or its write load) can outgrow what a single machine can hold
or handle, even ignoring durability/availability entirely:

- **Storage capacity:** a table with billions of rows and terabytes of
  data may simply not fit on one machine's disk (or fit, but make every
  operation slow due to the index/working-set no longer fitting in memory,
  Module 01's memory hierarchy concern).
- **Write throughput:** even with fast disks, a single machine has a
  ceiling on I/O operations/sec — recall Module 04's numbers (even a fast
  NVMe SSD tops out in the hundreds-of-thousands-of-IOPS range) — and a
  single-leader replication setup (Module 09) funnels *all* writes through
  one machine, however many followers exist.

**Partitioning** splits the dataset into independent chunks (**partitions**
or **shards**), each held by a different machine (or group of machines,
often each partition is itself replicated per Module 09 — partitioning and
replication are typically combined, not alternatives). Each machine only
needs to handle its own partition's share of storage and write load,
letting the system scale by adding more machines, each taking on a
smaller slice.

> **⚠️ Trap:** "Sharding" and "replication" are often confused or
> conflated by beginners, but they solve orthogonal problems: replication
> puts *copies of the same data* on multiple machines (for durability/
> availability/read-scaling); partitioning puts *different subsets of
> data* on different machines (for storage/write scaling). Production
> systems at scale need both simultaneously: each partition/shard is
> typically *also* replicated (e.g., a 3-way replicated shard, per Module
> 09's leader/follower model, for each of many shards) — one axis
> (sharding) increases scale, the other (replication) increases
> resilience of each shard independently.

---

## 2. Partitioning strategies: deciding what goes where

The central question: given a piece of data (identified by some **key**),
which partition does it belong to?

### 2.1 Range-based partitioning

Assign contiguous ranges of the key space to each partition (e.g.,
usernames A-F → partition 1, G-M → partition 2, etc.), often with
partition boundaries adjusted dynamically as data grows.

- **Pros:** Range queries (e.g., "all records between X and Y") are
  efficient — likely served by one or a few contiguous partitions.
- **Cons:** **Hotspots** — if writes aren't uniformly distributed across
  the key range (e.g., keys are timestamps, and all *current* writes have
  today's timestamp), one partition absorbs a disproportionate share of
  load while others sit idle. This is the single most common
  range-partitioning pitfall.

### 2.2 Hash-based partitioning

Apply a hash function to the key, then assign the partition based on the
hash value (e.g., `partition = hash(key) % num_partitions`). This spreads
keys pseudo-randomly and roughly uniformly across partitions, regardless
of the key's actual distribution.

- **Pros:** Naturally avoids the hotspot problem range partitioning has
  for skewed key distributions — a good hash function makes load
  distribution close to uniform, independent of the actual key values.
- **Cons:** Range queries become expensive/impossible to serve
  efficiently — consecutive keys are now scattered essentially randomly
  across every partition, so "all records between X and Y" requires
  querying *every* partition and merging results, defeating the purpose of
  partitioning for that query pattern.

> **🎤 Interview Angle:** "Range vs. hash partitioning" is a direct,
> concrete instance of Module 00's trade-off framing — the right choice
> depends entirely on the actual query pattern: does this system need
> efficient range scans (favoring range partitioning, accepting hotspot
> risk that must be separately managed), or does it need uniform load
> distribution with only point lookups (favoring hash partitioning,
> accepting the loss of efficient range queries)? Naming this trade-off
> explicitly, rather than defaulting to "hash partitioning is better,"
> is what a strong answer does.

### 2.3 Directory-based (lookup-service) partitioning

A separate service maintains an explicit mapping of key → partition,
looked up on every request rather than computed algorithmically. Offers
maximum flexibility (arbitrary, dynamically adjustable assignment), at
the cost of that lookup service becoming a new dependency (and potential
bottleneck/SPOF, unless it's itself made highly available — a common
approach in real systems is to cache this mapping aggressively on clients,
refreshing only occasionally).

---

## 3. Consistent hashing: the mechanism that makes rebalancing cheap

### 3.1 The problem naive hash partitioning has

Simple hash partitioning (`hash(key) % num_partitions`) has a serious
flaw: if you add or remove a partition (changing `num_partitions`), the
modulo result changes for **almost every key**, meaning almost the entire
dataset must be reshuffled/moved to new partitions — an expensive, often
system-wide operation just to add one more machine.

### 3.2 The consistent hashing solution

Instead of hashing keys directly into a fixed number of "buckets,"
**consistent hashing** maps both keys *and* partitions (nodes) onto the
same conceptual **ring** (a circular hash space, e.g., 0 to 2^32-1). A
key belongs to the first node found by walking clockwise around the ring
from the key's hash position.

- **Why this minimizes rebalancing:** adding a new node only affects the
  keys that fall between the new node's ring position and the *previous*
  node clockwise from it — only that slice of keys needs to move, to the
  new node. Every other key, on every other node, is completely
  unaffected. Removing a node similarly only affects the keys it was
  responsible for, which simply move to the next node clockwise —
  nowhere else on the ring is disturbed.

### 3.3 Virtual nodes: fixing consistent hashing's own hotspot risk

Plain consistent hashing with one ring-position per physical node has its
own problem: with relatively few nodes, their random ring positions can
end up unevenly spaced, meaning some nodes are responsible for a much
larger arc of the ring (and thus a disproportionate share of keys) than
others — a structural hotspot risk baked into the ring layout itself,
independent of the key distribution issue from Section 2.1.

**Virtual nodes (vnodes)** fix this: each *physical* machine is assigned
many positions on the ring (not just one) — e.g., 100-200 virtual
positions per physical node, scattered around the ring. This averages out
uneven spacing (a physical node's total share of the ring converges
toward the fair average as its number of virtual positions increases) and
also makes rebalancing smoother when nodes are added/removed — the load
being moved is now spread across many small virtual-node-sized pieces
distributed across many other nodes, rather than being one large
contiguous chunk moved to a single neighbor.

> **📐 Numbers:** With, say, 10 physical nodes and no virtual nodes, random
> ring placement could plausibly leave one node responsible for 25% of
> the ring's key space while another gets only 3% — a severe imbalance.
> With 200 virtual nodes per physical node (2,000 total ring positions),
> the law of large numbers makes each physical node's *total* share
> converge much closer to the fair 10% each, since its share is now the
> sum of 200 independently-scattered small arcs rather than one large,
> potentially-unlucky one.

> **🤖 AI/ML Callout:** Consistent hashing (with virtual nodes) is the
> standard technique behind sharding a distributed vector database
> (Module 26) or a distributed key-value feature store (Module 24) across
> many nodes — as new nodes are added to absorb a growing embedding
> corpus or feature volume, consistent hashing ensures only a small,
> bounded fraction of vectors/features need to be physically moved to
> rebalance, rather than requiring a full, expensive re-partitioning of
> the entire dataset every time the cluster scales out.

---

## 4. Rebalancing: moving data when the cluster changes

Even with consistent hashing minimizing *how much* data moves, actually
moving it (**rebalancing**) is an operational process with its own
considerations:

- **Automatic vs. manual rebalancing:** fully automatic rebalancing
  (the system detects imbalance or a node change and moves data on its
  own) is convenient but risky if it triggers unexpectedly during already-
  high load (the rebalancing process itself consumes network/disk I/O,
  potentially worsening an already-stressed system) — many production
  systems prefer rebalancing to be triggered deliberately by an operator,
  or at least throttled/rate-limited even when automatic.
- **Fixed number of partitions, variable node count:** an alternative
  strategy some systems use: create far more partitions than there are
  currently nodes (e.g., 1,000 partitions across 10 nodes, 100 partitions
  per node), and when a new node joins, simply move some *whole existing
  partitions* to it — since partitions are already fixed-size chunks, "add
  a node" becomes "reassign N whole partitions to the new node" rather
  than recomputing a hash ring boundary, simplifying the rebalancing logic
  considerably (at the cost of needing to choose the number of partitions
  upfront, ideally larger than you'll ever need nodes, since splitting an
  existing partition further later is more disruptive than just
  reassigning whole ones).

---

## 5. Hotspots: the recurring failure mode

Even with a well-chosen partitioning strategy, **hot keys** (a small
number of keys receiving disproportionate traffic) remain a real,
frequently-encountered production problem — this is the partitioning-layer
analog of Module 08's cache stampede: load concentrated on a narrow target
that the system wasn't provisioned to handle at that concentration, even
if aggregate system capacity is fine.

- **Causes:** a celebrity's social media post going viral (everyone reads/
  writes to that one post's key), a popular product during a flash sale
  (Module 06's ticket-booking example, revisited at the partitioning
  layer), or poor key design (e.g., partitioning by a low-cardinality
  attribute like "country," where one country's users vastly outnumber
  others).
- **Mitigations:**
  - **Key salting/splitting:** append a random suffix to a hot key,
    spreading its writes across multiple *sub-keys* on different
    partitions (e.g., `post:123:0` through `post:123:9`), then aggregate
    across the sub-keys on read. Trades read-side complexity (must query
    and combine multiple sub-keys) for write-side load distribution.
  - **Caching the hot key** (Module 08) — if the hot key is
    read-dominated, a cache in front of the partition can absorb the vast
    majority of read traffic before it ever reaches the storage layer.
  - **Application-level awareness and special-casing** — some systems
    explicitly detect known-hot keys (e.g., a promoted item ahead of a
    flash sale) and proactively provision extra capacity or route them
    differently, rather than relying purely on the generic partitioning
    scheme to absorb the spike.

> **🎤 Interview Angle:** If asked to design a system with an obviously
> skewed access pattern (celebrity accounts, viral content, flash sales),
> proactively raising the hotspot risk and naming a specific mitigation
> (salting is the most commonly expected answer) is a strong, expected
> signal — this is one of the most frequently tested "does this candidate
> think about real-world skew, not just the average case" checkpoints
> across system design interviews generally.

---

## 6. Secondary indexes under partitioning

Partitioning by a primary key is straightforward, but what about queries
filtering on a *different*, non-partition-key attribute? Two approaches:

- **Local (document-partitioned) secondary indexes:** each partition
  maintains its own secondary index, covering only the data it holds.
  Writes stay local to one partition (fast), but a query on the secondary
  index must be broadcast to *every* partition and results merged
  (**scatter-gather**) — since the matching data could be on any
  partition.
- **Global (term-partitioned) secondary indexes:** the secondary index
  itself is partitioned separately (e.g., by the indexed attribute's
  value, not the primary key), so a query on it only needs to hit the
  specific partition(s) holding that index range. Reads are efficient
  (no scatter-gather), but writes become more expensive/complex — a
  single write may need to update a secondary index partition different
  from (and possibly on a different machine than) the primary data
  partition, often asynchronously, introducing its own replication-lag-
  style consistency window (echoing Module 09's anomalies, now for index
  freshness specifically).

> **⚠️ Trap:** This is the exact same read/write trade-off pattern from
> Module 00 and Module 05 recurring yet again, now at the partitioning
> layer — local secondary indexes optimize writes at the cost of
> expensive scatter-gather reads; global secondary indexes optimize reads
> at the cost of more complex, often asynchronous writes. There's no
> universal right answer, only the right answer for a given system's
> actual read/write ratio and latency requirements for that specific
> query pattern.

---

## Cheat Sheet (for fast revision)

- Partitioning (sharding) splits *different* data across machines (for
  storage/write scaling); replication puts *copies of the same* data on
  multiple machines (for durability/availability/read scaling) — they're
  orthogonal and typically combined (each shard is itself replicated).
- Range partitioning: efficient range queries, but hotspot risk if key
  distribution/access pattern is skewed (e.g., timestamp keys).
- Hash partitioning: near-uniform load distribution, but destroys
  efficient range queries (scatter-gather across all partitions).
- Consistent hashing: maps keys and nodes onto the same ring; adding/
  removing a node only moves the keys in its immediate ring neighborhood,
  not the whole dataset — this is what makes rebalancing cheap.
- Virtual nodes: give each physical node many ring positions, fixing
  consistent hashing's own uneven-spacing hotspot risk and smoothing
  rebalancing load across many nodes instead of one neighbor.
- Rebalancing: prefer controlled/throttled over fully automatic-and-
  unbounded; fixed-partition-count-with-node-reassignment is a common
  simplification over recomputing ring boundaries.
- Hotspots (hot keys) are a recurring failure mode independent of
  partitioning strategy quality — mitigate via key salting/splitting,
  caching the hot key, or explicit application-level special-casing for
  known hot items.
- Secondary indexes under partitioning: local/document-partitioned (fast
  writes, scatter-gather reads) vs. global/term-partitioned (fast reads,
  complex/async writes) — the same read/write trade-off axis, one layer
  deeper.

---

## Quiz

1. A colleague says "we're already using replication, so we don't need to
   worry about sharding." Explain precisely what's wrong with this
   statement, in terms of what each mechanism actually scales.
2. You partition a table of user activity events by timestamp using
   range partitioning, expecting efficient "events in the last hour"
   queries. In production, one partition (today's) receives 95% of all
   write traffic while others sit nearly idle. Explain exactly why this
   happened and name the fundamental trade-off it illustrates.
3. Explain precisely why naive hash partitioning (`hash(key) %
   num_partitions`) makes adding a single new node extremely disruptive,
   and how consistent hashing avoids this.
4. Why does plain consistent hashing (one ring position per physical
   node) still have a hotspot risk of its own, separate from the
   key-distribution hotspot problem in Question 2? How do virtual nodes
   fix it?
5. A single celebrity's post receives far more read and write traffic
   than any other key in the system, overwhelming the one partition that
   owns it, even though the overall cluster has plenty of spare capacity.
   Name this phenomenon and walk through how key salting addresses it,
   including what it costs you on the read side.
6. Contrast local (document-partitioned) and global (term-partitioned)
   secondary indexes in terms of what becomes expensive in each — connect
   your answer explicitly to a trade-off axis introduced in an earlier
   module.
7. A system is designed with a fixed 1,000 logical partitions distributed
   across a starting cluster of 10 physical nodes. Explain why this
   design choice simplifies rebalancing when an 11th node is added,
   compared to a system that computes partition boundaries dynamically
   from a hash ring.

<details>
<summary><b>Answers — click to expand</b></summary>

1. Replication and partitioning solve **orthogonal** problems: replication
   puts full copies of the *same* dataset on multiple machines, which
   improves durability (surviving a machine/disk loss), availability
   (surviving a node going down), and read throughput (spreading read
   load across replicas) — but every replica still holds the *entire*
   dataset and, in the common single-leader case, all writes still funnel
   through one leader machine. It does nothing to address a dataset that's
   too large to fit on one machine, or write throughput exceeding what a
   single leader can absorb, since replication doesn't split the data or
   the write path at all. Partitioning (sharding) is what actually
   addresses those two problems, by splitting the dataset into
   independent subsets held by different machines, so each machine only
   needs to store and accept writes for its own slice — "we use
   replication" says nothing about whether the total dataset size or
   write volume has been addressed at all.

2. This happened because of a **hotspot** — since events are partitioned
   by timestamp using range partitioning, and essentially all *current*
   writes have a timestamp close to "now," nearly every new event falls
   into the same narrow, recent range, which range partitioning assigns
   to the same one (or few) partitions — so that partition absorbs almost
   all write load while partitions covering older time ranges receive
   almost none. This illustrates the fundamental **range-partitioning
   hotspot trade-off** from Section 2.1: range partitioning's efficient-
   range-query benefit comes at the cost of vulnerability to hotspots
   whenever the access/write pattern is skewed relative to the key
   ordering — which is essentially guaranteed for any naturally
   time-ordered, "mostly recent data is hot" workload, making naive
   timestamp-based range partitioning a recognizable anti-pattern for
   exactly this reason.

3. With naive hash partitioning, the partition a key belongs to is
   `hash(key) % num_partitions` — the moment `num_partitions` changes
   (e.g., from 10 to 11 because a node was added), the modulo result
   changes for the overwhelming majority of keys, even though only one
   new machine was added — because the divisor itself changed, essentially
   every key's assigned partition shifts to a different value than before.
   This means adding a single node forces nearly the *entire* dataset to
   be reshuffled and moved to new partitions, an extremely expensive,
   disruptive operation for what should be a routine capacity-scaling
   action. Consistent hashing avoids this because it doesn't use modulo
   arithmetic against the *current count* of nodes at all — instead, keys
   and nodes are both mapped onto fixed positions on a hash ring, and a
   key's assignment (walk clockwise to the first node) only depends on the
   *relative positions* of the key and nearby nodes on the ring, not on
   the total node count — so adding a node only affects the specific,
   bounded arc of keys between it and its immediate predecessor on the
   ring, leaving every other key's assignment completely untouched.

4. Plain consistent hashing places each physical node at exactly one
   (essentially random) position on the ring — with a relatively small
   number of nodes, this random placement can easily be uneven, leaving
   some nodes responsible for a disproportionately large arc of the ring
   (and thus a disproportionately large share of keys) purely due to
   unlucky positioning, completely independent of whether the actual key
   values/access pattern are skewed (which is the Question 2 problem) —
   this is a **structural** hotspot risk baked into the ring layout
   itself. **Virtual nodes** fix this by giving each physical node many
   scattered positions on the ring (e.g., 100-200) instead of just one —
   since a physical node's total share of the ring is now the sum of many
   independently-placed small arcs rather than one large, potentially-
   unlucky one, the law of large numbers makes each physical node's total
   share converge much closer to the fair average share, regardless of
   how few physical nodes there are.

5. This is a **hot key** (hotspot) problem. **Key salting** addresses it
   by appending a random suffix to the hot key at write time, effectively
   turning one key into several sub-keys (e.g., `post:123:0` through
   `post:123:9`) that consistent hashing (or whatever partitioning scheme
   is in use) will naturally distribute across *different* partitions,
   since they're now different keys entirely — spreading what was a single
   partition's concentrated write load across multiple partitions instead.
   The cost on the read side: since the "real" data for the hot key is now
   split across N sub-keys on potentially N different partitions, reading
   the complete picture (e.g., the total like count on that post) requires
   querying all N sub-keys and aggregating/summing the results client-side
   or via a coordinating layer, rather than a single simple lookup —
   trading write-side load distribution for added read-side complexity and
   cost, a direct instance of Module 00's general trade-off framing.

6. **Local (document-partitioned) secondary indexes** keep the index
   entries for each partition's data co-located with that partition —
   writes stay entirely local and fast (updating the secondary index is
   just a local operation alongside the primary write), but a query using
   the secondary index must be sent to *every* partition (since matching
   data could be on any of them) and the results merged
   (**scatter-gather**), making reads expensive and involving every node
   in the cluster for even a narrow query. **Global (term-partitioned)
   secondary indexes** partition the index itself separately (typically by
   the indexed value), so a read only needs to query the specific
   partition(s) holding the relevant index range — fast, targeted reads —
   but a single write may now need to update an index partition that
   lives on a *different* machine than the primary data partition,
   typically requiring asynchronous propagation and introducing its own
   consistency window. This is explicitly the same **read/write
   optimization trade-off axis** first introduced in Module 00 (and
   revisited in Module 05's denormalization discussion) — local indexes
   optimize the write path at the read path's expense, global indexes
   optimize the read path at the write path's expense, and neither is
   universally correct.

7. With a fixed, larger-than-needed number of logical partitions (1,000)
   spread across fewer physical nodes (10, meaning 100 partitions per
   node initially), adding an 11th node doesn't require recomputing any
   hash ring boundaries or determining which specific keys need to move
   based on new ring positions — it simply means reassigning some whole,
   already-existing partitions (e.g., moving roughly 91 of the 1,000
   fixed partitions, so each of the 11 nodes ends up with ~91 partitions)
   from their current nodes to the new node. Since partitions are
   pre-defined, fixed-size units, "rebalancing" reduces to a much simpler
   operation: copy some already-self-contained partition's data to the new
   node and update which node is now responsible for serving it, rather
   than needing to calculate new boundary positions and determine, key by
   key, which specific keys now fall on which side of a shifted boundary —
   the operational logic is simpler because the unit of movement (a whole
   pre-existing partition) is fixed and already well-defined in advance,
   rather than being computed dynamically at rebalance time.

</details>

---

## What's next

**Module 11 — CAP, PACELC & Consistency Models** gives formal, rigorous
names to the trade-offs Modules 09 and 10 have been demonstrating
concretely all along — replication lag, quorum tuning, and the
consistency-vs-availability tension are all specific instances of the CAP
theorem, and this module finally states it precisely, along with its
commonly misunderstood nuances and PACELC's extension of it.
