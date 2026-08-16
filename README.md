# System Design 101 — From Level 0 to Advanced

A complete, self-paced curriculum in system design, written as a durable archive.
Every module is a standalone deep-dive with theory, worked numbers, diagrams,
a quiz, and a Q&A section.

**Audience:** engineers who can code but have never been taught distributed
systems formally. Starts at absolute zero (what is a server?) and ends at
designing planet-scale and AI/ML inference systems.

**Method:** classical (non-AI) examples are used to teach fundamentals, because
they are the cleanest teaching vehicles. Where a concept maps onto ML/LLM
infrastructure, a `🤖 AI/ML Callout` section makes the connection explicit.
Part VII is dedicated entirely to AI systems.

---

## How to use this repo

| If you want to... | Do this |
|---|---|
| Learn from scratch | Read modules in order. Do not skip Part I. |
| Revise before an interview | Read the `TL;DR` + `Cheat Sheet` at the top/bottom of each module |
| Self-test | Each module ends with a quiz; answers are in `<details>` blocks |
| Look up a term | See [glossary.md](glossary.md) |
| Estimate capacity | See [cheatsheets/numbers.md](cheatsheets/numbers.md) |
| Read real-world engineering write-ups alongside a module | See [cheatsheets/further-reading.md](cheatsheets/further-reading.md) |

Progress tracker: [PROGRESS.md](PROGRESS.md)

---

## Curriculum

### Part 0 — Foundations of Thinking
| # | Module | Status |
|---|---|---|
| 00 | [Mental Models, Vocabulary & Back-of-the-Envelope Math](modules/00-foundations.md) | ✅ |

### Part I — The Machine and The Network
| # | Module | Status |
|---|---|---|
| 01 | [Inside One Machine: CPU, Memory, Disk, OS, Concurrency](modules/01-inside-one-machine.md) | ✅ |
| 02 | [Networking for System Designers: IP, TCP/UDP, DNS, TLS, HTTP/1-2-3](modules/02-networking.md) | ✅ |
| 03 | [API Design & Communication Styles: REST, RPC, gRPC, GraphQL, WebSockets](modules/03-api-design.md) | ✅ |

### Part II — Storage
| # | Module | Status |
|---|---|---|
| 04 | [Storage Engine Internals: B-Trees, LSM-Trees, WAL, Indexes, Erasure Coding](modules/04-storage-engines.md) | ✅ |
| 05 | [Relational Databases & Data Modeling](modules/05-relational-databases.md) | ✅ |
| 06 | [Transactions: ACID, Isolation Levels, MVCC, Pessimistic/Optimistic Locking](modules/06-transactions.md) | ✅ |
| 07 | [The NoSQL Landscape: KV, Document, Wide-Column, Graph, Time-Series, Vector](modules/07-nosql-landscape.md) | ✅ |
| 08 | [Caching: Patterns, Eviction, Redis, CDNs, Stampedes](modules/08-caching.md) | ✅ |

### Part III — Distribution
| # | Module | Status |
|---|---|---|
| 09 | Replication: Leader/Follower, Multi-Leader, Leaderless, Quorums | ⬜ |
| 10 | Partitioning & Sharding: Consistent Hashing, Rebalancing, Hotspots | ⬜ |
| 11 | CAP, PACELC & Consistency Models | ⬜ |
| 12 | Consensus, Coordination & Time: Raft, etcd, Distributed Locks, Failure Detection (Heartbeats/Gossip/Phi-Accrual), Vector Clocks | ⬜ |
| 13 | Distributed Transactions: 2PC, Sagas, Outbox, Idempotency | ⬜ |

### Part IV — Scale and Traffic
| # | Module | Status |
|---|---|---|
| 14 | Load Balancing, Proxies & Service Discovery | ⬜ |
| 15 | Messaging & Streaming: Queues vs Logs, Kafka, Delivery Semantics | ⬜ |
| 16 | Batch & Stream Processing: MapReduce, Spark, Flink, CDC, Event Sourcing | ⬜ |
| 17 | Rate Limiting, Quotas & Admission Control | ⬜ |
| 18 | Resiliency: Timeouts, Retries, Circuit Breakers, Degradation | ⬜ |

### Part V — Running It in Production
| # | Module | Status |
|---|---|---|
| 19 | Observability: Metrics, Logs, Traces, SLI/SLO/Error Budgets | ⬜ |
| 20 | Deployment & Infrastructure: Containers, K8s, Canary, Multi-Region, DR | ⬜ |
| 21 | Security: AuthN/AuthZ, OAuth/OIDC, Encoding vs Encryption vs Tokenization, Password Storage, Multi-Tenancy | ⬜ |

### Part VI — Architecture at Large
| # | Module | Status |
|---|---|---|
| 22 | Monoliths, Microservices, DDD, Gateways & Service Mesh | ⬜ |
| 23 | Data Architecture: OLTP vs OLAP, Warehouses, Lakes, Lakehouses | ⬜ |

### Part VII — AI/ML Systems
| # | Module | Status |
|---|---|---|
| 24 | ML Systems Foundations: Training/Serving Split, Feature Stores, Skew | ⬜ |
| 25 | Inference at Scale: Batching, KV Cache, GPU Scheduling, Autoscaling | ⬜ |
| 26 | Vector Search & RAG Systems: HNSW/IVF, Hybrid Search, Freshness | ⬜ |
| 27 | LLM Application Architecture: Agents, Guardrails, Evals, Cost Control | ⬜ |

### Part VIII — Interview Craft & Case Studies
| # | Module | Status |
|---|---|---|
| 28 | The Interview Framework: A Repeatable 45-Minute Structure | ⬜ |
| 29 | Case Study: URL Shortener + Distributed ID Generation | ⬜ |
| 30 | Case Study: Chat System (WhatsApp/Slack) | ⬜ |
| 31 | Case Study: News Feed / Timeline (Twitter, Instagram) | ⬜ |
| 32 | Case Study: Video Platform (YouTube/Netflix) | ⬜ |
| 33 | Case Study: Geospatial & Ride Hailing (Uber) | ⬜ |
| 34 | Case Study: Search, Typeahead & Web Crawler | ⬜ |
| 35 | Case Study: Payments & Double-Entry Ledger | ⬜ |
| 36 | Case Study: Object Store (S3) & Distributed File Systems | ⬜ |
| 37 | Case Study: Collaborative Editing (OT vs CRDT) | ⬜ |
| 38 | Case Study: Ad Click Aggregation & Real-Time Analytics | ⬜ |
| 39 | Case Study: Recommendation System at Scale | ⬜ |
| 40 | Case Study: LLM Serving Platform (ChatGPT-scale) | ⬜ |
| 41 | Case Study: Enterprise RAG over 100M Documents | ⬜ |
| 42 | Bonus Case Study: Proximity Service & Geospatial Indexing (Quadtree/Geohash) | ⬜ |
| 43 | Bonus Case Study: Design Gmail (Email at Scale) | ⬜ |

### Part IX — Curated Industry Case Studies (Supplementary Reading)

Not standalone modules — these are real engineering blog write-ups mapped to
the module they reinforce, to read *after* that module so the theory has
somewhere concrete to land. Tracked in
[cheatsheets/further-reading.md](cheatsheets/further-reading.md).

---

## Conventions used in these notes

- **`⚠️ Trap`** — a common misconception or interview failure mode.
- **`🤖 AI/ML Callout`** — how the concept maps to ML/LLM infrastructure.
- **`📐 Numbers`** — quantitative reasoning you should be able to reproduce.
- **`🎤 Interview Angle`** — how this is probed in an interview.

---

*Maintained as a personal learning archive. Feel free to fork.*

---