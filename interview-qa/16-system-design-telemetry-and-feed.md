# 16 — System Design Round: Real-Time Telemetry Collection, and a Twitter-like Feed

**Why this matters for an Event Hub role:** the system-design round is the one that decides your
level. Coding proves you can build; design proves you can be *trusted with a decision*. For this role
there are two prompts that are close to certain: **"design a real-time telemetry/event collection
system for a streaming service"** — which is literally the job description — and **"design a
Twitter-like feed"**, which Blind reports as this company's standard design question. This file works
both of them end to end, in the order you should say them out loud.

**How this file differs from the others:** no Q&A template. Two full worked designs, plus the
framework and the vocabulary to narrate them. The JD emphasises communication, and in this round
*how* you drive the conversation is graded as heavily as the boxes you draw.

---

> ## A standing note about every number in this file
>
> Every quantity below is either (a) an **assumption I have chosen for the exercise** and labelled as
> such, or (b) **arithmetic derived from those assumptions**, shown in full. None of them are facts
> about any real company or system. In the interview you do the same: state the assumption, do the
> arithmetic out loud, and invite correction — *"I'm assuming 50 million peak concurrent viewers; if
> your real number is 5 million, everything downstream divides by ten and the architecture is the
> same shape but a lot cheaper."* Interviewers grade the **method**, not the digits. A candidate who
> announces a number with no derivation has told them nothing.

---

## Part 0 — The framework

### The seven steps

```
1. CLARIFY REQUIREMENTS      functional + non-functional. Ask questions. Write the scope down.
2. ESTIMATE SCALE            QPS, storage, bandwidth. Show the arithmetic. Label assumptions.
3. API DESIGN                the contract at the edge. This constrains everything after it.
4. DATA MODEL                entities, access patterns, keys. Access patterns FIRST, schema second.
5. HIGH-LEVEL ARCHITECTURE   boxes and arrows. One diagram. Walk the happy path end to end.
6. DEEP DIVES                the 2-3 hard parts. Let the interviewer choose; offer a menu.
7. FAILURE MODES & TRADE-OFFS what breaks, what the user sees, what you chose and what it cost.
```

Steps 1–5 build a *defensible baseline*. Step 6 is where seniority shows. Step 7 is what separates a
senior from a staff answer — and it is the step most candidates run out of time for, which is why the
timebox below front-loads everything else.

### Timeboxing 45 minutes

| Minutes | Step | What "done" looks like |
|---|---|---|
| 0–5 | 1. Clarify | A written scope list on the board: 4–6 functional bullets, 4–6 NFRs, 2–3 explicit **non-goals**. |
| 5–10 | 2. Estimate | 4–6 numbers derived out loud. QPS, bytes/s, storage/day, and the one number that drives the design. |
| 10–14 | 3. API | 2–4 endpoints with the interesting fields. Not a full OpenAPI spec. |
| 14–18 | 4. Data model | Entities + the key/partitioning decision + which store for which access pattern. |
| 18–26 | 5. Architecture | One diagram. Walk one event/request end to end while you draw. |
| 26–38 | 6. Deep dives | Offer a menu, let them pick, go deep on 2. Have your own favourite ready if they don't pick. |
| 38–43 | 7. Failure & trade-offs | The failure table, the 10×/100× question, and what you'd do differently with more time. |
| 43–45 | Close | "Here's how I'd phase the build." Leaves them with a delivery instinct, not just a diagram. |

**Two timing rules that save the round:**
- If you are past minute 12 and still clarifying, you are failing. Announce your assumptions and move:
  *"I'll assume X, Y, Z — stop me if any of those are wrong."*
- Reserve the last 7 minutes. Set a literal reminder. An answer with no failure-mode discussion reads
  as inexperienced no matter how good the boxes are.

### How to narrate

This is the part that is actually being graded. The pattern that works:

1. **Say the step you are on.** *"I'm going to clarify requirements for about five minutes, then do
   the estimates, then draw."* Now they know you have a plan, and they know when to interrupt.
2. **Think out loud, including the rejected options.** *"Three ways to store aggregates: Postgres,
   DynamoDB, or a time-series database. I'm going to rule out Postgres for the write path because…"*
   A design where nothing was rejected looks like a design where nothing was considered.
3. **Name the trade-off before they ask for it.** *"This buys me sub-second freshness; it costs me
   exactly-once semantics, and here's why I think that's the right trade for a dashboard."*
4. **Ask before you go deep.** *"I can go deeper on the partitioning scheme, on the windowing and
   late-data handling, or on the serving layer. Which is most useful?"* This is the single highest-
   signal sentence in the round: it shows you know there is more than one hard part.
5. **Write on the board what you decide.** A running decision list means you can point at it in
   minute 40 instead of re-deriving it.
6. **Say "I don't know" cleanly when you don't.** *"I don't know the exact throughput ceiling of a
   single Kafka partition — I'd measure it. What I know is that the partition count is set by
   consumer parallelism, not by broker bandwidth, and here's why."* Bounded honesty reads as senior.
   A confident wrong number is the worst outcome in this round.

---

# DESIGN 1 — Real-time telemetry / event collection for a streaming service

*This is the StreamPulse design. If you have built it, you are not designing in the abstract — you are
describing something you have run. Use that: "I built a version of this locally; let me tell you where
the design bit me."*

## 1.1 Clarify requirements

**Questions I would actually ask** (ask 4–6, not all of them — pick the ones that change the design):

| Question | Why it changes the design |
|---|---|
| "What are the events, and who emits them?" | Player SDKs on TVs/mobile/web → unreliable networks, batching, clock skew, and you cannot force an SDK upgrade. |
| "What's the peak concurrency, and how spiky?" | A 10× live-sports spike is a different system from a flat VOD load. Spike shape decides autoscaling vs over-provisioning. |
| "How fresh must the dashboard be — seconds or minutes?" | Sub-10s forces a streaming hot path. Minutes lets you batch, which is far cheaper. This is the single biggest cost lever. |
| "Can we lose events? What's the tolerance?" | Lossless costs replication, acks, and idempotency everywhere. "99.9% is fine for dashboards, but billing must be exact" is the answer that produces a *two-path* design. |
| "Is this multi-tenant, and do tenants have different SLAs?" | Decides isolation: shared topics with tenant keys vs dedicated infrastructure. |
| "What's the retention, and who queries the history?" | Ad-hoc analyst queries over a year of data is a data-lake problem, not a database problem. |
| "Is billing computed from this data?" | If yes, you need exactness and an audit trail, and you build a reconciled batch path alongside the streaming one. |
| "Global or single-region? Is there a data-residency rule?" | GDPR-style residency forces regional pipelines, which changes everything about the aggregation layer. |

**Functional requirements (the scope I'd write on the board):**
1. Ingest batched telemetry events from millions of player clients: `session_start`, `heartbeat`,
   `buffering_start/end`, `bitrate_change`, `error`, `session_end`.
2. Compute near-real-time aggregates: concurrent viewers by title/region/device, rebuffer ratio,
   startup-time distribution, error rate — in 10-second tumbling and 1-minute hopping windows.
3. Serve a live dashboard (push, not poll) and historical queries.
4. Evaluate alert rules on the aggregates and notify.
5. Retain raw events for ad-hoc analysis and reprocessing.

**Non-functional requirements:**
- **Freshness:** event timestamp → visible on the dashboard, p99 < 5 s.
- **Availability of ingest:** the client-facing write path is the thing that must never be down —
  target four nines; the dashboard can degrade.
- **Durability:** once we return 2xx, the event is durable. No silent loss.
- **Correctness:** at-least-once delivery plus idempotent consumers ⇒ *effectively once* per event id.
- **Multi-tenancy:** one tenant's spike must not degrade another's freshness.
- **Cost:** bytes on the wire and on broker disk are the dominant cost; this is a real constraint, not
  a footnote.

**Explicit non-goals (say these — scoping is a senior signal):** not designing the player SDK; not
designing the CDN; no per-event exactly-once billing in v1 (a separate reconciled batch path handles
billing); no ML/anomaly detection in v1 (threshold rules only).

## 1.2 Estimate scale — with the arithmetic shown

> **Assumptions chosen for this exercise.** All labelled A1…A7. Change any one and I'll redo the
> arithmetic live.

```
A1  Peak concurrent viewers                         50,000,000
A2  Heartbeat interval per session                  1 per 10 s
A3  Lifecycle events (start/stop/bitrate/error)     +20% on top of heartbeats
A4  Average serialized event size (Avro)            200 bytes
A5  Peak : average ratio over a day                 3 : 1
A6  Compression ratio (lz4/zstd on batched Avro)    5 : 1
A7  Kafka replication factor                        3
```

**Event rate**

```
heartbeats   = 50,000,000 viewers ÷ 10 s      = 5,000,000 events/s
+ lifecycle  = 5,000,000 × 1.20               = 6,000,000 events/s   ← PEAK
average      = 6,000,000 ÷ 3 (A5)             = 2,000,000 events/s
per day      = 2,000,000 × 86,400             ≈ 1.7 × 10¹¹ events/day  (≈ 173 billion)
```

**Bandwidth**

```
peak, uncompressed = 6,000,000/s × 200 B      = 1.2 × 10⁹ B/s = 1.2 GB/s ≈ 9.6 Gbps
peak, compressed   = 1.2 GB/s ÷ 5 (A6)        = 240 MB/s      ≈ 1.9 Gbps
```

**Storage**

```
raw per day, compressed = 240 MB/s × 86,400 ÷ 3 (A5 → average is 1/3 of peak)
                        ≈ 240 MB/s × 28,800   ≈ 6.9 TB/day        ← call it 7 TB/day
Kafka disk (RF=3)       = 7 TB/day × 3        = 21 TB/day of broker disk
7-day raw retention     = 21 × 7              = 147 TB of broker disk   ← EXPENSIVE
30 days in object store = 7 × 30              = 210 TB                  ← CHEAP
1 year in object store  = 7 × 365             ≈ 2.6 PB
```

**➜ The number that drives the design.** 147 TB of replicated broker disk for one week of raw events
is the expensive line item, and it is why the architecture splits: **short retention on the broker
(hours to a few days, sized for reprocessing and consumer outages), long retention in object storage.**
Say this out loud — it is the moment the interviewer sees you costing a design rather than drawing one.

**Request rate (this is what the ingest tier is sized for)**

```
A8  Events per client batch                         500
batch requests = 6,000,000 events/s ÷ 500     = 12,000 requests/s   ← peak
payload/request = 500 × 200 B                 = 100 KB uncompressed, ~20 KB gzipped

A9  Batch requests one ingest instance sustains     1,500/s  (TO BE VALIDATED BY LOAD TEST)
instances      = 12,000 ÷ 1,500               = 8
+ headroom and AZ redundancy (×2.5)           ≈ 20 instances
```

**Partition count — and why the obvious calculation is the wrong one**

```
By bandwidth:
  A10  A single Kafka partition sustains ~10 MB/s comfortably   (ASSUMPTION — measure yours)
  240 MB/s ÷ 10 MB/s = 24 partitions

By consumer parallelism:
  A11  One consumer thread processes ~20,000 events/s          (ASSUMPTION — measure yours)
  6,000,000 ÷ 20,000 = 300 consumer threads
  A consumer group cannot have more useful members than partitions
  → we need at least 300 partitions

  → choose 384. Partition count is set by CONSUMER PARALLELISM, not broker bandwidth.
    And it is effectively one-way: increasing partitions later breaks key→partition affinity
    and therefore per-key ordering, so size it with headroom now.
```

That last point is a genuinely senior observation and worth stating unprompted.

**Serving layer**

```
A12  Concurrent dashboard sessions                  10,000
A13  Series watched per session                     5
A14  Update cadence                                 1 per 10 s
messages/s = 10,000 × 5 ÷ 10                  = 5,000 SSE messages/s
bytes/s    = 5,000 × 500 B                    = 2.5 MB/s        ← trivial bandwidth
```

**➜ Second design-driving insight:** the serving layer's problem is **10,000 long-lived connections**,
not bandwidth. That changes what you optimise: connection handling, backpressure and fan-out — which is
why the serving tier is reactive/non-blocking and the ingest tier is not.

## 1.3 API design

```
POST /v1/events:batch
Headers:
  Authorization: Bearer <JWT>          # tenant + app claims — authorization is per tenant
  Idempotency-Key: <uuid>              # per BATCH: safe client retry
  Content-Encoding: gzip
Body:
{
  "sdkVersion": "3.4.1",
  "sentAt": "2026-09-28T10:00:03.221Z",   # client clock — used ONLY to estimate skew, never to window
  "events": [
    { "eventId": "01J...", "sessionId": "s-99", "tenantId": "sportsnow",
      "titleId": "t-42", "region": "eu-west", "deviceType": "tv",
      "ts": "2026-09-28T10:00:01.100Z", "type": "heartbeat",
      "payload": { "bitrateKbps": 4200, "bufferedMs": 3100 } }
  ]
}

202 Accepted   { "accepted": 500, "rejected": 0 }
207            per-event results when some events fail validation (partial success)
400            malformed batch (ProblemDetail / RFC 7807)
413            batch too large
429            over quota or backpressure → Retry-After: <seconds>
```

**Design decisions to justify out loud, because each one will be probed:**

| Decision | Why |
|---|---|
| **Batch, not per-event** | 12,000 req/s instead of 6,000,000. TLS handshakes, HTTP headers and per-request overhead dominate a 200-byte payload. It also lets the client compress, which is where the 5:1 comes from. The cost is latency (up to one batch interval) and blast radius (one bad event can poison a batch — hence 207). |
| **202, not 200** | We accept for asynchronous processing; we have not aggregated it yet. Don't imply more than you did. |
| **Client-generated `eventId`** | Deduplication has to be possible after a client retry, and only the client knows the event is the same one. A ULID/UUIDv7 gives you a sortable id for free. |
| **`Idempotency-Key` per batch AND `eventId` per event** | Two layers: the batch key makes a whole-request retry free (the common case); the per-event id catches partial retries and cross-batch duplicates. |
| **429 + `Retry-After`** | Backpressure that reaches the client. Rejecting fast is a *feature*: it keeps the queue in the client, where it is cheapest, instead of in your heap. |
| **Server-assigned ingest timestamp alongside client `ts`** | Client clocks are wrong — badly wrong on TVs. Keep both: window on event time, but keep ingest time to measure skew and to bound lateness. |
| **Schema in a registry, with BACKWARD compatibility** | You cannot force millions of devices to upgrade. Old SDKs will be sending v1 events for years. This is a hard constraint, not a nice-to-have. |

Read path:

```
GET  /v1/metrics/{tenant}/{metric}?titleId=&region=&from=&to=&granularity=10s|1m|1h
GET  /v1/stream/metrics?tenant=&titleId=          # SSE: text/event-stream
     Last-Event-ID: <cursor>                      # resume after a reconnect
GET  /v1/alerts?tenant=&state=active
```

**Why SSE and not WebSocket:** the data flow is one-directional (server → browser), SSE is plain HTTP
so it traverses proxies and load balancers without upgrade negotiation, it reconnects automatically
with `Last-Event-ID` built into the browser API, and it is dramatically simpler to operate. I'd choose
WebSocket only if the client needed to send a high rate of messages back. Say the rule: *"bidirectional
and chatty → WebSocket; server push only → SSE; infrequent updates → long-poll or just poll."*

## 1.4 Data model and storage choices

**Access patterns first. Schema second. Always.**

| # | Access pattern | Latency need | Volume | Store |
|---|---|---|---|---|
| P1 | "Current value of metric M for title T" (dashboard tile) | < 10 ms | 4M keys hot | **Redis** — key/value, TTL |
| P2 | "Last 24h of metric M for title T" (chart) | < 100 ms | moderate | **DynamoDB** (or Cassandra) — key+range |
| P3 | "Push me every update for tenant X" | continuous | 5k msg/s | **in-memory fan-out from a Kafka consumer** |
| P4 | "Ad-hoc: rebuffer ratio by device type for all of last month" | seconds–minutes | PB-scale | **S3/object store + Athena/Trino/Spark** |
| P5 | "Replay the last 3 days into a new aggregation" | batch | 21 TB | **Kafka retention + S3 raw** |
| P6 | "Tenant, title, alert-rule configuration" | < 10 ms, transactional | tiny | **Postgres** (write model) + CDC to caches |

The aggregate key, and its cardinality:

```
key = (tenantId, metric, titleId, region, deviceType, windowStart)

A15  Concurrently active titles at peak     200,000
A16  Regions                                20
A17  Device types                           6      → but aggregate device separately, not crossed
cardinality (tenant × title × region)    ≈ 200,000 × 20 ≈ 4,000,000 series
aggregate records emitted = 4,000,000 per 10 s window = 400,000 records/s
state store size = 4,000,000 × ~200 B    ≈ 800 MB, spread over 300 tasks ≈ 2.7 MB each  ✓ fits
```

**➜ Third insight:** 400,000 aggregate records/s is a 15× reduction from 6M raw events/s, but it is
still far too much to write to DynamoDB directly at a sane cost. So: **roll up before you persist.**
Write the 10-second windows to Redis (hot, TTL 2 h) for the dashboard, and only persist the
**1-minute** rollups to DynamoDB — a further 6× reduction to ~67,000 writes/s, batched. Say the cost
sentence: *"Granularity in the durable store is a cost decision, and 10-second granularity beyond an
hour is something nobody actually queries."*

DynamoDB single-table shape (this is BS-04 / Phase 4 in StreamPulse):

```
PK = TENANT#sportsnow#METRIC#rebuffer_ratio#TITLE#t-42
SK = WINDOW#2026-09-28T10:00:00Z
attrs = { value, count, region, deviceType, ttl }
GSI1: PK = TENANT#sportsnow#REGION#eu-west , SK = WINDOW#...     # "by region" access pattern
TTL attribute expires warm rows after 30 days → cold queries go to S3
```

Note what this shape *cannot* do: no ad-hoc "group by anything". That is deliberate — P4 lives in the
data lake. **Naming what a store cannot do is more convincing than listing what it can.**

S3 layout:

```
s3://telemetry-raw/tenant=sportsnow/dt=2026-09-28/hour=10/part-00013-<uuid>.ndjson.gz
                                     ▲             ▲
                                     └── partition columns: a query for one day reads one prefix
```

The small-files problem is real and worth naming: a flush every 30 seconds across 300 tasks is 36,000
objects per hour, which makes any query engine miserable. Hence a compaction job (hourly, once the
window is closed plus a grace period) that merges into large columnar files — Parquet if the query
pattern is columnar, which for analytics it is. This is StreamPulse BS-06 / REQ-AW-01.

## 1.5 High-level architecture

```
                                    ┌──────────── control plane (Postgres) ───────────┐
                                    │ tenants · quotas · titles · alert rules         │
                                    │        │ outbox (same txn)                      │
                                    └────────┼────────────────────────────────────────┘
                                             │ Debezium CDC
                                             ▼  config.changes.v1  ───────┐ (cache refresh,
                                                                          │  everyone subscribes)
  player SDKs                                                             │
  (TV/mobile/web)                                                         ▼
      │  HTTPS, gzipped batches of ~500 events
      ▼
  ┌──────────┐    ┌────────────────────────────────────────┐
  │ CDN /    │───▶│  INGEST TIER  (stateless, autoscaled)   │
  │ L7 LB /  │    │  · authn/authz (JWT → tenant)           │
  │ API GW   │    │  · schema + semantic validation          │
  └──────────┘    │  · per-tenant quota (distributed bucket) │
                  │  · dedupe: SET NX EX on eventId (Redis)  │
                  │  · bounded in-flight → 429 + Retry-After │
                  │  · idempotent producer, acks=all         │
                  └───────────────┬────────────────────────┬─┘
                                  │ key = sessionId        │ invalid / unparseable
                                  ▼                        ▼
             ┌──────────────────────────────────┐    telemetry.raw.dlq
             │  telemetry.raw.v1                │
             │  384 partitions · RF=3 · Avro    │
             │  retention: 72 h (reprocess win) │
             └───┬──────────────────────┬───────┘
                 │ HOT PATH             │ COLD PATH
                 ▼                      ▼
   ┌──────────────────────────┐   ┌─────────────────────────────┐
   │ STREAM PROCESSOR         │   │ ARCHIVER                    │
   │ Kafka Streams / Flink    │   │ batch by size+time           │
   │ · repartition by agg key │   │ → S3 gzip NDJSON             │
   │ · 10 s tumbling windows  │   │   tenant=/dt=/hour=          │
   │ · 1 m hopping windows    │   └──────────┬──────────────────┘
   │ · grace period for late  │              │ hourly, after window close
   │ · suppress until close   │              ▼
   └──────────┬───────────────┘   ┌─────────────────────────────┐
              │ telemetry.agg.v1  │ COMPACTION (serverless)      │
              ▼                   │ small files → Parquet        │
   ┌──────────────────────────┐   └──────────┬──────────────────┘
   │ SINK WRITER              │              ▼
   │ · Redis  (10 s, TTL 2 h) │        ┌──────────────┐
   │ · DynamoDB (1 m rollup)  │        │ query engine │  (Athena / Trino / Spark)
   │ · idempotent writes      │        │ ad-hoc, P4   │
   └──────────┬───────────────┘        └──────────────┘
              │
              ├──▶ ALERT ENGINE ──▶ alerts.v1 ──▶ notification fan-out (SQS / RabbitMQ / webhook)
              │     rules from CDC cache; dependency-graph BFS suppresses downstream alerts
              ▼
   ┌────────────────────────────────────────────┐
   │ QUERY / SERVING TIER (reactive, non-block) │
   │  · Kafka consumer → in-memory per-tenant   │
   │    multicast sink                          │
   │  · SSE fan-out, 15 s heartbeat             │
   │  · onBackpressureLatest (drop stale, keep  │
   │    newest — correct for a dashboard)       │
   │  · Last-Event-ID resume from a ring buffer │
   │  · REST reads: Redis → DynamoDB → S3       │
   └────────────────────┬───────────────────────┘
                        ▼
                   dashboards, MCP server, partner APIs
```

**Walk one event through it while you draw** — this is the narration that makes the diagram land:

> A TV player buffers 500 events over 5 seconds and POSTs them gzipped. The LB terminates TLS and
> routes to any ingest instance — they're stateless. The instance validates the JWT, extracts the
> tenant, checks the distributed quota bucket, validates each event against the Avro schema, and does
> a Redis `SET NX EX` on each `eventId` to drop duplicates from a client retry. Survivors go to the
> idempotent Kafka producer keyed on `sessionId` — so all of one session's events land on one
> partition and stay ordered, which the session-duration and rebuffer calculations depend on. If the
> in-flight semaphore is saturated we return 429 with `Retry-After` rather than buffering; the client
> queue is cheaper than our heap. We return 202.
>
> Two consumers read that topic independently. The archiver batches to S3 — that's the cold path and
> it's the durable source of truth for replay. The stream processor repartitions by the aggregation
> key, because the *storage* key (`sessionId`) and the *aggregation* key (`tenant|title|region`) are
> different — that repartition is a real shuffle and a real cost, and I'd call it out. It maintains
> 10-second tumbling windows with a grace period for late arrivals, suppresses emission until the
> window closes so downstream sees one result per window rather than a stream of partials, and
> publishes to the aggregate topic.
>
> The sink writer fans that out: Redis for the current value, a 1-minute rollup to DynamoDB for
> history. The serving tier consumes the same aggregate topic into an in-memory per-tenant sink and
> pushes to every subscribed SSE connection. End to end, event timestamp to pixel, the budget is
> under 5 seconds and most of it is the 10-second window itself — which is worth saying, because it
> means "make it fresher" means "make the window smaller", not "make the code faster."

## 1.6 Deep dives

Offer the menu: *"The parts I think are hardest here are the partition-key choice and hot partitions,
the windowing and late-data semantics, the delivery guarantees, and backpressure through the serving
tier. Which would you like?"*

### Deep dive A — Partition key choice and hot partitions

The key decides three things at once: **ordering**, **parallelism**, and **hot-spotting**.

| Candidate key | Ordering you get | Parallelism | Hot-partition risk |
|---|---|---|---|
| `sessionId` | per session ✅ — exactly what rebuffer/duration need | excellent: millions of distinct sessions spread evenly | low |
| `tenantId` | per tenant | **terrible** — one tenant is 40% of traffic → one partition takes 40% | **severe** |
| `titleId` | per title | poor during a live event: one title *is* the spike | **severe** |
| random / round-robin | none | perfect | none — but you lose ordering, which kills session-state logic |
| `tenantId + sessionId` composite | per session, with tenant locality | good | low, and it enables per-tenant topic isolation later |

**Choose `sessionId`.** Reasons in order: (1) the aggregations that need ordering are per-session
(session duration, rebuffer ratio over a session); (2) session cardinality is enormous and uniformly
distributed, so partitions stay balanced; (3) it makes the dedup check partition-local.

**The hot-partition failure, concretely.** Suppose a single mega-event puts 30% of traffic on one
title. With `sessionId` keying that is fine — the sessions still spread. But the *aggregation* stage
repartitions by `(tenant, title, region)`, and now one key is 30% of the stream. That partition's
consumer falls behind, its lag grows, and freshness degrades **for that title only**. Mitigations, and
their costs:

```
1. Two-stage aggregation (the standard fix — this is what I'd do):
     stage 1: key = (tenant, title, region, shard)  where shard = hash(sessionId) % 16
              → 16 partial aggregates computed in parallel, load spread 16×
     stage 2: key = (tenant, title, region)
              → merge 16 partials into the final value
   Cost: one extra shuffle and one extra window's worth of latency.
   Works because our aggregates are COMMUTATIVE and ASSOCIATIVE (counts, sums, min, max, HLL).
   It would NOT work for a non-mergeable aggregate like an exact median — for which you'd use
   a mergeable sketch (t-digest) and accept approximate percentiles. Say this: knowing WHICH
   aggregates are mergeable is the actual insight.

2. Detect and isolate: measure per-key throughput, route known-hot keys to a dedicated topic.
   Cost: operational complexity, and it only handles hotness you predicted.

3. Over-partition up front (384 rather than 32) so a hot key is a smaller share of one consumer's
   work. Cost: more metadata, more open file handles, more rebalance time.
```

### Deep dive B — Windowing, watermarks and late data

Three clocks, and the whole difficulty lives in the gap between them:

| Clock | What it is | Problem |
|---|---|---|
| **Event time** | the client's `ts` | client clocks are wrong (TVs are notoriously bad), and events arrive late or out of order |
| **Ingest time** | when the server received it | correct but wrong *semantically* — it moves an event into the wrong window during a network delay |
| **Processing time** | when the operator handled it | useless for correctness, useful for SLOs |

**Window on event time.** It is the only one that produces the same answer on a replay, and replay is
non-negotiable for a pipeline you will fix bugs in.

```
                    grace period (e.g. 30 s)
window [10:00:00 – 10:00:10) ├──────────────────┤
                             ▲                  ▲
                      window closes        watermark passes: emit final result
   arrives 10:00:04  ──▶ included ✅
   arrives 10:00:25  ──▶ late, but within grace ──▶ included, window updated ✅
   arrives 10:02:00  ──▶ beyond grace ──▶ DROPPED (and counted!) ❌
```

Decisions to state and justify:
- **Grace period** is a direct trade of *completeness* against *freshness and state size*. 30 seconds
  covers mobile-network hiccups; it does not cover a device that was offline for an hour. Pick the
  number from the measured arrival-delay distribution, not from intuition.
- **Suppression until window close** means downstream sees one final value per window instead of a
  stream of increasing partials. Without it, a dashboard chart visibly climbs during each window,
  which users report as a bug. The cost is that you hold state for the whole window plus grace.
- **Dropped-late events must be counted as a metric.** A silently dropped event is the worst kind of
  data-quality bug — nobody notices for a quarter. `late_events_dropped{tenant}` on a dashboard, with
  an alert. State this unprompted; it is a small detail that reads as operational maturity.
- **Clock skew:** compare client `ts` against server ingest time and export the distribution. A
  broken SDK build shows up here first, and it is also how you decide the grace period empirically.
- **Very late data** (a device offline for hours) goes to the cold path only. The dashboard is a live
  view; the data lake is the record of truth. Reprocessing the lake gives the corrected historical
  number. That is a **lambda-style** split and I would name it as such along with its cost: two code
  paths that can disagree, which is exactly the criticism of lambda architecture and the argument for
  a kappa (replay-the-log) approach instead. Which one is right depends on whether you can afford to
  reprocess from the log — at 147 TB/week, you can, for 3 days.

### Deep dive C — Delivery semantics

```
Client → Ingest:     at-least-once. The client retries on timeout; it MUST, because it cannot know
                     whether a timed-out request was processed. Hence eventId + Idempotency-Key.
Ingest → Kafka:      idempotent producer (enable.idempotence=true, acks=all). The broker dedupes
                     retries by producer-id + sequence number, so a retry after a network blip
                     does not duplicate.
Kafka → Aggregator:  exactly-once processing within Kafka (transactional read-process-write) is
                     available and appropriate here, because the whole cycle is inside Kafka.
Aggregator → Sinks:  at-least-once + IDEMPOTENT WRITES. There is no exactly-once across a system
                     boundary to Redis/DynamoDB/S3. Conditional writes keyed by (key, windowStart)
                     make redelivery harmless. This is the honest answer and the one interviewers
                     are listening for.
```

Say the sentence that shows you understand the boundary: **"'Exactly-once' is a property of a closed
system. Across a boundary you get at-least-once plus idempotency, which is *effectively* once, and
that is what I'd build."** Then name the cost of Kafka's exactly-once mode: transactional overhead,
higher latency, and a read-committed consumer that cannot see records until the transaction commits —
which directly eats into the 5-second freshness budget. That cost is why I would use EOS for the
aggregation stage and *not* pretend it extends to the sinks.

### Deep dive D — The serving layer and backpressure

The interesting property: a dashboard does **not** want a queue. If the browser is slow, the correct
behaviour is "show the newest value", not "deliver every value you missed".

```
Kafka consumer ──▶ per-tenant multicast sink ──▶ N SSE subscribers
                        │
                        └─ onBackpressureLatest: keep only the most recent element per slow subscriber
                           (contrast: onBackpressureBuffer = unbounded memory growth → OOM
                                      onBackpressureError  = kill the connection
                                      onBackpressureDrop   = drop newest, which is exactly backwards)
```

- **Slow consumer** (a laptop that went to sleep): the sink drops all but the latest for *that*
  subscriber. Other subscribers are unaffected. Memory is bounded by construction.
- **Heartbeat every 15 s** so idle connections aren't reaped by intermediate proxies, and so the
  client can detect a dead connection.
- **`Last-Event-ID` resume**: keep a small per-tenant ring buffer of recent updates so a reconnecting
  browser gets the gap rather than a blank chart. Bounded — if the gap is bigger than the buffer, tell
  the client to do a full REST read. Say the bound out loud; unbounded resume is a memory leak.
- **Connection distribution**: 10,000 long-lived connections across N instances means a rolling
  deploy drops all of them at once. Stagger the rollout, reconnect with **jittered** backoff (without
  jitter, 10,000 clients reconnect in lockstep and you've built a thundering herd into your deploy
  process), and keep instances behind an LB that supports long-lived HTTP.

### Deep dive E — Multi-tenancy and noisy neighbours

The requirement is "`sportsnow`'s spike must not degrade `kidsplus`". Mechanisms, cheapest first:

| Mechanism | Isolates | Cost |
|---|---|---|
| Per-tenant quota at ingest (distributed token bucket in Redis) | the write path | needs a distributed counter; adds a Redis round trip |
| Per-tenant metrics and MDC tags everywhere | *diagnosis*, not isolation — but you cannot fix what you cannot see | negligible; do it always |
| Per-tenant bulkheads on outbound calls (alert webhooks) | one partner's slowness from all tenants | thread/permit overhead per tenant |
| Separate consumer groups per tenant tier | aggregation lag | more consumers, more partitions |
| Separate topics (or clusters) for premium tenants | everything | real money and real ops; only for contractual SLAs |

The escalation ladder is the answer: quotas and observability for everyone, bulkheads where a third
party is involved, physical separation only where a contract requires it. **Say why you would not
start at the bottom:** dedicated infrastructure per tenant is the most effective and least efficient
isolation there is.

## 1.7 Failure modes — and what each one does to the user

| Failure | Blast radius | System behaviour | What the user sees | Mitigation |
|---|---|---|---|---|
| One ingest instance dies | ~1/20 of in-flight batches | LB removes it; clients retry | one retried request | stateless tier, health checks, graceful drain (StreamPulse BS-01) |
| Kafka broker lost | partitions it led | leader election from ISR, brief unavailability | a few seconds of 5xx/429 on ingest | RF=3, `min.insync.replicas=2`, `acks=all` |
| Kafka cluster unavailable | **the whole write path** | ingest cannot publish | 503s; clients buffer and retry | short local spool-to-disk with a hard bound, then 429; honest degradation, no silent loss |
| Aggregator consumer lag grows | freshness | lag climbs, dashboards go stale | stale charts + a "data delayed" banner | autoscale on lag (KEDA), over-partition, two-stage aggregation for hot keys |
| Poison message (unparseable, or a bug that throws on one record) | the partition it's on — **everything behind it stalls** | retries, then route to DLQ after N attempts | brief stall, then recovery | `@RetryableTopic` + DLT, alert on DLT rate, tooling to inspect and replay |
| Redis down | hot reads | serving tier falls through to DynamoDB | slower dashboard, higher cost | circuit breaker with fallthrough; treat Redis as a cache, never as truth |
| DynamoDB throttles | historical queries and rollup writes | retry with exponential backoff + jitter | charts load slowly | adaptive capacity, backoff+jitter, buffer the write path |
| S3 unavailable | cold path | archiver spools to local disk (bounded), replays on recovery | nothing immediately; analytics gap if prolonged | breaker + bounded spool + replay (StreamPulse BS-04 / REQ-RS-02) |
| Slow SSE consumer | that one connection | `onBackpressureLatest` drops intermediate values | skipped frames in a chart — correct behaviour | bounded per-subscriber buffers |
| Region loss | everything in that region | failover to the standby region | outage until DNS/traffic shifts | active-active ingest with regional pipelines; cross-region replication for aggregates; **state the RPO/RTO** |
| A client SDK bug floods us | potentially everything | per-tenant quota holds the line; 429s | that tenant's events are throttled | quotas are load-bearing, not decorative; plus per-SDK-version metrics to spot it |
| Schema change breaks old SDKs | all clients on the old version | registry rejects the incompatible change **at build time** | nothing — the deploy never happens | BACKWARD compatibility enforced in CI |

**The one I'd raise unprompted:** *poison message and consumer lag are the two failures that hurt
most, because both are silent for a while and both cascade.* Lag is invisible until the dashboard is
stale enough for someone to complain, and by then you are hours behind with a retention clock
running. So: alert on consumer lag and on DLT rate as *first-class SLIs*, not as afterthoughts.

## 1.8 Scaling to 10× and 100×

**10× (60M events/s peak).** Mostly mechanical, and that is the point of the design:

| Component | What changes | What breaks first |
|---|---|---|
| Ingest tier | 20 → ~200 instances. Stateless, so it just scales. | the load balancer's connection limit; per-tenant quota lookups hammering Redis → move to a local token bucket with periodic global reconciliation |
| Kafka | 384 → ~3,840 partitions | **this is the first real wall.** Partition count per cluster has practical limits (controller metadata, rebalance time, open file handles). Answer: multiple clusters partitioned by tenant, or a topic per tenant tier. |
| Aggregation | 300 → 3,000 tasks | shuffle volume during repartition; rebalance storms on every deploy. Answer: static membership + cooperative rebalancing so a rolling deploy doesn't reshuffle everything. |
| Storage | 7 → 70 TB/day | Kafka retention becomes unaffordable; drop raw retention to hours and lean on S3 for replay |
| Serving | 10k → 100k connections | connection count per instance; move to a dedicated push tier, consider a pub/sub edge |

**100× (600M events/s).** At this point the honest answer is *"the architecture changes, it doesn't
stretch"*, and saying so is the senior move:

1. **Sample or pre-aggregate at the edge.** The biggest win available: have the SDK send a
   pre-aggregated summary every 10 s instead of raw heartbeats. This is a 10–100× reduction *before
   any of your infrastructure is involved*, and it is the thing a real streaming company does. The
   cost is losing per-event detail and needing an SDK rollout you cannot force — which is why you
   design for it early, even if you don't enable it.
2. **Regional pipelines with regional aggregation**, and only roll up globally. Global centralisation
   at this rate is a cross-region bandwidth bill nobody will sign.
3. **Tiered fidelity:** full per-event capture for a sampled percentage of sessions, aggregates for
   the rest. Most questions are answerable from a 1% sample.
4. **Sketches instead of exact values:** HyperLogLog for unique viewers, t-digest for percentiles.
   Bounded memory, mergeable, approximate. Name the trade: you can no longer answer "exactly how
   many", and for a dashboard nobody ever needed to.
5. **Purpose-built storage**, e.g. a columnar real-time analytics store, rather than
   Redis + DynamoDB + S3 glued together.

## 1.9 Explicit trade-offs

| Decision | What I chose | What it cost | When I'd choose differently |
|---|---|---|---|
| Push vs poll to the dashboard | SSE push | long-lived connections complicate deploys and LBs | < 1,000 users and a 30 s refresh → just poll; it's a tenth of the work |
| Stream processing vs micro-batch | streaming (Kafka Streams / Flink) | stateful operations, rebalances, more ops burden | if freshness could be 5 minutes, a micro-batch job is far cheaper and simpler |
| Kafka vs a managed queue (Kinesis/SQS) | Kafka | you run it (or pay for managed); ops burden is real | Kinesis if the org is AWS-native and the shard model fits; SQS never — no replay, no ordering, no consumer groups |
| Exactly-once everywhere vs idempotency | idempotency at the boundaries | duplicates are possible in transit and must be tolerated by design | if this were billing, I'd add a reconciled batch path, not try to make streaming exact |
| Redis + DynamoDB + S3 vs one store | three stores, one per access pattern | three failure modes, three ops stories, consistency questions between them | one store if the scale allowed it — complexity is a cost you pay every day |
| 10 s window granularity | 10 s hot, 1 m persisted | "why can't I see 1-second resolution from last March" | shorter windows if freshness is the product; longer if cost is |
| Batch ingest | 500 events/batch | up to one batch-interval of added latency; partial-failure handling | per-event if events are rare and individually important (payments, not heartbeats) |

## 1.10 How I'd phase the build

```
Phase 1 (weeks 1-3)   Contracts + ingest + Kafka + S3 archive.
                      Nothing is aggregated yet, but NOTHING IS LOST. Durability first:
                      you can always compute later from data you kept, and never from data you dropped.
Phase 2 (weeks 4-6)   Stream aggregation, one metric end to end, Redis hot store, a REST read API.
                      One metric proves the whole pipeline; ten metrics prove nothing extra.
Phase 3 (weeks 7-8)   SSE serving + the dashboard. Now it's a product, not a pipeline.
Phase 4 (weeks 9-10)  Alerting + the control plane + CDC-driven config.
Phase 5 (weeks 11-12) Historical store, compaction, the query engine.
Phase 6 (ongoing)     Load testing, tuning, chaos drills, multi-region.

Cross-cutting from day one, not bolted on: schema registry, per-tenant metrics, consumer-lag
alerting, DLQ handling, and a replay tool. A pipeline without a replay tool is a pipeline you
cannot fix.
```

Ending on a phasing plan is deliberate: it tells the interviewer you ship systems, not just draw them.

---

# DESIGN 2 — A Twitter-like feed

*Blind reports this exact prompt at this company. It is also the canonical test of one idea —
**fan-out on write vs fan-out on read, and the hybrid** — so the interviewer is listening for that
specific reasoning. Get there quickly and then go deeper than the textbook answer.*

## 2.1 Clarify requirements

**Questions to ask:**
- "Is the feed chronological or ranked?" (Ranked changes the serving layer completely.)
- "Is it a following-graph feed, or a recommendation feed?" (The second is a different system.)
- "Can the feed be slightly stale — seconds? minutes?" (This is the single biggest lever.)
- "Do we need to handle celebrity accounts with tens of millions of followers?" (If yes — and the
  answer is always yes — the hybrid is mandatory, not optional.)
- "Media, or text only?" (Media is a separate CDN/blob-storage sub-problem; usually scope it out.)
- "Read-your-own-writes — must my own post appear instantly in my own feed?" (Yes, always, and it's
  a special case worth handling explicitly.)

**Functional:** post; follow/unfollow; view a home timeline of people you follow, newest-first,
paginated; view a user's own timeline.
**Non-functional:** home-timeline read p99 < 200 ms; a post visible to followers within a few seconds;
availability over consistency (a feed that is 5 seconds stale is fine; a feed that 500s is not);
massively read-heavy.
**Non-goals:** search, DMs, notifications, media processing, ads, the ranking *model* itself (I'll
design the ranking *pipeline*, not the model).

## 2.2 Estimate scale

```
B1  Daily active users                              300,000,000
B2  Feed opens per user per day (incl. refreshes)   15
B3  Fraction of DAU who post, per day               10%
B4  Posts per posting user per day                  2
B5  Median following-count (→ fan-out factor)       200
B6  Peak : average ratio                            3 : 1
B7  Celebrity account follower count                50,000,000
```

**Reads**

```
feed reads/day = 300,000,000 × 15        = 4.5 × 10⁹
average reads/s = 4.5e9 ÷ 86,400         ≈ 52,000 reads/s
peak reads/s    = 52,000 × 3             ≈ 156,000 reads/s   → call it 150k/s
```

**Writes**

```
posts/day      = 300,000,000 × 10% × 2   = 60,000,000
average/s      = 60,000,000 ÷ 86,400     ≈ 700 posts/s
peak/s         = 700 × 3                 ≈ 2,100 posts/s
```

**➜ The number that drives the entire design:**

```
read : write  =  52,000 : 700  ≈  75 : 1
```

Seventy-five reads for every write. **Do the work on the write path.** Precompute each user's timeline
when a post is created, so a read is a single cache lookup instead of a query across 200 authors'
timelines. That is fan-out-on-write, and the read:write ratio is the whole justification.

**Fan-out write amplification — the arithmetic that forces the hybrid**

```
Normal user:
  peak timeline inserts = 2,100 posts/s × 200 followers (B5) = 420,000 inserts/s
  → large but very shardable: sorted-set inserts across a Redis cluster. Fine.

Celebrity (B7):
  ONE post = 50,000,000 timeline inserts.
  At the cluster's 420,000 inserts/s total fan-out budget:
        50,000,000 ÷ 420,000 ≈ 119 SECONDS
  …of the ENTIRE cluster's write capacity, for a single tweet.
  And there are thousands of such accounts, several of whom post in the same minute.

  → Fan-out-on-write ALONE does not work. This arithmetic is the answer to the question,
    and it is why you show it rather than just asserting "celebrities are a problem."
```

**Timeline cache sizing**

```
B8  Timeline entries cached per user                800
B9  Bytes per entry in a Redis sorted set (id + score + overhead)   ~64 B  (ASSUMPTION — encoding-dependent)
B10 Users whose timeline we cache (active in last 3 days)           150,000,000

per user  = 800 × 64 B                   = 51.2 KB
total     = 150,000,000 × 51.2 KB        ≈ 7.7 TB
nodes     = 7.7 TB ÷ 50 GB usable/node   ≈ 154 nodes, × 2 for replicas ≈ 308 nodes

→ That is a LOT of money, so both caps are load-bearing decisions, not arbitrary:
    · 800 entries: nobody scrolls past ~800 posts. Deeper history → fall back to a read-time query.
    · only active users: materialising a timeline for a user who hasn't opened the app in a
      month is pure waste. Build it lazily on their next login.
  Halving B8 to 400 halves the bill to ~154 nodes. Say that sensitivity out loud — it shows
  you know which knob actually controls the cost.
```

**Post storage**

```
B11 Bytes per stored post (text + metadata)          500 B
per day   = 60,000,000 × 500 B           = 30 GB/day
per year  = 30 GB × 365                  ≈ 11 TB/year     ← trivial. Posts are NOT the storage problem.
                                                             TIMELINES are (7.7 TB of cache, live).
```

That inversion is worth stating: the source-of-truth data is small; the *derived, precomputed* data is
what costs money. It reframes the whole design as a caching problem.

## 2.3 API design

```
POST /v1/posts
  { "text": "...", "replyToId": null }
  → 201 { "postId": "01J...", "createdAt": "..." }     # snowflake/ULID: time-sortable

GET  /v1/feed?limit=20&cursor=<opaque>
  → 200 {
      "items": [ { "postId", "authorId", "text", "createdAt", "metrics" } ],
      "nextCursor": "eyJ0cyI6MTc..."                   # opaque: encodes (score, postId)
    }

POST   /v1/users/{id}/follow
DELETE /v1/users/{id}/follow
GET    /v1/users/{id}/posts?cursor=          # a user's own timeline
```

**Cursor pagination, not offset — and be ready to explain why properly, because this is a favourite
follow-up:**

| | Offset (`?page=3&size=20`) | Cursor (`?cursor=<opaque>`) |
|---|---|---|
| Correctness on a changing list | **broken**: new posts arrive at the head, so page 2 re-shows items from page 1, and items shift past the boundary and are never shown | stable: the cursor is a position in the *ordering*, not a count |
| Cost at depth | `OFFSET 100000` makes the database scan and discard 100,000 rows | O(1): seek directly to the key |
| Works with a Redis sorted set | no | yes: `ZREVRANGEBYSCORE key (lastScore -inf LIMIT 0 20` |
| Can you jump to page 50? | yes | no — and for an infinite feed nobody wants to |

The cursor must encode **(score, postId)**, not just the score, because scores collide — two posts
with the same timestamp would otherwise be skipped or repeated at a page boundary. Make it opaque
(base64 of a small struct) so you can change the encoding later without breaking clients. Mentioning
the tiebreaker unprompted is a strong signal.

## 2.4 Data model

```
users        (userId PK, handle, displayName, followerCount, isCelebrity)
posts        (postId PK = snowflake, authorId, text, createdAt, replyToId)
             sharded by postId; postId is time-sortable so range scans are cheap
follows      (followerId, followeeId, createdAt)
             TWO indexes, because both directions are hot and they answer different questions:
               by followerId  → "who do I follow?"      (read-time fan-in for celebrities)
               by followeeId  → "who follows me?"       (write-time fan-out)  ← the big one

timeline:<userId>   Redis SORTED SET
                    member = postId, score = rankingScore (or createdAt for chronological)
                    ZADD on fan-out; ZREMRANGEBYRANK to cap at 800; TTL on inactive users
```

**Store choices:**

| Data | Store | Why |
|---|---|---|
| Posts | sharded key-value / wide-column (Cassandra, DynamoDB), sharded by `postId` | write-heavy, immutable, always read by id or by author+time range. No joins wanted. |
| Follow graph | sharded relational, or a purpose-built graph/KV store, with both directions indexed | the write-time fan-out query ("who follows me") must be fast and can return 50M rows — it needs to be *paginated*, which is itself a design detail |
| Timelines | Redis cluster, sorted sets | the derived read cache; rebuildable from posts + follows, which is why it's acceptable to lose it |
| Media | object store + CDN | out of scope, but say it |

**"Rebuildable" is the key property of the timeline store.** It means you can treat it as a cache: no
replication guarantees needed, no backup, and a lost shard is a latency event rather than a data-loss
event. Say that — it's what justifies putting 7.7 TB of critical-path data in a cache.

## 2.5 Architecture

```
                                      ┌─────────────────────────────────────────┐
  POST /v1/posts ──▶ ┌───────────┐    │ FAN-OUT WORKERS (consume post events)    │
                     │ Post svc  │───▶│  1. look up followers (paginated!)       │
                     │ · validate│    │  2. is the author a CELEBRITY?           │
                     │ · store   │    │       yes → SKIP fan-out entirely        │
                     │ · emit    │    │       no  → for each ACTIVE follower:    │
                     └─────┬─────┘    │            ZADD timeline:<f> score postId│
                           │          │            ZREMRANGEBYRANK … cap at 800  │
                     posts.v1         │  3. push to any live WebSocket session   │
                     (Kafka)          └──────────────────┬──────────────────────┘
                           │                             │
                           ▼                             ▼
                   ┌──────────────┐            ┌──────────────────────┐
                   │ post store   │            │ Redis: timeline:<u>  │
                   │ (sharded KV) │            │  sorted sets, capped │
                   └──────┬───────┘            └──────────┬───────────┘
                          │                               │
  GET /v1/feed ──▶ ┌──────┴───────────────────────────────┴────────────┐
                   │ FEED SERVICE                                       │
                   │  1. ZREVRANGEBYSCORE timeline:<me>  → precomputed   │
                   │  2. celebrities I follow: query THEIR author        │
                   │     timelines directly (fan-out on READ)            │
                   │  3. MERGE the two by score  ← the hybrid, right here │
                   │  4. hydrate postIds → post objects (multi-get,       │
                   │     cached; this is where an N+1 would kill you)     │
                   │  5. filter (blocks, deletes, muted), then return     │
                   └───────────────────────┬────────────────────────────┘
                                           ▼
                                      client (cursor for the next page)
```

## 2.6 The core insight: fan-out on write vs read, and the hybrid

| | Fan-out on **write** (push) | Fan-out on **read** (pull) |
|---|---|---|
| On post | write the postId into every follower's timeline | write nothing but the post |
| On read | one cache lookup — **O(1)** | query the timelines of everyone you follow and merge — **O(following)** |
| Read latency | excellent (single `ZREVRANGEBYSCORE`) | poor: 200 queries + a merge, per feed open |
| Write cost | O(followers) — **catastrophic for celebrities** | O(1) |
| Storage | large: every post is duplicated per follower | small |
| Staleness | a follower who follows you *after* your post won't see it (unless you backfill) | always current |
| Fits | the 99.9% of users with hundreds of followers | the 0.1% with millions |

**The hybrid, and exactly where the line goes:**

```
if (author.followerCount < CELEBRITY_THRESHOLD)   // e.g. 100,000 — a TUNABLE, not a law
      → fan out on write to active followers
else  → do nothing at write time; the post sits only in the author's own timeline

At read time:
   precomputed = ZREVRANGEBYSCORE timeline:<me> ...                    ← 1 call
   celebs      = the (few) celebrity accounts I follow                 ← from my follow list, cached
   recent      = for each celeb: their author timeline, recent slice   ← ~5-20 calls, parallel
   feed        = merge-by-score(precomputed, recent), take 20
```

**Why the threshold is a tunable and what it trades:**

```
Lower threshold (more accounts treated as celebrities)
   → less write amplification, cheaper fan-out
   → more read-time work and higher read latency for everyone who follows them

Higher threshold
   → fast reads
   → write spikes when a big account posts; fan-out lag; timelines diverge

Pick it from the actual follower-count distribution, and make it a runtime config, not a constant.
Then measure: the right answer is wherever total cost (write amplification + read latency) is minimised,
and it moves as the user base grows.
```

**Two details that separate a good answer from a textbook one:**
1. **Only fan out to *active* followers.** Of 200 followers, maybe 60 opened the app this week.
   Writing to the other 140 is pure waste. Build an inactive user's timeline lazily on their next
   login (read-time fan-in, then cache it). This can cut fan-out volume by a large factor — the
   single cheapest optimization available, and most candidates miss it.
2. **Read-your-own-writes.** Your own post must appear in your own feed instantly, even though fan-out
   is asynchronous and takes seconds. Fix: the feed service merges the author's own recent posts from
   a local/session cache at read time. Small, and it's the kind of product detail that shows you've
   built user-facing systems.

## 2.7 Ranking

Design the **pipeline**, not the model — and say that you're doing that deliberately.

```
candidate generation → feature hydration → scoring → filtering/diversity → final ordering
(timeline + celebs)    (author affinity,    (a model    (blocks, mutes,      (no 5 posts from
                        recency, engagement, or a        dedupe, "not more    the same author
                        media type)         weighted     than N per author")  in a row)
                                            formula)
```

- **Score at write time or read time?** Affinity and recency are cheap enough to compute at read time
  for ~800 candidates. A heavy model runs offline and writes features that the read path looks up.
  The general rule: *precompute what is expensive and stable; compute at read time what is cheap or
  personal.*
- **Recency decay** is the honest starting point: `score = log(engagement + 1) + createdAt/decay`.
  It is explainable and debuggable. Say you'd start there and earn the right to something fancier.
- **The chronological escape hatch** matters: give users a "latest" toggle. It is a product decision,
  but it is also an *operational* one — a chronological feed is a pure sorted-set read and keeps
  working when the ranking service is down. Free graceful degradation.

## 2.8 Failure modes

| Failure | What happens | What the user sees | Mitigation |
|---|---|---|---|
| **Viral post / thundering herd** | millions of clients request the same post at once; the cache entry expires and every request stampedes the database | slow load, or an error | request coalescing (single-flight), probabilistic early expiry, never-expire-just-refresh for hot keys, and a tiny in-process cache in front of Redis |
| Fan-out worker lag | timelines behind by minutes | "I posted and my friends don't see it" | separate queues by priority; autoscale on lag; **and monitor fan-out lag as a first-class SLI** |
| Redis timeline shard lost | those users' timelines are gone | slow feed while it rebuilds | rebuild lazily from posts + follows on first read — the reason "rebuildable" was a design goal |
| Celebrity posts during a spike | read-time fan-in load jumps | slower feeds for their followers | cache the celebrity's recent timeline aggressively — it is one object read by millions, which is the easiest cache in the system |
| A user follows 50,000 accounts | read-time merge is enormous | very slow feed | cap following, or treat these accounts as a special case with a precomputed timeline only |
| Unfollow / block / delete | already-fanned-out entries are stale | seeing content you shouldn't | **filter at read time** — never try to retract from millions of timelines. Read-time filtering is the only correct answer here, and it is why hydration and filtering are separate pipeline stages. |
| Hot-key on a single Redis shard | one shard saturates | latency spike for those users | consistent hashing, plus a local cache for the hottest keys |
| Post-store shard down | some posts can't be hydrated | gaps in the feed | render what hydrated; degrade, don't fail the whole page |

**The delete/block case is the one to raise unprompted.** Candidates who have only read about fan-out
propose "remove it from every timeline", which is the same 50-million-write problem in reverse.
Read-time filtering is the answer, and explaining *why* retraction is infeasible demonstrates you
actually understand the cost model you just built.

## 2.9 Trade-offs

| Decision | Chose | Cost | Would change if |
|---|---|---|---|
| Push vs pull | hybrid | two code paths, a merge step, a threshold to tune | a small social graph with no celebrities → pure push |
| Timeline cap at 800 | capped | deep scroll needs a slower fallback path | if storage were free (it isn't) |
| Cache all active users' timelines | yes | ~7.7 TB of Redis | lazily build for everyone if latency allowed |
| Ranked vs chronological | ranked, with a chronological fallback | complexity, and a feed that's hard to debug | chronological is dramatically simpler and a legitimate product choice |
| Consistency | eventual (seconds) | "my friend posted 3 seconds ago and I can't see it" | never strong — the cost is not payable at this scale |
| Cursor pagination | opaque cursor | no jump-to-page | offset is simply wrong here; I wouldn't change it |

---

# Part 3 — The estimation cheat sheet

## Powers of two

| Power | Approx value | Reads as |
|---|---|---|
| 2¹⁰ | 1,024 | 1 thousand · 1 KB |
| 2²⁰ | ~1.05 × 10⁶ | 1 million · 1 MB |
| 2³⁰ | ~1.07 × 10⁹ | 1 billion · 1 GB |
| 2⁴⁰ | ~1.10 × 10¹² | 1 trillion · 1 TB |
| 2⁵⁰ | ~1.13 × 10¹⁵ | 1 quadrillion · 1 PB |

## Time conversions you should not have to think about

```
1 day    = 86,400 s        ≈ 10⁵ s
1 month  ≈ 2.6 × 10⁶ s
1 year   ≈ 3.15 × 10⁷ s

Quick conversions (memorise these three, derive everything else):
   1 million/day   ≈     12 /s
 100 million/day   ≈  1,157 /s      → "100M/day is about a thousand per second"
   1 billion/day   ≈ 11,574 /s      → "1B/day is about ten thousand per second"
```

## Sizes to reach for

| Thing | Rough size |
|---|---|
| UUID / ULID | 16 B binary, 36 B as a string |
| A timestamp (epoch millis) | 8 B |
| A short text post | 100–300 B |
| A JSON API response for one item | 0.5–2 KB |
| A telemetry event (compact binary) | 100–300 B |
| A 1080p video segment | a few MB |

## Latency numbers — ORDER OF MAGNITUDE ONLY

> **Attribution and a strong caveat.** This table is the "Latency Numbers Every Programmer Should
> Know" list, widely attributed to **Jeff Dean** (popularised through his talks and Peter Norvig's
> writing, with several interactive versions since). The original figures date from roughly a decade
> and a half ago and **several are now materially wrong** — SSD latency and throughput, network
> bandwidth, and compression speed have all improved by large factors, while the speed-of-light
> numbers have not changed at all.
>
> **Use these only as orders of magnitude, and say so in the interview.** *"Order of magnitude, from
> the classic Jeff Dean list — an in-memory read is nanoseconds, a same-datacenter round trip is
> hundreds of microseconds, a cross-continent round trip is over a hundred milliseconds. I'd check
> current numbers before making a capacity decision on them."* That sentence is worth more than a
> precisely recited but outdated table. **Verify current figures before quoting any specific one.**

| Operation | Order of magnitude | Still roughly right? |
|---|---|---|
| L1 cache reference | ~1 ns | yes |
| Branch mispredict | a few ns | yes |
| Mutex lock/unlock (uncontended) | tens of ns | yes |
| Main memory reference | ~100 ns | yes |
| Compress 1 KB (fast codec) | ~1 µs | **optimistic now — codecs got faster** |
| Send 1 KB over a fast local network | ~1–10 µs | depends entirely on link speed |
| SSD random read | tens of µs | **the original 150 µs figure is pessimistic for NVMe** |
| Read 1 MB sequentially from memory | tens of µs | yes |
| Round trip within one datacenter | ~500 µs | yes |
| Read 1 MB sequentially from SSD | hundreds of µs | **much better than the original 1 ms** |
| Disk seek (spinning rust) | ~10 ms | yes — and increasingly irrelevant |
| Round trip California ↔ Netherlands | ~150 ms | **yes, and this one will never improve — it's physics** |

**The three ratios that actually matter in a design discussion, and that are safe to state:**
1. Memory is ~100,000× faster than a network round trip → cache aggressively.
2. A same-datacenter round trip is ~300× faster than a cross-continent one → keep chatty
   interactions inside one region; that is the entire argument for regional pipelines.
3. Sequential I/O is orders of magnitude faster than random I/O → this is why Kafka (append-only log)
   and LSM-tree stores are fast, and why `OFFSET 100000` is slow.

## Sanity checks to run on your own numbers

- **Does the bandwidth fit on a wire?** 1.2 GB/s is ~10 Gbps. If your arithmetic says 40 GB/s into
  one service, you made an error or you need a fundamentally different design.
- **Does the storage fit in a budget?** Petabytes per month is a real conversation with a real CFO.
- **Is the QPS per instance plausible?** If your design needs 500,000 requests/s from one box, redo it.
- **Is the fan-out bounded?** Any `for each follower` / `for each subscriber` loop is the place a
  system dies. Always compute the worst case, not the average.
- **Sanity-check against something you know.** "That's more events per second than there are people
  in the country" is a legitimate and disarming thing to say out loud.

---

# Part 4 — Phrase bank for narrating trade-offs

**Opening and framing**
- *"Let me spend about five minutes clarifying, then I'll estimate, then draw. Stop me any time."*
- *"Before I design, I want to know which of these matters most: freshness, cost, or completeness.
  They pull in different directions and I'd rather optimise the one you care about."*
- *"Let me state my assumptions explicitly so we can correct them together."*

**Estimating**
- *"I'll assume X. If the real number is 10× that, here's what changes and here's what doesn't."*
- *"The number that drives this design is ___, so let me derive that first."*
- *"That's a big number — let me sanity-check it against something concrete."*

**Choosing, and rejecting**
- *"Three options here: A, B, C. I'm ruling out C because ___. Between A and B it comes down to ___."*
- *"I'd start with the simple version — ___ — and I'd only add ___ when the measurement justified it."*
- *"That's a one-way door, so I'd want to be more careful about it than about the reversible parts."*

**Trade-offs — the sentences that earn the level**
- *"This buys me ___; it costs me ___; I think that's the right trade here because ___."*
- *"The failure mode of this choice is ___, and the user experience of that failure is ___."*
- *"I'm trading consistency for availability here, and I'm comfortable with that because a feed that's
  five seconds stale is fine and a feed that returns 500 is not."*
- *"That's a cost decision more than a technical one — I'd want the number in front of me."*

**Depth and control**
- *"I can go deeper on ___, ___ or ___. Which is most useful to you?"*
- *"There's a subtlety here I want to flag, even though it's not the main thread: ___."*
- *"I'm going to defer ___ — it's real, but it's not on the critical path for what we're designing."*

**Honesty — use these, they help you**
- *"I don't know the exact number; I'd measure it. What I do know is the shape: ___."*
- *"I've never operated one at that scale. Here's how I'd de-risk it: ___."*
- *"You're right — let me redo that. If ___ then ___ changes, and so does ___."*  ← **recovering from a
  correction gracefully scores higher than never being corrected.**

**Closing**
- *"To summarise the three decisions I'd defend: ___, ___, ___."*
- *"With more time I'd go deeper on ___, which I think is the riskiest part of this design."*
- *"Here's how I'd phase the build so there's something useful in production in week three."*

---

# Part 5 — The top 10 mistakes in this round

1. **Drawing before clarifying.** Boxes in minute two means you designed for a problem nobody asked
   about. Five minutes of questions saves the whole round.
2. **Estimates with no arithmetic.** "It's about a million QPS" is worthless. Show the division. The
   method is what is being graded, and it is the *only* part that survives a wrong assumption.
3. **Running out of time before failure modes.** The most common way a strong candidate gets a "mid"
   rating. Set a hard reserve of the last seven minutes.
4. **No trade-offs.** A design where every choice is presented as obviously correct reads as a design
   where nothing was considered. Every box should have a rejected alternative behind it.
5. **Buzzword architecture.** Naming Kafka, Redis, Cassandra and Kubernetes without saying what each
   one does *for this problem* and what it costs. Expect "why not just Postgres?" — and have a real
   answer, because for many designs the honest answer is "Postgres would be fine at this scale."
6. **Ignoring the hot key / celebrity / skew case.** Every fan-out system has one. Not raising it
   unprompted is the single clearest signal of inexperience in these two designs specifically.
7. **Claiming exactly-once across a system boundary.** Say "at-least-once plus idempotent writes,
   which is effectively once." Interviewers listen for exactly this sentence.
8. **Silent monologuing.** Long pauses while you think, then a finished answer. Narrate. They are
   evaluating collaboration, and an interviewer who cannot follow you cannot credit you.
9. **Defending a wrong answer.** When corrected, integrate it out loud and redo the affected
   arithmetic. That reads as senior. Digging in reads as junior — and as someone who will be hard to
   work with.
10. **Over-engineering for scale nobody asked for.** Multi-region active-active for 1,000 users is a
    red flag, not a strength. The senior move is to *name* the simple design, say when it breaks, and
    design for one order of magnitude beyond today — not five.

**One meta-point.** In both designs above, the highest-signal moments are single sentences:
*"Partition count is set by consumer parallelism, not broker bandwidth."* · *"147 TB of replicated
broker disk is why raw retention is short and the lake is long."* · *"One celebrity post is 119
seconds of the entire cluster's write capacity."* · *"You can't retract from 50 million timelines, so
you filter at read time."* Each is a piece of arithmetic or a mechanism, not an opinion. Collect a few
of those per design and deploy them deliberately.

---

## Key terms

| Term | One-line meaning |
|---|---|
| **Fan-out on write (push)** | Precompute every follower's timeline at post time. Fast reads, O(followers) writes. |
| **Fan-out on read (pull)** | Assemble the feed at read time by querying everyone you follow. Cheap writes, slow reads. |
| **Hybrid fan-out** | Push for normal accounts, pull for celebrities, merge at read time. The standard answer. |
| **Write amplification** | One logical write becoming N physical writes — the thing that kills naive fan-out. |
| **Hot partition / hot key** | One key carrying a disproportionate share of traffic; the classic streaming failure. |
| **Two-stage aggregation** | Shard a hot key, aggregate partials in parallel, merge. Requires commutative/associative aggregates. |
| **Tumbling / hopping window** | Non-overlapping fixed windows / overlapping fixed windows at a fixed hop. |
| **Watermark** | The processing-time signal that event time has advanced past a window; when results are emitted. |
| **Grace period** | How long after a window closes you still accept late events. Completeness vs freshness. |
| **Suppression** | Emitting one final result per window instead of a stream of partial updates. |
| **Event time vs ingest time vs processing time** | Client clock, server receipt, operator execution. Window on event time. |
| **Lambda vs kappa architecture** | Separate batch + streaming paths, vs one streaming path you replay. |
| **Effectively once** | At-least-once delivery plus idempotent writes. The honest guarantee across a boundary. |
| **Backpressure** | A slow consumer propagating "slow down" upstream — or being deliberately dropped. |
| **`onBackpressureLatest`** | Keep the newest value for a slow subscriber; correct for dashboards. |
| **Cursor pagination** | Paginate by a position in the ordering, not by offset. Stable on a changing list. |
| **Thundering herd** | A cache expiry or a reconnect storm sending every client at the origin simultaneously. |
| **Single-flight / request coalescing** | Collapsing N concurrent identical cache misses into one origin call. |
| **Small-files problem** | Many tiny objects making an analytics query engine slow; fixed by compaction. |
| **Noisy neighbour** | One tenant's load degrading another's; fixed with quotas, bulkheads, or separation. |
| **RPO / RTO** | How much data you can lose / how long you can be down. State both in any DR discussion. |

---

## Self-check (answer out loud, no notes, with a whiteboard)

1. From 50M concurrent viewers and a 10-second heartbeat, derive events/s, peak bandwidth and raw
   storage per day — out loud, showing every division. Then say which single number drives the
   architecture and why.
2. Why is the Kafka partition count set by consumer parallelism rather than by broker bandwidth, and
   why is increasing it later expensive?
3. Explain the three clocks in a streaming pipeline and what a grace period trades against. What do
   you do with an event that arrives an hour late?
4. Do the celebrity fan-out arithmetic from memory: one post, 50M followers, a cluster budget of
   420k inserts/s. What does the number come out at, and what does it force you to build?
5. Name five failure modes for each design and, for each one, say what the *user* experiences — not
   what the system does.
