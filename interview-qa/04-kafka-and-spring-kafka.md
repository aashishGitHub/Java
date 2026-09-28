# 04 — Kafka & Spring Kafka: Partitions, Delivery Semantics, Rebalancing, DLT, Tuning, Streams

**Why this matters for an Event Hub role:** this is the file the role is actually about. Almost
anyone can write a `@KafkaListener`. The senior signal is being able to say, without hedging, *what
guarantee you have and what you paid for it* — why ordering is per-partition and not global, why
"exactly-once" is a lie at the edges, why your consumer group is thrashing, and what number you'd put
on a dashboard to know the pipeline is healthy before a customer tells you. An Event Hub team lives
and dies on **consumer lag**, **rebalance behaviour** and **poison messages**; everything else is
commentary.

**Scenario anchor — BS-03 (REQ-KT-01…06):** finance flags the Kafka bill (broker storage + network)
during a championship final; product needs event→dashboard freshness < 5s p99 at 10× baseline load.
The mandate is to cut bytes on the wire/disk by ≥ 30% **without** weakening delivery guarantees. That
forces you to understand every producer and consumer knob as a trade, not a default. Secondary
anchors: **BS-01** (offsets are only committed after a durable write), **BS-04** (poison messages and
DLT), **Phases 2, 3, 4 and 9**.

> **Version note, read once:** Kafka moved off ZooKeeper (KRaft became production-ready in 3.3 and
> ZooKeeper mode was removed in the 4.x line), the consumer got a new server-side rebalance protocol
> (KIP-848), and several producer defaults flipped in 3.0 (`enable.idempotence`, `acks`). I give the
> mechanisms confidently and flag **every default value** for verification — quoting a stale default
> in an interview is worse than saying "I'd check it".
>
> **Verify:** your broker version, your client version, and whether your cluster runs KRaft or (on an
> old install) ZooKeeper. Defaults below are "as documented for recent 3.x/4.x" — confirm against the
> version you pin.

---

## Q1. Explain Kafka's storage model: topics, partitions, offsets, replicas.

**30-second answer:** A topic is a logical stream; a **partition** is the actual unit of storage,
parallelism and ordering — an append-only log on disk. Every record in a partition gets a
monotonically increasing **offset**, and the offset is the consumer's cursor, not a message ID the
broker tracks per consumer. Each partition has one **leader** replica that serves reads and writes
and N−1 followers that replicate it; the set of replicas that are caught up is the **ISR** (in-sync
replicas), and `acks=all` means "acknowledged by all in-sync replicas", not "all replicas". Kafka is
a *log*, not a queue: consuming does not delete anything, so multiple independent consumer groups
read the same data, and I can replay history by resetting offsets. That replay property is the single
biggest reason to pick Kafka over RabbitMQ or SQS.

**Deeper:**

```
topic: telemetry.raw.v1   (12 partitions — REQ-KT-01 / Phase 1)

 partition 0 │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │◀── log end offset (LEO)
             │           ▲       ▲
             │           │       └─ high water mark (HWM): last offset replicated to all ISR;
             │           │          consumers can NEVER read past this
             │           └─ committed offset for group "sink-writer"  → lag = HWM − committed
 partition 1 │ 0 │ 1 │ 2 │ 3 │
 ...
 partition 11│ 0 │ 1 │
```

Things that follow from this picture and that interviewers probe:

| Property | Truth | Common wrong answer |
|---|---|---|
| Ordering | Total order **within a partition only** | "Kafka preserves order" |
| Parallelism ceiling | One partition → at most one consumer *in a group* | "Add consumers to go faster" |
| Retention | Time (`retention.ms`) or size (`retention.bytes`) based, independent of consumption | "Messages are deleted when consumed" |
| Durability | `acks=all` + `min.insync.replicas=2` on RF=3 | "`acks=all` means all 3 replicas" |
| Offset storage | In the internal `__consumer_offsets` topic (compacted), keyed by (group, topic, partition) | "The broker remembers what each consumer read" |

The `acks=all` + `min.insync.replicas` pairing is the one people get wrong. With RF=3 and
`min.insync.replicas=2`, a write needs 2 acks and you survive one broker loss. If you set
`min.insync.replicas=1`, `acks=all` degenerates to `acks=1` when followers fall behind — you have the
config that *looks* durable and isn't. Conversely `min.insync.replicas=3` on RF=3 means a single
broker restart makes the topic unwritable (`NotEnoughReplicasException`). RF=3/minISR=2 is the
standard balance.

Physically, each partition is a directory of **segment** files (`.log`, `.index`, `.timeindex`).
Retention and compaction operate per segment, and the **active segment is never deleted** — which is
why a low-traffic topic can hold data well past its `retention.ms`.

**Code:** the StreamPulse topic contract, expressed as a Spring `NewTopic` (Phase 1 AsyncAPI is the
source of truth; this is the executable version):

```java
@Configuration
class TopicConfig {

    /**
     * telemetry.raw.v1 — REQ-KT-01.
     * 12 partitions: chosen as "max expected consumer instances × headroom", NOT arbitrarily.
     * You can increase partitions later, but you can NEVER decrease them, and increasing them
     * breaks key→partition stability (see Q2), so pick with headroom on day one.
     */
    @Bean
    NewTopic telemetryRaw() {
        return TopicBuilder.name("telemetry.raw.v1")
                .partitions(12)
                .replicas(3)                                    // 1 locally, 3 in any real cluster
                .config(TopicConfig.MIN_IN_SYNC_REPLICAS_CONFIG, "2")
                .config(TopicConfig.RETENTION_MS_CONFIG,  "259200000")   // 3 days = replay window
                .config(TopicConfig.COMPRESSION_TYPE_CONFIG, "producer") // keep producer's codec,
                                                                         // don't re-compress (Q11)
                .build();
    }
}
```

**Follow-ups they'll ask:**
- *"How many partitions should a topic have?"* — driven by three things: the maximum consumer
  parallelism you want in one group, the per-partition throughput your consumers can sustain, and the
  cost (each partition is open file handles, memory on the broker, and a unit of rebalance work).
  I'd size for peak consumer count with ~2× headroom and measure. Tens of thousands of partitions per
  cluster is a real operational cost, not free.
- *"Can you reduce partition count?"* — no. You create a new topic and migrate. Another reason to add
  headroom up front.
- *"What's the high water mark for?"* — consumers can't read uncommitted-to-ISR data, so a leader
  failover can't expose records that then disappear.

**Pitfalls / red flags:** calling Kafka "a message queue", or saying messages are removed on consume.

**Where in StreamPulse:** Phase 1 (AsyncAPI topic contracts: partitions, keys, retention,
compaction), Phase 3–4.

**Verify:** whether `min.insync.replicas` defaults to 1 on your broker, and the exact segment-rolling
config names (`segment.ms`, `segment.bytes`) for your version.

---

## Q2. How do you choose a partition key, and what does that buy you? Why `sessionId` in StreamPulse?

**30-second answer:** The partition key determines ordering and co-location, nothing else. Kafka
hashes the key (murmur2 by default in the Java client) modulo the partition count, so all records
with the same key land on the same partition and are therefore **totally ordered relative to each
other**. In StreamPulse I key `telemetry.raw.v1` on **`sessionId`**, because the ordering that
actually matters to the domain is within a viewing session: `session_start` →
`buffering_start` → `buffering_end` → `bitrate_change` → `session_end`. Rebuffer ratio and
startup-time are computed from the *sequence* of events in one session, so if those get reordered the
metric is wrong. I do **not** key on `tenantId`, because `sportsnow` alone would be a third of the
traffic on one partition — a guaranteed hot partition and an artificial throughput ceiling.

**Deeper:**

The default Java producer partitioner:
- key present → `murmur2(keyBytes) % numPartitions` (the `DefaultPartitioner` behaviour).
- key null → in modern clients, the **sticky** behaviour: batch to one partition until the batch is
  sent, then switch. This is a throughput optimisation — it makes batches fuller — not round-robin
  per record.

The candidate keys for StreamPulse, and why `sessionId` wins:

| Key | Ordering you get | Cardinality | Hot-partition risk | Verdict |
|---|---|---|---|---|
| `tenantId` | per tenant | ~3 (fictional tenants) | **Severe** — `streamflix` is 40% of traffic | ❌ |
| `titleId` | per title | thousands, but Zipfian | **High** during a live final: one title dominates | ❌ |
| `sessionId` | per viewing session | millions, near-uniform | Low | ✅ |
| `eventId` / null | none | effectively infinite | None | ❌ loses the ordering the domain needs |
| `tenantId + titleId` | per title within tenant | same as titleId | High | ❌ |

**The hot-partition failure mode, concretely:** a hot key means one partition's leader broker takes
disproportionate write and read traffic. The symptoms are a *skewed* lag chart — 11 partitions at
near-zero lag and one climbing — and you cannot fix it by adding consumers, because a partition is
consumed by exactly one member of the group. You have to change the key (new topic) or add an
artificial salt (`sessionId#bucket`), which destroys the ordering you keyed for. **This is a design
decision you cannot cheaply reverse**, which is why interviewers like it.

The second irreversibility: **adding partitions changes `hash % n` for existing keys.** After a
resize, a session's later events can land on a different partition than its earlier events, so
ordering breaks *across the resize boundary*. Options: over-provision partitions up front (what I do),
use a custom partitioner with a stable mapping, or accept a bounded window of disorder and design the
consumer to tolerate it.

A `Partitioner` is pluggable if you need custom behaviour (e.g. keeping a whole tenant on a subset of
partitions for noisy-neighbour isolation):

```java
/**
 * Only write one of these if you have a real reason — a custom partitioner is a piece of
 * logic that must be identical in every producer forever. StreamPulse uses the default.
 */
public class TenantAwarePartitioner implements Partitioner {
    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                         Object value, byte[] valueBytes, Cluster cluster) {
        int n = cluster.partitionCountForTopic(topic);
        // ... map tenant → partition range, then hash sessionId within that range
        return 0;
    }
    @Override public void close() {}
    @Override public void configure(Map<String, ?> configs) {}
}
```

**Code:** the actual send, with the key made explicit and the ordering intent commented:

```java
@Component
class KafkaEventPublisher implements EventPublisher {   // hexagonal port impl (see file 01, Q10)

    private final KafkaTemplate<String, TelemetryEvent> template;

    @Override
    public CompletableFuture<Void> publish(TelemetryEvent e) {
        // KEY = sessionId. This is the single most consequential line in the ingest path:
        //  - guarantees per-session ordering (buffering_start before buffering_end)
        //  - keeps partitions balanced (millions of sessions, roughly uniform hash)
        //  - does NOT give per-tenant or global ordering — the stream processor must not assume it
        return template.send("telemetry.raw.v1", e.sessionId(), e)
                       .thenApply(result -> null);
    }
}
```

**Follow-ups they'll ask:**
- *"What if you need global ordering?"* — one partition, and accept the throughput ceiling of a
  single broker's log. Usually the right answer is "the business doesn't actually need global
  ordering, it needs ordering per entity" — and you find the entity.
- *"How do you detect a hot partition?"* — per-partition lag and per-partition bytes-in in Grafana
  (Phase 8). A skew chart, not an aggregate. Aggregate lag hides it completely.
- *"Does keying guarantee ordering end-to-end?"* — only up to the partition. If your consumer hands
  work to a thread pool, you have thrown the ordering away downstream. That's why StreamPulse's
  sink-writer does the ordering-sensitive work on the listener thread and only offloads the
  *order-independent* S3 upload (BS-02 / REQ-CC-04).
- *"What about `max.in.flight` and ordering?"* — see Q5; retries can reorder unless idempotence is on.

**Pitfalls / red flags:** "I'd key on tenant so each customer's data stays together." That's the hot
partition, stated as a feature. Also: claiming key→partition mapping survives a partition increase.

**Where in StreamPulse:** Phase 1 (`api/asyncapi/*.yaml` documents key + partitions + retention per
topic), Phase 2 (the producer), BS-03.

---

## Q3. Explain consumer groups and rebalancing. What's the difference between eager and cooperative-sticky?

**30-second answer:** A consumer group is a set of consumers that share a subscription; the group
coordinator (a broker) assigns each partition to exactly one member, so the group's parallelism is
capped at the partition count. A **rebalance** is the protocol that recomputes that assignment
whenever membership or topic metadata changes. The classic protocol is **eager**: every member
revokes *all* its partitions, then the whole group re-joins — a stop-the-world pause for the entire
group even if one pod restarted. **Cooperative-sticky** (incremental cooperative rebalancing) instead
computes the new assignment first and only revokes the partitions that actually have to move, in a
second round — so 11 of 12 partitions keep processing while one moves. For anything with non-trivial
scale I use cooperative-sticky; the cost is that it takes two rounds and your revocation callback can
be called with a *subset* rather than everything.

**Deeper:**

```
EAGER (RangeAssignor / RoundRobinAssignor)
  pod C joins
    ├─ ALL members revoke ALL partitions      ← entire group stops consuming
    ├─ JoinGroup / SyncGroup
    └─ ALL members get a new assignment        ← "stop the world"

COOPERATIVE-STICKY (CooperativeStickyAssignor)
  pod C joins
    ├─ round 1: leader computes target assignment; members revoke ONLY partitions
    │           that must move (A keeps 0-3, B keeps 4-7, B revokes 8-11)
    ├─ round 2: the revoked partitions are assigned to C
    └─ A and B never stopped
```

**What actually triggers a rebalance** — worth being able to list, because most incidents are here:

| Trigger | Notes |
|---|---|
| A consumer joins or leaves gracefully | `close()` sends LeaveGroup → immediate rebalance |
| A consumer's heartbeat stops for `session.timeout.ms` | The coordinator evicts it |
| `poll()` not called within `max.poll.interval.ms` | The member is *removed from the group* even though heartbeats were fine |
| Partition count of a subscribed topic changes | Metadata refresh |
| A consumer subscribed by pattern and a new matching topic appears | Sneaky one |

The third row is the classic production bug, and BS-03 / REQ-KT-04 turns it into a test.
`heartbeat.interval.ms` is sent by a **background thread**, so your consumer looks alive while your
handler is stuck; but `poll()` is called from the **application thread**, so if processing a batch
takes longer than `max.poll.interval.ms`, the coordinator kicks you out mid-batch. You then finish,
try to commit, get `CommitFailedException`, and the partitions you were working on are reassigned to
someone else who reprocesses them. If the cause is systemic (everyone is slow), every member does this
in a loop: a **rebalance storm** where the group spends all its time rebalancing and no time
consuming. Lag goes vertical.

The governing inequality, which I'd write on the whiteboard:

```
max.poll.records  ×  worst-case per-record processing time   <   max.poll.interval.ms
```

Three levers: reduce `max.poll.records`, make processing faster/bounded (timeouts on every
downstream call — BS-04 / REQ-RS-06), or raise `max.poll.interval.ms` (which delays genuine
failure detection — a trade, not a free fix).

**Static membership** (`group.instance.id`) is the other big lever. Normally a pod restart = leave +
join = two rebalances. With a stable `group.instance.id` (in Kubernetes, the StatefulSet ordinal or
the pod name), the coordinator keeps the member's assignment reserved for `session.timeout.ms`, so a
rolling restart causes **zero** rebalances as long as each pod comes back inside that window. The
trade: a genuinely dead pod isn't detected until the session timeout expires, so you tune
`session.timeout.ms` to "longer than a pod restart, shorter than your tolerance for stalled
partitions".

**KIP-848** replaces this client-driven protocol with a broker-driven one (the group coordinator
computes assignments and pushes them incrementally; no more group leader, no more stop-the-world join
barrier). If your cluster and clients support it, it substantially reduces rebalance impact.
**Verify:** which Kafka and client versions in your stack have the new protocol enabled, and the
config to opt in (`group.protocol`) — this changed across the 3.7→4.x line.

**Code:**

```yaml
# sink-writer — REQ-KT-04 / BS-03. Every value here is a decision, not a default.
spring:
  kafka:
    consumer:
      group-id: sink-writer
      auto-offset-reset: earliest      # a NEW group replays; 'latest' would silently skip history
      enable-auto-commit: false        # we commit after a durable write (BS-01 / REQ-LC-02)
      max-poll-records: 200            # 200 × 40ms worst case = 8s  ≪ max.poll.interval.ms
      properties:
        partition.assignment.strategy: org.apache.kafka.clients.consumer.CooperativeStickyAssignor
        max.poll.interval.ms: 120000   # 2 min: generous vs 8s budget, still fails fast enough
        session.timeout.ms: 45000      # > pod restart time, so static membership actually helps
        heartbeat.interval.ms: 3000    # rule of thumb: ~1/3 of session.timeout.ms
        # group.instance.id is set per-pod from the K8s downward API, NOT here:
        #   -Dspring.kafka.consumer.properties.group.instance.id=$(POD_NAME)
```

```java
/**
 * Rebalance listener. With cooperative-sticky, onPartitionsRevoked is called with the
 * SUBSET being moved — code that assumed "revoked == everything I had" is a real bug
 * when you migrate from eager.
 */
@Component
class DrainOnRevokeListener implements ConsumerAwareRebalanceListener {

    @Override
    public void onPartitionsRevokedBeforeCommit(Consumer<?, ?> consumer,
                                                Collection<TopicPartition> partitions) {
        // Flush only the buffers for the partitions we are losing, then let Spring commit.
        s3Buffer.flush(partitions);
    }

    @Override
    public void onPartitionsAssigned(Consumer<?, ?> consumer,
                                     Collection<TopicPartition> partitions) {
        meterRegistry.counter("kafka.partitions.assigned").increment(partitions.size());
    }
}
```

**Follow-ups they'll ask:**
- *"Range vs round-robin vs sticky?"* — `RangeAssignor` assigns contiguous ranges per topic and is
  the classic default; it skews badly when partition count isn't a multiple of member count and when
  you subscribe to several topics. `RoundRobinAssignor` balances better across topics.
  `StickyAssignor` minimises movement but is still eager. `CooperativeStickyAssignor` is sticky *and*
  incremental. **Verify:** the default `partition.assignment.strategy` for your client version — it
  changed in the 3.x line to a list that includes the cooperative assignor for upgrade purposes.
- *"How do you migrate from eager to cooperative safely?"* — it's a documented two-rolling-restart
  upgrade (first deploy adds the cooperative assignor to the list while keeping the old one, second
  removes the old one). You cannot flip it in one deploy. **Verify** the exact procedure in the docs
  for your version before doing it.
- *"More consumers than partitions?"* — the extras sit idle. Not an error, just wasted pods; a
  legitimate pattern for fast failover.
- *"Why does `enable.auto.commit=true` lose data?"* — it commits on a timer based on what `poll()`
  *returned*, not what you *processed*. A crash between the commit and the processing loses those
  records. It can also, less famously, duplicate. Manual ack after the durable write is the only
  honest option.

**Pitfalls / red flags:** thinking heartbeats keep you in the group while you're slow. Or "we set
`max.poll.interval.ms` to an hour" — that's hiding a slow consumer, and now a hung pod holds its
partitions for an hour.

**Where in StreamPulse:** Phase 4 (sink-writer), BS-03 / REQ-KT-04, `SlowConsumerRebalanceIT`
(deliberately violates the inequality and asserts the storm, then fixes it and asserts stability).

---

## Q4. At-most-once, at-least-once, exactly-once — what does Kafka's EOS actually cover?

**30-second answer:** The three semantics are a function of *when you commit relative to when you
process*. Commit first, then process = at-most-once (crash loses the record). Process first, then
commit = at-least-once (crash reprocesses). Kafka's **exactly-once semantics (EOS)** is narrower than
the name suggests: it covers the **consume→transform→produce cycle inside Kafka**, using an
idempotent producer plus transactions that atomically write output records *and* the consumer offsets
to the same transaction. What it does **not** cover is any side effect outside Kafka — a DynamoDB
write, an S3 PUT, an HTTP call to a partner, an email. The moment your handler touches an external
system, EOS gives you nothing there, and the correct design is **at-least-once delivery plus an
idempotent consumer** — what people call *effectively-once*. That's what StreamPulse's sink-writer
does; EOS is used only in the Kafka Streams stage, where input and output are both Kafka.

**Deeper:**

| | Mechanism | Failure outcome | Where StreamPulse uses it |
|---|---|---|---|
| **At-most-once** | commit offset *before* processing | record lost | nowhere — telemetry loss is the thing BS-01 exists to prevent |
| **At-least-once** | process, durable write, *then* commit | record reprocessed | ingest→Kafka, and sink-writer→Redis/DynamoDB/S3 |
| **EOS (Kafka→Kafka)** | idempotent producer + transaction covering output records **and** the offset commit | atomic: either both or neither | `stream-processor`, `processing.guarantee=exactly_once_v2` (Phase 3) |
| **Effectively-once** | at-least-once + idempotent write (conditional put / dedup key) | duplicate delivery, single *effect* | sink-writer → DynamoDB conditional write, S3 deterministic key (Phase 4) |

The clearest way to say the EOS boundary out loud:

```
        ┌──────────────────── the EOS transaction boundary ──────────────────┐
        │                                                                    │
 Kafka ─┼─▶ consumer ──▶ your transform ──▶ producer ──▶ Kafka output topic ─┼─▶
        │                    │                    + offset commit            │
        └────────────────────┼───────────────────────────────────────────────┘
                             │
                             ▼
                      DynamoDB / S3 / HTTP   ◀── NOT covered. At all. Ever.
                      (these can run twice)
```

Three more caveats I'd volunteer, because they show you've actually operated this:

1. **EOS is not free.** Transactions add a commit round-trip per batch, and `read_committed`
   consumers wait for the **LSO** (last stable offset) — they cannot read past an open transaction.
   So an in-flight transaction adds latency to every downstream reader. In Kafka Streams the default
   commit interval drops when EOS is on to keep that latency bounded, which means more, smaller
   commits. **Verify** the exact `commit.interval.ms` default under EOS in your version.
2. **EOS does not survive a non-deterministic transform.** If your transform calls
   `System.currentTimeMillis()` or an external service, a replay produces different output. The
   *atomicity* still holds; the *idempotence of the result* doesn't.
3. **EOS does not de-duplicate your producer's inputs.** If the ingest gateway sends the same logical
   event twice (client retry at the HTTP layer), Kafka happily stores two records with two offsets.
   That's why StreamPulse dedups on `eventId` in Redis at ingest (Phase 2) *and* uses conditional
   writes at the sink — two independent layers, because the guarantee gaps are at different places.

**Code:** the at-least-once + idempotent-write pattern, which is the one you'll write most often:

```java
/**
 * BS-03 / REQ-KT-05. At-least-once + idempotent write = effectively-once.
 * Note the ORDER: durable write FIRST, ack SECOND. Reversing these two lines
 * converts the service from at-least-once to at-most-once — i.e. silent data loss.
 */
@KafkaListener(topics = "telemetry.agg.v1", containerFactory = "batchFactory")
public void onAggregates(List<ConsumerRecord<String, MetricAggregate>> records,
                         Acknowledgment ack) {

    for (var record : records) {
        var agg = record.value();

        // Conditional write: "only if this (tenant, metric, window) is not already written".
        // A redelivered record hits the condition and is a no-op, not a duplicate row.
        dynamo.putItem(PutItemRequest.builder()
                .tableName("streampulse-metrics")
                .item(toItem(agg))
                .conditionExpression("attribute_not_exists(PK) AND attribute_not_exists(SK)")
                .build());
    }

    ack.acknowledge();   // only now. If we crash above, the whole batch is redelivered — fine.
}
```

**Follow-ups they'll ask:**
- *"So is exactly-once a myth?"* — end-to-end across heterogeneous systems, essentially yes, unless
  every sink is idempotent or participates in the same transaction. Within Kafka it is real and
  well-implemented. The senior framing: *"I don't chase exactly-once delivery, I design for
  at-least-once delivery with exactly-once effect."*
- *"Could you use two-phase commit with DynamoDB?"* — you could reach for an XA-style protocol, but
  it's a distributed-transaction coordinator you now own, with blocking failure modes. Idempotent
  writes are cheaper and more available. That's the CAP-flavoured trade.
- *"What's the cheapest idempotency key here?"* — a deterministic one derived from the data, so it
  survives a replay: `alertId = hash(ruleId, windowStart)` in BS-08 / REQ-SF-05; the
  `(tenant, metric, windowStart)` composite key here. A random UUID generated at processing time is
  useless — it's different on the retry.

**Pitfalls / red flags:** "we turned on exactly-once so duplicates can't happen" while writing to
DynamoDB from the same handler. That's the answer that ends the topic.

**Where in StreamPulse:** Phase 3 (EOS v2 in Kafka Streams), Phase 4 (effectively-once sinks),
BS-03 / REQ-KT-05.

---

## Q5. How does the idempotent producer work internally?

**30-second answer:** With `enable.idempotence=true` the producer gets a **PID** (producer ID) from
the broker and stamps every record batch with `(PID, epoch, sequence number)` per partition. The
partition leader keeps the last sequence number it accepted per PID, so if a producer retries a batch
after a network timeout — where the write actually succeeded but the ack was lost — the broker sees a
duplicate sequence and **acknowledges it without appending**. That kills the classic duplicate:
retry-after-lost-ack. It also preserves ordering under retries, because the broker rejects an
out-of-order sequence (`OutOfOrderSequenceException`) instead of appending it. The constraint people
remember is on `max.in.flight.requests.per.connection`: with idempotence enabled it must stay at or
below a small limit (5, as I recall) for the broker to be able to deduplicate and reorder within its
window — **verify the exact limit in the current docs before quoting it.**

**Deeper:**

The failure this fixes, drawn out:

```
WITHOUT idempotence                        WITH idempotence
producer ──batch(seq=?)──▶ leader          producer ──batch(PID=7,epoch=0,seq=42)──▶ leader
          appended ✔                                 appended ✔, lastSeq[7,p0] = 42
          ack ──✗ lost (network)                     ack ──✗ lost
producer retries ──────────▶ leader        producer retries (PID=7,epoch=0,seq=42) ─▶ leader
          appended AGAIN ✔  ← DUPLICATE              seq 42 ≤ lastSeq → ack, DO NOT append ✔
```

Scope, precisely — this is where candidates over-claim:

| Idempotent producer guarantees | Does NOT guarantee |
|---|---|
| No duplicates from **producer retries**, per partition, **per producer session** | No duplicates if your application calls `send()` twice |
| Ordering preserved under retries within the in-flight window | Anything across a producer restart, unless you use a `transactional.id` |
| Per-partition sequence tracking | Cross-partition atomicity (that's transactions, Q6) |

The **epoch** is what makes zombie fencing work: when a producer with a `transactional.id` restarts,
it bumps the epoch, and the broker rejects writes from the old epoch with
`ProducerFencedException`. Without a `transactional.id`, a restarted producer just gets a new PID and
the broker has no way to know the two sessions are the same logical producer — so a batch in flight
during a crash *can* be duplicated by the restarted instance. Idempotence is a session-scoped
guarantee.

The broker keeps this state in memory per partition (with a bounded snapshot on disk), and it expires
after `transactional.id.expiration.ms` / producer-state expiry. So a producer that is silent for a
very long time and then retries an old batch is not protected. **Verify** the exact expiry config
names for your broker version.

Since Kafka 3.0, `enable.idempotence` defaults to **true** for the Java producer, which also forces
`acks=all`, `retries > 0` and the in-flight cap. If you explicitly set `acks=1` *and*
`enable.idempotence=true`, the producer fails to start with a config error rather than silently
downgrading — a good design. **Verify** the 3.0 default-change list for your client version.

**Code:**

```yaml
# ingest-gateway — REQ-KT-03. These four are a package deal; changing one breaks the guarantee.
spring:
  kafka:
    producer:
      acks: all                       # required by idempotence; wait for all in-sync replicas
      properties:
        enable.idempotence: true      # PID + sequence numbers  → no retry duplicates
        max.in.flight.requests.per.connection: 5   # must not exceed the idempotence limit (verify)
        delivery.timeout.ms: 120000   # total budget: send → ack, INCLUDING all retries
        request.timeout.ms: 30000     # per attempt; must be ≤ delivery.timeout.ms
        # retries is effectively Integer.MAX_VALUE by default; delivery.timeout.ms is the real bound.
      # BS-03 tuning knobs live here too — see Q11
      linger-ms: 20
      batch-size: 65536
      compression-type: zstd
```

```java
/**
 * BS-03 / REQ-KT-03. Why NOT max.in.flight=10 for throughput?
 * Because the broker's dedup/reorder window per PID is bounded. Above the supported limit
 * the producer cannot guarantee ordering on retry, so Kafka refuses the combination.
 * The throughput you wanted comes from linger.ms + batch.size + compression instead (Q11) —
 * bigger batches, not more concurrent un-acked batches.
 */
```

**Follow-ups they'll ask:**
- *"Why would `max.in.flight > 1` reorder without idempotence?"* — batch 1 fails and is retried while
  batch 2 already succeeded, so batch 1's records land *after* batch 2's. Classic.
- *"What does `delivery.timeout.ms` bound versus `request.timeout.ms`?"* — `request.timeout.ms` is
  one attempt; `delivery.timeout.ms` is the whole lifecycle including retries and time in the
  accumulator. Setting `retries` without setting `delivery.timeout.ms` is how people get sends that
  take minutes to fail.
- *"What happens when the producer buffer is full?"* — `send()` blocks up to `max.block.ms` then
  throws `TimeoutException`. In StreamPulse that's a backpressure signal: it surfaces as
  **HTTP 429 + Retry-After** rather than a 500 (Phase 2), because the client should slow down, not
  treat it as a server bug.
- *"Does idempotence cost throughput?"* — the sequence bookkeeping is cheap; the real cost is that it
  forces `acks=all`, so you pay replication latency. Usually worth it.

**Pitfalls / red flags:** saying "idempotent producer means no duplicates ever". It means no
duplicates *from producer retries in one session*.

**Where in StreamPulse:** Phase 2 (producer), BS-03 / REQ-KT-03.

**Verify:** the `max.in.flight.requests.per.connection` limit under idempotence (I believe 5), the
Kafka 3.0 default changes, and producer-state expiry config names.

---

## Q6. What do Kafka transactions add, and what does `read_committed` change for the consumer?

**30-second answer:** Transactions extend idempotence from "one partition, one session" to
"**atomic multi-partition writes plus an atomic offset commit**". The producer declares a
`transactional.id`, calls `beginTransaction()`, sends to any number of partitions, calls
`sendOffsetsToTransaction()` to include the input offsets, then `commitTransaction()`. Either every
output record *and* the offset commit becomes visible, or none does. On the read side, a consumer
with `isolation.level=read_committed` filters out aborted records and will not advance past the
**last stable offset**, so it never sees data from an in-flight transaction. That's the whole
consume-transform-produce loop, and it's exactly what Kafka Streams' `exactly_once_v2` automates for
you.

**Deeper:**

```
producer.initTransactions()                    // fences any zombie with the same transactional.id
  │
  ├─ beginTransaction()
  ├─ send(telemetry.agg.v1, p3, aggA)          ┐
  ├─ send(telemetry.agg.v1, p7, aggB)          │ all-or-nothing
  ├─ sendOffsetsToTransaction(offsets, groupMd)│ ← the input offsets ride INSIDE the transaction
  └─ commitTransaction()                       ┘
        │
        └─ broker writes a COMMIT marker to every touched partition + __consumer_offsets
```

Two implementation details worth naming:

- **Control records / markers.** A commit or abort writes a tiny control record into each affected
  partition. `read_committed` consumers use these to decide what to surface. Consequence: offsets are
  *not* contiguous for a `read_committed` consumer — there are gaps where markers and aborted
  batches live. Code that assumes `offset + 1 == next offset` is broken.
- **LSO (last stable offset).** `read_committed` consumers can only read up to the LSO, which is the
  offset of the *oldest still-open transaction*. A hung producer holding a transaction open therefore
  **stalls every downstream `read_committed` consumer** on that partition until
  `transaction.timeout.ms` aborts it. This is a real availability coupling and a great thing to
  mention unprompted.

`exactly_once_v2` (the Streams setting, formerly `exactly_once_beta`) improved on the original by
using **one producer per Streams instance** instead of one producer per input partition — which is
why EOS in Streams stopped being prohibitively expensive at high partition counts. The older
`exactly_once` and `exactly_once_beta` values are deprecated/removed depending on version.
**Verify** which values your Kafka Streams version accepts.

**Code:**

```yaml
# stream-processor — Phase 3. This is the ONLY place StreamPulse uses EOS,
# because it is the only stage whose input and output are both Kafka.
spring:
  kafka:
    streams:
      application-id: streampulse-aggregator   # also becomes the transactional.id prefix
      properties:
        processing.guarantee: exactly_once_v2
        # With EOS, Streams manages transactions, offsets and the producer for you.
        # Do NOT hand-roll transactions inside a Streams processor.
```

```java
// The hand-rolled equivalent, for the "can you do it without Streams?" follow-up.
producer.initTransactions();
while (running) {
    var records = consumer.poll(Duration.ofMillis(200));
    producer.beginTransaction();
    try {
        var offsets = new HashMap<TopicPartition, OffsetAndMetadata>();
        for (var r : records) {
            producer.send(new ProducerRecord<>("telemetry.agg.v1", r.key(), transform(r.value())));
            offsets.put(new TopicPartition(r.topic(), r.partition()),
                        new OffsetAndMetadata(r.offset() + 1));
        }
        // Offsets committed THROUGH the producer, inside the transaction — never consumer.commitSync().
        producer.sendOffsetsToTransaction(offsets, consumer.groupMetadata());
        producer.commitTransaction();
    } catch (ProducerFencedException e) {
        producer.close();          // a newer instance took over; we are the zombie. Exit.
        throw e;
    } catch (KafkaException e) {
        producer.abortTransaction();   // downstream read_committed consumers never see these records
    }
}
```

Spring's `KafkaTransactionManager` + `@Transactional` wires the same thing into a `@KafkaListener`
when you set `spring.kafka.producer.transaction-id-prefix`.

**Follow-ups they'll ask:**
- *"Can a Kafka transaction include a database write?"* — not atomically. You can chain a Kafka
  transaction and a DB transaction, but there is a window where one committed and the other didn't.
  The correct pattern for "DB + message" is the **transactional outbox** (write the row and the
  outbox entry in one DB transaction, let CDC publish it) — BS-05 / REQ-DL-06, and file 07.
- *"What's `transaction.timeout.ms` for?"* — it bounds how long a transaction can stay open before
  the coordinator aborts it, which bounds how long LSO can stall consumers. It must be ≤ the broker's
  `transaction.max.timeout.ms`.
- *"Why is `transactional.id` stability important in Kubernetes?"* — fencing only works if the
  restarted instance uses the *same* id. A random id per pod means no fencing and possible duplicate
  processing by a zombie. Derive it from a stable identity (StatefulSet ordinal).

**Pitfalls / red flags:** committing offsets with `consumer.commitSync()` inside a transactional
loop — that breaks atomicity and silently defeats the whole thing.

**Where in StreamPulse:** Phase 3; contrasted in the Phase 4 ADR with the non-transactional
sink-writer.

**Verify:** accepted `processing.guarantee` values and `transaction.timeout.ms` defaults in your
version.

---

## Q7. Your consumer will see duplicates. How do you make the consumer idempotent?

**30-second answer:** Assume duplicates and design the *write* so that applying it twice equals
applying it once. Four techniques, in order of how often I use them: (1) a **conditional write** —
DynamoDB `attribute_not_exists` or Postgres `INSERT … ON CONFLICT DO NOTHING` keyed on a deterministic
business key; (2) a **deterministic object key** — S3 `PUT` to the same key overwrites rather than
appends, so a replay is a no-op; (3) an explicit **processed-keys table** with a TTL, checked before
the side effect, for operations that aren't naturally idempotent (sending an email, calling a partner
webhook); (4) **upsert with a monotonic version** — `UPDATE … WHERE version < :incoming` so a stale
replay loses. The critical requirement in all four is that the key is **derived from the data**, not
generated at processing time.

**Deeper:**

| Sink | Idempotency mechanism | StreamPulse key |
|---|---|---|
| DynamoDB aggregates | conditional `PutItem` | `PK=TENANT#t#METRIC#m`, `SK=WINDOW#ts` (Phase 4) |
| Redis hot cache | `SET` is naturally idempotent; use last-write-wins with an event timestamp | `tenant:metric:window` |
| S3 raw NDJSON | deterministic key `tenant=/dt=/hour=/part-<partition>-<firstOffset>.ndjson.gz` | replay overwrites the same object |
| Postgres `usage_hourly` | `INSERT … ON CONFLICT (tenant, hour) DO UPDATE` | natural key |
| Partner webhook (BS-07) | processed-keys set in Redis + partner-side `Idempotency-Key` header | `alertId = hash(ruleId, windowStart)` |

The S3 row is the subtle one and a good thing to raise: naming a part-file after
`(partition, firstOffset)` makes it deterministic, because a redelivery after a crash replays the
*same* offsets from the same partition. If you name it `part-<UUID>.ndjson.gz` or
`part-<timestamp>`, every replay creates a new object and your "raw archive" quietly double-counts.
That's a data-quality bug you find months later in a billing report.

The dedup-table approach has a failure mode you must be able to state: **the write to the dedup store
and the side effect are not atomic.** If you record "processed" then crash before the side effect,
you've lost the record. So you record *after* the side effect and accept that a crash in between
means one duplicate — back to at-least-once, but now bounded to a crash window rather than every
rebalance. If the side effect genuinely cannot tolerate that, you need the side-effect system itself
to be idempotent (which is why every decent payment API takes an idempotency key).

**Code:**

```java
/**
 * BS-07 / BS-04 — the non-idempotent sink: a partner webhook. Two layers:
 *  1. a local processed-set so a Kafka redelivery is cheap to detect,
 *  2. an Idempotency-Key header so the PARTNER can also dedup (defence in depth —
 *     our Redis could be flushed; their store shouldn't be).
 */
@KafkaListener(topics = "alerts.v1")
public void onAlert(AlertEvent alert, Acknowledgment ack) {

    // alertId is DERIVED (hash of ruleId + windowStart), so a replay produces the same value.
    String key = "dedup:alert:" + alert.alertId();

    // SET NX EX — atomic "claim". Returns false if someone already processed it.
    Boolean claimed = redis.opsForValue().setIfAbsent(key, "1", Duration.ofHours(24));

    if (Boolean.FALSE.equals(claimed)) {
        meterRegistry.counter("alerts.duplicate.suppressed").increment();
        ack.acknowledge();
        return;
    }

    try {
        partnerClient.deliver(alert, alert.alertId());   // → Idempotency-Key header
        ack.acknowledge();
    } catch (RuntimeException e) {
        // Release the claim so the redelivery can retry. Forgetting this line turns a
        // transient failure into a permanently swallowed alert — a genuinely nasty bug.
        redis.delete(key);
        throw e;
    }
}
```

**Follow-ups they'll ask:**
- *"Where does the dedup key come from at ingest?"* — the client supplies `eventId` per event and an
  `Idempotency-Key` per batch (Phase 2, REQ). We `SET NX EX` on `eventId` in Redis. A test fires
  duplicate batches concurrently and asserts exactly one publish.
- *"What TTL?"* — longer than your maximum realistic replay window (retention + the time you'd take
  to notice and reset offsets), balanced against Redis memory. StreamPulse uses 24h for alerts and a
  shorter window for per-event ingest dedup; both are configurable and both are guesses until
  measured.
- *"Isn't Redis a single point of failure for correctness?"* — it's a *performance* optimisation for
  dedup; correctness must also hold at the sink (conditional write). If Redis is down I fail open on
  dedup and rely on the sink's condition, and I say so in the design doc.

**Pitfalls / red flags:** a dedup key that's a random UUID generated in the handler; or "we don't get
duplicates because we use exactly-once".

**Where in StreamPulse:** Phase 2 (ingest dedup), Phase 4 (conditional writes), BS-03 / REQ-KT-05,
BS-08 / REQ-SF-05.

---

## Q8. Walk me through Spring Kafka's listener container. How does `concurrency` relate to partitions?

**30-second answer:** `@KafkaListener` is a thin annotation over a **message listener container**.
`ConcurrentKafkaListenerContainerFactory` creates N child `KafkaMessageListenerContainer`s, each with
**its own `KafkaConsumer` on its own thread** — they are all members of the same consumer group, so
the group coordinator distributes partitions across them exactly as it would across separate pods.
The consequence is the number that trips people up: **`concurrency` above the partition count does
nothing** — the extra containers get no assignment and sit idle. Effective parallelism is
`min(concurrency × instances, partitions)`. And because each container polls on its own thread, your
listener method must be thread-safe or stateless; the container does not serialise across
partitions.

**Deeper:**

```
ConcurrentKafkaListenerContainerFactory (concurrency = 3)
   ├─ KafkaMessageListenerContainer #0 ── KafkaConsumer ── thread "sink-0-C-1" ── partitions 0..3
   ├─ KafkaMessageListenerContainer #1 ── KafkaConsumer ── thread "sink-0-C-2" ── partitions 4..7
   └─ KafkaMessageListenerContainer #2 ── KafkaConsumer ── thread "sink-0-C-3" ── partitions 8..11

  poll() ─▶ records ─▶ [RecordInterceptor] ─▶ MessageConverter ─▶ your @KafkaListener method
                                                   │
                                                   ├─ normal return  → ack per AckMode
                                                   └─ throws         → CommonErrorHandler (Q9)
```

**AckMode** — the choice that decides your delivery semantics:

| `AckMode` | Commits when | Use when |
|---|---|---|
| `RECORD` | after each record's listener returns | small volumes, tight duplicate control; slowest |
| `BATCH` | after all records from one `poll()` are processed (the container default) | the sane default for auto-ack |
| `TIME` | on `ackTime` elapsed | rarely |
| `COUNT` | every `ackCount` records | rarely |
| `COUNT_TIME` | whichever comes first | rarely |
| `MANUAL` | you call `Acknowledgment.acknowledge()`; queued and committed at the end of the poll batch | manual control, batched commits |
| `MANUAL_IMMEDIATE` | you call `acknowledge()`; committed **immediately** on the consumer thread | when you must not lose the commit — StreamPulse's choice for the S3 sink |

BS-03 / REQ-KT-05 requires manual ack and asks you to justify `MANUAL` vs `MANUAL_IMMEDIATE`.
My justification: `MANUAL_IMMEDIATE` costs an extra commit round-trip per batch, but it means the
commit point is *exactly* the line after the durable write — there's no window where the container
defers the commit and a shutdown reorders it. Given BS-01's zero-loss requirement, paying one
round-trip per batch (not per record) is cheap. For the Redis/DynamoDB sink, where reprocessing is
genuinely free, `BATCH` with auto-ack would also be defensible.

**Batch listeners** are the throughput lever: with `factory.setBatchListener(true)` your method
receives `List<ConsumerRecord<K,V>>` — the whole `poll()` result, up to `max.poll.records`. That lets
you do one `BatchWriteItem` to DynamoDB or one multipart S3 part instead of N single writes. The cost
is coarser error handling: one poison record in a batch of 200 fails the batch, which is exactly what
`BatchListenerFailedException` exists to solve (Q9).

**Code:**

```java
@Configuration
class KafkaListenerConfig {

    /**
     * BS-03 / REQ-KT-05 + BS-01 / REQ-LC-02.
     * concurrency=3 with 12 partitions and 4 pods → 12 consumers total = 1 per partition.
     * Going to concurrency=6 would give 24 consumers for 12 partitions: 12 idle threads
     * and 12 more members to rebalance. More is not faster.
     */
    @Bean
    ConcurrentKafkaListenerContainerFactory<String, MetricAggregate> batchFactory(
            ConsumerFactory<String, MetricAggregate> cf,
            CommonErrorHandler errorHandler) {

        var factory = new ConcurrentKafkaListenerContainerFactory<String, MetricAggregate>();
        factory.setConsumerFactory(cf);
        factory.setConcurrency(3);
        factory.setBatchListener(true);                       // one call per poll()
        factory.setCommonErrorHandler(errorHandler);
        factory.getContainerProperties()
               .setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);
        // Stop the container as part of the ordered drain (BS-01): SmartLifecycle phase 200.
        factory.getContainerProperties().setPhase(200);
        return factory;
    }
}

@Component
class AggregateSink {

    @KafkaListener(topics = "telemetry.agg.v1",
                   groupId = "sink-writer",
                   containerFactory = "batchFactory")
    public void consume(List<ConsumerRecord<String, MetricAggregate>> records,
                        Acknowledgment ack) {
        dynamo.batchWrite(records.stream().map(ConsumerRecord::value).toList());  // idempotent (Q7)
        ack.acknowledge();     // MANUAL_IMMEDIATE: committed right here, after the durable write
    }
}
```

**Follow-ups they'll ask:**
- *"How do you pause a consumer?"* — `KafkaListenerEndpointRegistry.getListenerContainer(id).pause()`
  (or `stop()`). `pause()` keeps the assignment and keeps heartbeating, so it does **not** trigger a
  rebalance — the right tool when a downstream dependency is down (pairs naturally with BS-04's
  circuit breaker going OPEN).
- *"Can you use virtual threads for listeners?"* — Spring for Apache Kafka can run listener
  containers on a virtual-thread executor, but it changes nothing about the polling model: one
  consumer per container, still one thread doing the poll loop. Virtual threads help if your
  *handler* is I/O-blocking and you fan out inside it. **Verify** the exact property/API in the
  Spring Kafka version you pin.
- *"How does a `@KafkaListener` deserialize?"* — `ErrorHandlingDeserializer` wrapping the real
  deserializer is the pattern you want, so a malformed payload becomes a handled record with a
  `DeserializationException` header instead of an infinite poison loop in the consumer's own
  deserializer, where your error handler can't reach it.

**Pitfalls / red flags:** setting `concurrency` to 50 on a 12-partition topic and expecting 50× the
throughput.

**Where in StreamPulse:** Phase 4, BS-03 / REQ-KT-05, BS-01 / REQ-LC-01.

---

## Q9. A record fails processing. Walk me through your error handling — retries, DLT, poison messages.

**30-second answer:** First I classify: is the failure **transient** (DynamoDB throttled, S3 503) or
**permanent** (schema violation, missing tenant, a null the code can't handle)? Transient failures get
retried; permanent ones must never be retried, because retrying a poison message blocks its partition
forever. Spring Kafka gives me two retry models. **Blocking retry** (`DefaultErrorHandler` with a
`BackOff`) retries in place on the consumer thread — it preserves ordering but stalls the whole
partition for the backoff duration, and if the backoff exceeds `max.poll.interval.ms` you trigger a
rebalance. **Non-blocking retry** (`@RetryableTopic`) publishes the failed record to a `-retry-0`,
`-retry-1`… topic with increasing delays and acks the original immediately — the partition keeps
moving, at the cost of losing ordering for retried records. My default for StreamPulse: short blocking
retry for genuinely instant-transient errors, then `@RetryableTopic` for anything slower, then a DLT
with an alert on its rate.

**Deeper:**

```
BLOCKING (DefaultErrorHandler + ExponentialBackOff)
  poll ─▶ rec#5 fails ─▶ sleep 1s ─▶ retry ─▶ sleep 2s ─▶ retry ─▶ recover(DLT)
          ▲ partition 3 is STOPPED for the whole 3s+. Ordering preserved.
          ▲ if total backoff > max.poll.interval.ms → rebalance storm (Q3)

NON-BLOCKING (@RetryableTopic)
  main topic ─▶ rec#5 fails ─▶ publish to alerts.v1-retry-0 ─▶ ACK main ─▶ partition keeps moving
                                     │ (consumer with 1s delay)
                                     └─ fails ─▶ -retry-1 (2s) ─▶ fails ─▶ -retry-2 (4s) ─▶ -dlt
  Ordering of retried records vs the main stream is LOST. That is the price.
```

| | Blocking retry | Non-blocking (`@RetryableTopic`) |
|---|---|---|
| Ordering | preserved | **lost** for retried records |
| Head-of-line blocking | yes — partition stalls | no |
| Rebalance risk | high if backoff is long | low |
| Extra topics | none | one per retry level + DLT |
| Good for | sub-second transients | 1s–minutes recovery (S3 outage, throttling) |
| Observability | error-handler metrics | topic-level lag per retry tier — you can *see* the retry backlog |

**Poison messages.** A record that will fail on every attempt until the code changes. Three defences:
1. `ErrorHandlingDeserializer` so deserialization failures become handled records, not consumer-loop
   exceptions.
2. `DefaultErrorHandler.addNotRetryableExceptions(...)` — classify `ValidationException`,
   `SerializationException`, `IllegalArgumentException` as fatal so they go straight to the DLT with
   zero retries. This is the single highest-value line in the config.
3. **Alert on DLT rate**, not just DLT depth. A DLT that is quietly filling is a silent outage.

**Batch listeners + poison** is the trap: if record 137 of a 200-record batch fails, a naive handler
retries all 200, and the first 136 side effects happen again. `BatchListenerFailedException(message,
index)` tells `DefaultErrorHandler` exactly which record failed, so it commits up to that record,
sends the bad one to the DLT, and resumes at index+1.

**Code:**

```java
@Bean
CommonErrorHandler errorHandler(KafkaTemplate<Object, Object> template) {

    // Recoverer: publish the failed record to "<topic>-dlt", same partition number.
    // Spring adds headers: original topic/partition/offset, exception class, stacktrace.
    var recoverer = new DeadLetterPublishingRecoverer(template,
            (record, ex) -> new TopicPartition(record.topic() + "-dlt", record.partition()));

    // 3 retries at 200ms, 400ms, 800ms = 1.4s total — deliberately far below
    // max.poll.interval.ms (120s, Q3) so blocking retry can never cause a rebalance.
    var backOff = new ExponentialBackOffWithMaxRetries(3);
    backOff.setInitialInterval(200L);
    backOff.setMultiplier(2.0);
    backOff.setMaxInterval(2_000L);

    var handler = new DefaultErrorHandler(recoverer, backOff);

    // POISON CLASSIFICATION — these will never succeed, so retrying is pure waste and
    // pure head-of-line blocking. Straight to the DLT.
    handler.addNotRetryableExceptions(
            ValidationException.class,
            DeserializationException.class,
            UnknownTenantException.class);

    // Transient ones we DO want retried (everything not listed above is retried by default;
    // being explicit documents intent for the next reader).
    handler.setBackOffFunction((record, ex) ->
            ex instanceof ProvisionedThroughputExceededException ? longerBackOff() : backOff);

    return handler;
}
```

```java
/**
 * BS-04 / Phase 4: the partner-delivery path, where recovery takes seconds to minutes
 * (S3 503s for 3 minutes, a throttled partner). Blocking here would stall the partition
 * for minutes; non-blocking keeps the main stream flowing.
 */
@RetryableTopic(
        attempts = "4",                                  // 1 original + 3 retry topics
        backoff = @Backoff(delay = 1000, multiplier = 3.0, maxDelay = 30_000),
        autoCreateTopics = "true",
        exclude = { ValidationException.class },          // poison → straight to DLT
        dltStrategy = DltStrategy.FAIL_ON_ERROR)
@KafkaListener(topics = "alerts.v1", groupId = "partner-delivery")
public void deliver(AlertEvent alert) {
    partnerClient.deliver(alert, alert.alertId());
}

@DltHandler
void parked(AlertEvent alert,
            @Header(KafkaHeaders.ORIGINAL_TOPIC) String topic,
            @Header(KafkaHeaders.EXCEPTION_MESSAGE) String reason) {
    meterRegistry.counter("alerts.dlt", "reason", classify(reason)).increment();
    dltRepository.park(alert, topic, reason);      // an admin endpoint can replay from here
}
```

**Follow-ups they'll ask:**
- *"What's the default retry behaviour if you configure nothing?"* — Spring Kafka installs a
  `DefaultErrorHandler` with a fixed back-off; the historical default was 10 delivery attempts with no
  delay, then log-and-skip. **Verify** the default `BackOff` for your Spring Kafka version — do not
  rely on it in production either way; configure it explicitly.
- *"How do you replay a DLT?"* — a small admin consumer that reads the DLT and republishes to the
  original topic (the original topic/partition/offset headers tell you where it came from), gated
  behind an admin endpoint and audited. Never an automatic loop — that's an infinite retry with extra
  steps.
- *"How do you stop the whole pipeline when the *dependency* is down, rather than DLT-ing everything?"*
  — pause the container. If the circuit breaker for DynamoDB is OPEN (BS-04 / REQ-RS-02), sending 50k
  records to a DLT is the wrong response: pause, let lag build (lag is recoverable, a DLT flood is
  manual work), resume when the breaker closes.
- *"How does ordering survive `@RetryableTopic`?"* — it doesn't, and for a keyed stream that matters.
  For StreamPulse's alert delivery, alerts are independent per `alertId`, so it's acceptable. For
  `telemetry.raw.v1`, where session ordering is the point, I use blocking retry + pause instead.

**Pitfalls / red flags:** infinite retry with no classification, or a DLT nobody monitors.

**Where in StreamPulse:** Phase 4 (`@RetryableTopic` + DLT + poison-message test), BS-04.

**Verify:** `DefaultErrorHandler`'s default `BackOff`, and `@RetryableTopic` attribute names/defaults
in your Spring Kafka version (this annotation has evolved).

---

## Q10. What single metric tells you the pipeline is healthy? How do you act on it?

**30-second answer:** **Consumer lag** — per topic, per partition, per group. Lag is
`highWaterMark − committedOffset`, i.e. how many records the group hasn't processed. I care about it
because it's the only metric that's *causal*: rising lag means consumption is slower than production,
and every user-visible symptom (stale dashboard, delayed alert) is downstream of it. But I watch two
refinements. First, **per-partition** lag, not the sum — an aggregate hides a hot partition or a stuck
member. Second, **lag in time, not records**: 100k records of lag means nothing without throughput.
`lag / consumption rate` gives "how far behind in seconds", which is what the SLO is written in.
StreamPulse's real SLI is **freshness** — event timestamp to SSE delivery — and lag is the leading
indicator that predicts it.

**Deeper:**

```
records  ─── produced ─────────────────────────────────────────▶ HWM
                                        ▲
                               lag ─────┤
                                        ▼
         ─── consumed & committed ─────▶ committed offset

health signals, in the order I'd add them to a dashboard:
  1. lag per (group, topic, partition)          — the causal metric
  2. lag-in-seconds = lag / records-consumed-rate — the SLO-shaped version
  3. records-consumed-rate per member           — is any member at zero? (stuck/idle)
  4. rebalance rate per group                   — > 0 steady-state means something is wrong
  5. DLT publish rate                           — silent data loss if unwatched
  6. end-to-end freshness (event.ts → SSE)      — the actual user-facing SLI (BS-10 / REQ-SL-01)
```

**Lag goes up — the triage tree** (this is the question behind the question; they're asking whether
you've been on call):

| Observation | Likely cause | Action |
|---|---|---|
| All partitions rising evenly | Genuine load increase, or a slow downstream | Scale consumers (up to partition count); check downstream p99 |
| One partition rising, others flat | Hot key (Q2), or a stuck member | Check per-key distribution; check that member's thread dump |
| Lag rising *and* rebalance rate > 0 | Poll-interval violation (Q3) | Lower `max.poll.records`, bound handler time |
| Lag flat but freshness bad | Producer-side delay (`linger.ms` too high) or a stalled LSO (Q6) | Check producer batch metrics, check for open transactions |
| Lag at zero and freshness bad | The problem isn't Kafka — look at the SSE fan-out or the query path | |
| Lag rising, consumers idle, CPU low | Downstream circuit breaker OPEN and container paused | That's the system working as designed (BS-04) — alert on breaker state, not lag |

**Scaling on lag** is what KEDA does in Phase 11: the `kafka` scaler reads the group's lag and scales
the deployment. Two things to say about it: (1) you cannot scale past the partition count, so the HPA
max must be `partitions / concurrency`; (2) every scale event is a rebalance, so a twitchy autoscaler
on a stateful consumer causes more harm than the lag did — you need stabilisation windows. For Kafka
Streams (stateful, RocksDB), scaling also means **state restoration**, which is why standby replicas
matter (Q15).

**Code:**

```java
/**
 * Phase 8. Kafka's client already exposes lag via JMX (records-lag / records-lag-max per
 * partition), and Spring Boot's Micrometer Kafka metrics bridge them. This bean adds the
 * TIME-based view, which is what the SLO is written against (BS-10 / REQ-SL-01).
 */
@Component
class FreshnessMetrics {

    private final Timer freshness;

    FreshnessMetrics(MeterRegistry registry) {
        this.freshness = Timer.builder("streampulse.freshness")
                .description("event timestamp → processed, the user-visible staleness")
                .publishPercentiles(0.5, 0.95, 0.99)
                .register(registry);
    }

    void record(TelemetryEvent e) {
        // NOTE: this is wall-clock across machines, so it includes clock skew. Fine as an SLI
        // when your hosts run NTP; say so in the design doc rather than pretending it's exact.
        freshness.record(Duration.between(e.ts(), Instant.now()));
    }
}
```

```yaml
# Prometheus alert (Phase 8). Alert on the TREND and the SLO, not on a raw record count.
groups:
  - name: streampulse-kafka
    rules:
      - alert: ConsumerLagGrowing
        # 15 min of sustained positive derivative = we are falling behind, not just spiky
        expr: deriv(kafka_consumergroup_lag{group="sink-writer"}[15m]) > 0
              and kafka_consumergroup_lag{group="sink-writer"} > 50000
        for: 10m
      - alert: RebalanceStorm
        expr: rate(kafka_consumer_coordinator_rebalance_total[5m]) > 0.05
        for: 5m
      - alert: DltRateNonZero
        expr: rate(kafka_producer_record_send_total{topic=~".*-dlt"}[5m]) > 0
        for: 2m
```

**Follow-ups they'll ask:**
- *"Where do you get lag from?"* — the consumer's own JMX metrics (`records-lag-max`) are per-member
  and disappear when the member dies; a broker-side exporter (`kafka_exporter`, Burrow, or
  `kafka-consumer-groups.sh --describe`) survives that and is what you alert on. Use both: client
  metrics for per-member debugging, broker-side for alerting. **Verify** the exact Micrometer metric
  names your Spring Boot version exposes.
- *"Is zero lag good?"* — zero lag with high freshness latency means the producer is the bottleneck.
  Zero lag on a topic that should have traffic means the producer is *dead*. "No lag" is not "no
  problem"; you also need a throughput floor alert.
- *"Lag spiked to 2M overnight — what's your first move?"* — check whether it's rising or draining
  (derivative), check per-partition skew, check rebalance rate, check whether the consumer is paused
  by a breaker. Then decide: scale, or fix the downstream. Don't reset offsets — that's data loss
  dressed as a fix.

**Pitfalls / red flags:** naming "throughput" or "CPU" as the health metric, or not distinguishing
lag-in-records from lag-in-time.

**Where in StreamPulse:** Phase 8 (dashboards), Phase 11 (KEDA), BS-10 / REQ-SL-01.

---

## Q11. BS-03: cut Kafka bytes by 30% without breaking freshness. Which knobs, and what do they trade?

**30-second answer:** Bytes on the wire and on disk are dominated by **batching and compression**, so
the two knobs that matter are `linger.ms` and `compression.type`, with `batch.size` as the ceiling.
`linger.ms` is the producer deliberately *waiting* to accumulate a fuller batch — it trades latency
for compression ratio, because compression is applied per batch and a bigger batch compresses far
better. `compression.type=zstd` usually gives the best ratio for JSON-ish/Avro records, `lz4` the best
ratio-per-CPU. On the consumer side, `fetch.min.bytes` + `fetch.max.wait.ms` do the mirror-image job:
fewer, larger fetches instead of many tiny ones. My method for BS-03 isn't to pick values from a blog
— it's to run the matrix under Gatling and record `compression-rate-avg`, `record-size-avg`, broker
log size, p99 freshness and lag per configuration, then pick the point that hits 30% bytes reduction
while staying inside the 5s freshness budget.

**Deeper:**

| Knob | Side | Raising it… | Costs |
|---|---|---|---|
| `linger.ms` | producer | fuller batches → better compression, fewer requests | adds up to that much latency per record |
| `batch.size` | producer | allows bigger batches (bytes, per partition) | memory in the accumulator; no effect if `linger.ms=0` and traffic is sparse |
| `compression.type` | producer | fewer bytes on wire **and disk** (brokers store the compressed batch) | producer + consumer CPU |
| `buffer.memory` | producer | more headroom before `send()` blocks | memory; hides backpressure |
| `fetch.min.bytes` | consumer | broker waits for that much data → fewer, bigger fetches | adds up to `fetch.max.wait.ms` latency |
| `fetch.max.wait.ms` | consumer | bounds the above wait | — |
| `max.poll.records` | consumer | more records per handler call → better batch writes | must respect the Q3 inequality |
| `concurrency` (Spring) | consumer | more consumers per pod | capped by partitions; more rebalance units |

**The key insight about compression and the broker:** Kafka brokers normally store the batch
*as the producer compressed it* (when the topic's `compression.type=producer`), and serve it
compressed to consumers. So producer-side compression reduces **network in, disk, and network out** —
three costs for one CPU price. If you set a *different* codec on the topic, the broker must decompress
and recompress, which costs broker CPU and defeats zero-copy on the read path. That's why the topic
config in Q1 says `producer`.

**The interaction people miss:** `linger.ms` and compression multiply. At `linger.ms=0` with sparse
traffic you compress batches of one record, where the codec overhead can make the batch *larger*. At
`linger.ms=20` under load you might compress 500 records at once. The bytes saving isn't from turning
compression on — it's from turning compression on **over a batch worth compressing**. That's the whole
BS-03 story in one sentence, and it's the sentence I'd say in the interview.

**The arithmetic I'd show** (illustrative, showing the method — the real numbers come from Phase 9):

```
baseline : linger.ms=0,  compression=none, avg record 800 B, 50k ev/s
           → 50,000 × 800 B  = 40 MB/s produced
             × replication factor 3 on the broker network path
             + 2 consumer groups reading   → the disk and egress bill

tuned    : linger.ms=20, compression=zstd
           measured compression-rate-avg   = R   (Kafka reports this — do not guess it)
           → 40 MB/s × R
           bytes reduction = (1 − R)

  To hit REQ-KT's ≥30% target you need R ≤ 0.70. Whether zstd achieves that on YOUR payload
  depends entirely on the payload's entropy — Avro binary with repeated tenant/title strings
  compresses well; already-compressed blobs don't compress at all. MEASURE IT.

freshness budget check:
  added producer latency ≤ linger.ms          = 20 ms
  added consumer latency ≤ fetch.max.wait.ms  = 100 ms
  total added ≈ 120 ms against a 5,000 ms p99 budget → 2.4% of budget. Acceptable.
```

**Code:**

```yaml
# REQ-KT-01: the two profiles the benchmark switches between. Same code, same load, two configs.
---
spring:
  config.activate.on-profile: kafka-baseline
  kafka:
    producer:
      properties: { linger.ms: 0, batch.size: 16384, compression.type: none }
    consumer:
      properties: { fetch.min.bytes: 1, fetch.max.wait.ms: 500, max.poll.records: 500 }
---
spring:
  config.activate.on-profile: kafka-tuned
  kafka:
    producer:
      acks: all
      properties:
        enable.idempotence: true          # REQ-KT-03 — non-negotiable, both profiles
        linger.ms: 20                     # ← the latency we are deliberately spending
        batch.size: 65536                 # 64 KB ceiling per partition batch
        compression.type: zstd            # ← the bytes we are buying
        max.in.flight.requests.per.connection: 5
    consumer:
      properties:
        fetch.min.bytes: 65536            # don't wake up for 200 bytes
        fetch.max.wait.ms: 100            # ...but never wait more than 100ms for it
        max.poll.records: 200             # Q3 inequality: 200 × 40ms = 8s ≪ 120s
    listener:
      concurrency: 3
```

```java
/**
 * REQ-KT-02. The metrics that PROVE the change — all of them come from Kafka's own
 * producer/consumer metrics, not from a guess. This is what goes in docs/kafka-tuning.md.
 */
record TuningResult(
        String profile,
        double recordSizeAvg,        // producer metric: record-size-avg
        double compressionRateAvg,   // producer metric: compression-rate-avg  ← the money metric
        double outgoingByteRate,     // producer metric: outgoing-byte-rate
        long   brokerLogBytes,       // from the broker / kafka-log-dirs.sh
        double p99FreshnessMs,       // our own Timer (Q10)
        long   maxLag) {}
```

**Follow-ups they'll ask:**
- *"gzip vs snappy vs lz4 vs zstd?"* — the ordering is roughly: gzip = best ratio historically but
  slowest; snappy and lz4 = fast, moderate ratio; zstd = usually the best ratio-per-CPU on modern
  hardware and tunable by level. But this is payload-dependent and CPU-dependent: **measure on your
  data**, that's the whole point of REQ-KT-02. I would not quote a ratio in an interview without
  saying where I measured it.
- *"What did raising `linger.ms` do to p99?"* — bounded it by exactly `linger.ms` in the worst case,
  and often *improved* throughput enough that queueing delay elsewhere fell. Under load the producer
  is already batching, so `linger.ms` mostly matters for the sparse tail.
- *"How do you keep the guarantee while tuning?"* — REQ-KT-03: `enable.idempotence=true` and
  `acks=all` are fixed in both profiles; only latency/size knobs vary. Tuning must not be allowed to
  silently buy throughput with durability.
- *"What about `max.request.size` and `message.max.bytes`?"* — if you raise `batch.size` a lot you can
  hit the broker's per-request limit. They must be raised together (producer, broker, and the topic's
  `max.message.bytes`), or you get confusing `RecordTooLargeException`s.

**Pitfalls / red flags:** quoting "linger.ms=5 and snappy is best practice" with no measurement. Also:
optimising bytes while quietly dropping `acks` to 1.

**Where in StreamPulse:** BS-03 / REQ-KT-01…06, Phase 9, `docs/kafka-tuning.md`,
`KafkaTuningBenchmark`.

**Verify:** default values for `linger.ms`, `batch.size`, `fetch.min.bytes`, `fetch.max.wait.ms`,
`max.poll.records`, and which compression codecs your broker version supports.

---

## Q12. Log compaction vs retention — when do you use which?

**30-second answer:** They answer different questions. **Retention** (`cleanup.policy=delete`) is
"keep the last N hours/GB of the stream and then drop the oldest" — it's for *event* topics where old
events stop mattering. **Compaction** (`cleanup.policy=compact`) is "keep at least the latest value
for every key, forever" — it turns the topic into a durable, replayable **changelog / key-value
snapshot**, so a new consumer can bootstrap complete current state by reading from the beginning. In
StreamPulse, `telemetry.raw.v1` is `delete` with a 3-day retention (the replay window), while
`config.changes.v1` — tenants, alert rules from Debezium CDC — is **compacted**, because a service
starting up must be able to rebuild its entire config cache from the topic without a database call.
Kafka Streams' internal changelog topics for state stores are compacted for exactly the same reason.

**Deeper:**

```
cleanup.policy=delete                     cleanup.policy=compact
 k1 k2 k1 k3 k2 k1                         k1 k2 k1 k3 k2 k1
 └──── dropped ───┘ └─ kept ─┘             └── compacted (background) ──┐
 (by age or total size)                     result:  k3 k2 k1   ← latest value per key survives
                                            head of the log is NEVER compacted (it's "dirty")
```

| | `delete` | `compact` |
|---|---|---|
| Unit of removal | whole segments, by age/size | individual records superseded by a later same-key record |
| Needs keys? | no | **yes** — a null key is undeletable garbage in a compacted topic |
| Deletion of a key | n/a | a **tombstone**: a record with that key and a `null` value |
| Replay gives you | a window of history | current state (plus whatever history hasn't compacted yet) |
| Order preserved | yes | yes, for surviving records |
| Guarantee | "last N hours" | "at least the latest value per key" — *not* "exactly one record per key" |

The two most misunderstood details:

1. **Compaction is asynchronous and best-effort.** The log cleaner runs in the background, and the
   *active* segment plus anything newer than `min.compaction.lag.ms` is never compacted. So a consumer
   reading from offset 0 will see duplicate keys — it just sees the latest one *last*, which is all
   you need for building a map. Never claim "a compacted topic has one record per key".
2. **Tombstones expire.** A `null`-valued record marks a key deleted, and it is itself removed after
   `delete.retention.ms`. If a consumer is offline longer than that and then does a full replay, it
   will never learn about the deletion and will resurrect deleted state. That's a genuine production
   bug class, and naming it is a strong senior signal.

`cleanup.policy=compact,delete` is legal and useful: keep the latest value per key *and* drop anything
older than N days. Kafka Streams windowed-store changelogs use exactly this, because window state
should not live forever.

**Code:**

```java
/**
 * config.changes.v1 — BS-08 / REQ-SF-02, Phase 6.
 * Debezium's outbox router publishes tenant/rule changes here. Every service consumes the
 * WHOLE topic at startup to build its config cache, then follows the tail. Compaction is what
 * makes that bootstrap bounded: without it, startup would replay years of change history.
 */
@Bean
NewTopic configChanges() {
    return TopicBuilder.name("config.changes.v1")
            .partitions(6)
            .replicas(3)
            .config(TopicConfig.CLEANUP_POLICY_CONFIG, TopicConfig.CLEANUP_POLICY_COMPACT)
            .config(TopicConfig.MIN_CLEANABLE_DIRTY_RATIO_CONFIG, "0.1")  // compact eagerly
            .config(TopicConfig.DELETE_RETENTION_MS_CONFIG, "604800000")  // tombstones live 7 days
            .config(TopicConfig.MIN_COMPACTION_LAG_MS_CONFIG, "60000")
            .build();
}
```

```java
// Deleting an alert rule = a TOMBSTONE, not an "op=DELETE" payload.
// Sending a real payload with a deleted flag would keep the record alive forever
// and grow the compacted topic without bound.
kafkaTemplate.send("config.changes.v1", "rule:" + ruleId, null);
```

**Follow-ups they'll ask:**
- *"Can you use a compacted topic as a database?"* — as a *changelog* you replay into a local
  materialised view, yes — that's exactly what a `KTable` and a Kafka Streams state store are. As a
  query-serving store with random reads, no: Kafka has no index by key.
- *"What breaks if a producer to a compacted topic sends null keys?"* — those records can never be
  compacted away and the topic grows unbounded. Enforce non-null keys in the contract.
- *"How does this relate to Kafka Streams?"* — every persistent state store gets a compacted changelog
  topic; on a rebalance, a new instance restores its RocksDB store by replaying that changelog. That
  restore time is why standby replicas exist (Q15).

**Pitfalls / red flags:** "compaction deduplicates my topic" — it deduplicates *by key, eventually,
and only outside the active segment*.

**Where in StreamPulse:** Phase 1 (topic contracts), Phase 6 (CDC), Phase 3 (Streams changelogs).

**Verify:** default `delete.retention.ms`, `min.cleanable.dirty.ratio`, `segment.ms` for your broker.

---

## Q13. How do you evolve an event schema without breaking consumers?

**30-second answer:** Put the schema in a registry, pick a **compatibility mode**, and let the build
fail on an incompatible change. StreamPulse uses **Avro + Schema Registry with BACKWARD
compatibility** (Phase 1), which means a new schema can read data written with the previous schema —
so you can upgrade **consumers first**, then producers. Practically, BACKWARD allows *deleting* a
field and *adding a field with a default*, and forbids adding a required field or changing a type
incompatibly. Producers register the writer schema; the payload on the wire carries a 4-byte schema
ID, not the schema, so the encoding stays small, and the consumer fetches the writer schema by ID and
Avro resolves writer→reader. Versioning the *topic name* (`telemetry.raw.v1`) is the escape hatch for
a genuinely breaking change: you run v1 and v2 side by side and migrate consumers, rather than
pretending a breaking change is compatible.

**Deeper:**

| Mode | New schema can read old data | Old schema can read new data | Upgrade order | Allows |
|---|---|---|---|---|
| `BACKWARD` (common default) | ✅ | ❌ | **consumers first** | delete field; add field **with default** |
| `FORWARD` | ❌ | ✅ | **producers first** | add field; delete field **with default** |
| `FULL` | ✅ | ✅ | either | add/delete optional (defaulted) fields only |
| `*_TRANSITIVE` | vs **all** previous versions, not just the last | | | the strict version |
| `NONE` | — | — | — | anything; i.e. no protection |

The non-transitive modes check only against the **latest** version. That's a real trap: three
successive BACKWARD-compatible changes can leave you unable to read the *oldest* data still in
retention. If you keep 3 days of data and deploy daily, `BACKWARD_TRANSITIVE` is the honest setting.

**Wire format** (Confluent's): `[magic byte 0x00][4-byte big-endian schema ID][Avro binary payload]`.
Consequences worth knowing: the schema is not in the message, so you cannot decode a Kafka message
with `cat` and you cannot decode it at all without registry access; and the registry becomes a
runtime dependency of every producer and consumer (clients cache aggressively, so a registry blip
doesn't immediately stop traffic, but a cold-started pod during a registry outage will fail —
mention this, it's an availability coupling people forget).

**Avro vs Protobuf vs JSON Schema** — all three are supported by Confluent Schema Registry:

| | Avro | Protobuf | JSON Schema |
|---|---|---|---|
| Wire size | small (binary, no field names) | small (binary, tag numbers) | large (text) |
| Schema evolution | defaults + writer/reader resolution | field numbers + `optional` | additive-ish, weaker |
| Tooling in Java | Avro Gradle plugin → generated `SpecificRecord` | protoc → generated classes | POJOs |
| Cross-language | good | excellent | universal |
| StreamPulse choice | ✅ Kafka payloads (`libs/contracts`) | gRPC ingest (BS-09) | REST DTOs via OpenAPI |

Using Avro for Kafka and Protobuf for gRPC in the same repo is a deliberate choice worth defending:
Protobuf is the native gRPC IDL, Avro is the native Kafka/Schema-Registry ecosystem, and `libs/contracts`
owns both plus the mapping so the two never drift (the `ProtocolParityTest` in BS-09 / REQ-GR-02 proves
it).

**Code:**

```json
// schemas/avro/TelemetryEvent.avsc — Phase 1.
// Every added field has a DEFAULT. That single rule is what makes BACKWARD work.
{
  "type": "record",
  "name": "TelemetryEvent",
  "namespace": "com.streampulse.contracts",
  "fields": [
    { "name": "eventId",    "type": "string" },
    { "name": "tenantId",   "type": "string" },
    { "name": "sessionId",  "type": "string" },
    { "name": "titleId",    "type": ["null", "string"], "default": null },
    { "name": "region",     "type": "string" },
    { "name": "ts",         "type": { "type": "long", "logicalType": "timestamp-millis" } },
    { "name": "type",       "type": { "type": "enum", "name": "EventType",
                                      "symbols": ["SESSION_START","HEARTBEAT","BUFFERING_START",
                                                  "BUFFERING_END","BITRATE_CHANGE","ERROR",
                                                  "SESSION_END"],
                                      "default": "HEARTBEAT" } },
    // v2 addition — has a default, so a v2 reader can read v1 data. BACKWARD-compatible.
    { "name": "cdnPop",     "type": ["null", "string"], "default": null }
  ]
}
```

```java
/**
 * Phase 1: the test that makes the BUILD fail on an incompatible change, rather than
 * discovering it when a consumer pod crash-loops in staging. This is the deliverable.
 */
@Test
void addingARequiredFieldIsRejected() {
    var v1 = new Schema.Parser().parse(resource("TelemetryEvent.avsc"));
    var v2 = new Schema.Parser().parse(resource("TelemetryEvent-bad.avsc"));  // new field, NO default

    var result = new SchemaValidatorBuilder()
            .canReadStrategy()            // "can the new schema read data written with the old one?"
            .validateLatest()
            .validate(v2, List.of(v1));   // BACKWARD semantics

    // (Avro's validator throws on failure; assert that it does.)
    assertThatThrownBy(() -> result).isInstanceOf(SchemaValidationException.class);
}
```

**Enum default** is the detail that catches people: adding a new enum symbol is only readable by an
old consumer if the enum declares a `default` symbol (unknown symbols resolve to it). Without that,
adding `SESSION_PAUSE` breaks every older reader.

**Follow-ups they'll ask:**
- *"You must add a required field. What do you do?"* — either add it as optional-with-default and
  backfill, enforcing it at the application layer during a migration window, or publish
  `telemetry.raw.v2` and dual-write while consumers migrate. Never `NONE` compatibility "just for
  this one".
- *"Where does the compatibility check run?"* — in CI against the registry (`maven/gradle` schema
  registry plugin's `testCompatibility` task), so the PR fails, not the deploy. **Verify** the plugin
  name/task for your build and registry.
- *"Subject naming strategy?"* — default is `TopicNameStrategy` (`<topic>-value`), which means one
  schema type per topic. `RecordNameStrategy` / `TopicRecordNameStrategy` let you put multiple event
  types on one topic — useful when ordering across types matters (all events for one session on one
  topic), which is actually StreamPulse's case if we ever split event types.

**Pitfalls / red flags:** "we use JSON so we don't need schemas" — you still have a schema, it's just
undocumented and enforced at 3am.

**Where in StreamPulse:** Phase 1 (`libs/contracts`, BACKWARD enforced, a test proves an incompatible
change fails the build), BS-09 (Protobuf side).

**Verify:** the default compatibility mode of your registry deployment (it is configurable globally
and per subject), and the exact Gradle plugin/task names.

---

## Q14. Kafka Streams: what is a topology, and what's the difference between `KStream`, `KTable` and `GlobalKTable`?

**30-second answer:** A **topology** is a DAG of processing nodes — sources, processors, sinks — that
Kafka Streams compiles from your DSL calls and then runs as **tasks**, one task per input partition,
distributed across the instances of your application (they form a consumer group, so scaling is the
same story as Q3). The three abstractions differ in how they interpret a record: a **`KStream`** is an
unbounded sequence of independent facts (append semantics — "a buffering event happened"); a
**`KTable`** is a changelog where each record *replaces* the previous value for that key (upsert
semantics — "the current rebuffer ratio for this title is X"); a **`GlobalKTable`** is a `KTable`
replicated in full to *every* instance rather than partitioned, so you can join against it by any
attribute without repartitioning — at the cost of every instance holding the whole dataset.

**Deeper:**

| | `KStream` | `KTable` | `GlobalKTable` |
|---|---|---|---|
| Semantics | append / fact | upsert / current state | upsert / current state |
| Partitioning | partitioned (one task per partition) | partitioned | **fully replicated to every instance** |
| Null value means | a record whose value is null | **delete** (tombstone) | delete |
| Join requires co-partitioning | yes | yes | **no** |
| Join is | keyed | keyed | by any extractable key, non-keyed |
| Time semantics on join | windowed for stream-stream | non-windowed for table joins | non-windowed |
| Backed by | nothing (stateless) unless aggregated | a state store + compacted changelog | a state store, bootstrapped fully at startup |
| Good for | events | dimension/config data that's partitioned the same way | small, global reference data |

**Co-partitioning** is the requirement that trips everyone up: to join two `KStream`s or `KTable`s,
they must have the **same number of partitions** and the **same key and partitioning strategy**, so
that matching keys live on the same task. If they don't, Kafka Streams either errors or silently needs
a **repartition** — it writes the data to an internal `<app-id>-<name>-repartition` topic keyed
correctly and reads it back. That's a full network round-trip through the broker, plus disk, plus
latency, per repartition. Anything that changes the key triggers it: `selectKey`, `map` (as opposed to
`mapValues`), `groupBy` (as opposed to `groupByKey`), `flatMap`.

**The practical rule I follow:** prefer `mapValues`/`groupByKey`/`flatMapValues` — the value-only
variants — because they tell Kafka Streams the key is unchanged and no repartition is needed. Reach
for `selectKey`/`groupBy` only when you genuinely need a different key, and then be aware you just
added a topic to the pipeline.

**State stores** are RocksDB (an embedded LSM key-value store) on local disk, with an in-memory option
for small state. Two things follow: (1) a Kafka Streams pod is **stateful**, so in Kubernetes it wants
a StatefulSet with a PVC, not a stateless Deployment — otherwise every restart re-downloads the whole
changelog; (2) each store is backed by a **compacted changelog topic**, so state is durable and
recoverable even if the local disk is lost.

**Code:** the StreamPulse topology (Phase 3), with the repartition decisions called out:

```java
@Component
class AggregationTopology {

    @Autowired
    void build(StreamsBuilder builder) {

        KStream<String, TelemetryEvent> raw =
                builder.stream("telemetry.raw.v1",
                        Consumed.with(Serdes.String(), avroSerde)
                                // Event time, not wall clock: late/out-of-order events land in the
                                // window they BELONG to, not the window they arrived in (Q15).
                                .withTimestampExtractor((rec, partitionTime) ->
                                        ((TelemetryEvent) rec.value()).getTs()));

        // Branch out invalid events to a DLT instead of poisoning the aggregation.
        Map<String, KStream<String, TelemetryEvent>> branches = raw.split(Named.as("b-"))
                .branch((k, v) -> v.getTenantId() == null, Branched.as("invalid"))
                .defaultBranch(Branched.as("valid"));

        branches.get("b-invalid").to("telemetry.raw.v1-dlt");

        KStream<String, TelemetryEvent> valid = branches.get("b-valid");

        // ⚠ REPARTITION: input is keyed by sessionId (Q2), but we aggregate by
        // (tenant, title, region). selectKey changes the key → Kafka Streams inserts an
        // internal repartition topic here. This is a REAL COST (extra write + read of the
        // full stream) and it is a deliberate, documented decision, not an accident.
        KTable<Windowed<String>, Long> concurrentViewers = valid
                .selectKey((sessionId, e) -> e.getTenantId() + "|" + e.getTitleId() + "|" + e.getRegion())
                .groupByKey(Grouped.with(Serdes.String(), avroSerde))
                .windowedBy(TimeWindows.ofSizeAndGrace(Duration.ofSeconds(10), Duration.ofSeconds(30)))
                .count(Materialized.as("concurrent-viewers-store"));   // → RocksDB + changelog topic

        // GlobalKTable: title metadata is small (thousands of rows) and every instance needs
        // to enrich by titleId regardless of partitioning. Fully replicated → no repartition.
        GlobalKTable<String, TitleMeta> titles =
                builder.globalTable("config.titles.v1",     // compacted (Q12)
                        Materialized.as("titles-store"));

        concurrentViewers.toStream()
                .map((windowedKey, count) -> KeyValue.pair(windowedKey.key(),
                        MetricAggregate.of(windowedKey, count)))
                .join(titles,
                        (key, agg) -> agg.getTitleId(),   // ← join on ANY attribute; this is the
                                                          //   GlobalKTable superpower
                        (agg, meta) -> agg.withTitleName(meta.getName()))
                .to("telemetry.agg.v1", Produced.with(Serdes.String(), aggSerde));
    }
}
```

**Follow-ups they'll ask:**
- *"When is a `GlobalKTable` the wrong choice?"* — when the dataset is large, because every instance
  holds a full copy in RAM/RocksDB and bootstraps it fully at startup (slow starts). Also, global
  tables have weaker time-synchronisation semantics on joins — the join uses whatever the global table
  currently holds, not a time-aligned view. Small, slow-changing reference data only.
- *"How does Kafka Streams scale?"* — tasks = max partitions across the sub-topology; instances share
  tasks via the consumer group. So the same rule as Q3: you cannot scale past partition count.
  `num.stream.threads` controls tasks per instance.
- *"What are standby replicas?"* — `num.standby.replicas=1` makes another instance passively follow a
  task's changelog and keep a warm copy of its state store. Without it, losing an instance means the
  new owner restores RocksDB from the changelog topic from scratch — minutes of unavailability for a
  large store. With it, failover is near-immediate at the cost of extra disk and network. For any
  stateful Streams app in production I'd set at least 1.
- *"How do you test a topology?"* — `TopologyTestDriver`: no broker, no Docker, millisecond tests. You
  pipe records into a `TestInputTopic` with explicit timestamps and read a `TestOutputTopic`. It is
  the only sane way to test out-of-order and late-arrival behaviour, because you control event time
  directly.

**Pitfalls / red flags:** not knowing that `map`/`selectKey`/`groupBy` cause a repartition, or
treating a `GlobalKTable` as "a KTable but easier".

**Where in StreamPulse:** Phase 3 (topology, EOS v2, state stores, standby replicas documented),
Phase 11 (StatefulSet + state restoration on scale).

**Verify:** `TimeWindows.ofSizeAndGrace` / `ofSizeWithNoGrace` replaced the older `TimeWindows.of(...)`
— confirm the API for your Kafka Streams version, and the default `num.standby.replicas` (I believe 0).

---

## Q15. Windowing: tumbling vs hopping vs session. What is a grace period, and what happens to late events?

**30-second answer:** Windows group records by **event time** into buckets. **Tumbling** windows are
fixed-size and non-overlapping (10s, 10s, 10s) — every record belongs to exactly one. **Hopping**
windows are fixed-size with a smaller advance (1m window hopping every 10s) — they overlap, so a
record belongs to `size / advance` windows, which multiplies your state and output volume. **Session**
windows are data-driven: they group activity separated by less than an inactivity gap, so the window
size is determined by the data, which is exactly right for "a viewing session". The **grace period**
is how long after a window's end you still accept records whose event time falls inside it; once
grace expires the window is **closed** and anything later is dropped and counted as a late-arrival
metric. StreamPulse uses 10s tumbling (the dashboard tick) plus 1m hopping (the smoothed trend), with
a 30s grace for mobile clients that buffer and upload late.

**Deeper:**

```
event time ──────────────────────────────────────────────────────▶

TUMBLING  size=10s
 │ w[0,10) │ w[10,20) │ w[20,30) │        each record → exactly 1 window

HOPPING   size=60s, advance=10s
 │ w[0,60)                          │
      │ w[10,70)                         │
           │ w[20,80)                         │      each record → 6 windows (60/10)
  ⚠ state and output volume × 6

SESSION   inactivityGap=30s
 │ ev ev  ev │                    │ ev ev │        window boundaries come from the DATA
              └── 30s of silence ─┘                sessions MERGE when a late event bridges a gap

GRACE
 window [0,10)  closes at event-time 10 + grace 30 = 40
   record with ts=7 arriving when stream time is 35 → ACCEPTED, window updated & re-emitted
   record with ts=7 arriving when stream time is 45 → DROPPED, counter incremented
```

**"Stream time"** is the crucial concept: Kafka Streams advances time based on the maximum event
timestamp it has *observed*, not on wall clock. Consequences:

- A partition that stops receiving data **stops advancing stream time**, so its windows never close
  and never emit. This is the "my aggregates stopped appearing on a quiet topic" mystery.
  `max.task.idle.ms` controls how long a task waits for a lagging partition before advancing time
  with what it has — a correctness-vs-liveness trade.
- Replaying historical data processes a day of windows in seconds, correctly, because time is derived
  from the data. That's the payoff of event-time semantics, and it's why a replay after a bug fix
  produces the same aggregates.

**`suppress`** answers "do I emit an update on every record, or one final result per window?". By
default a windowed aggregation emits a new result on **every input record** — great for a live
dashboard that wants continuous updates, terrible for an alerting pipeline that would fire on a
partial window. `suppress(Suppressed.untilWindowCloses(...))` buffers and emits exactly one final
record per window when grace expires. The trade is latency (you now wait `size + grace`) and memory
(the buffer). And `untilWindowCloses` requires an **unbounded** buffer or it throws — if you bound it
with `shutDownWhenFull` you crash on overflow, and `emitEarlyWhenFull` breaks the "exactly one final
result" guarantee. That trilemma is a good thing to name.

StreamPulse uses both deliberately:
- `telemetry.agg.v1` for the **dashboard** → no suppression, continuous updates, SSE shows the window
  filling in real time (freshness is the SLO).
- the **alert engine** input → suppressed, one final value per window, so a rule like "error rate
  > 5% for a 1-minute window" can't fire on the first two errors of the window.

**Code:**

```java
// Phase 3. Three window types, three different questions.

// 1. TUMBLING + grace — the dashboard tick. No suppress: emit on every update.
KTable<Windowed<String>, Long> perTenSeconds = valid
        .groupByKey()
        .windowedBy(TimeWindows.ofSizeAndGrace(Duration.ofSeconds(10),   // size
                                               Duration.ofSeconds(30)))  // grace: mobile uploads late
        .count(Materialized.as("viewers-10s"));

// 2. HOPPING — the smoothed 1-minute trend, updated every 10s.
//    ⚠ size/advance = 6 → every record updates SIX windows. State and output are 6×.
//    That multiplication is the whole reason to think before choosing hopping.
KTable<Windowed<String>, Long> smoothed = valid
        .groupByKey()
        .windowedBy(TimeWindows.ofSizeAndGrace(Duration.ofMinutes(1), Duration.ofSeconds(30))
                               .advanceBy(Duration.ofSeconds(10)))
        .count(Materialized.as("viewers-1m-hop"));

// 3. SESSION — "how long was this viewing session, and how much of it was rebuffering?"
//    Window boundaries come from the data. A late event arriving in the gap MERGES two
//    sessions, which emits a tombstone for the old windows and a new merged record —
//    downstream consumers must tolerate that. This is the subtlest of the three.
KTable<Windowed<String>, SessionStats> sessions = valid
        .groupByKey()
        .windowedBy(SessionWindows.ofInactivityGapAndGrace(Duration.ofSeconds(30),
                                                           Duration.ofSeconds(30)))
        .aggregate(SessionStats::empty, SessionStats::add, SessionStats::merge,
                   Materialized.as("session-stats"));

// 4. SUPPRESSED — the alert-engine feed: exactly one final value per window.
perTenSeconds
        .suppress(Suppressed.untilWindowCloses(Suppressed.BufferConfig.unbounded()))
        .toStream()
        .to("telemetry.agg.final.v1");
```

```java
/**
 * Phase 3 test — TopologyTestDriver. No broker, no Docker, ~10ms, and it can do the one
 * thing an integration test cannot do easily: control event time exactly.
 */
@Test
void lateEventWithinGraceUpdatesTheWindow_beyondGraceIsDropped() {
    var input  = driver.createInputTopic("telemetry.raw.v1", stringSer, avroSer);
    var output = driver.createOutputTopic("telemetry.agg.v1", stringDe, aggDe);

    input.pipeInput("s1", event(t(0)),  t(0));      // window [0,10)
    input.pipeInput("s2", event(t(5)),  t(5));      // window [0,10) → count 2
    input.pipeInput("s3", event(t(35)), t(35));     // advances STREAM TIME to 35

    input.pipeInput("s4", event(t(7)),  t(35));     // late, ts=7, stream time 35 < 10+30 grace
    assertThat(lastCountFor(output, window(0, 10))).isEqualTo(3);   // ✅ accepted

    input.pipeInput("s5", event(t(50)), t(50));     // stream time → 50, window [0,10) now CLOSED
    input.pipeInput("s6", event(t(8)),  t(50));     // ts=8, too late
    assertThat(lastCountFor(output, window(0, 10))).isEqualTo(3);   // ✅ still 3 — dropped
    assertThat(metric("late-records-dropped")).isEqualTo(1);
}
```

**Follow-ups they'll ask:**
- *"How do you pick the grace period?"* — measure the actual arrival-delay distribution of your
  events (event time vs ingest time) and set grace at a high percentile of it. It's a
  correctness-vs-latency dial: longer grace = more correct aggregates, later final results, more
  state. Guessing is how you get an SLO you can't meet.
- *"Where do dropped late events go?"* — by default they're silently dropped and counted in a Streams
  metric. In StreamPulse I branch them to a `telemetry.late.v1` topic before the aggregation where it
  matters, so a late-arrival spike (a whole region's clients buffering) is visible and re-processable
  rather than invisible data loss.
- *"Window retention vs grace?"* — the window store keeps windows for a retention period (must be ≥
  size + grace) so late updates can find them. Setting store retention shorter than grace is a
  configuration bug.
- *"How does this interact with EOS?"* — the suppression buffer and the state store are part of the
  transaction, so a crash doesn't emit a partial window twice. That's the case where EOS earns its
  cost.

**Pitfalls / red flags:** not knowing the difference between event time and processing time, or
describing hopping windows without mentioning the state multiplication.

**Where in StreamPulse:** Phase 3 (10s tumbling + 1m hopping + grace + suppress), `TopologyTestDriver`
tests for out-of-order, late and duplicate events.

**Verify:** `SessionWindows.ofInactivityGapAndGrace` and `TimeWindows.ofSizeAndGrace` API names for
your version (the older `.of(...)`/`.grace(...)` forms are deprecated), and the default
`max.task.idle.ms`.

---

## Q16. Walk me through how Kafka is used end-to-end in StreamPulse, and what you'd change at 10×.

*(The applied question — this is where knowledge becomes evidence.)*

**30-second answer:** Four topics, four different sets of trade-offs, each chosen for a reason I can
defend. `telemetry.raw.v1`: 12 partitions, keyed by **sessionId** for per-session ordering without a
hot partition, `delete` retention of 3 days as the replay window, Avro with BACKWARD compatibility.
The producer is idempotent with `acks=all`, and the ingest path applies **backpressure** — a bounded
in-flight semaphore that returns 429 + Retry-After rather than buffering unboundedly.
`telemetry.agg.v1`: written by Kafka Streams with `exactly_once_v2`, because that stage is Kafka→Kafka
and EOS actually applies there. `config.changes.v1`: **compacted**, fed by Debezium from the
control-plane outbox, so any service can rebuild its whole config cache from the topic at startup.
`alerts.v1`: consumed by a non-blocking `@RetryableTopic` chain into a DLT, because partner delivery
can fail for minutes and must not block the partition. The sink-writer is deliberately **not**
transactional: it's at-least-once with conditional DynamoDB writes and deterministic S3 keys —
effectively-once, which is both cheaper and more honest than pretending EOS covers an external store.
BS-03 then tunes the whole thing for cost: `linger.ms`+`zstd` to trade a bounded ~20ms of latency for
a measured bytes reduction, validated against a 5s p99 freshness budget.

**The topic table I'd draw on the whiteboard:**

| Topic | Key | Partitions | Policy | Guarantee | Why |
|---|---|---|---|---|---|
| `telemetry.raw.v1` | `sessionId` | 12 | delete, 3d | at-least-once, idempotent producer | ordering per session; 3d = replay window; no hot key |
| `telemetry.agg.v1` | `tenant\|title\|region` | 12 | delete, 1d | **EOS v2** | Kafka→Kafka, so EOS is real here |
| `config.changes.v1` | `rule:<id>` / `tenant:<id>` | 6 | **compact** | at-least-once + tombstones | bounded bootstrap of config caches |
| `alerts.v1` | `alertId` | 6 | delete, 7d | at-least-once + dedup on `alertId` | partner retry story; 7d for replay of missed pages |
| `*-retry-N`, `*-dlt` | inherited | inherited | delete, 14d | — | non-blocking retry + parking |

**What I'd change at 10× (500k ev/s), in priority order:**

1. **Partitions first, before anything clever.** 12 partitions caps the consumer group at 12
   members. At 10× I'd need a new topic at a higher partition count (you can't shrink, and an
   *increase* breaks key→partition stability for in-flight sessions, so I'd cut over via a v2 topic
   with dual consumption rather than resizing in place).
2. **Kill the repartition in the Streams topology** if it becomes the bottleneck — the `selectKey`
   in Q14 doubles the data volume through the broker. The alternative is to produce with a
   composite key from the start and accept two topics (one keyed by session, one by
   tenant|title|region) — trading duplicated ingest bytes for removed repartition bytes. Which wins
   is a measurement, not an opinion.
3. **Re-measure the tuning.** BS-03's optimum at 50k ev/s is not the optimum at 500k: at higher
   throughput, batches fill on their own, so `linger.ms` matters less and CPU for zstd matters more.
   I'd expect to move from zstd to lz4 at some throughput, and I'd want the number, not the guess.
4. **Standby replicas and a StatefulSet** for the Streams app become mandatory, because restoring a
   10× state store from the changelog is no longer a "few minutes" event.
5. **Separate clusters or at least separate topics by tenant tier.** At 10× the noisy-neighbour risk
   from `sportsnow` during a final is real; isolating the enterprise tenant protects the BS-10 SLA.
6. **Tiered storage** for the replay window, so 3 days of retention doesn't mean 3 days of local SSD
   on every broker. **Verify** tiered-storage availability and maturity in your Kafka/MSK version —
   this moved from KIP to GA relatively recently.

**Follow-ups they'll ask:**
- *"Why not one topic with an event-type field?"* — that *is* what `telemetry.raw.v1` does for the
  seven telemetry types, precisely because they share a session and must stay ordered. I split
  `agg`/`alerts`/`config` out because they have genuinely different keys, retention, compaction
  policies and consumers. The rule: split on *policy*, not on *type*.
- *"What's the failure you're most worried about?"* — a poison message pattern that affects a whole
  tenant (a client SDK bug emitting a bad field), because it would fill the DLT faster than anyone
  reads it. Mitigation: classify at the deserializer, alert on DLT *rate* with a per-tenant tag, and
  have the replay tooling ready before you need it.
- *"How do you prove any of this?"* — `docs/kafka-tuning.md` with the measured matrix,
  `SlowConsumerRebalanceIT`, `TopologyTestDriver` tests for late/out-of-order/duplicate events, the
  poison-message test, and the Grafana lag/freshness dashboards.

> **Your story:** if you've operated Kafka, Kinesis, or a comparable log at Couchbase, JustAnswer or
> NetApp, this is where it goes — in the same shape: what the topic keys were, what broke, what you
> measured, what you changed. Keep "I built/owned" for what you personally did. StreamPulse is a
> portfolio project, so present it as one: *"I built this to prove the pattern end to end."* Do not
> present BS-03 as an incident you lived through — it is a design scenario.

**Where in StreamPulse:** BS-03 / REQ-KT-01…06 end to end; Phases 1, 2, 3, 4, 9, 11.

---

## Key terms

| Term | One-line meaning |
|---|---|
| **Partition** | The unit of storage, ordering and parallelism. One partition → one consumer per group. |
| **Offset** | A record's position in a partition; the consumer's cursor, stored in `__consumer_offsets`. |
| **ISR** | In-sync replicas — the replicas caught up with the leader; what `acks=all` actually waits for. |
| **HWM / LSO** | High water mark = last offset replicated to all ISR. Last stable offset = the read limit for `read_committed`. |
| **Consumer group** | A set of consumers sharing a subscription; partitions are assigned exclusively within it. |
| **Rebalance** | Recomputing the partition→member assignment. Eager = stop-the-world; cooperative-sticky = incremental. |
| **Static membership** | A stable `group.instance.id` so a pod restart doesn't trigger a rebalance. |
| **Idempotent producer** | PID + per-partition sequence numbers so broker-side dedup kills retry duplicates. |
| **EOS / `exactly_once_v2`** | Atomic consume-transform-produce **inside Kafka**. Covers no external side effect. |
| **Effectively-once** | At-least-once delivery + an idempotent write. The pattern you'll actually use. |
| **`read_committed`** | Consumer isolation level that hides aborted/in-flight transactional records. |
| **Consumer lag** | `HWM − committedOffset`. The causal health metric; watch it per partition and in *seconds*. |
| **Poison message** | A record that fails every attempt; must be classified and DLT'd, never retried forever. |
| **Blocking vs non-blocking retry** | In-place retry (keeps ordering, stalls partition) vs retry topics (keeps flowing, loses ordering). |
| **DLT / DLQ** | Dead-letter topic for parked records. Alert on its *rate*, not just its depth. |
| **Log compaction** | Keep at least the latest value per key; `null` value = tombstone. Turns a topic into a changelog. |
| **Repartition topic** | An internal topic Kafka Streams inserts when you change the key. A real, measurable cost. |
| **Stream time** | Kafka Streams' clock, derived from observed event timestamps — not wall clock. |
| **Grace period** | How long after a window ends you still accept records for it. Then the window closes. |
| **`suppress`** | Emit one final result per window instead of an update per record. Costs latency + memory. |
| **Standby replica** | A warm passive copy of a Streams state store, so failover doesn't mean a full changelog restore. |

---

## Self-check (answer out loud, no notes)

1. Why `sessionId` and not `tenantId` as the partition key — and what exactly breaks if you add
   partitions later?
2. Draw the EOS boundary. Name two things inside it and two things outside it.
3. A consumer group's lag is climbing and the rebalance rate is non-zero. What is the inequality you
   check, and what are your three levers?
4. Blocking retry vs `@RetryableTopic`: which one preserves ordering, and which one would you use for
   an S3 outage that lasts three minutes?
5. `linger.ms=20` and `compression.type=zstd`: explain, in one sentence each, what you bought and what
   you paid.
