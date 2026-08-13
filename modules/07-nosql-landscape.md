# Module 07 — The NoSQL Landscape: KV, Document, Wide-Column, Graph, Time-Series, Vector

> **TL;DR:** "NoSQL" isn't one thing — it's a grab-bag term for databases
> that relax some part of the relational model (Module 05) or ACID
> guarantees (Module 06) in exchange for something else: horizontal
> scalability, schema flexibility, or a data model that fits a specific
> access pattern more naturally than tables and joins do. This module
> covers the six families you actually need to recognize, what each one
> optimizes for, and — most importantly — how to pick between them based
> on your actual query patterns, which is the single most common weakness
> in interview answers about NoSQL.

---

## 1. Why NoSQL exists: what's actually being traded away

Every NoSQL family in this module gives up *something* a relational
database provides by default, in exchange for something else:

| What's often relaxed | In exchange for |
|---|---|
| Joins (data is denormalized/embedded instead) | Faster reads for known access patterns, no join cost |
| Strong (ACID) transactions across multiple rows/documents | Horizontal scalability, higher write throughput |
| A fixed schema | Flexibility to store heterogeneous or evolving records |
| Strong consistency (Module 11) | Availability and lower latency, especially across regions |

None of this is "worse" than relational — it's Module 00's trade-off
framing applied at the database-selection level: **there is no
universally best database, only the best database for a stated access
pattern and consistency requirement.** The rest of this module is a tour
of which family fits which pattern.

> **⚠️ Trap:** "NoSQL is more scalable than SQL" is an oversimplified,
> often wrong statement — modern relational databases scale substantially
> (vertical scaling, read replicas, and even horizontal sharding, Module
> 10) and some NoSQL databases (e.g., MongoDB in certain configurations)
> can still offer relatively strong consistency. The actually meaningful
> distinction per-database is: what's the data model, what consistency
> guarantees does it make, and what access patterns is its storage engine
> and API built around? "SQL vs. NoSQL" is a much less useful framing than
> naming the specific family and its actual trade-offs.

---

## 2. Key-Value stores

The simplest model: a giant hash map — `get(key)` and `put(key, value)`,
where the value is typically an opaque blob the database doesn't inspect
or index into (no querying by value content).

- **Examples:** Redis (also a cache, Module 08), DynamoDB (also spans
  wide-column-like features), Riak, etcd (also used for coordination,
  Module 12).
- **Why it's fast:** the simplicity of the access pattern (exact-key
  lookup only) means the storage engine doesn't need to support arbitrary
  queries — often backed by a hash index or an LSM-tree (Module 04) tuned
  purely for point lookups and writes.
- **Best for:** session storage, caching, feature flags, shopping carts —
  anything naturally addressed by a single unique key with no need to
  query by other attributes.

> **⚠️ Trap:** Key-value stores don't support secondary queries
> (`WHERE` on a non-key field) without either scanning everything or
> building your own secondary index scheme on top. If your access pattern
> needs "find all X where Y," a pure KV store is the wrong tool unless you
> can restructure the problem to always look up by a known key.

---

## 3. Document stores

Store semi-structured, self-contained **documents** (usually JSON/BSON),
each identified by a key but queryable by their internal fields too,
unlike a pure KV store.

- **Examples:** MongoDB, Couchbase, (and Elasticsearch, though it's more
  specialized toward search).
- **Data modeling philosophy:** favors **embedding** related data inside
  one document (e.g., an order document containing its line items nested
  directly inside it) rather than normalizing into separate
  tables/collections joined at query time — the document itself is
  designed around "what does one read of this entity need to return?"
  which is exactly the "model for the query" mindset introduced at the end
  of Module 05.
- **Trade-off:** embedding avoids joins for the common read pattern, but
  duplicates data if the same sub-object needs to appear in multiple
  parent documents (the same denormalization-sync problem from Module 05,
  Section 4.2 — now a first-class modeling decision instead of an
  optimization applied later) and most document databases only offer
  strong transactional guarantees *within* a single document, not across
  multiple documents (though this has been improving in modern versions of
  some document databases).

> **🎤 Interview Angle:** "Should I embed or reference?" is the document-
> database equivalent of "should I normalize or denormalize?" from Module
> 05 — the answer follows the same logic: embed when the nested data is
> almost always read/written together with its parent and doesn't need
> independent querying; reference (store an ID and look it up separately)
> when the nested data is large, frequently updated independently, or
> needs to be queried/reused across many different parent documents.

---

## 4. Wide-column stores

Organize data as rows identified by a key, but where each row can have a
**different, sparse set of columns**, and columns are physically grouped
into **column families** — a hybrid between a table and a nested map.

- **Examples:** Cassandra, HBase, Google Bigtable.
- **Storage engine:** almost universally **LSM-tree-based** (Module 04) —
  wide-column stores are built for exactly the write-heavy, high-throughput
  ingestion workloads LSM-trees excel at.
- **Data modeling philosophy:** unlike document/relational modeling
  (start from entities, then figure out queries), wide-column modeling is
  explicitly and unapologetically **query-first**: you design your
  **partition key** and **clustering columns** based on the *exact* queries
  you need to serve, often creating multiple denormalized tables holding
  the same underlying data, each organized for a different query pattern
  (a technique sometimes called "query-driven modeling" or, informally,
  "one table per query").
- **Trade-off:** extremely fast, horizontally scalable writes and reads
  *for the specific queries you designed the schema around* — but queries
  outside that pattern (e.g., an ad-hoc filter on a non-indexed column)
  are expensive or outright unsupported without a full scan.

> **📐 Numbers:** Cassandra is a canonical choice for systems needing
> to sustain very high sustained write throughput (hundreds of thousands
> of writes/sec across a cluster) with predictable low latency, precisely
> because its LSM-tree engine (Module 04) converts that write volume into
> sequential disk operations, and its partition-key-based sharding (Module
> 10) distributes that load horizontally across many nodes with no single
> coordinating bottleneck for writes.

> **🤖 AI/ML Callout:** Wide-column stores are a common choice for storing
> raw ML training event logs and time-stamped feature history at massive
> scale (e.g., "every user interaction event, ever") — the write pattern
> (constant high-volume append, rarely updated after the fact) and the
> query pattern (fetch all events for a specific entity ID within a time
> range — a natural partition-key + clustering-column design) map directly
> onto what wide-column stores are built for.

---

## 5. Graph databases

Model data explicitly as **nodes** (entities) and **edges** (relationships
between them, which can themselves carry properties), optimized for
efficiently traversing relationships — "find all of X's friends' friends
who also like Y" — which would require expensive, deeply-nested joins in
a relational database.

- **Examples:** Neo4j, Amazon Neptune, ArangoDB (multi-model).
- **Why relational struggles here:** each "hop" in a relationship
  traversal is a join in a relational model; a query traversing many hops
  (common in social networks, fraud detection, recommendation graphs)
  means many chained joins, each one adding cost, and the number of hops
  is often not known in advance (variable-depth traversal, like "find any
  path connecting A and B"), which relational query languages handle
  awkwardly.
- **Why graph databases are faster for this:** they typically store
  direct physical pointers/references between connected nodes (**index-
  free adjacency** — a node "knows" its neighbors directly, without a
  separate join/index lookup), so traversing a relationship is a fast
  pointer-following operation rather than a query-planner-mediated join.

> **🎤 Interview Angle:** "Design a social network's friend recommendation
> feature" or "design a fraud detection system tracing connections between
> accounts" are the classic cues for graph databases — the signal to
> listen for in the problem statement is **relationship traversal depth
> that's unknown or large**, not just "the data has relationships" (nearly
> all data has relationships; a graph database earns its place when
> *traversing* those relationships, potentially many hops deep, is the
> actual, frequent query pattern).

---

## 6. Time-series databases

Optimized specifically for data points indexed by **time** — metrics,
sensor readings, logs, financial ticks — where the overwhelming access
pattern is: high-volume appends of new timestamped points, and queries
that aggregate/downsample over time ranges (e.g., "average CPU usage per
5-minute bucket over the last 24 hours").

- **Examples:** InfluxDB, TimescaleDB (a Postgres extension), Prometheus.
- **Storage optimizations specific to this workload:**
  - **Time-based partitioning** — data is physically organized into
    chunks by time range, so a query for "the last 24 hours" only touches
    the relevant recent chunks, and old chunks can be cheaply dropped
    entirely once past their retention period (much cheaper than deleting
    individual rows).
  - **Downsampling/rollups** — older data is often automatically
    aggregated into coarser-grained summaries (e.g., raw per-second data
    kept for a week, then rolled up to per-hour averages kept for a year),
    trading precision for storage cost over time — a time-specific
    instance of Module 05's denormalization-for-reads trade-off.
  - **Specialized compression** — consecutive timestamped values are
    often very similar to each other (e.g., a slowly changing sensor
    reading), enabling much higher compression ratios than general-purpose
    row storage, using techniques like delta encoding (store the
    difference from the previous value, not the full value).

> **🤖 AI/ML Callout:** Time-series databases are a natural fit for storing
> ML model **monitoring metrics** — prediction latency, request volume,
> feature drift scores, model accuracy over time — exactly the
> "high-volume timestamped writes, range-aggregated reads" pattern this
> family is built for, which is why observability stacks (Module 19) for
> ML systems frequently sit on top of a time-series database underneath.

---

## 7. Vector databases

Store high-dimensional numeric vectors (**embeddings** — the numeric
representation of text, images, or other content produced by an ML model)
and support **approximate nearest-neighbor (ANN) search**: "find the k
vectors most similar to this query vector," using a similarity metric
like cosine similarity or Euclidean distance.

- **Examples:** Pinecone, Weaviate, Milvus, Qdrant, and vector-search
  *extensions* to existing databases (pgvector for Postgres, Elasticsearch/
  OpenSearch vector fields).
- **Why this needs a specialized index:** a brute-force nearest-neighbor
  search (compare the query vector against every stored vector) is O(n)
  per query and becomes impractically slow at millions/billions of
  vectors — vector databases instead build **approximate** indexes (HNSW,
  IVF — the specific algorithms are covered in depth in Module 26) that
  trade a small amount of accuracy for dramatically faster lookups,
  conceptually similar to how a B-tree (Module 04) trades some structure
  for fast approximate-in-the-navigational-sense (not accuracy-approximate,
  but *structurally* similar in spirit — trading exhaustive search for a
  navigable structure) lookups, though the underlying algorithms are quite
  different.
- This module only introduces *what* a vector database is and why it
  exists; Module 26 (Vector Search & RAG Systems) covers the indexing
  algorithms, hybrid search, and freshness trade-offs in full depth — this
  is intentionally left brief here to avoid duplicating that module.

> **🤖 AI/ML Callout:** Vector databases are the storage layer underneath
> essentially every modern **RAG (Retrieval-Augmented Generation)**
> system — an LLM application retrieves the most semantically relevant
> chunks of a document corpus (via vector similarity search) to include as
> context in a prompt, rather than relying purely on the model's trained-in
> knowledge. This is likely the single most directly relevant NoSQL family
> to your background as an AI engineer, which is exactly why it gets an
> entire dedicated module (26) later in this curriculum.

---

## 8. Choosing a database: the decision framework

Given a real access pattern, ask, in order:

1. **What's the natural shape of a single unit of data?** A flat
   key→value pair, a nested document, sparse wide rows, connected
   nodes/edges, timestamped points, or high-dimensional vectors?
2. **What are the actual query patterns, and how often does each run?**
   Point lookups by key? Filtering by non-key attributes? Deep relationship
   traversal? Time-range aggregation? Similarity search? (Recall Module
   05, Section 5 — "model for the query," now the *primary* decision
   driver for most NoSQL families, not an optimization applied after the
   fact.)
3. **What consistency guarantee does this data actually need?** Financial
   balances likely need strong consistency (favoring relational + ACID,
   Module 06); a social media "like count" can tolerate eventual
   consistency (favoring a NoSQL store optimized for availability/
   throughput instead) — this question is formalized fully in Module 11.
4. **What's the write volume and pattern?** Steady moderate writes with
   frequent point reads (B-tree/relational territory) vs. extremely high
   sustained write throughput (LSM-tree/wide-column territory)?

> **🎤 Interview Angle:** A weak answer picks a database by name-recognition
> ("we'll use MongoDB because it's popular" or "Cassandra because it
> scales"). A strong answer walks through this four-question framework
> explicitly, out loud, arriving at a specific family (not necessarily a
> specific product) and justifying it from the actual stated requirements
> — and it's completely fine, and often correct, for the answer to be
> "a relational database, because the access patterns here are exactly
> what B-trees and ACID transactions are built for" — NoSQL is not the
> default "senior" answer; the *reasoning* is what's senior.

---

## Cheat Sheet (for fast revision)

- NoSQL isn't one model — it's six-ish families, each relaxing a
  different relational/ACID guarantee for a specific benefit. Always name
  the specific family, not "NoSQL" generically.
- Key-Value: exact-key lookups only, no secondary query support. Best for
  caching, sessions, simple lookups.
- Document: semi-structured, queryable-by-field documents; "embed vs.
  reference" mirrors "normalize vs. denormalize" from Module 05.
- Wide-Column: LSM-tree-based, query-first schema design (partition key +
  clustering columns per query), extreme write throughput, poor at ad-hoc
  queries outside the designed pattern.
- Graph: nodes + edges with index-free adjacency, built for deep/unknown-
  depth relationship traversal — the trigger phrase is "traverse
  relationships," not just "data has relationships."
- Time-Series: time-partitioned storage, downsampling/rollups, delta
  compression — built for high-volume timestamped writes + range
  aggregation reads.
- Vector: stores embeddings, supports approximate nearest-neighbor search
  via specialized indexes (HNSW/IVF, detailed in Module 26) since
  brute-force search doesn't scale; the storage layer underneath RAG
  systems.
- Decision framework: (1) natural shape of one unit of data, (2) actual
  query patterns and frequency, (3) required consistency level, (4) write
  volume/pattern. Reasoning through these beats picking by name
  recognition — and "relational is correct here" is a legitimate,
  sometimes best, answer.

---

## Quiz

1. A key-value store can't efficiently answer "find all users where
   `age > 30`." Explain precisely why, in terms of what the storage engine
   is built to support, and what you'd need to add to answer that query
   efficiently.
2. Explain the document-database "embed vs. reference" decision using the
   same underlying logic as Module 05's normalize/denormalize trade-off —
   when does embedding make sense, and when does it recreate the
   denormalization-sync problem?
3. Wide-column stores are described as "query-first" schema design, often
   creating multiple denormalized tables for the same underlying data.
   Explain why this approach is necessary for Cassandra-style databases
   specifically, connecting it to their LSM-tree storage engine and
   partition-key-based access model.
4. What is "index-free adjacency" in a graph database, and why does it
   make multi-hop relationship traversal faster than the equivalent
   relational query? Give an example of a query where this advantage is
   large, and one where it wouldn't matter much.
5. Explain two specific storage-level optimizations time-series databases
   use that a general-purpose relational database typically doesn't, and
   what workload characteristic each optimization exploits.
6. Why does brute-force nearest-neighbor search not scale for a vector
   database with a billion embeddings, and what's the general trade-off
   the algorithms covered in Module 26 make instead?
7. A team wants to choose a database for a new feature and says "let's
   use Cassandra, it's what scales." Using the four-question decision
   framework from Section 8, what would you actually want to know before
   agreeing or disagreeing, and under what conditions would you argue for
   a plain relational database instead?

<details>
<summary><b>Answers — click to expand</b></summary>

1. A pure key-value store's storage engine is built around exact-key
   lookups — it maps a key directly to a value (often via a hash index or
   an LSM-tree tuned purely for point gets/puts, Module 04) and generally
   treats the value itself as an opaque blob it doesn't parse, index, or
   query into. Answering "`age > 30`" requires inspecting the *contents*
   of every value to check a field inside it — since there's no index over
   that field, the only way to answer this with a pure KV store is
   scanning every single value, an expensive O(n) operation the store
   isn't optimized for. To answer this efficiently, you'd need either a
   database that supports secondary indexes over document/value fields
   (a document store, Section 3), or you'd need to build and maintain your
   own secondary index structure on top of the KV store manually (e.g., a
   separate `age_index` key/value entries you update alongside every
   write) — extra complexity a document or relational database gives you
   natively.

2. The underlying logic is identical to normalize/denormalize: embedding
   is denormalization (duplicating/nesting data directly where it's read),
   referencing is closer to normalization (storing a pointer/ID and
   looking the related data up separately). **Embedding makes sense** when
   the nested data is almost always read and written together with its
   parent, is naturally "owned" by that one parent (not shared/reused
   across many different parent documents), and doesn't need to be
   queried independently of its parent — e.g., an order's line items
   embedded directly in the order document, since you almost always want
   them together and they don't exist meaningfully apart from that order.
   **Referencing is better**, and embedding recreates the
   denormalization-sync problem, when the nested data is large, updated
   independently and frequently, or needs to be shared/queried across many
   different parent documents — e.g., embedding a full product document
   inside every order line item that references it would mean every price
   change on that product requires finding and updating every order
   document that ever embedded it, exactly the same update-anomaly risk
   Module 05 described for a denormalized `user_email` field.

3. Cassandra's LSM-tree storage engine (Module 04) is highly optimized for
   fast sequential writes and reasonably fast reads *when the read pattern
   matches how data is physically organized on disk* — specifically, reads
   for a given **partition key** are efficient because all data for that
   partition is stored together, sorted by the clustering columns. But
   Cassandra deliberately doesn't support the kind of flexible ad-hoc
   query planning a relational database's B-tree + secondary indexes
   provide (Module 05) — there's no general-purpose join, and querying by
   a non-partition-key attribute either requires a full cluster scan or
   simply isn't efficiently supported. Because of this, if an application
   needs to query the same underlying data in multiple different ways
   (e.g., "get a user's orders by user ID" *and* "get all orders in a
   date range regardless of user"), each distinct query pattern needs its
   *own* table, with the data physically organized (partitioned/clustered)
   specifically for that query — hence "query-first," and hence multiple
   denormalized copies of conceptually the same data, each shaped for one
   access pattern, is the normal and expected design approach rather than
   an anti-pattern.

4. **Index-free adjacency** means each node in a graph database stores
   direct physical references (pointers) to its connected neighbor nodes,
   as part of the node's own data — so traversing from a node to its
   neighbor is a direct pointer-follow, not a separate index lookup or
   join. In a relational database, the equivalent traversal (e.g.,
   "friend of a friend") requires a **join** for each hop — looking up an
   index, finding matching foreign key rows, repeating for each additional
   hop — and the cost/complexity compounds with each hop, especially for
   deep or variable-depth traversals (e.g., "find any path between A and
   B," where you don't know the depth in advance). The advantage is large
   for **deep, multi-hop traversal queries** — e.g., "find connections up
   to 4 hops away in a social network" or "find any path between two
   accounts for fraud detection" — where a relational query would need
   several chained joins (or a recursive query, which is often slow at
   scale) versus the graph database simply walking pointers hop by hop.
   The advantage **wouldn't matter much** for simple, shallow, one-hop
   lookups (e.g., "get this user's direct friends list") — a well-indexed
   relational join handles that just as efficiently as a graph traversal,
   since there's only one hop to resolve either way.

5. **(a) Time-based partitioning**: data is physically organized into
   chunks by time range, so a query for a recent time window only touches
   the relevant chunks instead of scanning the whole dataset, and entire
   old chunks can be dropped cheaply once past retention — this exploits
   the workload characteristic that time-series queries overwhelmingly
   filter by a recent or specific time range, and that data has a natural,
   predictable expiration/retention policy by age. **(b) Delta encoding /
   specialized compression**: consecutive timestamped values from the same
   series are often very close to each other (e.g., a slowly-changing
   sensor reading, or gradually incrementing counters), so storing only
   the *difference* from the previous value, rather than the full value
   each time, achieves much higher compression than general-purpose row
   storage — this exploits the workload characteristic that time-series
   data tends to have high temporal locality/similarity between
   consecutive points, which a general-purpose relational database's
   storage format doesn't specifically assume or exploit.

6. Brute-force nearest-neighbor search compares the query vector against
   *every single stored vector* to find the true nearest neighbors — an
   O(n) operation per query. At a billion vectors, this means a billion
   distance computations for every single search, which is far too slow
   for any interactive or high-QPS use case, and this cost scales linearly
   (gets worse) as the dataset grows, unlike a B-tree's logarithmic lookup
   cost (Module 04). The general trade-off vector search algorithms make
   instead (HNSW, IVF — detailed in Module 26) is **accepting approximate
   results** — the "A" in ANN (approximate nearest neighbor) — building an
   index structure that can find vectors *very likely* to be among the
   true nearest neighbors, but without an absolute guarantee of exact
   correctness, in exchange for dramatically faster (often sub-linear,
   effectively near-constant-ish in practice for a well-tuned index)
   query time — a direct instance of Module 00's general "you can often
   trade a small amount of correctness/precision for a large gain in
   speed" pattern, applied specifically to similarity search.

7. Before agreeing, you'd want to know: **(1) What's the natural shape of
   the data** — is it naturally wide, sparse rows with a clear partition
   key and a small number of well-defined query patterns, or does it need
   flexible ad-hoc querying/joins? **(2) What are the actual query
   patterns and their frequency** — is this genuinely a high-volume,
   narrow-query-pattern write workload (Cassandra's sweet spot), or does
   the feature need varied queries across different attributes that would
   force creating many denormalized tables, or worse, force inefficient
   full-cluster scans? **(3) What consistency does this data need** — can
   this feature tolerate eventual consistency (Module 11), or does it need
   strong transactional guarantees across multiple related pieces of data
   (favoring relational + ACID instead)? **(4) What's the actual write
   volume** — is it genuinely at a scale (tens/hundreds of thousands of
   writes/sec+) where a single well-tuned relational database with proper
   indexing and read replicas would actually struggle, or is "it scales"
   being invoked without a concrete number backing it up? You'd argue for
   a **plain relational database instead** if: the feature needs strong
   consistency across multiple related writes (e.g., financial
   transactions, inventory decrements tied to orders), the query patterns
   are varied/ad-hoc rather than a small fixed set known upfront, or the
   actual measured/projected write volume comfortably fits within what a
   well-indexed relational database and its read replicas can handle —
   "it scales" is not, by itself, a reason to accept the real costs
   Cassandra imposes (query-first modeling rigidity, weaker consistency
   guarantees, operational complexity) if the workload doesn't actually
   need that scale.

</details>

---

## What's next

**Module 08 — Caching: Patterns, Eviction, Redis, CDNs, Stampedes** covers
the layer that sits in front of *any* of these storage systems — how to
avoid hitting them at all for repeated reads, the specific failure modes
caching introduces (staleness, stampedes, cache-aside vs. write-through),
and how caching decisions connect back to the freshness-vs-efficiency
trade-off first introduced with DNS TTLs in Module 02.
