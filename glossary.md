# Glossary

Terms are added here as they're introduced in modules. Each entry links back
to the module where it was first taught.

<!-- Format:
### Term
One-line definition. See [Module 00](modules/00-foundations.md).
-->

### Latency
Time for one request/operation to complete, start to finish; usually reported as p50/p95/p99. See [Module 00](modules/00-foundations.md).

### Throughput
Number of requests/operations a system handles per unit time. Often traded against latency via batching. See [Module 00](modules/00-foundations.md).

### Availability
The fraction of time a system successfully responds to requests — says nothing about whether the response is correct. See [Module 00](modules/00-foundations.md).

### Reliability
The system does the correct thing consistently, even under failure — distinct from availability. See [Module 00](modules/00-foundations.md).

### Durability
Once a write is acknowledged, it is not lost, even across crashes — distinct from availability (durable data can be briefly unreadable). See [Module 00](modules/00-foundations.md).

### Fault Tolerance
The system keeps working correctly when components fail, achieved via redundancy rather than failure prevention. See [Module 00](modules/00-foundations.md).

### Consistency
Overloaded term — means something different in ACID vs. CAP vs. everyday English; always state which sense you mean. See [Module 00](modules/00-foundations.md), refined in [Module 11](modules/00-foundations.md) and [Module 06](modules/00-foundations.md).

### Scalability (Vertical vs Horizontal)
Ability to handle growth by adding resources. Vertical = bigger machine (hard ceiling, SPOF). Horizontal = more machines (no ceiling, but introduces distributed coordination problems). See [Module 00](modules/00-foundations.md).

### SPOF (Single Point of Failure)
Any component whose failure takes down the whole system; a primary design goal is eliminating these via redundancy. See [Module 00](modules/00-foundations.md).

### Idempotency
An operation that can be applied multiple times without changing the result beyond the first application — critical for safe retries over unreliable networks. See [Module 00](modules/00-foundations.md).

### Stateless vs Stateful
A stateless service keeps no client-specific data between requests (easy to scale horizontally); a stateful service (e.g. a database) remembers data across requests. See [Module 00](modules/00-foundations.md).
