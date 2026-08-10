# Module 04 — Storage Engine Internals: B-Trees, LSM-Trees, WAL, Indexes, Erasure Coding

> **TL;DR:** Every database — relational, NoSQL, vector, time-series — is
> ultimately a thin layer of query language and API on top of a **storage
> engine** that answers one question: how do I organize bytes on disk so
> that reads and writes are both reasonably fast? There are exactly two
> dominant families of answer (B-trees and LSM-trees), and the difference
> between them is a direct, concrete application of Module 01's
> sequential-vs-random-I/O principle. Understanding this module is what
> lets you *predict* whether a database will be read-optimized or
> write-optimized from its storage engine alone, before ever touching its
> query language.

---

## 1. The problem: an index is a trade-off, not a free lookup

A **database index** is a data structure that lets you find a record
without scanning the entire table — trading extra storage and write cost
for faster reads. Every index design in this module is answering the same
question: *given that random disk I/O is expensive (Module 01, Section 3),
how do we minimize how much of it a lookup or a write requires?*

Without an index, finding a row means a **full table scan** — O(n) reads.
With an index, you want something closer to O(log n) — but *how* you
achieve that, and what it costs on writes, is exactly where B-trees and
LSM-trees diverge.

---

## 2. B-Trees: the read-optimized default

A **B-tree** (and its common variant, the **B+tree**, used by nearly every
relational database — PostgreSQL, MySQL/InnoDB, SQLite) is a balanced tree
structure where each node holds many keys (not just 1-2, as in a binary
tree) and points to child nodes, kept balanced so every leaf is the same
distance from the root.

### 2.1 Why "many keys per node," not binary?

Recall from Module 01 that disk (and even RAM-to-cache) access happens in
fixed-size chunks — a disk **page** (commonly 4KB-16KB). A B-tree node is
sized to fit exactly one disk page, and holds as many keys as fit in that
page. This means each level of the tree you descend corresponds to
**exactly one disk read**, and because each node has many keys (high
**fanout**, often hundreds), the tree stays extremely shallow even for
huge datasets — a B-tree over billions of rows is typically only 3-4
levels deep, meaning a lookup costs only 3-4 disk reads.

### 2.2 Reads and writes in a B-tree

- **Read:** descend from root to leaf following keys, ~3-4 page reads for
  huge tables. Fast, and the *access pattern is what it looks like from
  the top* — mostly logical navigation, not raw sequential scanning.
- **Write (insert/update):** find the correct leaf page (same cost as a
  read), then **update that page in place**. If the page is full, it must
  **split** into two pages and update the parent — an occasional, more
  expensive operation, but the common case is one page write.

> **⚠️ Trap:** "Update that page in place" is precisely a **random write**
> — the page being updated could be anywhere on disk, unrelated to
> whatever was written just before it. This is the core weakness B-trees
> have and LSM-trees exist to fix: B-trees are read-optimized because
> lookups are cheap (shallow tree, few reads), but every write potentially
> costs a random disk write, which (Module 01, Section 3.2) is the
> expensive kind.

---

## 3. Write-Ahead Logs (WAL): making random writes crash-safe

Before discussing how LSM-trees dodge the random-write problem, there's a
prerequisite concept both B-trees and LSM-trees rely on for durability:
the **Write-Ahead Log**.

### 3.1 The problem WAL solves

A B-tree's in-place page update is not atomic from a crash's perspective —
if the machine crashes mid-write, a page could be left in a **corrupted,
half-written state**, with no way to tell what it should have looked like.

### 3.2 The mechanism

Before modifying the actual B-tree page in place, the database first
**appends a record describing the intended change** to a separate,
append-only log file — the WAL. Only after that log entry is safely
persisted (Module 01's `fsync`) does the engine modify the actual page.

This gives two guarantees:

- **Crash recovery:** if the machine crashes after the WAL entry is
  fsync'd but before the actual page update completes, the database can
  replay the WAL on restart to redo the change — nothing acknowledged as
  committed is ever lost.
- **Sequential writes for durability:** appending to a log is a
  **sequential write** (Module 01, Section 3.2) — fast — even though the
  actual page update it's protecting is a random write. The expensive
  random write can happen lazily/in the background; the durability
  guarantee is achieved cheaply via the sequential log.

> **🤖 AI/ML Callout:** This "append a durable, sequential log of intended
> changes, apply them lazily" pattern reappears constantly — it's the same
> idea behind Kafka's log-structured storage (Module 15) and behind
> **checkpointing in ML training**: a training job periodically writes a
> checkpoint (a durable snapshot) so that if a long-running, expensive GPU
> job crashes, it resumes from the last checkpoint instead of restarting
> from scratch — conceptually the same insurance-against-crashes trade-off
> as a WAL, applied at a much coarser granularity.

> **⚠️ Trap:** "The WAL entry was written" is the actual moment durability
> is achieved for most databases claiming "durable writes" (recall Module
> 01's `fsync` discussion) — if a database acknowledges a write to the
> client *before* the WAL entry is fsync'd, that acknowledgment is
> premature and the write can be lost on crash. This is a very common,
> very concrete thing to probe when someone claims a system is "durable":
> ask exactly when, relative to fsync, the client gets its acknowledgment.

---

## 4. LSM-Trees: the write-optimized alternative

An **LSM-tree (Log-Structured Merge-tree)** — used by Cassandra, RocksDB,
LevelDB, HBase, and as an option in many modern engines — takes a
fundamentally different approach: **never do random writes to the main
data structure at all.**

### 4.1 The mechanism

1. **Writes go to an in-memory structure** (a sorted structure, often
   called a **memtable**), which is fast (RAM) and naturally handles
   random-key writes cheaply since it's not disk-bound.
2. Every write is *also* appended to a WAL on disk first (Section 3),
   for crash safety — the memtable itself is not yet durable on its own.
3. When the memtable reaches a size threshold, it's **flushed to disk as
   an immutable, sorted file** (often called an **SSTable — Sorted String
   Table**) — written with one large **sequential write**, not scattered
   random writes.
4. Over time, many SSTables accumulate on disk. A background process
   called **compaction** periodically merges multiple SSTables into fewer,
   larger ones, discarding overwritten/deleted entries — itself done via
   sequential reads and a sequential write of the merged output.

### 4.2 Why this is write-optimized

Every write path here — memtable (RAM), WAL append (sequential),
memtable flush (sequential), compaction (sequential) — **never requires a
random disk write**. This is the entire point: LSM-trees convert what
would be scattered random writes (as in a B-tree) into batched sequential
writes, which (Module 01, Section 3.2) can be orders of magnitude faster,
especially on spinning disks, and still meaningfully faster on SSDs.

### 4.3 The cost: reads get more expensive

A read for a given key might need to check the memtable, *and* potentially
multiple SSTables on disk (the key could have been written at any point in
the past, landing in any of several SSTables, with the most recent write
"winning" if the key appears in more than one) — this is strictly more
work than a B-tree's single, direct descent from root to leaf.

Two techniques mitigate this:

- **Bloom filters** — a compact, probabilistic data structure kept in
  memory per SSTable that can definitively say "this key is *not* in this
  SSTable" (with zero false negatives, some false positives), letting a
  read skip SSTables that certainly don't contain the key, without
  touching disk at all for those.
- **Compaction** (Section 4.1, step 4) also directly reduces the number of
  SSTables a read must check, which is why compaction tuning is a real
  operational lever for LSM-based databases — too little compaction and
  reads slow down (**read amplification**); too much compaction and you
  waste disk I/O rewriting data repeatedly (**write amplification**) —
  this three-way tension (read amplification, write amplification, and
  space amplification from not-yet-compacted duplicate/stale entries) is
  the central tuning problem of LSM-tree-based systems.

### 4.4 B-Tree vs. LSM-Tree — the decision table

| | B-Tree | LSM-Tree |
|---|---|---|
| **Reads** | Fast, predictable (few page reads) | Slower, variable (may check memtable + multiple SSTables), mitigated by bloom filters/compaction |
| **Writes** | Slower — random in-place page updates | Fast — sequential appends only |
| **Best for** | Read-heavy workloads, traditional OLTP (Module 23) | Write-heavy workloads (logging, time-series, event ingestion) |
| **Used by** | PostgreSQL, MySQL/InnoDB, SQLite | Cassandra, RocksDB, LevelDB, HBase |
| **Background cost** | Occasional page splits | Ongoing compaction (real, tunable CPU/I/O cost) |

> **🎤 Interview Angle:** "Why would you choose Cassandra over PostgreSQL
> for a high-volume event-logging system?" is directly answerable from
> this table: event logging is overwhelmingly write-heavy and append-only
> in nature (you rarely need to update or point-query old individual
> events with the same rigor a transactional system needs), which plays
> exactly to an LSM-tree's strength and away from a B-tree's. Answering
> with "Cassandra is more scalable" is vague; answering with "Cassandra's
> LSM-tree storage engine converts writes into sequential appends instead
> of B-tree's random page updates, which matches this workload's
> write-heavy, append-mostly access pattern" is precise and demonstrates
> real understanding.

> **🤖 AI/ML Callout:** Vector databases and embedding stores (Module 26)
> frequently favor LSM-tree-like or LSM-inspired storage engines
> (RocksDB is a very common embedded choice) for their metadata/vector
> storage layer, precisely because embedding pipelines are often
> **write-heavy, append-mostly workloads** — continuously ingesting new
> embeddings from a growing corpus — which is the exact profile LSM-trees
> are built for, while the actual similarity-search index on top (HNSW,
> IVF — Module 26) is a separate, specialized structure layered above this
> storage-engine choice.

---

## 5. Indexes beyond the primary structure

The B-tree/LSM-tree choice describes the *primary* storage structure, but
production databases also build **secondary indexes** — additional lookup
structures over non-primary-key columns, so you can efficiently query
`WHERE email = ?` even though rows are physically organized by primary key.

- A secondary index is, structurally, its own B-tree or LSM-tree (of the
  form: indexed-column-value → primary-key), incurring its own write cost
  on every insert/update (every index you add makes writes slower, since
  each one must also be kept up to date) — a direct, tangible instance of
  the **read/write optimization trade-off** from Module 00: every index
  speeds up a specific read pattern and slows down every write.
- A **composite index** (multiple columns, e.g., `(user_id, created_at)`)
  is ordered by the first column, then the second within ties — meaning it
  efficiently serves queries filtering on the first column (optionally plus
  the second), but generally can't efficiently serve a query filtering
  *only* on the second column alone — column order in a composite index is
  a real design decision, not an arbitrary convenience.

> **⚠️ Trap:** "Just add an index" is not a free performance fix. Every
> additional index adds write-side cost (it must be updated on every
> insert/update/delete to the indexed columns) and storage cost. A common
> interview and real-world mistake is over-indexing a write-heavy table
> because "indexes make things faster," without acknowledging *which*
> operations get faster (reads matching that index) and which get slower
> (every write).

---

## 6. Erasure coding: durability without full replication cost

Module 09 will cover replication in depth (keeping full copies of data on
multiple machines for durability/availability), but there's a
storage-layer alternative worth introducing here since it's fundamentally
a storage-encoding technique, not a distribution-protocol one:
**erasure coding**.

### 6.1 The idea

Instead of storing N full copies of your data (**replication** — e.g., 3x
replication means 3x the storage cost), erasure coding splits data into
`k` fragments and computes additional `m` **parity fragments**, storing
all `k + m` fragments across different disks/machines. The original data
can be reconstructed from *any* `k` of the `k + m` total fragments — you
can tolerate up to `m` fragment losses.

This is conceptually the same idea as **RAID** (specifically RAID 5/6 use
this principle) applied at a larger, distributed-systems scale, and is the
mechanism behind how systems like Amazon S3 or HDFS achieve high
durability without simply tripling storage cost.

### 6.2 The trade-off vs. replication

| | Replication (e.g., 3 full copies) | Erasure Coding (e.g., 6 data + 3 parity) |
|---|---|---|
| **Storage overhead** | High (3x for 3 copies) | Lower (e.g., 1.5x for a 6+3 scheme, tolerating 3 losses) |
| **Read latency** | Fast — read directly from any full copy | Slower on the recovery path — reconstructing from fragments requires computation |
| **CPU cost** | Minimal (just copying) | Real — encoding/decoding requires computation |
| **Best for** | Hot, frequently-accessed, latency-sensitive data | Cold/warm, large-volume data where storage cost matters more than access latency (archival storage, large object stores) |

> **📐 Numbers:** A common erasure coding scheme might be "10 data
> fragments + 4 parity fragments," tolerating any 4 simultaneous fragment
> losses while using only 1.4x the raw storage — versus 3x-4x for
> equivalent-durability full replication. This is why large-scale object
> stores (Module 36) holding petabytes of infrequently-read data
> overwhelmingly favor erasure coding over full replication: at that
> scale, the storage cost difference (1.4x vs 3x+) is enormous in absolute
> dollar terms, and the data isn't latency-sensitive enough to need
> replication's faster direct-read path.

> **🎤 Interview Angle:** If asked to design a large object store (Module
> 36) or discuss storage cost at scale, mentioning erasure coding as an
> alternative to naive replication for durability — and correctly stating
> *when* you'd still prefer replication instead (hot/latency-sensitive
> data) — is a strong signal you're thinking about storage cost as a first-
> class constraint, not just availability in the abstract.

---

## Cheat Sheet (for fast revision)

- An index trades write cost and storage for faster reads — never a free
  lunch; every index added slows down every write to that table.
- B-trees: shallow, balanced, page-sized nodes → ~3-4 disk reads for huge
  tables. Writes update pages in place → random writes → B-trees are
  read-optimized, write-costly.
- WAL: append intended changes to a sequential log before modifying data
  in place, enabling crash recovery and converting the *durability*
  guarantee into a cheap sequential write, even when the actual data
  update is a random write.
- LSM-trees: writes go to an in-memory memtable + WAL, flushed to disk as
  immutable sorted SSTables via sequential writes; reads may need to check
  multiple SSTables (mitigated by bloom filters); background compaction
  merges SSTables and tunes the read/write/space amplification trade-off.
  LSM-trees are write-optimized, read-costlier.
- B-tree (PostgreSQL/MySQL/SQLite) → read-heavy OLTP. LSM-tree
  (Cassandra/RocksDB/LevelDB) → write-heavy ingestion/logging/time-series.
- Secondary indexes are their own B-tree/LSM structures; composite index
  column order matters (ordered by first column, then second within ties).
- "The write succeeded" is durable only once the WAL entry (or replica)
  is actually fsync'd/persisted — same durability-gap trap as Module 01.
- Erasure coding: split data into k fragments + m parity fragments,
  reconstructable from any k of k+m; much lower storage overhead than full
  replication, at the cost of read/reconstruction latency and CPU —
  favored for cold/large-volume data (object stores, archival); full
  replication still wins for hot, latency-sensitive data.

---

## Quiz

1. Explain precisely why a B-tree lookup over a table with a billion rows
   typically costs only 3-4 disk reads. What specific design choice makes
   this true?
2. A B-tree's write path is described as "read-optimized, write-costly."
   Explain exactly what makes a B-tree write expensive, using a Module 01
   concept by name.
3. What specific problem does a Write-Ahead Log solve, and why is
   appending to the WAL cheap even when the actual page update it protects
   is expensive? What would happen without a WAL if the machine crashed
   mid-update?
4. Walk through what happens, step by step, when a client writes a new
   key to an LSM-tree-based database (e.g., Cassandra), from the write
   call to the data eventually residing in a compacted SSTable on disk.
5. An LSM-tree read for a given key might need to check several SSTables
   before finding it (or confirming it doesn't exist). Name two specific
   mechanisms that mitigate this cost, and explain what each one actually
   does.
6. You're choosing a database for a service that ingests millions of raw
   sensor readings per second (write-heavy, rarely updated or
   individually queried afterward, mostly read later in large sequential
   batches for analytics) versus a service managing user account
   profiles (moderate write volume, frequent single-record reads by ID).
   Which storage engine family fits each, and why?
7. Contrast replication and erasure coding as two ways to achieve
   durability against disk/machine failure. Give a concrete scenario
   where each would be the better choice, referencing the actual
   trade-off (not just "one is cheaper").

<details>
<summary><b>Answers — click to expand</b></summary>

1. A B-tree node is sized to match a disk page (commonly 4-16KB) and packed
   with as many keys as fit — a high **fanout** (often hundreds of keys
   per node). Because each node holds so many keys, the tree needs very
   few levels to index a huge number of rows — with a fanout of, say, 300,
   even 4 levels can index over 300^4 (well into the tens of billions) of
   rows. Since each level of the tree corresponds to exactly one disk page
   read as you descend from root to leaf, a lookup over a billion-row
   table costs only as many disk reads as the tree is deep — typically
   3-4 — regardless of how large the table grows beyond that, as long as
   fanout stays high.

2. A B-tree write (insert/update) finds the correct leaf page (a
   read, same low cost as above) and then modifies that page **in
   place**. That page could physically be located anywhere on disk,
   completely unrelated to whatever page was written immediately before or
   after it in time — this is precisely a **random write** (Module 01,
   Section 3.2), which is the expensive access pattern on both HDD (seek +
   rotational latency) and, to a lesser but still real degree, SSD. This
   is what makes B-tree writes costly relative to a design that could
   instead batch writes sequentially.

3. The WAL solves the **crash-safety of in-place updates**: without it, if
   a machine crashes in the middle of modifying a B-tree page, that page
   could be left in a corrupted, half-written state with no way to know
   what it should have contained — data could be silently lost or
   corrupted. The WAL fixes this by first **appending** a record of the
   intended change to a separate log file, which is a **sequential write**
   (fast, per Module 01) — this is cheap regardless of how random/expensive
   the actual downstream data-page update will be, because the durability
   guarantee is achieved the moment the sequential log entry is fsync'd,
   not when the (potentially still-pending) random page update completes.
   On restart after a crash, the database replays the WAL to redo any
   changes that were durably logged but not yet reflected in the actual
   data pages, guaranteeing nothing acknowledged as written is lost.

4. (1) The write is first appended to the WAL on disk (a sequential write,
   for crash durability) and the client can be acknowledged at this point.
   (2) The write is applied to the in-memory **memtable**, a sorted
   in-memory structure, which handles it cheaply since it's RAM, not disk.
   (3) Once the memtable grows past a size threshold, it is **flushed to
   disk as an immutable, sorted SSTable**, written via one large
   sequential write. (4) Over time, as more memtables get flushed, many
   SSTables accumulate on disk; a background **compaction** process
   periodically merges multiple SSTables into fewer, larger ones,
   discarding stale/overwritten/deleted entries along the way — itself
   performed via sequential reads of the input SSTables and a sequential
   write of the merged output. At no point in this entire path does a
   random disk write occur.

5. **Bloom filters**: a compact, probabilistic in-memory structure kept
   per SSTable that can definitively say "this key is *not* present" (no
   false negatives, some false positives) — allowing a read to skip
   SSTables that certainly don't contain the key without touching disk at
   all for those, dramatically cutting how many SSTables must actually be
   read from disk. **Compaction**: by periodically merging multiple
   SSTables into fewer, larger ones, compaction directly reduces the
   *number* of distinct SSTables that might contain a given key in the
   first place, so a read has fewer places to check even before bloom
   filters help skip the ones that don't apply.

6. The sensor-ingestion service is a textbook **LSM-tree** fit
   (Cassandra/RocksDB-style): extremely write-heavy, append-mostly,
   rarely updated or point-queried by individual key afterward, and later
   read in large sequential batches — exactly the access pattern LSM-trees
   are optimized for (fast sequential writes; even LSM's costlier point
   reads matter less here since the analytics reads are large sequential
   scans, not scattered point lookups). The user-profile service is a
   better fit for a **B-tree** (PostgreSQL/MySQL-style): moderate write
   volume with frequent, latency-sensitive single-record reads by primary
   key — exactly where a B-tree's shallow, few-disk-read lookup path
   shines, and its comparatively higher write cost is acceptable since
   writes aren't the dominant, extreme-volume workload here.

7. **Replication** stores full copies of data on multiple machines;
   **erasure coding** stores data split into fragments plus computed
   parity fragments, reconstructable from any subset of them, using much
   less total storage for equivalent failure tolerance (e.g., ~1.4x
   overhead vs. 3x+ for replication) but at the cost of slower/more
   CPU-intensive reads on the reconstruction path. A concrete scenario
   favoring **replication**: a primary transactional database serving
   live, latency-sensitive user reads — you want to read directly from a
   full copy instantly, without paying any reconstruction computation cost,
   and the data volume is small enough that 3x storage overhead is an
   acceptable price for that speed. A concrete scenario favoring
   **erasure coding**: a large object store holding petabytes of
   infrequently-accessed archival data (e.g., old log files, backups,
   media assets) — here the storage cost difference between 1.4x and 3x+
   overhead is enormous in absolute terms at that scale, and the data
   isn't accessed often enough or with tight-enough latency requirements
   to justify paying for replication's faster direct-read path.

</details>

---

## What's next

**Module 05 — Relational Databases & Data Modeling** builds on this
module's storage-engine foundation to cover how relational databases
organize data logically (normalization, schemas, joins) on top of the
B-tree storage engines just discussed, and when denormalization is the
right trade-off.
