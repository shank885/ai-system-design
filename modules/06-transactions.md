# Module 06 — Transactions: ACID, Isolation Levels, MVCC, Pessimistic/Optimistic Locking

> **TL;DR:** Module 01 introduced race conditions between threads on one
> machine. A database transaction is the same fundamental problem —
> concurrent operations touching shared, related data — solved at the
> database layer, with formal guarantees instead of ad-hoc locks. This
> module covers what those guarantees (ACID) actually mean, the specific
> anomalies that show up when they're relaxed for performance (isolation
> levels), and the two dominant mechanisms — MVCC and locking — databases
> use to deliver them. This is one of the most commonly misunderstood
> areas in system design interviews, because "consistency" and
> "isolation" get used loosely; this module fixes that with precision.

---

## 1. ACID: the four guarantees, precisely

A **transaction** groups multiple operations into one logical unit that
either fully happens or doesn't happen at all. **ACID** names the four
guarantees a transactional database provides:

- **Atomicity** — a transaction's operations all succeed, or all are
  rolled back as if none happened. No partial transaction is ever visible.
  Achieved via the WAL/undo-log mechanisms from Module 04 — if a
  transaction fails partway, the database uses the log to undo whatever
  partial changes it already made.
- **Consistency** — a transaction takes the database from one valid state
  to another valid state, respecting all defined constraints (foreign
  keys, unique constraints, application-defined invariants). Note: this is
  the **ACID sense** of "consistency" — narrower and different from CAP's
  sense (Module 11) or the "is the data fresh/correct" sense from Module
  00. This is genuinely the single most overloaded word in this
  curriculum; always know which sense is meant.
- **Isolation** — concurrent transactions behave as if they ran one at a
  time (serially), even though the database may actually interleave them
  for performance. *How strictly* this is enforced is a spectrum, covered
  in Section 2 — this is the guarantee most worth understanding deeply,
  because it's the one most commonly relaxed in practice.
- **Durability** — once a transaction commits, its changes survive a crash
  (the same durability concept from Module 00/01/04 — concretely, the
  commit isn't acknowledged until the relevant WAL entries are fsync'd).

> **⚠️ Trap:** People say "the database is ACID" as if it's binary, but
> **Isolation** in particular is configurable, and most databases don't
> default to the strictest level (full **Serializable** isolation) because
> it costs performance. "Is this database ACID?" is a much less useful
> question than "what isolation level is this transaction running at, and
> which anomalies does that permit?" — Section 2 gives you the vocabulary
> to ask the better question.

---

## 2. Isolation levels and the anomalies they permit

Running transactions fully serially (one at a time) would be perfectly
isolated but terrible for throughput — this is the same
concurrency-vs-correctness tension from Module 01, now at the database
layer. Isolation levels are a spectrum of how much interleaving is allowed
in exchange for performance, each one defined by *which anomalies it
still permits*.

### 2.1 The anomalies, defined precisely

- **Dirty read:** Transaction A reads data written by Transaction B, which
  hasn't committed yet — if B later rolls back, A read data that never
  "really" existed.
- **Non-repeatable read:** Transaction A reads the same row twice, and gets
  different values, because Transaction B committed an update to that row
  in between A's two reads.
- **Phantom read:** Transaction A runs the same query twice (e.g., "count
  all orders over $100"), and gets a different *set of rows* the second
  time, because Transaction B committed an insert/delete matching that
  query's criteria in between.
- **Lost update:** Transactions A and B both read the same value, both
  compute a new value based on it, and both write back — one of the
  updates is silently overwritten/lost. (This is exactly Module 01's
  race-condition example — two threads incrementing a counter — now
  happening between two database transactions instead of two threads.)
- **Write skew:** A subtler anomaly where two transactions each read
  overlapping data, and each makes a decision (and writes) based on what
  they read, and each individual write is valid on its own — but the
  *combination* of both violates an invariant neither transaction could
  see the other violating. (Classic example: a hospital rule "at least one
  doctor must be on call" — two doctors, both on call, both independently
  check "is at least one *other* doctor on call?", both see yes, both go
  off call — now zero doctors are on call, even though neither
  transaction did anything individually wrong.)

### 2.2 The four standard isolation levels

| Level | Dirty Read | Non-repeatable Read | Phantom Read | Notes |
|---|---|---|---|---|
| **Read Uncommitted** | Possible | Possible | Possible | Rarely used in practice; almost no real guarantee |
| **Read Committed** | Prevented | Possible | Possible | Common default (PostgreSQL, Oracle) — each individual read sees only committed data, but re-reading can still see a different value |
| **Repeatable Read** | Prevented | Prevented | Possible in theory (MySQL/InnoDB's implementation actually prevents most phantoms too, via gap locking — a detail worth knowing exists) | MySQL/InnoDB's default |
| **Serializable** | Prevented | Prevented | Prevented | Strongest — behaves as if transactions ran one at a time; highest cost |

> **⚠️ Trap:** Even **Serializable** isolation doesn't automatically save
> you from **write skew** in every database's actual implementation —
> true serializability prevents it by definition, but some databases
> implement "serializable" via mechanisms (like snapshot isolation plus
> extra checks) that have historically had edge cases. The practical
> takeaway: for invariants spanning multiple rows or tables (like the
> on-call doctor example), don't assume isolation level alone protects
> you — verify with your specific database's documentation, or enforce the
> invariant via an explicit constraint/lock rather than relying purely on
> isolation semantics.

> **🎤 Interview Angle:** If a system design question involves any
> "read a value, compute something, write it back" pattern under
> concurrency (inventory counts, account balances, seat reservations),
> naming the specific anomaly at risk (usually **lost update**, sometimes
> **write skew**) and the specific isolation level or locking mechanism
> that prevents it is exactly the kind of precision that separates a
> strong answer from a vague "we'll use a transaction" answer.

---

## 3. MVCC: how modern databases achieve isolation without blocking everyone

**MVCC (Multi-Version Concurrency Control)** — used by PostgreSQL,
MySQL/InnoDB, and most modern relational databases — is the dominant
mechanism for delivering strong isolation without simply blocking all
concurrent readers and writers against each other.

### 3.1 The core idea

Instead of a single, current value for each row, the database keeps
**multiple versions** of a row as it's updated over time (each update
creates a new version rather than overwriting in place, at least
logically — old versions are cleaned up later by a background process,
e.g., PostgreSQL's **VACUUM**). Each transaction, when it starts, gets a
**consistent snapshot** — effectively "the state of the database as of the
moment this transaction began" — and reads from that snapshot throughout
its lifetime, regardless of what other transactions commit in the
meantime.

### 3.2 Why this is powerful

- **Readers never block writers, and writers never block readers** — a
  read transaction just reads its own snapshot's version of each row; a
  concurrent write creates a *new* version without needing to lock the
  row against the reader. This is a massive practical win over naive
  locking, where a long-running read could otherwise stall writes (and
  vice versa) for its entire duration.
- This directly explains **Read Committed** and **Repeatable Read**
  from Section 2.2: Read Committed takes a *new* snapshot for every
  individual statement within a transaction (so it can see other
  transactions' commits between statements — enabling non-repeatable
  reads); Repeatable Read takes *one* snapshot for the whole transaction
  (so every read within it sees the same consistent point-in-time view —
  preventing non-repeatable reads).

> **🤖 AI/ML Callout:** MVCC's "give every reader a consistent
> point-in-time snapshot without blocking writers" pattern is
> conceptually close to a recurring need in ML feature stores (Module 24):
> a model training job needs a **consistent, point-in-time snapshot** of
> features as they existed at each historical training example's
> timestamp (to avoid "future" data leaking into training — a serious,
> common bug called **label/feature leakage**), while the online feature
> pipeline keeps writing new feature values concurrently. The
> point-in-time-correct join techniques used in feature stores are solving
> a snapshot-isolation-shaped problem, just applied to historical feature
> retrieval rather than live transaction reads.

---

## 4. Pessimistic vs. optimistic locking

MVCC handles read/write isolation elegantly, but concurrent **writes** to
the *same* row still need a resolution strategy — two approaches, with a
real trade-off between them.

### 4.1 Pessimistic locking

Acquire a lock on a row **before** reading/modifying it (`SELECT ... FOR
UPDATE` in SQL), preventing any other transaction from touching that row
until the lock is released (transaction commits/rolls back). Other
transactions wanting the same row simply **wait**.

- **Pros:** Guarantees no conflict ever happens — simple to reason about.
- **Cons:** Reduces concurrency (other transactions wait, even if they
  might not have actually conflicted), and introduces **deadlock risk**
  (the same risk from Module 01's mutex discussion, now between database
  transactions instead of threads — Transaction A waits for a lock B
  holds, while B waits for a lock A holds).

### 4.2 Optimistic locking

Don't lock anything upfront. Read the row (often including a **version
number** or timestamp column), do your work, then when writing back,
check: "is the version/timestamp still what I read?" If yes, commit the
write and increment the version. If no (someone else modified it in the
meantime), **reject the write** and let the application retry.

- **Pros:** No blocking, no deadlock risk, higher concurrency when
  conflicts are actually rare.
- **Cons:** Under high contention (many transactions frequently
  conflicting on the same rows), you get **repeated wasted work and
  retries**, which can be worse than pessimistic locking would have been.

### 4.3 The decision

| Choose Pessimistic when... | Choose Optimistic when... |
|---|---|
| Conflicts are frequent/likely on the same data | Conflicts are rare (most transactions touch different rows, or the same row infrequently) |
| The cost of retrying a failed transaction is high (e.g., complex multi-step business logic) | Retries are cheap and simple |
| You need a guarantee, not just a check | High-read, low-write-contention workloads |

> **📐 Numbers:** This is a direct real-world instance of the throughput-
> under-contention lesson from Module 01, Section 5.2: locks/mutexes don't
> scale linearly under heavy contention because of waiting/context-switch
> overhead. Pessimistic database locks have the identical shape of cost —
> the more transactions contend for the same rows, the more waiting
> (reduced throughput) you pay, exactly mirroring why "more threads on a
> lock-heavy critical section" doesn't scale.

> **🎤 Interview Angle:** A classic scenario: "design a concert ticket
> booking system where two users might try to book the last seat
> simultaneously." This is a textbook **lost update** risk under high
> contention on a *specific*, popular row (the last seat) — a strong
> answer recognizes this is exactly the frequent-conflict case favoring
> **pessimistic locking** (`SELECT ... FOR UPDATE` on that seat row) rather
> than optimistic locking, which would just produce a storm of failed
> retries all fighting over the same hot row. Contrast this with, say,
> updating a user's own profile — conflicts on any single user's row are
> rare (only that user typically writes to it), so optimistic locking
> fits better there.

---

## Cheat Sheet (for fast revision)

- ACID: Atomicity (all-or-nothing, via WAL/undo logs), Consistency
  (constraints respected — ACID-specific meaning, distinct from CAP's or
  the everyday sense), Isolation (concurrent transactions act as if
  serial — configurable spectrum), Durability (commit survives crash, via
  fsync'd WAL).
- Anomalies, weakest to strongest guarantee needed to prevent: dirty read
  → non-repeatable read → phantom read → lost update / write skew (the
  latter two need explicit attention even at strong isolation levels).
- Isolation levels: Read Uncommitted (almost no guarantee) → Read
  Committed (no dirty reads; common default) → Repeatable Read (no dirty
  or non-repeatable reads; MySQL default) → Serializable (strongest, acts
  fully serial, highest cost).
- MVCC: multiple row versions + per-transaction consistent snapshot lets
  readers and writers never block each other; explains why Read Committed
  (new snapshot per statement) vs. Repeatable Read (one snapshot per
  transaction) behave differently.
- Pessimistic locking: lock upfront, others wait; safe but reduces
  concurrency and risks deadlock — best under frequent contention on the
  same data.
- Optimistic locking: check a version/timestamp at write time, reject and
  retry on conflict; no blocking, but wasteful under high contention —
  best when conflicts are rare.
- Lost update = Module 01's race condition, replayed at the database
  transaction layer. Same root cause (non-atomic read-modify-write),
  different scale.

---

## Quiz

1. Explain, precisely, why "the database is ACID" is a less useful
   statement than naming the isolation level a specific transaction runs
   at. What guarantee varies the most across isolation levels?
2. Give a concrete example (different from the ones in this module) of a
   **lost update**, and name the exact isolation level (from Section 2.2)
   that would prevent it.
3. Explain why write skew can occur even when neither individual
   transaction violates any constraint it can see — what makes this
   anomaly fundamentally different from a lost update?
4. Explain how MVCC allows a long-running read transaction and a
   concurrent write transaction to both proceed without blocking each
   other, and connect this to the difference between Read Committed and
   Repeatable Read.
5. A ticket-booking system has 10,000 users trying to book 50,000
   different unique event seats (low contention per seat, since most
   seats are only contended by a handful of users at once) versus a flash
   sale with 10,000 users all trying to buy the same 10 units of one
   specific item (extremely high contention on those 10 rows). Which
   locking strategy fits each scenario, and why would applying the wrong
   one to the flash-sale case be a problem?
6. Connect this module's "lost update" anomaly to a specific concept from
   Module 01, by name, and explain what's identical about the underlying
   root cause despite happening at different layers of the stack.
7. What's the ML-relevant analog to MVCC's "consistent point-in-time
   snapshot for a reader" concept, and what specific bug does it prevent
   in that context?

<details>
<summary><b>Answers — click to expand</b></summary>

1. "The database is ACID" implies a fixed, binary guarantee, but in
   practice **Isolation** is configurable via isolation levels, and most
   databases don't default to the strongest one (Serializable) for
   performance reasons — so two transactions on the same "ACID" database
   can have meaningfully different real guarantees against concurrency
   anomalies depending on which isolation level they're running at.
   Atomicity and Durability are typically non-negotiable guarantees the
   database always provides; Consistency (ACID sense) is enforced via
   constraints regardless of isolation level. **Isolation is the
   guarantee that varies the most** across levels — it's a genuine
   spectrum (Read Uncommitted through Serializable), each point on it
   permitting a different, specific set of anomalies, which is why naming
   the actual isolation level in use is far more informative than the
   blanket claim "it's ACID."

2. Example: two customer support agents both view the same support
   ticket's "status" and independently decide to update unrelated fields
   but both also happen to reset a shared `last_updated_by` counter/field
   based on the value they read — or more classically: two bank tellers
   both read an account balance of $100, one processes a $30 withdrawal
   (computing new balance $70 based on the $100 they read) and writes
   $70; the other processes a $50 deposit (computing new balance $150
   based on the *same* stale $100 they read) and writes $150 — whichever
   write happens second overwrites the other, and the account ends up
   reflecting only one of the two operations instead of both ($70 or $150
   instead of the correct $120). **Serializable** isolation (or, in
   practice, an explicit locking mechanism like `SELECT ... FOR UPDATE`)
   prevents this — Repeatable Read alone does not automatically prevent
   lost updates in all database implementations, since the anomaly
   depends on the specific read-then-write pattern, not just repeatable
   reads of an unchanged value; this is exactly why many production
   systems use explicit row locking or optimistic version checks for
   this specific pattern rather than relying on isolation level alone.

3. In a lost update, one transaction's write is silently and directly
   overwritten by another's — a clear, single-value conflict on the exact
   same piece of data. In write skew, each transaction reads a *set* of
   data (not necessarily the exact row the other transaction will modify),
   makes a decision based on what it individually observed, and writes to
   a *different* row/value than the other transaction — so from either
   transaction's local point of view, nothing was overwritten and no
   individual write conflicts with the other's write. What makes it a
   genuine anomaly is that a **cross-transaction invariant** (e.g., "at
   least one doctor on call," spanning both doctors' rows) is violated by
   the *combination* of both transactions' otherwise-individually-valid
   decisions — neither transaction could observe, from its own read, that
   the other was about to make the invariant-violating decision too. This
   is fundamentally different from a lost update because no single value
   was "lost" — every write succeeded and is individually consistent with
   what that transaction read; the violated correctness lives at a level
   above any single row.

4. MVCC keeps multiple versions of each row as it's updated over time,
   rather than overwriting a row's single current value in place. A
   read transaction, instead of needing to lock rows against concurrent
   writers, is simply given a consistent snapshot to read from — when a
   writer commits a change, it creates a *new* version of the row rather
   than mutating the version the reader's snapshot is looking at, so the
   reader's long-running transaction keeps seeing its own consistent view
   throughout, completely unaffected by (and not blocking) the writer's
   concurrent commit. This directly explains the difference between the
   two isolation levels: **Read Committed** takes a *fresh* snapshot for
   every individual statement within the transaction, so a later statement
   in the same transaction can see a different (newer, since-committed)
   version of a row than an earlier statement did — allowing
   non-repeatable reads. **Repeatable Read** takes just *one* snapshot at
   the start of the whole transaction and keeps reading from that same
   snapshot for every statement within it, so every read within the
   transaction sees the exact same point-in-time view regardless of what
   commits happen concurrently elsewhere — preventing non-repeatable
   reads.

5. The low-contention seat-booking scenario (50,000 largely
   independently-contended seats) fits **optimistic locking** well:
   conflicts on any specific seat are rare (most seats aren't being fought
   over by many users simultaneously), so the "check version at write
   time, retry on conflict" approach rarely triggers a retry and avoids
   the overhead/reduced-concurrency cost of locking rows upfront that
   usually wouldn't have conflicted anyway. The flash-sale scenario
   (10,000 users contending for only 10 units, i.e., 10 rows/or one
   heavily-contended inventory counter) fits **pessimistic locking**
   better: conflicts here are the norm, not the exception, on that small
   set of hot rows. Applying optimistic locking to the flash-sale case
   would be a problem because nearly every transaction would read a
   version, do its work, then find at write time that the version already
   changed (since thousands of others are hitting the same few rows
   concurrently) — producing a storm of failed writes and retries, most of
   which fail *again* on the retry too, wasting enormous work for very
   little successful throughput; pessimistic locking's upfront serialization
   of access to those specific hot rows is actually the more efficient
   choice precisely because contention is the expected, common case there.

6. This is **Module 01's race condition**, specifically the exact
   "two threads increment a shared counter without a lock" example. The
   identical root cause: a **non-atomic read-modify-write sequence**
   (read a value, compute something based on it, write it back) is
   interrupted by another concurrent actor performing the same sequence
   in between the read and the write, causing one of the two updates to
   be silently overwritten/lost. At the OS/threading layer (Module 01)
   the concurrent actors are threads sharing memory; at the database
   layer (this module) the concurrent actors are transactions sharing
   rows — but the underlying failure mode, an operation that isn't a
   single atomic step being corrupted by uncontrolled interleaving, is
   exactly the same problem recurring at a different layer of the stack,
   just as Module 00 first flagged non-idempotency as this same root
   cause showing up in network retries too.

7. The ML-relevant analog is **point-in-time-correct feature retrieval**
   in a feature store (Module 24): giving a model training pipeline a
   consistent, "as of this historical timestamp" view of feature values,
   even while the online feature pipeline keeps writing newer values
   concurrently — structurally the same problem MVCC solves (a consistent
   snapshot for a reader, undisturbed by concurrent writers), just applied
   to historical feature lookups instead of live transactional reads. The
   specific bug it prevents is **label/feature leakage** — where a
   training example accidentally uses feature values that were only
   available *after* the point in time that example represents (i.e., the
   model effectively "sees the future" during training), which produces
   a model that looks artificially accurate during training/evaluation
   but performs far worse in real, live inference, where those future
   values genuinely aren't available yet.

</details>

---

## What's next

**Module 07 — The NoSQL Landscape: KV, Document, Wide-Column, Graph,
Time-Series, Vector** shifts from the relational/transactional model this
module and Module 05 covered, into the family of databases that
deliberately relax some of these guarantees (often isolation and
sometimes even full ACID) in exchange for horizontal scalability and
schema flexibility — and how to decide which NoSQL family actually fits a
given access pattern.
