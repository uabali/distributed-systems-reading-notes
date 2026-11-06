# Designing Data-Intensive Applications — notes in my own words

These notes paraphrase themes from Martin Kleppmann’s *Designing Data-Intensive Applications*.  
**No long quotes. No invented page numbers.** Re-read the book for nuance.

Each block: **Idea → Why it matters → Link to fal / Distributed Systems work.**

---

## 1. Reliability, scalability, maintainability

**Idea.** A data system is “good” when it keeps working under faults (reliability), stays useful as load or data grows (scalability), and stays changeable by humans (maintainability). Throughput and latency are load dimensions; “works on my laptop” is not a scaling story.

**Why it matters.** Product marketing loves “always on.” Engineering has to define *which* faults you absorb (disk, node, network, human), *what* you measure under load (p99 latency vs QPS), and *how* you keep the system operable (observability, operability, simplicity).

**fal / DS link.** Inference APIs fail on GPU OOM, queue lag, and partial region loss. Labs already practice this: `rust-queue-worker` retries with backoff; `mini-CDN` sheds load; `inference-worker-lab` scales on lag — all reliability/scalability levers, not just “add more machines.”

---

## 2. Data models and query languages

**Idea.** Relational, document, graph, and columnar models encode different assumptions about relationships and access patterns. The “best” model is the one that matches your queries and your write shape — not the newest logo.

**Why it matters.** Wrong model → awkward joins, N+1 reads, or painful migrations. Schema-on-write vs schema-on-read is a tradeoff about who pays for consistency of shape: writers at ingest time, or readers later.

**fal / DS link.** Job metadata in Redis/Postgres, embeddings in a vector store, and CDN object keys are three models. Mixing them without a clear boundary creates “distributed spaghetti.” Prefer explicit ownership of each store.

---

## 3. Storage and retrieval

**Idea.** Log-structured vs page-oriented storage, indexes (B-trees, LSM, hash), and compaction/vacuum are how you trade write amplification, read amplification, and space amplification.

**Why it matters.** “Just put it in the DB” hides whether you optimized for append-heavy writes, point lookups, or analytical scans. Understanding the storage engine explains why p99 spikes during compaction or why secondary indexes hurt write throughput.

**fal / DS link.** Redis Streams (`inference-worker-lab`) is an append-oriented log with consumer groups — great for queues, not a replacement for a durable OLTP store of customer state. Choose engines by access pattern.

---

## 4. Encoding and schema evolution

**Idea.** JSON, Protobufs, Avro, Thrift, etc. differ in schema enforcement, evolution rules, and language support. Rolling upgrades need forward/backward compatibility: old readers + new writers (and vice versa) must not explode.

**Why it matters.** Microservices and queues live forever. A breaking field rename in a job payload will strand workers mid-deploy. Compatibility is a distributed-systems problem, not a “serialization library” detail.

**fal / DS link.** Version your queue messages (`job_v1`, `job_v2`). Keep workers able to ignore unknown fields. Same discipline as API versioning for an inference gateway.

---

## 5. Replication

**Idea.** Copy data to more nodes for latency (geo), availability, or read scale. Leader/follower, multi-leader, and leaderless (quorum) designs differ in who accepts writes and how conflicts resolve.

**Why it matters.** Replication is not free consistency. Async replicas lag; sync replicas amplify write latency and failure coupling. You must name the failure mode you accept (stale read, lost acknowledged write, conflict).

**fal / DS link.** CDN edge caches (`mini-CDN`) are a form of replication with eventual freshness. For model weights or feature stores, decide explicitly: read-your-writes for a user session vs “eventually consistent embeddings are OK.”

---

## 6. Partitioning (sharding)

**Idea.** Split a dataset across nodes so no single node holds everything. Key choice (hash, range, directory) decides hotspots, rebalancing cost, and cross-partition queries/transactions.

**Why it matters.** Hot keys kill clusters. Rebalancing moves data and load; done badly it causes thundering herds. Secondary indexes become local or global — another design fork.

**fal / DS link.** Consistent hashing in `mini-CDN` is partitioning for cache ownership with graceful remapping when nodes die. Same mental model as sharding an inference job queue by tenant or model id — watch for hot models.

---

## 7. Transactions

**Idea.** ACID is a contract about what concurrent clients may observe. Isolation levels (read committed, snapshot, serializable) trade anomalies avoided vs throughput. Distributed transactions (2PC) couple availability to coordinator health.

**Why it matters.** “We use transactions” is meaningless without the isolation level. Many “bugs” are allowed anomalies under weaker isolation. Spanning a transaction across queue + DB + object store is usually a saga / outbox problem, not a single ACID tx.

**fal / DS link.** Accept a job (API + Redis XADD) then update status after worker completion — that is an eventual workflow, not one DB transaction. Design for idempotent consumers (exactly-once *effect*, at-least-once delivery).

---

## 8. The trouble with distributed systems

**Idea.** Networks lie (partial failure, unbounded delay). Clocks skew. Processes pause (GC, noisy neighbors). You cannot assume perfect failure detection; timeouts are guesses.

**Why it matters.** Algorithms that look correct on a whiteboard fail when messages reorder or a node is “zombie alive.” Fencing tokens, generation numbers, and leases exist because “ping failed” ≠ “node is dead.”

**fal / DS link.** Linux lab (cgroups, namespaces) shows *why* a neighbor can steal CPU and inflate your timeout. Load shedding and retries without jitter amplify outages — `mini-CDN` and `rust-queue-worker` practice the opposite habits.

---

## 9. Consistency and consensus

**Idea.** Linearizability, causal consistency, eventual consistency — different promises about order and freshness. Consensus (Raft/Paxos-class) gives agreement on a value/log among majority; it is expensive and needs careful quorum membership.

**Why it matters.** You rarely need full linearizability everywhere. Put strong consistency where money, locks, or leader election live; allow weaker models for feeds, metrics, and caches. Mislabeling “eventually consistent” as “strong” causes silent product bugs.

**fal / DS link.** Autoscaler desired-replica count can be eventually consistent; “which worker owns this GPU lease” cannot. Separate control-plane consistency needs from data-plane caching.

---

## 10. Batch processing

**Idea.** MapReduce-style / Spark-style jobs scan large inputs offline, write derived outputs, and retry whole tasks. Batch is about throughput and correctness of a snapshot, not interactive latency.

**Why it matters.** Great for training pipelines, backfills, and large joins. Bad as a substitute for low-latency serving. Exactly-once *outputs* often mean deterministic task IDs + atomic commit of results.

**fal / DS link.** Offline evaluation of models, log compaction, and rebuilding a search index are batch. Don’t put user-facing inference on a daily batch cron.

---

## 11. Stream processing

**Idea.** Continuous processing of unbounded events. Challenges: ordering, late data, exactly-once effects, stateful operators, and joining streams with tables. Event time ≠ processing time.

**Why it matters.** Product features (fraud, metrics, personalization) want freshness without waiting for a nightly job. Watermarks and windowing are how you admit that reality is late and out of order.

**fal / DS link.** Redis Streams workers are a minimal stream consumer. Lag-based autoscaling is a stream-ops pattern: measure backlog, not just CPU. Same idea scales up to Kafka + Flink/Spark Structured Streaming.

---

## 12. Derived data and the “unbundled” database

**Idea.** Many systems are pipelines: source of truth → derived views (search index, cache, feature store, warehouse). Event logs can be the integration backbone. The hard part is keeping derived data correct as sources evolve.

**Why it matters.** Dual writes (API writes DB *and* cache) race and diverge. Better: write once to a log/DB, derive the rest with idempotent consumers and clear ownership.

**fal / DS link.** Inference results, Prometheus metrics, and CDN caches are all derived. Treat “source of truth” as sacred; treat caches as disposable. That mindset prevents the classic “cache and DB disagree, blame the GPU” incident.

---

## Quick self-check (interview style)

1. Name one consistency model weaker than linearizability and when you’d pick it.
2. Why is hash partitioning friendly to rebalancing but bad for range scans?
3. What’s the difference between at-least-once delivery and exactly-once *processing effects*?
4. When is a batch job the wrong tool for an inference product?

If you can answer those without quoting a textbook page, the notes did their job.
