# High-Level Design (HLD)

## 1. Disclaimer

Disclaimer - thoughts are my own. Trust carefully.

### Good answers

| Signal          | What "good" looks like                                                                   |
| --------------- | ---------------------------------------------------------------------------------------- |
| Requirements    | Asks clarifying questions; converges on 3–5 functional requirements without hand-holding |
| Building blocks | Correctly deploys standard components: LB, cache, DB, queue, CDN, blob store             |
| Data modeling   | Sensible schema, correct choice of SQL vs NoSQL with _a_ justification                   |
| Scale reasoning | Can do QPS/storage math; knows when to shard, when to cache                              |
| Trade-offs      | Names at least one alternative for major decisions and why they chose theirs             |
| Depth           | Goes deep in 1–2 areas **when prompted** by the interviewer                              |
| Communication   | Keeps a legible diagram; doesn't need rescuing more than once or twice                   |

### Excellent answers

Everything above, **plus**:

| Signal                | What "good" looks like                                                                      |
| --------------------- | ------------------------------------------------------------------------------------------- |
| Ambiguity ownership   | _Defines_ the problem scope themselves; proposes what's in/out and gets buy-in              |
| Quantified NFRs       | Turns vague asks into numbers: "10M DAU → ~1.2K writes/s avg, plan for 5x peak"             |
| Proactive deep dives  | Identifies the 2–3 hardest sub-problems _unprompted_ and drives into them                   |
| Failure thinking      | Discusses failure modes, degradation behavior, blast radius, recovery — without being asked |
| Consistency rigor     | Precise about consistency requirements per data path, not one blanket claim                 |
| Evolution & migration | "V1 looks like X; at 10x we'd change Y; here's the migration path"                          |
| Cost & ops awareness  | Mentions cost drivers, on-call burden, observability, deployment strategy                   |
| Alternatives          | Presents 2–3 genuinely viable designs and _argues_ for one, quantitatively where possible   |
| Pushback handling     | When challenged, either defends with reasoning or concedes gracefully with a revised design |

### The three axes every rubric reduces to

1. **Breadth** — do you know the standard toolbox and when each tool applies?
2. **Depth** — can you go 3 levels down in at least a couple of areas (e.g., _how_ consistent hashing rebalances, _how_ Kafka achieves ordering, _what_ happens during a failover)?
3. **Judgment** — do you make decisions proportionate to requirements, or do you cargo-cult (Kafka for 10 QPS, microservices for a CRUD app)?

---

## 2. The Answer Framework

This has worked for me multiple times.
Time is Claude estimation.

### The 8 steps (with time budget for a 45-min round)

| #   | Step                            | Time      | Output                                                                       |
| --- | ------------------------------- | --------- | ---------------------------------------------------------------------------- |
| 1   | **Functional requirements**     | 3–5 min   | 3–5 core features, explicitly de-scoped list                                 |
| 2   | **Non-functional requirements** | 2–3 min   | Scale, latency targets, availability, consistency, durability — _as numbers_ |
| 3   | **Back-of-envelope estimation** | 3–4 min   | QPS (avg + peak), storage/year, bandwidth, cache memory                      |
| 4   | **API design**                  | 3–4 min   | 3–6 endpoints or RPCs with request/response shapes                           |
| 5   | **Data model**                  | 3–5 min   | Entities, keys, indexes; DB choice with justification                        |
| 6   | **High-level design**           | 8–10 min  | End-to-end diagram covering every functional requirement                     |
| 7   | **Deep dives**                  | 12–15 min | 2–3 hard sub-problems solved in detail                                       |
| 8   | **Wrap-up**                     | 2–3 min   | Bottlenecks, failure modes, monitoring, future evolution                     |

**Important:** after step 2, say out loud which 2–3 deep dives you _intend_ to hit ("the interesting problems here are fan-out strategy, hot-key mitigation, and exactly-once delivery — I'll make sure we cover those").

### Step 1 — Functional requirements: how to do it well

- Ask **who** the users are (consumers? internal services? both?) and roughly **how many**.
- Ask about the **core flows**, then propose: "I'll focus on A, B, C and explicitly de-scope D, E — sound good?"
- Write requirements as _verbs_: "users can post tweets," "followers see tweets in near-real-time."
- **Trap to avoid:** listing 10 features. Pick the 3–5 that create the hard design problems.

### Step 2 — Non-functional requirements: the checklist

Go through this list every time, but only _elaborate_ on the ones that bite:

- **Scale**: DAU/MAU, read:write ratio, data size per object.
- **Latency**: p50/p99 targets per critical path ("feed load < 200ms p99").
- **Availability**: 99.9% vs 99.99% — say what an outage costs to justify the target.
- **Consistency**: per data path. "Payment ledger: strongly consistent. Follower counts: eventual is fine."
- **Durability**: can we ever lose a write? (Payments: no. Analytics events: sampled loss OK.)
- **Security/compliance**: mention when domain-relevant (payments → PCI, health → HIPAA, EU users → GDPR).
- **Cost**: one sentence acknowledging the dominant cost driver (egress? storage? compute?).

### Step 3 — API design conventions

- REST for public/external; gRPC for internal service-to-service; WebSocket/SSE for server push.
- Show idempotency where it matters: `POST /payments` with `Idempotency-Key` header.
- Pagination: cursor-based (not offset) for anything feed-like — and say _why_ (offset breaks under concurrent inserts and gets slow at depth).
- Auth: "requests carry a JWT validated at the gateway" — one sentence, don't rabbit-hole.

### Step 4 — High-level design discipline

- Draw left-to-right: client → edge (DNS/CDN) → gateway/LB → services → data stores → async infra.
- **Walk every functional requirement through the diagram.** "A tweet post hits the gateway, lands in the tweet service, writes to the tweets table, emits an event to Kafka…" If a requirement has no path through your diagram, the design is incomplete.
- Keep boxes coarse. Don't pre-shatter into 12 microservices; 3–6 logical services is right for 45 minutes.

### Step 5 — Deep dives: the menu

The deep dive is where levels separate. Common deep-dive dimensions (know how to run each):

1. **Scaling a hot path** — sharding strategy, hot partition mitigation, caching layers.
2. **Consistency mechanics** — how exactly do we prevent double-booking / double-spend / lost updates? (Locks vs OCC vs serializable transactions vs fencing.)
3. **Fan-out / delivery** — push vs pull vs hybrid; celebrity problem.
4. **Real-time channel** — WebSocket connection management, presence, reconnect, message ordering.
5. **Async pipeline** — queue choice, delivery semantics, idempotent consumers, outbox pattern, DLQs.
6. **Storage engine fit** — why LSM vs B-tree for this workload; index design; TTL/compaction.
7. **Failure & recovery** — what breaks first, what degrades gracefully, how we fail over, RPO/RTO.
8. **Multi-region** — active-active vs active-passive, data residency, conflict resolution.

### Step 6 — Wrap-up script (memorize the shape)

> "Bottlenecks: X is the first thing to fall over at 10x — we'd address it with Y. Failure modes: if the cache tier dies we degrade to Z with elevated latency rather than hard-failing. I'd monitor A, B, C as the key SLIs. Future evolution: at bigger scale I'd revisit D."

---

## 3. Back-of-Envelope Estimation Toolkit

You need to do this math _fluently and out loud_. Practice until each estimate takes < 60 seconds.

### Numbers to memorize

**Latency ladder (orders of magnitude — 2020s hardware):**

| Operation                                          | Latency     |
| -------------------------------------------------- | ----------- |
| L1 cache reference                                 | ~1 ns       |
| Main memory reference                              | ~100 ns     |
| Read 1 MB sequentially from RAM                    | ~10 µs      |
| SSD random read                                    | ~100 µs     |
| Read 1 MB sequentially from SSD                    | ~1 ms       |
| Intra-datacenter round trip                        | ~0.5 ms     |
| Read 1 MB sequentially from spinning disk          | ~10 ms      |
| Cross-continent round trip (e.g., India ↔ US-East) | ~150–250 ms |

**Capacity rules of thumb:**

- 1 day ≈ 86,400s → **shortcut: X per day ÷ 100K ≈ X per second** (within 15%).
- 1 million requests/day ≈ **12 QPS** average. Peak = 2–5x average (use 3x if unsure; 10x+ for spiky/flash-sale domains).
- A single beefy Postgres/MySQL node: ~**5–20K simple QPS** reads, ~**1–5K** writes (highly workload-dependent — say so).
- Redis node: ~**100K+ ops/s**. Kafka: **hundreds of MB/s per broker**, millions of msgs/s per cluster.
- One server holds ~**100K–1M concurrent WebSocket connections** (memory-bound; ~10KB/conn as a planning number).
- Char = 1 byte, UUID = 16 bytes, snowflake ID = 8 bytes; typical metadata row ≈ **1 KB**; image ≈ 200 KB–2 MB; 1 min of 1080p video ≈ **50–100 MB** raw, ~10 MB streamed.

### The 4 standard estimates (do all four, every interview)

Worked example — "Design Twitter," 200M DAU, each user posts 0.2 tweets/day, reads 100 timeline items/day:

1. **Write QPS**: 200M × 0.2 = 40M tweets/day ≈ **400 writes/s** avg, ~1.2–2K peak.
2. **Read QPS**: 200M × 100 = 20B reads/day ≈ **200K reads/s** avg → read:write ≈ 500:1 → _conclusion: this is a read-optimization problem; caching and precomputation dominate the design._
3. **Storage**: 40M tweets/day × 1 KB ≈ 40 GB/day ≈ **~15 TB/year** for text (media handled separately in blob storage, ~×100 that).
4. **Cache sizing**: 80/20 rule — cache 20% of daily read working set. Estimate hot set, multiply by object size, state how many Redis nodes that implies.

**Important:** always end estimation with a _design conclusion_, not just numbers. "500:1 read ratio → we precompute timelines" is the point of the exercise.

## 4. The Syllabus

### Tier 1 — Networking & the Edge

- **DNS**: resolution path, TTLs, GeoDNS / latency-based routing, DNS as a crude failover mechanism (and why TTL caching makes it slow).
- **TCP vs UDP vs QUIC**: handshakes, head-of-line blocking, why HTTP/3 moved to QUIC.
- **HTTP/1.1 vs HTTP/2 vs HTTP/3**: multiplexing, header compression; keep-alive.
- **TLS**: termination points (edge vs service), mTLS for service-to-service.
- **Client–server communication patterns** (know the decision tree cold):
    - Request/response (REST, gRPC, GraphQL) — when each: REST for public CRUD, gRPC for internal low-latency typed RPC, GraphQL when clients need flexible aggregation (and its costs: N+1, caching difficulty, query complexity limits).
    - Server push: **short polling vs long polling vs SSE vs WebSocket** — trade-offs in latency, connection cost, proxy-friendliness, bidirectionality. Classic probe: "why not just poll?"
- **Load balancing**: L4 vs L7; algorithms (round robin, least connections, weighted, consistent hashing for stickiness); health checks; global (GSLB/anycast) vs regional; where TLS terminates.
- **Reverse proxy vs API gateway**: gateway responsibilities — authN, rate limiting, routing, request shaping, TLS termination.
- **CDN**: pull vs push, cache keys, invalidation (purge vs versioned URLs — prefer versioned), signed URLs, edge compute; what's CDN-able (static, video segments, even API GETs with care).

### Tier 2 — Storage

**Relational internals:**

- B-tree indexes, covering indexes, composite index ordering; why indexes slow writes.
- WAL / redo logs; how a write actually becomes durable.
- Transactions & isolation levels: read committed → repeatable read → serializable; what anomalies each permits (dirty read, non-repeatable read, phantom, write skew). Know **write skew** — it's the senior-level trap in booking/inventory questions.
- Pessimistic locking (`SELECT … FOR UPDATE`) vs optimistic concurrency (version column / CAS): when each wins.

**Replication:**

- Leader–follower (async vs sync vs semi-sync); replication lag and its user-visible symptoms (read-your-writes violations); mitigations (pin user to leader briefly, session tokens, monotonic reads via sticky routing).
- Multi-leader: use cases (multi-region writes) and the conflict problem (LWW, CRDTs, app-level merge).
- Leaderless / quorum (Dynamo-style): N/R/W, sloppy quorum, hinted handoff, read repair, anti-entropy with Merkle trees.
- Failover mechanics: leader election, split-brain, fencing tokens, the danger of async-replica promotion (lost writes).

**Partitioning / sharding:**

- Strategies: range (order-preserving, hot-spot prone), hash (uniform, kills range scans), directory-based (flexible, extra hop).
- **Consistent hashing**: the ring, virtual nodes, how it minimizes remapping — be able to explain on a whiteboard in 2 minutes.
- Hot partitions: detection and fixes (key salting, split hot keys, dedicated shards for celebrities/whales).
- Resharding & rebalancing: how to move data live (dual-write, backfill, cutover) — a classic senior probe.
- Secondary indexes on sharded data: local (scatter-gather reads) vs global (fan-out writes) — DynamoDB GSI is the canonical example.

**Storage engines:**

- **B-tree vs LSM-tree**: write amplification vs read amplification vs space amplification; memtable → SSTable → compaction (leveled vs size-tiered); bloom filters for point-read short-circuiting; why Cassandra/RocksDB choose LSM and Postgres chooses B-tree.

**NoSQL taxonomy & flagship systems (know 1–2 deeply each):**

- Key-value: Redis, DynamoDB. Wide-column: Cassandra (partition key + clustering key modeling, tombstones). Document: MongoDB. Graph: Neo4j (mention-only). Search: Elasticsearch (inverted index, analyzers, relevance, why it's not a source of truth).
- **NewSQL / Spanner-class**: globally consistent transactions via TrueTime; CockroachDB/Spanner as the "I need SQL semantics at global scale" answer — plus its latency cost.
- Object storage (S3): what belongs in blob storage vs DB (anything > ~a few hundred KB); presigned URLs for direct client upload/download; multipart upload.

**Caching (asked in literally every interview):**

- Patterns: cache-aside (default), read-through, write-through, write-behind (durability risk), refresh-ahead.
- Eviction: LRU/LFU/TTL; TTL jitter to prevent synchronized expiry.
- **Failure modes — know all four and their fixes:**
    1. _Stampede / thundering herd_ (hot key expires → dogpile on DB): request coalescing / single-flight, lock-and-recompute, stale-while-revalidate.
    2. _Penetration_ (queries for nonexistent keys bypass cache): negative caching, bloom filter in front.
    3. _Avalanche_ (cache tier dies): circuit breakers, degrade gracefully, warm-up plan, multi-tier caching.
    4. _Hot key_ (one key exceeds a node's capacity): local/in-process cache layer, key replication with client-side random pick.
- Consistency: invalidate vs update-on-write; why delete-then-write is usually safer; CDC-driven invalidation as the robust approach.
- Layers: browser → CDN → gateway → service-local (in-process) → distributed (Redis) → DB buffer pool. Name the layers, choose deliberately.

### Tier 3 — Async, Queues & Streaming

- **Queue vs log**: RabbitMQ/SQS (competing consumers, per-message ack/delete, no replay) vs Kafka (partitioned append-only log, consumer groups, offsets, replay, ordering _per partition only_).
- When a queue at all: decouple producers/consumers, absorb bursts, retry safely, fan out to multiple consumers, smooth over downstream outages.
- **Delivery semantics**: at-most-once, at-least-once (the practical default), exactly-once (really: at-least-once + **idempotent consumers**, or Kafka transactions/EOS within its ecosystem). Senior answer: "we design for at-least-once and make handlers idempotent via idempotency keys / dedup tables."
- **Transactional outbox + CDC (Debezium)**: the correct answer to "how do you atomically update the DB _and_ publish an event?" Never say "we write to the DB then publish to Kafka" without addressing the dual-write problem.
- Ordering: per-key ordering via partition keys; what to do when you need global order (you almost never do — say why).
- Backpressure: bounded queues, consumer lag monitoring, load shedding, slow-consumer isolation.
- DLQs, poison pills, retry with exponential backoff + jitter, retry budgets.
- **Stream processing** (Flink/Kafka Streams/Spark Streaming): tumbling/sliding/session windows, event time vs processing time, watermarks, late data; the canonical use: real-time aggregation (ad clicks, metrics, fraud features).
- Fan-out patterns: pub/sub topic per interest; fan-out-on-write vs fan-out-on-read (feeds).

### Tier 4 — Distributed Systems Theory

- **CAP** (precisely: during a _partition_, choose availability or consistency) and **PACELC** (else, latency vs consistency) — use PACELC; it's the grown-up version.
- **Consistency models** ladder: linearizable → sequential → causal → eventual; session guarantees (read-your-writes, monotonic reads/writes). Map real systems onto the ladder.
- **Consensus**: Raft in enough depth to sketch it (leader election with terms, log replication, commit index, why majority quorums); where consensus actually lives in your designs (etcd/ZooKeeper for metadata, Kafka controller, DB failover) — and why you don't run Paxos per user request.
- **Distributed locks & leases**: Redis SETNX + TTL and its dangers (Redlock controversy — know the outline), fencing tokens as the fix, ZooKeeper/etcd leases as the robust option; and the senior insight: _prefer designing locks away_ (idempotency, uniqueness constraints, single-writer partitions) over distributed locking.
- **Time**: NTP drift, why wall-clock ordering is unsafe, logical clocks (Lamport), vector clocks (Dynamo conflict detection), Spanner TrueTime as the exotic option; hybrid logical clocks (mention).
- Failure detection: heartbeats, timeouts, gossip protocols (Cassandra), phi-accrual (mention).
- **Unique ID generation**: auto-increment (single point / doesn't shard), UUIDv4 (random, index-unfriendly), **Snowflake** (time-sortable 64-bit: timestamp + machine + sequence — know the layout), UUIDv7, TicketServer, pre-allocated ranges. This shows up as its own mini-question constantly.

### Tier 5 — Architecture Patterns & Operations

- Monolith vs microservices: honest trade-offs (org scaling, deploy independence vs distributed-systems tax); modular monolith as a legitimate senior answer for v1.
- Service discovery (DNS, Consul, k8s services), sidecars/service mesh (one sentence, don't rabbit-hole).
- **Resilience patterns**: timeouts everywhere, retries with backoff + jitter _only on idempotent ops_, circuit breakers, bulkheads, load shedding, graceful degradation, retry storms & metastable failures (senior gold: mention cascading failure and how retry budgets prevent it).
- **Rate limiting algorithms**: fixed window (bursty at edges), sliding window log (exact, memory-heavy), sliding window counter (practical), token bucket (allows bursts — usually the answer), leaky bucket (smooths). Distributed enforcement: Redis + Lua atomicity, local buckets with async sync for lower latency, race conditions at the boundary.
- **Distributed transactions**: 2PC (blocking coordinator, why it's avoided across services), **Sagas** (choreography vs orchestration, compensating transactions, the "pending" state), TCC. Default senior answer: sagas + idempotency + outbox.
- Event sourcing & CQRS: what they buy (audit, temporal queries, independent read models) and their cost (complexity, eventual consistency of read models) — use judiciously, don't lead with them.
- **Observability**: metrics (RED: rate/errors/duration; USE for resources), structured logs, distributed tracing (trace ID propagation), SLI/SLO/error budgets — sprinkle SLO language into wrap-ups.
- Deployments: blue-green, canary, feature flags, backward-compatible schema migrations (expand → migrate → contract).
- **Multi-region**: active-passive (simpler, RTO/RPO trade) vs active-active (conflict handling, data residency, home-region pinning per user); async cross-region replication lag; GDPR/data-residency as a design constraint.
- Cell-based architecture / shuffle sharding: blast-radius limitation (a strong senior mention for very-high-availability designs).
- Security in 60 seconds: OAuth2/OIDC, JWT (stateless, revocation weakness) vs opaque sessions, mTLS internally, encryption at rest/in transit, secrets management, principle of least privilege.

### Tier 6 — Specialized Components (pull in when the question demands)

- **Search**: inverted index, tokenization/analyzers, TF-IDF/BM25 relevance, index sharding & replicas, keeping ES in sync with source-of-truth via CDC.
- **Geospatial**: geohash (prefix search, boundary problem), quadtrees, S2/H3 cells, Redis GEO, PostGIS; "find nearby X" query patterns; moving-object updates (driver locations: in-memory + batched writes).
- **Probabilistic structures**: bloom filter (membership, no false negatives), count-min sketch (approximate counts / heavy hitters), HyperLogLog (distinct counts) — the top-K/leaderboard and crawler questions live here.
- **Time-series**: downsampling, retention tiers, delta/gorilla compression ideas, why generic RDBMS struggles.
- **WebSockets at scale**: connection gateways as a stateful tier, connection registry (user → gateway mapping in Redis), pub/sub backbone between gateways, heartbeats, resume/sync protocol on reconnect (client sends last-seen message ID).
- Push notifications: APNs/FCM as external delivery, token management, collapsing/batching, delivery receipts.
- Video pipeline: chunked/multipart upload → transcoding DAG (workers, spot instances) → ABR ladder (HLS/DASH segments) → CDN; separately: view counting, watch-position resume.
- Full-text autocomplete: prefix tries with precomputed top-K per node, periodic offline rebuild + fast delta layer.

---

## 5. Question Bank

Organized by **archetype**. Each archetype trains a reusable skill; once you can do the flagship question, the variants are 80% solved. For each flagship: the core challenge, the design skeleton, and the probes interviewers actually ask.

### Archetype A — Read-heavy, cache-and-precompute systems

**Flagship: News Feed / Twitter Timeline** _(the single most-asked question at multiple levels)_

- **Core challenge**: read:write ratio ~100–1000:1; assemble personalized feeds at low latency.
- **Skeleton**: post service → outbox/Kafka → fan-out service → per-user timeline cache (Redis lists/zsets) → feed service merges timeline cache + hydrates post content from a separate post cache.
- **The famous trade-off**: fan-out-on-write (precompute timelines; fast reads; write amplification for celebrities) vs fan-out-on-read (compute at read; slow reads) → **hybrid**: push for normal users, pull-and-merge for celebrity authors above a follower threshold.
- **Probes**: How do you handle a user with 100M followers? What happens when a user deletes a post (tombstones vs lazy filtering)? Feed ranking — where does ML scoring slot in? Cold start for inactive users whose timeline cache expired? How big is the timeline cache (do the math)?
- **Twist**: consistency of counters (likes/retweets) — sharded counters / CRDT-ish approximate counts with periodic reconciliation; pagination stability while new posts arrive.

**Variants**: Instagram, Facebook feed, Reddit (adds ranking/hot algorithms), LinkedIn feed, news aggregator.

**Flagship: URL Shortener (TinyURL)** _(the warm-up classic — you must be flawless)_

- **Core challenge**: tiny problem, so the interview is entirely about _rigor_: ID generation, redirect latency, analytics.
- **Skeleton**: write path generates a short code (base62 of a Snowflake ID, or pre-generated key pool) → KV store (DynamoDB/Redis+DB) → read path is a cached 301/302 redirect at the edge.
- **Probes**: 301 vs 302 (301 caches at the browser → you lose analytics; 302 preserves them — know this). Collision handling for custom aliases. Hot links → CDN/edge caching. Analytics without slowing redirects → async event emission. Predictability of sequential codes (enumeration attacks) → randomize.
- **Twist**: exact click counts vs approximate; multi-region read latency; abuse/malware URL scanning pipeline.

**Variants**: Pastebin (adds blob storage + TTL/expiry sweeps).

**Flagship: Typeahead / Search Autocomplete**

- **Core challenge**: p99 < 100ms per keystroke at massive QPS; freshness of trending queries.
- **Skeleton**: offline pipeline aggregates query logs → builds trie/finite-state structure with **precomputed top-K per prefix** → sharded by prefix → served from memory; a lightweight real-time layer overlays trending deltas; aggressive client-side debouncing + CDN caching of hot prefixes.
- **Probes**: Why precompute top-K instead of ranking at query time? Shard by prefix — what about hot prefixes ("a", "the")? How fresh is trending data (lambda-style batch + speed layer)? Personalization?

### Archetype B — Real-time communication

**Flagship: Chat System (WhatsApp / Messenger)**

- **Core challenge**: stateful connections, message ordering, delivery guarantees, offline delivery, group fan-out.
- **Skeleton**: clients hold WebSockets to a fleet of **chat gateways**; a connection registry (Redis: user → gateway) routes messages; message service persists to a write-optimized store (Cassandra/HBase — partition by conversation, cluster by message ID); cross-gateway delivery via pub/sub (Redis pub/sub or Kafka); offline users → store + push notification (APNs/FCM); sync protocol on reconnect (client sends last message ID per conversation, server returns the delta).
- **Probes**: Message IDs & ordering within a conversation (per-conversation sequence vs Snowflake — what does "ordered" mean with mobile clock skew?). Delivery states (sent/delivered/read) — how receipts flow back. Group chat: fan-out to 500 members (write per member vs per conversation + per-member cursors). End-to-end encryption — what the server can and can't do then (no server-side search, key distribution hand-wave is fine). How many WebSocket servers for 500M concurrent (do the connection math)?
- **Twist**: exactly-once _display_ (client dedup by message ID on top of at-least-once delivery); thundering reconnect after a gateway dies (jittered backoff, connection draining); multi-device sync (per-device cursors).

**Variants**: Slack (adds channels, history search, org model), live comments, presence/online status (its own mini-question: heartbeats + TTL'd presence keys + subscriber notification — and the fan-out cost of a popular user going online).

**Flagship: Collaborative Editor (Google Docs)**

- **Core challenge**: concurrent edits converging — **OT** (server-ordered transforms, central sequencer per doc) vs **CRDT** (commutative ops, no central sequencer, higher metadata cost).
- **Skeleton**: doc sessions pinned to a "doc leader" server (single-writer per doc simplifies OT) → op log persisted → periodic snapshots + op-log compaction → presence/cursors via the same channel.
- **Probes**: Why is per-doc single-writer OK (a doc's edit rate is human-bounded)? Snapshotting strategy? Offline edits merging? Permission changes mid-session?
- _(This is a hard question — usually senior-only. If you know Raft and CRDTs already, this is a chance to shine.)_

### Archetype C — Media / heavy content

**Flagship: YouTube / Netflix**

- **Core challenge**: two nearly-independent systems — an **upload/processing pipeline** and a **serving/streaming path**; design both, spend time where the interviewer steers.
- **Upload**: resumable multipart upload direct-to-blob via presigned URLs → event triggers a **transcoding DAG** (split into chunks → parallel transcode into ABR ladder: 240p→4K, multiple codecs → stitch → generate manifests) on a worker fleet (queue + autoscaling, spot instances for cost) → outputs to origin blob storage.
- **Serving**: manifest (HLS/DASH) → client adaptive bitrate → segments from **CDN** (99%+ offload); signed URLs for access control.
- **Probes**: Why chunk-level transcoding (parallelism, retry granularity)? View-count pipeline (async, approximate then reconciled — never on the serving path). Resume watch position (tiny KV write, batched). Copyright/content-matching hook in the pipeline. Cold vs hot content tiering.
- **Twist**: cost math (egress dominates → CDN contract framing), pre-positioning popular content regionally (Netflix Open Connect story), live streaming differences (latency ladder: HLS ~10s → LL-HLS ~3s → WebRTC sub-second).

**Variants**: image hosting (Instagram media path), Spotify (adds audio licensing windows, offline sync), Dropbox/Google Drive (**different problem**: chunk-level dedup via content hashes, delta sync, metadata DB as the hard part, client sync-state machine, conflict copies).

### Archetype D — Geospatial

**Flagship: Uber / Ride-hailing**

- **Core challenge**: millions of moving objects; low-latency "nearby drivers" matching; trip state machine correctness.
- **Skeleton**: driver apps send location every 3–5s → location gateway → **in-memory geosharded index** (geohash/H3 cells → driver sets in Redis; drivers move between cells) with batched async persistence; rider request → matching service queries neighboring cells → ranks (ETA via routing service) → dispatch with driver-accept timeout; trip service owns a strict **state machine** (requested → matched → en-route → in-trip → completed) with idempotent transitions.
- **Probes**: Why not store locations in Postgres (write volume; staleness tolerance)? Cell size trade-off and the geohash boundary problem (search neighbor cells). Surge pricing inputs (supply/demand per cell — a streaming aggregation). Preventing double-dispatch of a driver (atomic claim: Redis Lua / conditional write). ETA computation (precomputed routing graph, contraction hierarchies — name-drop, don't derive).
- **Twist**: degradation when the matching layer is overloaded (queue riders vs widen search radius); location data privacy/retention.

**Variants**: food delivery (three-sided: adds restaurant capacity & prep-time estimation), Yelp/nearby places (static objects → simpler: PostGIS/ES geo queries + caching), Google Maps (usually scoped down to one subsystem — clarify!), Strava-style activity tracking.

### Archetype E — Consistency-critical / transactional

**Flagship: Payment System**

- **Core challenge**: money must never be lost or duplicated; external PSPs (Stripe/bank rails) are slow and flaky; everything must be idempotent and reconcilable.
- **Skeleton**: payment API with **idempotency keys** (unique constraint → return prior result on retry) → payment intent stored in `pending` → call PSP with timeout + bounded retries → webhook/callback (verified, itself idempotent) advances state → **double-entry ledger** (append-only, immutable, debits = credits, balances derived) → async **reconciliation** job diffs internal ledger vs PSP settlement files → saga/compensation for multi-step flows (auth → capture → refund).
- **Probes**: PSP call times out — did the charge happen? (Unknown state → query/retry with same idempotency key; never blindly re-fire.) Why a ledger instead of updating a balance column (auditability, no lost updates, temporal queries)? Exactly-once payment execution (at-least-once + idempotency, spelled out). Where's strong consistency mandatory vs relaxable (ledger writes: serializable/single-partition; merchant dashboards: eventual)? PCI scope minimization (tokenization — card data never touches your servers).
- **Twist**: hot merchant accounts (balance contention → sharded sub-accounts or ledger-derived async balances), cross-currency, chargeback flows, and the ops story (reconciliation break alerting).

**Flagship: Ticket / Hotel Booking (Ticketmaster, Booking.com)**

- **Core challenge**: finite inventory under contention; prevent double-booking without destroying throughput; flash-sale spikes.
- **Skeleton**: browse path is heavily cached (availability approximate) → **reservation step** places a short-TTL hold (row-level `SELECT FOR UPDATE`, or Redis hold with TTL + fencing, or OCC version check) → payment within the hold window → confirm converts hold to booking; expired holds auto-release.
- **Probes**: Pessimistic vs optimistic locking here — and why the answer differs for airline seats (chosen seat, high contention on one row) vs hotel room _types_ (counter decrement, contention spread). **Write skew** trap: two transactions each read "1 room left" and both book under snapshot isolation → need serializable, unique constraints, or atomic decrements with a `>= 0` check. Virtual waiting queue for flash sales (admission control before the app tier). Double-charge vs double-book — which is worse and how the flow biases toward the recoverable failure.
- **Twist**: overselling as a _business_ choice (airlines) — model it explicitly; multi-night/multi-leg atomicity (saga across resources).

**Flagship: Stock Exchange / Order Matching** _(your home turf — expect follow-ups to go deep)_

- **Core challenge**: deterministic matching, microsecond latency, total ordering, and fault tolerance without giving up speed.
- **Skeleton**: gateway → risk checks → **sequencer** assigns total order → single-threaded matching engine per symbol (in-memory order book: price-level map + FIFO queues) → events out (fills, book updates) via an event log → market data fan-out on a separate, lossy-tolerant path; recovery via event-sourced replay from the sequenced log + snapshots; hot-warm engine pairs consuming the same sequenced stream.
- **Probes**: Why single-threaded per symbol (determinism, cache locality, no locks)? Order book data structure choice and complexity of match/cancel. How replicas stay identical (state machine replication over the sequenced input log). Cancel-heavy workloads. Fairness (FIFO at the sequencer; what "fair" even means).
- _(Leverage Coinbase International Exchange experience explicitly here — real war stories about trade reporting, risk checks, and derivatives margining read as strong senior signal.)_

**Variants**: e-commerce checkout (inventory + cart + payment saga), digital wallet (P2P transfer atomicity across two shards), flash sale (admission control + inventory), ad auction.

### Archetype F — Infrastructure components ("design X itself")

These test whether you understand your tools' internals. Increasingly common at senior level.

**Flagship: Distributed Rate Limiter**

- Algorithms (token bucket default), Redis + Lua for atomic check-and-decrement, latency cost of central check → local buckets with periodic sync (accepting slight over-admission), rule config distribution, fail-open vs fail-closed (senior: fail-open for availability, _except_ on security-sensitive endpoints — say this trade explicitly), response headers (429, Retry-After).

**Flagship: Distributed Message Queue (design Kafka-lite)**

- Partitioned append-only log, sequential I/O + page cache + zero-copy as the performance story, offset-based consumption, consumer groups & rebalancing, replication with ISR and leader election, retention/compaction, ordering scope, how to add delayed messages / DLQs on top.

**Flagship: Distributed KV Store (design Dynamo/Redis-cluster-lite)**

- Consistent hashing + vnodes, N/R/W quorums and the R+W>N intuition, hinted handoff, read repair, Merkle-tree anti-entropy, vector clocks vs LWW, LSM storage engine underneath, how a client routes (smart client vs proxy).

**Flagship: Web Crawler**

- Frontier queue with **politeness** (per-domain rate limits, robots.txt) → politeness routing (domain → dedicated queue/worker so one slow site doesn't block others) → fetch → parse → **dedup** (URL seen? bloom filter + exact store; content seen? simhash/checksums) → link extraction back to frontier; traps (spider traps, infinite calendars) via depth/URL-pattern budgets; freshness via recrawl scheduling by change frequency; DNS caching; storage pipeline to blob + index.
- **Probes**: BFS vs DFS and why priority queues really (importance/freshness scoring); how to crawl 1B pages/month (do the fetch-rate and bandwidth math); distributed frontier partitioning (by domain hash).

**Flagship: Metrics & Monitoring System (design Datadog-lite)**

- Agents → ingestion gateway → Kafka → stream aggregation (pre-aggregate per series per 10s) → time-series store (delta+XOR compression ideas, downsampling tiers: raw 10s→1m→1h, retention per tier) → query layer with fan-out/merge → alerting evaluator (streaming, avoid flapping via for-duration conditions); cardinality explosion as **the** core problem (label limits, rollups).

**Flagship: Distributed Job Scheduler (design cron-as-a-service)**

- Job definitions in DB → scheduler leaders per shard (leader election via etcd/ZK leases) scan due jobs → push to execution queue → worker fleet with heartbeats + visibility timeouts → retries/backoff, idempotent execution, misfire policy, **exactly-once-ish triggering** (claim row atomically: conditional update on `next_run_at`), priorities & rate limits per tenant, DAG dependencies (Airflow-flavored follow-up).

**Also be ready for (table form):**

| Question                          | Core problem                              | One-line senior hook                                                                 |
| --------------------------------- | ----------------------------------------- | ------------------------------------------------------------------------------------ |
| Notification service              | Multi-channel fan-out, preferences, dedup | Collapse + rate-limit per user; provider failover; delivery-receipt reconciliation   |
| Top-K / leaderboard               | Heavy hitters at scale                    | Count-min sketch + min-heap for approximate; Redis zset shards + merge for exact-ish |
| Ad click aggregator               | Exactly-once-ish counting for money       | Stream windows + late events + reconciliation against raw log (lambda architecture)  |
| Distributed counter (likes/views) | Write contention on one row               | Sharded counters, batched flush, approximate reads                                   |
| S3-like object store              | Metadata vs data separation               | Metadata DB + placement service + erasure coding vs 3x replication cost math         |
| ID generator service              | Uniqueness + rough ordering + HA          | Snowflake layout; clock-skew handling (refuse to go backwards)                       |
| Distributed lock service          | Safety vs liveness                        | Fencing tokens; lease renewal; "do you actually need a lock?"                        |
| Logging/log-search system         | Ingest firehose + cheap storage + search  | Hot ES tier + cold blob tier with on-demand hydration                                |
| API Gateway (design one)          | AuthN, routing, limiting at the edge      | Config propagation; per-route policies; latency budget for middleware                |
| Recommendation feed               | Candidate gen → rank → serve              | Two-tower retrieval + feature store freshness; fallback to popularity                |

---

## 6. Leveling Walkthrough

Same question — **"Design a rate limiter"** — answered at three quality levels. Study the deltas.

**Rejected / junior-reading answer:**

> "Use Redis. Store a counter per user, increment on each request, reject over the limit, reset every minute."

Problems: fixed-window burst flaw unmentioned, no atomicity (read-then-write race), no distribution story, no failure mode.

**Solid SDE-2 answer:**

> Clarifies where limiting happens (gateway), per-what (user + IP + endpoint). Compares fixed window vs sliding window vs token bucket; picks token bucket for burst tolerance. Uses Redis with a Lua script so check-and-decrement is atomic. Handles multi-instance gateways by sharing Redis. Mentions 429 + Retry-After. When asked "what if Redis is down?" — reasons through fail-open vs fail-closed.

**Senior answer (adds on top):**

> States the latency budget: "a synchronous Redis hop adds ~1ms; for a 10ms-budget edge, fine; for sub-ms internal RPC, not fine → local token buckets per gateway with async replenishment from a central budget, accepting bounded over-admission of ~N×burst." Quantifies Redis load and shards by key. Fail-open by default _with an explicit exception list_ for auth/payment endpoints (security > availability there). Discusses rule distribution (config service, versioned, hot-reload), multi-tenant fairness, and observability (emit limit-hit metrics; alert on sudden global throttle spikes as an incident signal). Closes with evolution: "start centralized-Redis for correctness, move hot paths to the local-bucket model when latency data justifies it."

**The pattern**: senior answers _quantify_, _segment the decision by context_ (fail-open except security paths), _own operations_, and _sequence the design over time_.

---

## 7. Cheat Sheets

### Database choice decision table

| Workload shape                                  | Default pick                 | Why / caveats                                                   |
| ----------------------------------------------- | ---------------------------- | --------------------------------------------------------------- |
| Transactional core (orders, payments, bookings) | Postgres/MySQL               | ACID, constraints, joins; shard later; boring is good           |
| Massive write throughput, known query patterns  | Cassandra/Scylla             | LSM writes, partition-key modeling; no ad-hoc queries           |
| Simple KV at scale, low ops burden              | DynamoDB                     | Predictable latency, GSIs; watch hot partitions & cost          |
| Sub-ms reads, ephemeral/hot data                | Redis                        | Cache, counters, sessions, geo, zsets; persistence is secondary |
| Flexible documents, mid-scale                   | MongoDB                      | Fine, but say why not Postgres+JSONB first                      |
| Full-text / faceted search                      | Elasticsearch                | Never the source of truth; sync via CDC                         |
| Analytics / OLAP                                | ClickHouse/BigQuery/Redshift | Columnar; don't run analytics on the OLTP DB                    |
| Global SQL with strong consistency              | Spanner/CockroachDB          | Pay in write latency; justify the need                          |
| Time-series                                     | Timescale/InfluxDB/M3        | Compression + downsampling built in                             |
| Blobs > few hundred KB                          | S3/GCS + CDN                 | Presigned URLs; store only metadata in DB                       |

### Queue/streaming choice

| Need                                                                         | Pick                          |
| ---------------------------------------------------------------------------- | ----------------------------- |
| Task queue, per-message ack, delays, low ops                                 | SQS (or RabbitMQ self-hosted) |
| High-throughput event backbone, replay, multiple consumers, ordering per key | Kafka                         |
| Simple pub/sub fan-out to online subscribers, fire-and-forget                | Redis pub/sub                 |
| Streaming compute over events                                                | Flink / Kafka Streams         |

### Consistency decision script (say this out loud per data path)

1. What breaks if a read is stale here? (Money/inventory → strong. Counts/feeds → eventual.)
2. What breaks if two writes race? (Uniqueness/limits → serializable, constraint, or atomic op. Independent appends → nothing.)
3. Can I confine strong consistency to a **single partition** (one account, one seat, one conversation)? If yes, single-writer + local transaction beats any distributed protocol.
4. Cross-entity flows → saga + idempotency, not distributed 2PC.

### Latency budget template

"End-to-end p99 target 200ms = TLS+edge ~30 + gateway 5 + service 20 + cache hit 2 (or DB 20) + fan-in/serialization 10, leaving headroom ~2x." Practicing one of these per mock makes you sound like you run production systems.

### Availability math

- 99.9% = 43 min down/month; 99.99% = 4.3 min/month.
- Serial components multiply: two 99.9% dependencies in the critical path ≈ 99.8%.
- Redundancy math and the honest caveat: correlated failures (same AZ, same bad deploy) break the independence assumption — mention it, it's senior-flavored.

---

## 8. Study Plan

8 weeks × ~1.5–2 hrs/day. Compress to 4 weeks by merging pairs. Given an already-strong distributed-systems base, bias time toward **mocks and articulation** — knowledge without out-loud fluency is the classic strong-engineer failure mode.

| Week | Study focus                                                                                                       | Practice (out loud, timed, on a whiteboard/Excalidraw)                  |
| ---- | ----------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| 1    | Framework + estimation drills (do 10 estimations); Tier 1 networking; caching end-to-end                          | URL shortener, Pastebin — full 45-min solo runs, recorded               |
| 2    | Tier 2 storage: replication, sharding, consistent hashing, LSM vs B-tree, indexes, isolation levels               | KV store, Typeahead                                                     |
| 3    | Tier 3 async: Kafka vs queues, delivery semantics, outbox/CDC, backpressure, stream windows                       | Notification service, Ad click aggregator, Web crawler                  |
| 4    | Feeds & caching failure modes (stampede/hot-key/penetration); pagination; counters                                | Twitter/news feed, Instagram, Top-K leaderboard                         |
| 5    | Real-time: WebSocket fleets, presence, chat sync, ordering; geospatial indexing                                   | WhatsApp, Slack presence, Uber, Yelp                                    |
| 6    | Consistency-critical: isolation traps (write skew), locks vs OCC, sagas, idempotency, ledgers                     | Payment system, Ticketmaster, hotel booking, wallet                     |
| 7    | Tier 4 theory sharpening: PACELC, consensus placement, fencing, clocks; multi-region patterns                     | Google Docs, distributed scheduler, rate limiter, S3-lite               |
| 8    | Media pipeline + metrics system; wrap-up polish; failure-mode drills ("kill any box in your diagram — now what?") | YouTube, Dropbox, Datadog-lite + **2 full mock interviews with humans** |

**Weekly rhythm:** 3 days study → 2 days practice questions → 1 day mock or recorded self-run → 1 day review notes into a personal "decision journal" (one page per question: the 3 key decisions + why). The journal is what you reread the morning of the interview.

**Mocks are non-negotiable for senior:** minimum 4 with a human (peers, Pramp, or paid). The senior bar is largely _communication under challenge_ — you can't practice being pushed back on alone.
