# Module 05 — Relational Databases & Data Modeling

> **TL;DR:** Module 04 covered how a database stores bytes on disk
> (B-trees/LSM-trees). This module covers how you organize your *data* on
> top of that storage engine so it's correct, non-redundant, and fast to
> query — the relational model, normalization, keys, joins, and the
> deliberate art of breaking those rules (denormalization) when read
> performance demands it. Data modeling mistakes are some of the most
> expensive to fix later, because they're load-bearing for everything
> built on top.

---

## 1. The relational model, briefly

A relational database organizes data into **tables** (relations), each
with a fixed set of **columns** (attributes) and any number of **rows**
(tuples/records). The power of the model comes from **relationships**
between tables, expressed via **keys**:

- **Primary key (PK)** — a column (or set of columns) that uniquely
  identifies each row in a table. Recall from Module 04 that most
  relational engines physically organize the table's B-tree around this
  key (a **clustered index**) — so choosing a primary key is not just a
  logical decision, it's a physical storage decision too.
- **Foreign key (FK)** — a column in one table that references a primary
  key in another, expressing a relationship (`orders.user_id` references
  `users.id`). The database can enforce **referential integrity**: it will
  reject an insert/update that would create a foreign key pointing to a
  non-existent row, and can be configured to cascade or restrict deletes
  of a referenced row.

> **⚠️ Trap:** Choosing a primary key that changes over time (e.g., a
> mutable "username" as PK instead of an immutable generated ID) causes
> real pain: every foreign key referencing it, and the clustered index
> physically organized by it, must be updated too. The standard practice
> — use an immutable, auto-generated ID (integer sequence or UUID) as the
> primary key, and enforce uniqueness on mutable business fields like
> username/email via a separate unique constraint/index — exists
> specifically to avoid this.

---

## 2. Normalization: organizing data to eliminate redundancy

**Normalization** is the discipline of structuring tables so that each
fact is stored in exactly one place, avoiding **update anomalies** (the
same fact stored in multiple rows, some of which get updated and others
forgotten, leaving inconsistent data). It's expressed as a series of
**normal forms**, each stricter than the last.

### 2.1 The normal forms that matter in practice

- **1NF (First Normal Form):** every column holds a single, atomic value —
  no comma-separated lists or nested structures crammed into one field.
  (`tags: "red,blue,green"` in one column violates 1NF; a separate
  `tags` table with one row per tag violates nothing.)
- **2NF:** (applies to tables with composite primary keys) every
  non-key column must depend on the *entire* primary key, not just part
  of it.
- **3NF:** every non-key column must depend **only** on the primary key —
  not on another non-key column. (If `orders` has both `user_id` and
  `user_email`, and `user_email` really belongs to the `users` table, then
  storing it redundantly in `orders` violates 3NF — if a user changes
  their email, every order row also needs updating, or you get
  inconsistent data.)

In practice, **"normalize to 3NF, then denormalize deliberately where
performance demands it"** is the standard working philosophy — few
production schemas go further into stricter normal forms (BCNF, 4NF), and
knowing 3NF by name and example is enough for the vast majority of design
conversations, interview or real.

> **🎤 Interview Angle:** You don't need to recite normal-form definitions
> verbatim. What signals real understanding is recognizing an update
> anomaly when you see one: "if this field is duplicated across many rows,
> what happens when the real-world fact it represents changes?" is the
> question underlying all of normalization, and articulating *that*
> question is more valuable than naming "3NF" correctly.

### 2.2 Why normalization has a cost

Normalized data is spread across multiple tables, which means retrieving
a complete "view" of something (e.g., an order with its user's name and
each line item's product details) requires **joins** (Section 3) —
correctness and non-redundancy are gained at the cost of read-time
assembly work. This tension — normalize for write-side correctness vs.
denormalize for read-side speed — is this module's central trade-off, and
a direct instance of Module 00's "read optimization vs. write
optimization" axis.

---

## 3. Joins: how relational databases reassemble normalized data

A **join** combines rows from two (or more) tables based on a related
column, most commonly the FK/PK relationship discussed above.

### 3.1 How a join actually executes (tying back to Module 04)

At a physical level, a join is not magic — the query planner picks a
strategy, commonly one of:

- **Nested loop join:** for each row in table A, scan/look up matching
  rows in table B (fast if B has an index on the join column — this
  becomes B-tree lookups per row, i.e., Module 04's ~3-4-disk-read cost,
  *times* the number of rows in A; slow without an index, since each
  lookup degrades toward a full scan of B).
- **Hash join:** build an in-memory hash table from the smaller table,
  then scan the larger table probing the hash table for matches. Often
  fast for large, unsorted inputs without a usable index.
- **Merge join:** if both inputs are already sorted (or become sorted via
  their index) on the join column, merge them in one linear pass — very
  efficient, but requires that sortedness.

> **⚠️ Trap:** "Joins are slow" is an oversimplification that causes
> people to prematurely denormalize. A join on properly indexed columns
> (Module 04's secondary indexes) can be very fast — the real danger is an
> *unindexed* join column forcing the planner into an expensive strategy.
> The correct diagnostic question isn't "should I avoid joins?" but "is
> the join column indexed, and does the query plan reflect that?" —
> checking a query's actual execution plan (`EXPLAIN` in most SQL
> databases) is the concrete way to answer this, rather than guessing.

### 3.2 The N+1 query problem, revisited

Module 03 introduced the N+1 problem in the context of GraphQL resolvers,
but it originates here: an application layer that fetches a list of N
orders, then issues N *separate* queries (one per order) to fetch each
order's user, instead of a single query with a `JOIN` (or a single batched
`WHERE user_id IN (...)` query), pays for N round trips to the database
that a single join-based query would have avoided entirely. This is a data
*modeling and querying* discipline issue as much as an API design one —
recognizing it here, at the database layer, is where the fix (write
proper joins/batched queries) actually happens.

---

## 4. Denormalization: the deliberate trade-off

**Denormalization** is intentionally introducing redundancy — storing the
same fact in multiple places — to avoid join costs on the read path, in
exchange for accepting more complex/costly writes (every place the
redundant fact lives must be kept in sync) and the *reintroduction* of the
update-anomaly risk normalization was designed to prevent.

### 4.1 When denormalization is the right call

- **Read-heavy, write-light workloads** where the same joined "view" is
  queried extremely often (Module 00's read/write optimization trade-off,
  concretely applied) — e.g., a product listing page that always needs
  product name + category name + current price together; storing a
  precomputed, denormalized `product_listing` table/view can be far
  cheaper than joining 3 tables on every single page load.
- **When strict consistency between the duplicated fields isn't critical**
  or can tolerate eventual correction (e.g., a display name cached
  alongside a foreign key, corrected on the next write/sync, rather than
  required to be instantaneously perfectly accurate).

### 4.2 The mechanisms used to keep denormalized data in sync

- **Application-level dual writes** — the application writes to both the
  normalized source of truth and the denormalized copy in the same
  request. Simple, but risky: if one write succeeds and the other fails,
  the copies diverge (this is exactly the kind of problem Module 13's
  distributed transaction patterns, like the outbox pattern, exist to
  address properly).
- **Database triggers** — the database itself automatically updates a
  denormalized copy whenever the source table changes. Keeps logic
  centralized in the DB, but can be harder to observe/debug and adds
  write-path overhead directly inside the transaction.
- **Materialized views** — a query result physically stored and
  periodically (or trigger-driven) refreshed, rather than recomputed on
  every read. This is the formalized, database-native version of manual
  denormalization, and reappears in Module 23 (OLAP/data warehousing)
  as a first-class technique.
- **Async propagation via CDC/event streams** — changes to the source
  table are captured (Change Data Capture) and asynchronously applied to
  denormalized copies elsewhere, accepting eventual consistency
  explicitly rather than pretending otherwise. This connects directly to
  Module 16 (CDC) and Module 09's eventual consistency concepts.

> **🤖 AI/ML Callout:** Feature stores in ML systems (Module 24) are, at
> their core, a large-scale, formalized denormalization pattern: raw
> normalized business data (users, transactions, events) is
> precomputed into flat, ready-to-serve feature vectors so that a model
> serving a real-time prediction doesn't have to perform expensive joins
> and aggregations at inference time — it reads one precomputed row. The
> "keep it in sync" problem from Section 4.2 reappears there as **training/
> serving skew**: if the batch pipeline computing training features and
> the online pipeline computing serving-time features drift out of sync,
> the model sees different data at training vs. inference time, and
> accuracy silently degrades — the same fundamental risk as any denormalized
> copy falling out of sync with its source of truth, just with a
> model-accuracy consequence instead of a data-correctness one.

> **📐 Numbers:** A join across 3-4 well-indexed tables for a single row
> might cost a handful of extra B-tree lookups (Module 04) — often still
> sub-millisecond. Denormalization typically only becomes clearly worth
> its complexity cost once you're doing that join **many thousands of
> times per second**, or the join pattern is not well-served by any
> practical index (e.g., full-text search across joined tables, or
> aggregation across millions of rows) — reach for denormalization as a
> targeted optimization backed by a measured bottleneck, not a default
> starting design.

---

## 5. Data modeling for access patterns: model the queries, not just the entities

A subtle but important shift, especially relevant once you get into NoSQL
(Module 07): relational modeling with normalization tends to start from
"what are the real-world entities and their relationships?" (an
entity-relationship, or **ER modeling**, approach). This is the right
starting point for a normalized relational schema, but production system
design ultimately also needs to ask: **"what are the actual queries this
system needs to serve, and how often?"**

- If a `GET /users/:id/dashboard` endpoint is hit 10,000 times/sec and
  needs data from 5 joined tables, that's a strong, measured signal to
  consider a denormalized or materialized read path specifically for that
  query, even while keeping the underlying normalized schema as the
  source of truth for writes.
- This "model for the query" mindset is the conceptual bridge to Module 07
  (NoSQL), where many databases (wide-column stores, document stores)
  explicitly *require* you to design your schema around access patterns
  from the start, rather than normalizing first and denormalizing later
  as an optimization.

> **🎤 Interview Angle:** A strong system design answer typically does
> both: sketch the normalized ER model first (to show you understand the
> real entities/relationships and get correctness right), *then*
> explicitly call out which specific read paths are hot enough to warrant
> denormalization, a cache (Module 08), or a separate read-optimized store
> — rather than picking one philosophy and applying it uniformly
> everywhere.

---

## Cheat Sheet (for fast revision)

- Primary key = unique row identifier, usually the clustered index
  (physical storage order) — prefer immutable generated IDs, not mutable
  business fields.
- Foreign key = enforces referential integrity between tables.
- Normalization (up to 3NF in practice): each fact stored once, avoiding
  update anomalies where a duplicated fact goes stale in some rows but not
  others.
- Normalization costs read-time assembly (joins); this is Module 00's
  read/write trade-off, concretely.
- Joins aren't inherently slow — an *unindexed* join column is what makes
  them slow (nested loop degrades to near-full-scan). Check `EXPLAIN`
  before assuming a join is the bottleneck.
- The N+1 query problem originates at the database/query layer: N separate
  per-row queries instead of one join or one batched `IN (...)` query.
- Denormalization = deliberate redundancy to avoid read-time joins, paid
  for with write-side complexity (keeping copies in sync) and reintroduced
  update-anomaly risk. Sync mechanisms: dual writes (risky), triggers,
  materialized views, async CDC/event propagation (explicit eventual
  consistency).
- Denormalize as a targeted, measured optimization for a specific hot
  read path — not a default starting design.
- Model both the entities (normalized ER model, for correctness) and the
  actual hot query patterns (for performance) — production schemas
  usually need both perspectives.

---

## Quiz

1. Why is using a mutable field (like username) as a primary key
   problematic, specifically in terms of what it's physically tied to in
   the storage engine?
2. Give a concrete example of an "update anomaly" that 3NF prevents —
   describe a denormalized table design where updating one real-world
   fact requires updating multiple rows, and explain what goes wrong if
   you miss one.
3. "Joins are slow, so avoid them and denormalize instead" — explain what's
   wrong with this as a blanket rule, and state the actual diagnostic
   question you should ask before denormalizing.
4. Explain where the N+1 query problem actually originates (which layer),
   and how a single well-indexed join or batched query fixes it.
5. You denormalize a `product_name` field into an `order_items` table
   (alongside the real source of truth in a `products` table) so that
   listing past orders doesn't require a join. Name two different
   mechanisms you could use to keep `product_name` in sync when a
   product's name changes, and state one risk of the simplest one.
6. A service serves a hot `GET /users/:id/dashboard` endpoint 10,000
   times/sec, assembling data from 5 normalized tables via joins. Would
   you recommend denormalizing this path immediately, or is there a step
   you should take first? Justify.
7. Connect this module's denormalization-sync problem to one concept from
   the AI/ML callout — what is the ML-specific name for "the denormalized
   copy and the source of truth drifting out of sync," and why does it
   matter for model accuracy specifically (not just data correctness)?

<details>
<summary><b>Answers — click to expand</b></summary>

1. Most relational engines physically organize the table's B-tree around
   the primary key — this is the **clustered index** (Module 04): rows are
   stored on disk in primary-key order. If the primary key is mutable
   (like a username that can be changed), then changing it isn't just a
   logical value update — it can require physically relocating the row's
   position within the clustered index/B-tree, and every foreign key in
   other tables referencing that primary key must also be updated to point
   to the new value, potentially cascading widely. An immutable,
   auto-generated ID avoids all of this — the physical storage position
   and every foreign-key reference stay stable for the row's entire
   lifetime.

2. Example: an `orders` table stores both `user_id` and `user_email`
   directly on every order row (denormalized/duplicated), instead of only
   storing `user_id` and looking up email via a join to `users` when
   needed. If a user changes their email address, the application must
   remember to update `user_email` on *every single order row* that user
   has ever placed. If even one order row is missed (a bug, a partial
   failure, an overlooked batch job), that row now shows a stale,
   incorrect email — the same real-world fact ("this user's email") now
   has two different stored values in the database simultaneously, with no
   indication which one is correct without manually cross-referencing
   against `users`. 3NF prevents this by requiring `user_email` be stored
   only once, in `users`, and always retrieved via a join/lookup rather
   than duplicated.

3. The blanket rule is wrong because joins are not inherently expensive —
   a join on a properly **indexed** column can resolve via fast B-tree
   lookups (Module 04), often sub-millisecond, and the query planner can
   choose an efficient strategy (merge join, indexed nested loop). What
   actually makes a join slow is an **unindexed** join column, forcing the
   planner into a near-full-table-scan strategy for every row being
   joined. The correct diagnostic question before reaching for
   denormalization is: "is the join column properly indexed, and does the
   query's actual execution plan (`EXPLAIN`) confirm it's using that
   index efficiently?" — not "should I avoid joins in general?"

4. The N+1 problem originates at the **database/query layer**: it happens
   when application code fetches a list of N parent records (e.g., N
   orders), then issues N *separate* follow-up queries — one per row — to
   fetch each row's related data (e.g., each order's user), instead of
   retrieving everything needed in a single query. A single well-indexed
   `JOIN` (or a single batched `WHERE user_id IN (id1, id2, ..., idN)`
   query) fetches all the needed related data in one round trip to the
   database, collapsing what would have been N+1 separate queries (and
   N+1 separate network round trips to the DB) into just 1 — this is the
   database-layer root cause of the same pattern Module 03 discussed
   reappearing inside GraphQL resolvers.

5. Two mechanisms: **(a) database triggers** — a trigger on the `products`
   table automatically updates the corresponding `product_name` value in
   every affected `order_items` row whenever a product's name changes,
   keeping the logic centralized in the database itself. **(b) Async
   propagation via CDC/event streams** — a change to `products.name` is
   captured as a change event and asynchronously applied to update
   `order_items.product_name` via a background consumer, explicitly
   accepting eventual (not immediate) consistency between the two. The
   simplest alternative not listed above, **application-level dual
   writes** (the app writes to both `products` and `order_items` in the
   same request), carries the risk that if one of the two writes succeeds
   and the other fails (e.g., a crash or network error between them), the
   two copies silently diverge with no automatic mechanism to detect or
   correct it — exactly the kind of partial-failure problem Module 13's
   outbox pattern is designed to solve properly.

6. Denormalizing immediately is premature. The correct first step is to
   **measure**: confirm the joins are actually the bottleneck (check the
   query's `EXPLAIN` plan, ensure all join columns are properly indexed
   per Question 3's diagnostic), and consider cheaper fixes first — proper
   indexing, or introducing a **cache** (Module 08) in front of this
   specific hot read path, which can often absorb the vast majority of the
   10,000/sec load without touching the schema at all. Denormalization
   (or a materialized view) becomes the right call specifically once
   you've confirmed indexing/caching alone can't hit the required latency
   or the query pattern is fundamentally not join-friendly (e.g., heavy
   cross-table aggregation) — Section 4.1's guidance is to treat
   denormalization as a targeted, measured optimization, not a default
   reflex the moment a path looks "hot."

7. The ML-specific name is **training/serving skew** (mentioned in the
   Module's AI/ML Callout, expanded in Module 24). It matters for model
   accuracy specifically — not just data correctness — because a machine
   learning model's predictions are only as reliable as the assumption
   that the data it sees at inference time statistically resembles the
   data it was trained on; if the batch pipeline computing training
   features and the online/real-time pipeline computing serving-time
   features drift out of sync (the same "denormalized copy diverges from
   source of truth" failure mode as Question 5), the model silently starts
   making predictions on data that doesn't match what it learned patterns
   from — degrading accuracy in a way that, unlike a stale `product_name`
   field, often produces no obvious error or exception, just quietly worse
   predictions that can go unnoticed without dedicated monitoring.

</details>

---

## What's next

**Module 06 — Transactions: ACID, Isolation Levels, MVCC, Pessimistic/
Optimistic Locking** builds directly on this module's relational
foundation to cover what happens when multiple operations touch related,
normalized data *concurrently* — how databases guarantee correctness
without serializing every operation one at a time.
