# 07 — Dual Writes, the Transactional Outbox, Debezium CDC, and SSE

**Why this matters for an Event Hub role:** this file is the spine of the job. An event hub's entire
value proposition is *"a change happened over there, and it reliably shows up over here, fast."* Every
naive implementation of that sentence is broken in the same way — the **dual-write problem** — and
every correct implementation is some variant of "write once, atomically, to a log, and derive
everything else from it." Then the last mile: getting the change into a browser without polling, over
a connection that proxies, load balancers and slow clients are all trying to kill.

**Scenario anchor — Phase 6 + BS-08:** the control plane owns tenants, titles and alert rules in
Postgres. When an operator edits a rule, every service's cache must refresh and the dashboard must
show it — with no missed and no duplicate events, including while the alert engine is being extracted
into its own service (REQ-SF-02: *rules reach the engine via `config.changes.v1`, never by a DB
join* — the data-ownership cut happens **before** the process cut). The same pipeline carries alerts
to the browser over SSE.

> This file is the one to internalise at **mechanism level**, because it maps directly to a
> CDC→SSE pipeline you can speak to from experience. The `> **Your story:**` blocks mark where that
> goes — the material below gives you the vocabulary; the specifics are yours.

---

## Q1. What is the dual-write problem? Be precise.

**30-second answer:** A dual write is any operation that must change two systems that do not share a
transaction — classically "commit to Postgres, then publish to Kafka". There is no atomicity across
them, so there is a window between the two writes. If the process dies, the pod is evicted, or the
broker is briefly unavailable in that window, you get a database row that no consumer ever hears
about — a **silent, permanent divergence** that no retry can discover, because nothing recorded that
the publish was owed. Reversing the order doesn't help: publish-then-commit gives you an event for a
change that never happened. The window is small, which is worse, not better — it means the bug is
rare enough to reach production and rare enough to be blamed on something else.

**Deeper:**

```
  ❌ commit-then-publish                      ❌ publish-then-commit
  ─────────────────────────                   ────────────────────────
  BEGIN                                       publish(RuleChanged)   ← consumers act on it
  UPDATE rule …                               BEGIN
  COMMIT            ✅ durable                 UPDATE rule …
  ──── crash window ────                       ROLLBACK               ✅ nothing changed
  publish(RuleChanged)   ✖ never happens       ──────────────────────
                                              → phantom event: engine now
  → DB says the rule changed;                    evaluates a rule that
    the engine never hears.                      does not exist.
    Divergence is PERMANENT.
```

**Why a try/catch does not fix it.** The obvious patch:

```java
// ❌ Still broken. This handles the FAILURE case; it does not handle the CRASH case.
@Transactional
public void update(RuleUpdate cmd) {
    ruleRepo.save(cmd.toEntity());
    try {
        kafka.send("config.changes.v1", event(cmd)).get();
    } catch (Exception e) {
        throw new PublishFailedException(e);     // rolls back the DB… sometimes
    }
}
```

Three separate things are wrong here:

1. **The crash case is untouched.** SIGKILL, an OOM-kill, a node failure or a network partition
   between `COMMIT` and `send()` leaves no code running to catch anything. A `catch` block only runs
   if your process is alive to run it.
2. **The failure you caught may not be a failure.** `send().get()` timing out does not mean the
   broker rejected the record — it may have been written and the ack lost. You now roll back a
   database change for an event that *was* published. You have swapped one divergence for the other.
3. **Rolling back on publish failure inverts the dependency.** Your ability to change a rule now
   requires Kafka to be up. You have coupled the availability of the write path to the availability of
   the message bus, which is usually the opposite of what you wanted.

The generalisation worth saying: *you cannot build an atomic operation out of two non-atomic
operations by adding error handling.* You need **one** atomic write, and a separate, retryable,
idempotent process that propagates it. That is the outbox.

**Follow-ups they'll ask:**
- *"What about `@TransactionalEventListener(AFTER_COMMIT)`?"* — strictly better than the code above,
  because it publishes only if the transaction really committed, so no phantom events. But the crash
  window is still there: commit succeeds, the process dies before the listener runs, the event is
  gone. It is best-effort, and it is fine for things like cache-warming hints where loss is tolerable.
  It is not fine for a rule change that drives alerting.
- *"Isn't this what XA / two-phase commit is for?"* — in theory. In practice Kafka has no XA resource
  manager in the usual sense, 2PC needs a recovery-capable transaction manager and blocks in-doubt
  transactions if the coordinator dies, and it couples the availability of both systems. Nobody
  operating a high-throughput pipeline runs it. See file 09 for the longer version of that argument.
- *"How would you *detect* that this is happening today?"* — a reconciliation job: compare the
  database's `updated_at` watermark against the last event's payload version per aggregate, and count
  the gaps. If you cannot detect it, you cannot claim it is not happening.

**Pitfalls / red flags:** "we retry the publish" (retries need a durable record of what to retry), or
not being able to name the crash window as distinct from the failure case.

**Where in StreamPulse:** the reason REQ-DL-06 exists — the outbox row is written **in the same
transaction** as the entity, and `OutboxAtomicityIT` proves that a rollback leaves neither.

---

## Q2. Explain the transactional outbox pattern. Show me the schema.

**30-second answer:** Instead of writing to two systems, you write to one: the business change and an
`outbox` row go into the **same** local database transaction, so they commit or roll back together —
atomicity, for free, from a transaction you were already having. A separate relay process then reads
the outbox and publishes to Kafka, retrying until it succeeds. The relay is at-least-once by design,
so every event carries a stable id and consumers dedupe on it. The trade you have made is explicit and
worth stating: **you traded atomicity-you-cannot-have for eventual consistency-you-can-operate**, and
the lag between commit and publish becomes a metric you watch instead of a bug you can't see.

**Deeper:**

```
 ┌──────────────── ONE local transaction ────────────────┐
 │  UPDATE alert_rule SET expression = '…' WHERE id = ?  │
 │  INSERT INTO outbox (id, aggregate_type, aggregate_id,│
 │                      type, payload, created_at) …     │
 │  COMMIT                                               │  ← atomic. Both, or neither.
 └───────────────────────┬───────────────────────────────┘
                         │
              ┌──────────┴───────────┐
              ▼                      ▼
     polling relay             log-based CDC (Debezium)
     (SELECT … SKIP LOCKED)    (reads the WAL — Q4/Q5)
              │                      │
              └──────────┬───────────┘
                         ▼
                 config.changes.v1  ──▶ every service refreshes its cache
                                    ──▶ alert-engine (BS-08, REQ-SF-02)
```

**The schema, and why each column is there:**

```sql
-- StreamPulse control-plane, Flyway V12__outbox.sql
CREATE TABLE outbox (
    id              uuid        PRIMARY KEY,                 -- the EVENT id = the dedup key
    aggregate_type  text        NOT NULL,                    -- 'AlertRule' → routes to a topic
    aggregate_id    text        NOT NULL,                    -- ruleId → becomes the Kafka KEY
                                                             --   (same key ⇒ same partition ⇒ order)
    type            text        NOT NULL,                    -- 'RuleUpdated' → event type header
    payload         jsonb       NOT NULL,                    -- the event body, NOT the row image
    trace_id        text,                                    -- W3C traceparent, so the trace spans CDC
    created_at      timestamptz NOT NULL DEFAULT now(),
    -- polling-relay only; a Debezium-based relay does not need these:
    status          text        NOT NULL DEFAULT 'PENDING',
    claimed_by      text,
    claimed_at      timestamptz,
    attempts        int         NOT NULL DEFAULT 0
);

-- Partial index: the relay only ever scans PENDING rows, so keep the index small.
CREATE INDEX idx_outbox_pending ON outbox (created_at) WHERE status = 'PENDING';
```

Design points I'd raise unprompted:

- **`payload` is a domain event, not a row image.** This is the single most important choice. A row
  image couples every consumer to your table schema; the moment you rename a column, downstream
  breaks (see Q7). The outbox payload is a versioned contract you publish deliberately. It is also why
  the outbox table's own schema can stay frozen while the business schema evolves (file 06, Q12).
- **`aggregate_id` becomes the Kafka message key**, which gives you **per-aggregate ordering** — all
  events for rule `r-42` land on one partition and are consumed in order. You do *not* get global
  ordering, and you should say so before they ask.
- **`id` is the idempotency key.** Generate it in the application (UUIDv7 if you want it
  time-sortable) so it is stable across retries and across the BS-08 cutover — REQ-SF-05 uses the
  same idea for alerts: `alertId = hash(ruleId, windowStart)`, so an overlap between the old and new
  engine produces the same id and consumers collapse it.

**Ordering, precisely.** With a polling relay there is a subtle hazard: transaction **commit order**
is not insert order. T1 inserts outbox row A at 10:00:00.100 and commits at 10:00:00.500; T2 inserts
row B at 10:00:00.200 and commits at 10:00:00.300. A relay polling on `created_at` can read B, commit
its offset, and only later see A — it published them out of order, and worse, if it tracks a
high-water mark by `created_at` it may **skip A entirely**. Fixes: use a monotonic sequence assigned
at commit rather than at insert; or claim rows with `FOR UPDATE SKIP LOCKED` and only mark them
`SENT`, never using a timestamp watermark; or use log-based CDC, which reads the WAL in **commit
order** and does not have this problem at all. That last point is a strong argument for Debezium.

**Cleanup.** The outbox grows forever if you let it. Options, in order of preference:
1. **Delete after publish** in the relay (polling relay only). Simple, keeps the table tiny, but you
   lose the audit trail and the partial index churns.
2. **Soft-delete + partitioned table**, dropping old partitions daily. Best of both: you keep a
   window of history and cleanup is a metadata operation, not a 10-million-row `DELETE` that bloats
   the heap and blocks autovacuum.
3. **`DELETE FROM outbox WHERE created_at < now() - interval '7 days'` in batches**, throttled.
   Acceptable, but watch for bloat.

With Debezium there is a trick worth knowing: you can `INSERT` and immediately `DELETE` the outbox row
**in the same transaction**. The insert is still in the WAL, so Debezium sees the event; the table
stays empty, so there is nothing to clean up. The cost is that the outbox is no longer readable as an
audit log, and debugging gets harder — I'd only do it at volumes where cleanup is genuinely a problem.

**Code:**

```java
/**
 * REQ-DL-06 — the whole point: one @Transactional method, two writes, one commit.
 * Nothing here talks to Kafka. The service cannot fail because a broker is down,
 * and it cannot succeed halfway.
 */
@Service
class RuleService {

    @Transactional
    public AlertRule update(UUID ruleId, RuleUpdate cmd) {
        AlertRule rule = ruleRepo.findById(ruleId).orElseThrow(RuleNotFound::new);
        rule.apply(cmd);                                   // dirty checking writes the UPDATE

        outboxRepo.save(OutboxEvent.of(
                UUID.randomUUID(),                         // event id == dedup key downstream
                "AlertRule",                               // aggregate_type → topic routing
                ruleId.toString(),                         // aggregate_id  → Kafka key → ordering
                "RuleUpdated",
                payload(rule),                             // a DOMAIN EVENT, not the row image
                MDC.get("traceId")));                      // keeps the trace continuous across CDC

        return rule;
    }   // single COMMIT — both rows, or neither
}
```

```java
@Test
@DisplayName("REQ-DL-06: rollback leaves neither the rule nor the outbox row")
void outboxIsAtomicWithTheEntity() {
    assertThatThrownBy(() -> ruleService.update(id, invalidUpdate()))
        .isInstanceOf(ValidationException.class);

    assertThat(ruleRepo.findById(id).orElseThrow().expression()).isEqualTo(ORIGINAL);
    assertThat(outboxRepo.count()).isZero();
}
```

**Follow-ups they'll ask:**
- *"Doesn't the outbox add write amplification?"* — yes: one extra insert per business write, plus WAL
  for it. On a control plane doing tens of writes per second that is nothing. On a table doing 50k
  writes/s I'd measure it, and consider making the log itself the source of truth instead.
- *"What guarantees do you end up with?"* — at-least-once delivery, per-aggregate ordering, and no
  loss. Not exactly-once (Q8), and not global ordering.
- *"What if the relay publishes and then crashes before marking the row sent?"* — it republishes.
  That is the at-least-once case, and it is why the event id is stable and consumers dedupe.

**Pitfalls / red flags:** describing the outbox as "a queue table" without mentioning that the
atomicity comes from it sharing the business transaction — that *is* the pattern. Also: putting the
row image in `payload` and calling it a contract.

**Where in StreamPulse:** BS-05 / REQ-DL-06; BS-08 / REQ-SF-02, REQ-SF-05; Phase 6.

---

## Q3. Polling relay or log-based CDC? How do you choose?

**30-second answer:** A polling relay is a scheduled job doing `SELECT … FOR UPDATE SKIP LOCKED`,
publishing, then marking rows sent. It needs no extra infrastructure, it is trivially debuggable, and
it is the right answer for a small service — I'd ship it first. Log-based CDC (Debezium reading the
Postgres WAL) puts zero query load on the table, sees changes in **commit order** with no polling
latency, and captures changes made by anything — including a manual `UPDATE` by a DBA. The cost is a
Kafka Connect cluster to run and a replication slot that will fill your disk if you stop consuming it.
StreamPulse uses Debezium because CDC *is* the thing being demonstrated, and because commit-order
delivery removes a whole class of ordering bugs.

**Deeper:**

| | Polling relay | Log-based CDC (Debezium) |
|---|---|---|
| Infrastructure | none (a `@Scheduled` method) | Kafka Connect + connector + a replication slot |
| Latency | poll interval (100 ms–5 s) | milliseconds after commit |
| Load on the source DB | a query per poll, per instance, forever | ~none on the table; WAL reading only |
| Ordering | insert order ≠ commit order — the hazard in Q2 | **commit order**, guaranteed by the log |
| Captures out-of-band changes | no (only what your app inserts) | yes — any writer, including psql |
| Failure modes | missed rows from a bad watermark; duplicate publishes | slot lag → unbounded WAL growth (Q4) |
| Scale-out | needs `SKIP LOCKED` or partitioned claims | one connector task per table set; slot is single-reader |
| Debuggability | excellent — it's a table and a query | you need Connect logs, slot lag, and the topic |
| Delete/cleanup | you own it | you own it (or insert-then-delete, Q2) |

**When I'd deliberately choose polling:** a single modest service, a team without a Connect cluster, or
a compliance environment where adding a replication slot to the production primary needs a three-week
review. "Ship the polling relay, keep the outbox schema identical, swap in Debezium later" is a
genuinely good migration story, because the *outbox contract* is the same either way — only the relay
changes.

**Code — the polling relay, done correctly:**

```java
/**
 * The claim query is the interesting part (file 06, Q9):
 *   - FOR UPDATE SKIP LOCKED lets N replicas each take a DISJOINT batch without blocking;
 *   - we claim by UPDATE (not a timestamp watermark), which is what makes the
 *     "commit order ≠ insert order" hazard harmless — a late-committing row is simply
 *     still PENDING and gets picked up on the next poll.
 *   - ORDER BY created_at gives approximate FIFO. Strict per-aggregate order still comes
 *     from the Kafka KEY (aggregate_id), not from the poll order.
 */
@Scheduled(fixedDelayString = "${streampulse.outbox.poll-interval:200ms}")
@Transactional
void relay() {
    List<OutboxEvent> batch = jdbcClient.sql("""
            UPDATE outbox SET status='SENDING', claimed_by=:me, claimed_at=now()
            WHERE id IN (SELECT id FROM outbox WHERE status='PENDING'
                         ORDER BY created_at LIMIT :limit
                         FOR UPDATE SKIP LOCKED)
            RETURNING id, aggregate_id, type, payload, trace_id
            """)
        .param("me", instanceId).param("limit", 200)
        .query(OutboxEvent.class).list();

    for (OutboxEvent e : batch) {
        // If we crash after send() but before the status update, we republish on restart.
        // At-least-once by design; the consumer dedupes on e.id().
        kafka.send(record(e)).join();
        markSent(e.id());
    }
}
```

**Follow-ups they'll ask:**
- *"Why `SKIP LOCKED` rather than a `claimed_by` + `WHERE claimed_by IS NULL` update?"* — you can do
  that too, but `SKIP LOCKED` avoids the lock convoy where every instance contends for the same head
  rows, and it is one statement instead of read-then-write.
- *"How do you monitor a relay?"* — outbox depth (`count(*) WHERE status='PENDING'`), oldest pending
  age (this is the real SLI: *"how stale can a config change be?"*), publish error rate, and rows
  stuck in `SENDING` (a crashed claimer — needs a reaper that resets `SENDING` rows older than N).
- *"Does the relay need to be leader-elected?"* — no, that is exactly what `SKIP LOCKED` buys you.
  Leader election would be the alternative if you needed strict global ordering, at the cost of
  throughput and a failover gap.

**Pitfalls / red flags:** a relay that tracks progress with `WHERE created_at > :lastSeen`. That is
the skip-a-row bug from Q2, and it is the most common outbox implementation error.

**Where in StreamPulse:** Phase 6; the ADR records polling-vs-Debezium and why Debezium won here.

---

## Q4. How does Postgres logical decoding actually work? What's the hazard?

**30-second answer:** Postgres already writes every change to the **write-ahead log** for crash
recovery and physical replication. Logical decoding is a second reader of that log: you set
`wal_level = logical`, create a **replication slot**, and an output plugin (`pgoutput`, built in since
PG 10) turns the physical WAL records into a logical stream of row changes grouped by transaction and
emitted **in commit order**. Debezium is a client of that stream. The hazard is the slot: it pins the
WAL. Postgres will not recycle WAL segments that a slot has not confirmed, so if the consumer stops —
Connect crashes, the connector is paused, someone deletes the topic — the WAL grows without bound
until the disk fills and the **primary goes down**. An inactive replication slot is the classic way
CDC takes out a database.

**Deeper:**

```
   client COMMIT
        │
        ▼
   ┌─────────┐   physical WAL records (page-level)
   │  WAL    │────────────┬──────────────────────────────┐
   └─────────┘            │                              │
                  crash recovery /            logical decoding
                  physical replica            ┌──────────────────┐
                                              │ output plugin    │  pgoutput / wal2json
                                              │ reassembles rows │  (decoderbufs, historically)
                                              │ per transaction, │
                                              │ in COMMIT order  │
                                              └────────┬─────────┘
                                                       ▼
                                           replication slot (named, durable)
                                             restart_lsn ─────► how far back WAL must be kept
                                             confirmed_flush_lsn ► how far the consumer has acked
                                                       ▼
                                                  Debezium → Kafka
```

Three properties that fall out of this and that interviewers like:

1. **Commit order, not insert order.** The decoder buffers a transaction's changes and emits them
   when it sees the `COMMIT` record. So the ordering hazard from Q2 simply does not exist. It also
   means a long transaction's changes appear *all at once*, at commit — a 10-minute batch job produces
   a burst, not a trickle.
2. **Uncommitted and rolled-back transactions never appear.** You cannot see dirty data.
3. **A slot is single-reader and stateful**, living on the primary. Before PG 16 it did **not**
   survive a failover to a physical replica (logical slots were not replicated), which was a real
   operational gotcha; PG 16 added failover-capable logical slots. Verify against your version — if
   you're on an older one, the failover plan is "recreate the slot and re-snapshot", and you should
   know that before the failover, not during it.

**The disk-growth hazard, and the three defences:**

```sql
-- The query to have in your runbook.
SELECT slot_name, active, restart_lsn,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
FROM   pg_replication_slots;
```

| Defence | What it does | Trade-off |
|---|---|---|
| **Alert on `retained_wal`** | pages you before the disk does | requires someone to act |
| **`max_slot_wal_keep_size`** (PG 13+) | caps retained WAL; the slot is **invalidated** past the cap | the database survives; CDC must re-snapshot |
| **Connector heartbeats** | Debezium periodically writes/acks so the slot advances even when your tables are idle | needs a heartbeat table or query on a low-traffic database |

That third one is genuinely non-obvious and worth volunteering: if the monitored tables are quiet but
*other* databases on the same instance are busy, the slot's `confirmed_flush_lsn` never advances,
because Debezium has nothing to ack — yet WAL keeps being generated. The whole cluster's WAL is then
pinned by an idle connector. Debezium's heartbeat interval (and, on Postgres, an action query that
writes a dummy row so there is something to ack) exists exactly for this.

Two more Postgres-specific requirements to name:
- **`REPLICA IDENTITY`** decides what the WAL contains for `UPDATE`/`DELETE`. The default is
  `DEFAULT` = primary key only, so "before" images of non-key columns are absent. `REPLICA IDENTITY
  FULL` logs the whole old row — useful for consumers that need the before-state, expensive in WAL
  volume. For the outbox pattern it doesn't matter, because you only ever `INSERT`.
- **TOAST columns**: large values stored out-of-line are not re-logged on an `UPDATE` that didn't
  change them, so they arrive as an "unchanged" placeholder. Consumers must handle that.

**Code — the minimum Postgres setup:**

```yaml
# docker-compose, StreamPulse `cdc` profile
postgres:
  image: postgres:16
  command:
    - "postgres"
    - "-c"; "wal_level=logical"          # the switch that enables logical decoding
    - "-c"; "max_replication_slots=4"
    - "-c"; "max_wal_senders=4"
    - "-c"; "max_slot_wal_keep_size=2GB" # PG 13+: sacrifice the slot, not the database
```

**Follow-ups they'll ask:**
- *"Does logical decoding slow down the primary?"* — the decoding work itself happens in a walsender
  process, so it costs CPU and memory on the database host but does not touch your tables. The real
  cost is the WAL retention and, with `REPLICA IDENTITY FULL`, extra WAL volume.
- *"MySQL equivalent?"* — the binlog in `ROW` format, read via a replication client. Same shape,
  different failure modes: binlog retention is time/size-based (`binlog_expire_logs_seconds`), so a
  lagging consumer gets a "binlog not found" error rather than filling the disk. Arguably a friendlier
  failure.
- *"What does the connector store to resume?"* — the LSN, in a Kafka Connect **offset topic**, plus the
  slot's `confirmed_flush_lsn` on the database side. Both must be consistent; deleting the offset
  topic while keeping the slot is how you accidentally re-snapshot.

**Pitfalls / red flags:** talking about CDC purely as "Debezium config" with no idea what a
replication slot is or why it is dangerous. The slot is *the* operational risk of this pattern.

**Where in StreamPulse:** Phase 6, compose profile `cdc`; the runbook entry "slot lag" in Phase 14.

**Verify:** the default `plugin.name` for your Debezium Postgres connector version (it has moved
between `decoderbufs` and `pgoutput`), PG 16's failover-slot behaviour, and
`max_slot_wal_keep_size` semantics, all in current docs.

---

## Q5. Walk me through Debezium: snapshot, streaming, and the Outbox Event Router.

**30-second answer:** A Debezium source connector runs inside Kafka Connect. On first start it takes a
**consistent snapshot** — it reads the current contents of the configured tables and emits them as
synthetic `read` events, recording the LSN at which the snapshot was consistent — then switches to
**streaming**, reading the WAL from that LSN onwards. So a new consumer gets the full current state
plus every change since, with no gap and no duplicate window. For the outbox pattern you add the
**Outbox Event Router**, a single-message transform that unwraps the CDC envelope: it takes the
`payload` column as the message value, the `aggregate_id` as the message key, and routes to a topic
based on `aggregate_type` — so consumers see clean domain events, not `{before, after, source, op}`.

**Deeper:**

**The raw CDC envelope** (what you get *without* the router):

```json
{
  "before": null,
  "after":  { "id": "…", "aggregate_type": "AlertRule", "aggregate_id": "r-42",
              "type": "RuleUpdated", "payload": "{\"ruleId\":\"r-42\",…}" },
  "source": { "connector": "postgresql", "lsn": 24857392, "txId": 9912,
              "ts_ms": 1790000000000, "table": "outbox", "snapshot": "false" },
  "op":     "c",                                   // c=create r=read(snapshot) u=update d=delete
  "ts_ms":  1790000000004
}
```

Note `op: "r"` — that is how you tell a snapshot row from a real change, which matters if your
consumer has side effects (you usually want snapshot rows to load state, not to fire alerts).

**Snapshot modes** (names vary by version — verify):

| Mode | Behaviour | When |
|---|---|---|
| `initial` | snapshot once, then stream | the default; a new pipeline |
| `never` | stream from the current LSN only | you already have the state elsewhere and only want the delta |
| `initial_only` | snapshot, then stop | one-off bulk load |
| `when_needed` | snapshot again if the stored offset is no longer valid | self-healing after a WAL gap |
| incremental snapshot (signal table) | chunked snapshot **while streaming continues** | re-snapshotting a big table without stopping the pipeline |

The incremental snapshot is the one worth knowing about: you write a row into a **signal table** and
the connector snapshots in chunks, interleaved with live events, deduplicating by primary key. Before
this existed, re-snapshotting a large table meant pausing the pipeline for hours.

**Topics.** Debezium names topics `<topic.prefix>.<schema>.<table>` — one topic per table. For the
outbox pattern that is one topic (`…public.outbox`), which the router then re-routes by
`aggregate_type` into `config.changes.v1` (or per-aggregate topics). Per-table topics matter for a
reason people miss: **ordering is per topic-partition**, so a change to `tenant` and a change to
`alert_rule` in the same database transaction land in different topics and can be consumed in either
order. If two tables must be consumed in a consistent order, route them to the same topic with the
same key — which is another argument for the outbox, where *you* choose the key and the topic rather
than inheriting the table layout.

**Code — the connector, with the router:**

```json
{
  "name": "control-plane-outbox",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres", "database.dbname": "control_plane",
    "topic.prefix": "streampulse",
    "plugin.name": "pgoutput",
    "slot.name": "streampulse_outbox",
    "publication.autocreate.mode": "filtered",
    "table.include.list": "public.outbox",

    "heartbeat.interval.ms": "10000",

    "transforms": "outbox",
    "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
    "transforms.outbox.table.field.event.key":     "aggregate_id",
    "transforms.outbox.table.field.event.payload": "payload",
    "transforms.outbox.route.by.field":            "aggregate_type",
    "transforms.outbox.table.fields.additional.placement": "type:header:eventType,id:header:eventId"
  }
}
```

What that produces on `config.changes.v1`:

```
key     = "r-42"                      ← aggregate_id: per-rule ordering, stable partition
value   = {"ruleId":"r-42", …}        ← the payload column, unwrapped: a clean domain event
headers = eventType: RuleUpdated
          eventId:  018f3c…           ← the dedup key the consumer uses (Q8)
```

**Follow-ups they'll ask:**
- *"What are the delivery guarantees?"* — Debezium into Kafka is **at-least-once** by default: after a
  crash the connector resumes from its last committed offset and may re-emit events it already
  published. Kafka Connect gained exactly-once support for source connectors (KIP-618), but it is an
  opt-in with its own requirements — verify before claiming it. The design should not depend on it:
  the consumer dedupes.
- *"What happens on a connector restart?"* — it resumes from the LSN in the Connect offset topic. If
  that offset is gone but the slot still exists, you may re-stream; if the slot is gone, you
  re-snapshot. Both are survivable *because* consumers are idempotent — which is the theme.
- *"How does this interact with schema changes?"* — see Q7. Short version: with the outbox, it
  doesn't, which is the point.
- *"Where does the trace context go?"* — I put `trace_id` in the outbox row and promote it to a Kafka
  header, so a Tempo trace runs from the HTTP request that changed the rule all the way to the SSE
  push. Without that, CDC is a hole in your tracing.

**Pitfalls / red flags:** treating Debezium as "replication" and shipping raw table rows to consumers
as the public contract. That is a distributed shared database with extra steps.

**Where in StreamPulse:** Phase 6, compose profile `cdc`; `config.changes.v1` refreshes the LRU cache
in query-api (Phase 5) and feeds the extracted alert engine (BS-08 / REQ-SF-02).

**Verify:** every connector option name above, the snapshot-mode names, the Outbox Event Router
config keys, and Connect's exactly-once source support — all have changed across Debezium/Connect
versions. Read the docs for the exact versions you pin before quoting them.

---

## Q6. Compare dual write, application events, polling, and CDC on one slide.

**30-second answer:** Four options, and the honest ranking depends on what loss you can tolerate. Dual
write is broken and should never be a deliberate choice. `@TransactionalEventListener(AFTER_COMMIT)`
removes phantom events but keeps the crash window — fine for hints, not for correctness. A polling
outbox relay is correct, needs no infrastructure, and is where I'd start. Log-based CDC is correct,
lower-latency, commit-ordered, and captures out-of-band changes, at the cost of a Connect cluster and a
replication slot you must monitor.

**Deeper:**

| | Dual write | `AFTER_COMMIT` event | Polling outbox | Log-based CDC |
|---|---|---|---|---|
| Lost events on crash | **yes** | **yes** (small window) | no | no |
| Phantom events | yes (publish-first) | no | no | no |
| Extra infrastructure | none | none | none | Connect + slot |
| Latency after commit | ~0 | ~0 | poll interval | ms |
| Ordering | none | none | approximate; per-key via Kafka key | **commit order** |
| Load on the DB | none | none | a poll query per instance | WAL read only |
| Captures non-app writes | no | no | no | **yes** |
| Operational risk | silent divergence | silent divergence | table growth | slot lag → disk full |
| Delivery semantics | undefined | best-effort | at-least-once | at-least-once |
| Verdict | never | hints and cache-warming only | **good default** | **best when CDC infra exists** |

**"Exactly-once", end to end, honestly.** This is where a lot of candidates over-claim, so here is the
chain, hop by hop, for StreamPulse:

```
1. app → Postgres        EXACTLY ONCE   (one local transaction; that's what ACID is)
2. Postgres → Debezium   AT LEAST ONCE  (connector restarts can re-emit; EOS for source
                                         connectors exists but is opt-in — verify)
3. Debezium → Kafka      AT LEAST ONCE  (idempotent producer removes broker-side dupes
                                         within a session, not across connector restarts)
4. Kafka → consumer      AT LEAST ONCE  (offsets commit after processing, so a crash replays)
5. consumer → its store  EXACTLY ONCE *IF* the write is idempotent:
                           - DynamoDB conditional write on the event id
                           - Postgres INSERT … ON CONFLICT DO NOTHING on the event id
                           - Redis SET NX
                         → "effectively once"
```

So the honest sentence to say in the room is: **"at-least-once delivery plus an idempotent consumer
equals effectively-once processing. There is no exactly-once delivery across a network — the
end-to-end guarantee comes from the consumer's write being idempotent, not from the transport."**
Kafka's `exactly_once_v2` is real, but it covers consume→process→produce **within Kafka** (offsets and
output records in one transaction); it says nothing about your DynamoDB write or your HTTP call.

**Code — the idempotent consumer, which is what actually buys the guarantee:**

```java
@KafkaListener(topics = "config.changes.v1", groupId = "query-api-cache")
void onConfigChange(ConsumerRecord<String, ConfigChange> rec) {

    String eventId = header(rec, "eventId");        // stable across retries AND across the
                                                    // BS-08 cutover (REQ-SF-05)

    // SET key value NX EX ttl → true only the first time. Redelivery is a no-op.
    if (!dedup.firstTime(eventId, Duration.ofHours(24))) {
        meter.counter("cdc.duplicate.skipped").increment();
        return;
    }
    titleCache.invalidate(rec.value().titleId());    // the actual side effect
}
```

Two cautions on that code that I'd volunteer: the dedup store must outlive your worst redelivery
window (a consumer group that was down for six hours will replay six hours of events), and if the
process dies *between* the dedup write and the side effect, you have marked it done without doing it.
For a cache invalidation that is harmless. For a payment it is not — there you need the dedup marker
and the side effect in **one** transaction, which is the outbox pattern again, pointing the other way.

**Follow-ups they'll ask:**
- *"Where would you accept a dual write?"* — a genuinely best-effort side channel: a Slack
  notification, an analytics ping, a cache pre-warm. Say the tolerance out loud when you choose it.
- *"How do you prove effectively-once in a test?"* — replay the same batch twice through the pipeline
  and assert the resulting state is identical, plus a counter showing the duplicates were detected.
  StreamPulse's `RollingRestartChaosIT` does exactly that shape for the ingest path.

**Pitfalls / red flags:** saying "we use exactly-once" without qualifying what it covers. It is the
fastest way to lose credibility in a streaming interview.

**Where in StreamPulse:** Phase 6 (CDC), Phase 4 (idempotent sinks: DynamoDB conditional writes),
Phase 2 (`SET NX` dedup on `eventId`).

---

## Q7. How do you handle schema changes flowing through CDC?

**30-second answer:** With the outbox pattern, mostly you don't have to — and that is the strongest
argument for it. The CDC stream carries the **outbox** table, whose schema is frozen and generic
(`aggregate_id`, `type`, `payload`), so renaming a column in `alert_rule` is invisible to every
consumer. The thing that *can* change is the **event payload contract**, and that is governed
deliberately: Avro (or JSON Schema) in the schema registry with `BACKWARD` compatibility, so you may
add optional fields and may not remove or retype required ones. If you ever do need a breaking change,
you publish a new event type or a new topic version (`config.changes.v2`) and run both until consumers
have moved.

**Deeper:**

| Change | Table-level CDC (row images) | Outbox CDC (domain events) |
|---|---|---|
| Rename a column | **breaks** every consumer | invisible |
| Add a column | consumers see a new field appear | invisible until you choose to publish it |
| Change a column type | breaks; the connector may emit a new schema mid-stream | invisible |
| Split one table into two | breaks completely | invisible — the event shape is unchanged |
| Add a field to the event | n/a | a `BACKWARD`-compatible Avro change: optional with a default |
| Remove a field from the event | n/a | **breaking**: publish `v2`, dual-publish, migrate, retire |

**Compatibility modes, which they will ask you to distinguish:**

| Mode | Meaning | Who upgrades first |
|---|---|---|
| `BACKWARD` | a **new schema** can read data written with the **old** one | consumers |
| `FORWARD` | an **old schema** can read data written with the **new** one | producers |
| `FULL` | both | either |
| `NONE` | anything goes | nobody, and you find out in production |

StreamPulse uses `BACKWARD` (Phase 1), which means: **add optional fields with defaults; never remove
a required field; never change a field's type.** The build enforces it — a test attempts an
incompatible change and asserts the registry rejects it.

**The expand/contract link.** File 06 Q12's zero-downtime column rename works *because* the outbox
decouples it: during the five releases where both `expression` and `expression_v2` exist, the event
payload keeps its single `expression` field, populated from whichever column is authoritative. The
schema migration and the event contract evolve on **independent schedules**. With raw table CDC they
are welded together, and every schema change becomes a cross-team coordination exercise.

**Code:**

```json
// libs/contracts/avro/ConfigChange.avsc — adding a field, BACKWARD-compatibly
{
  "type": "record", "name": "ConfigChange", "namespace": "com.streampulse.contracts",
  "fields": [
    { "name": "eventId",  "type": "string" },
    { "name": "ruleId",   "type": "string" },
    { "name": "changedAt","type": {"type":"long","logicalType":"timestamp-millis"} },
    // NEW in v1.3 — optional with a default, so old consumers reading new data are fine.
    // Removing it later, or making it required, would be BREAKING.
    { "name": "changedBy","type": ["null","string"], "default": null }
  ]
}
```

**Follow-ups they'll ask:**
- *"Debezium emits its own schema per table — where does that fit?"* — Connect has its own schema
  model, and with the Avro converter each table topic gets a registered schema. With the outbox router,
  the `payload` column is typically an opaque string or JSON, so **your** contract is the one in the
  registry for `config.changes.v1`, which is exactly the separation you want.
- *"What about a consumer that must handle both v1 and v2?"* — the event type header carries the
  version; the consumer branches on it and maps both into one internal model. Keep the branch for one
  release cycle, then delete it — and put a deletion date in the code comment, or it lives forever.
- *"How do you know when it's safe to retire v1?"* — consumer-group metrics on the v1 topic: zero lag
  and zero fetches for a full retention period. Don't guess.

**Pitfalls / red flags:** "the consumer just ignores fields it doesn't know" — true for adding, false
for removing and retyping, and useless as a policy without a registry enforcing it.

**Where in StreamPulse:** Phase 1 (`BACKWARD` enforced by a failing build), Phase 6 (the outbox keeps
schema and contract independent).

---

## Q8. Now the consumption side. Explain SSE at the wire level.

**30-second answer:** Server-Sent Events is one long-lived HTTP response with
`Content-Type: text/event-stream`, which the server never closes and writes into incrementally. The
body is a trivially simple line-based format: `event:`, `data:`, `id:` and `retry:` fields, with a
**blank line** terminating each event. It is unidirectional — server to client only — and text only,
no binary. The two features that make it production-grade are built into the protocol rather than
bolted on: the browser's `EventSource` reconnects automatically, and it sends back the last `id:` it
saw in a `Last-Event-ID` request header, so the server can resume from that point. You get resumable
streaming for about twenty lines of server code.

**Deeper — an actual wire trace:**

```http
GET /v1/stream/metrics?tenant=sportsnow HTTP/1.1
Accept: text/event-stream
Last-Event-ID: 1790000000123-7          ← sent automatically by EventSource on reconnect

HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
X-Accel-Buffering: no                    ← tells nginx not to buffer the response

: heartbeat                              ← a COMMENT line (starts with ':'). Keeps proxies
                                            from idling the connection out. Cheapest keepalive.

retry: 3000                              ← client should wait 3s before reconnecting

event: metric
id: 1790000000124-1
data: {"titleId":"t-9","viewers":18422,"rebufferRatio":0.012}
                                         ← BLANK LINE = end of event. This is the delimiter.

event: alert
id: 1790000000131-1
data: {"alertId":"a-77","severity":"CRITICAL","rule":"rebuffer>2%"}

```

| Field | Meaning | Gotcha |
|---|---|---|
| `data:` | the payload; repeated `data:` lines are joined with `\n` | **must not contain a raw newline** — split or JSON-encode it |
| `event:` | a named type; the client listens with `addEventListener("metric", …)` | without it, everything lands on `onmessage` |
| `id:` | the event id, echoed back as `Last-Event-ID` on reconnect | only useful if you can actually resume from it (Q11) |
| `retry:` | reconnect delay hint in ms | the client may still back off on its own |
| `: comment` | ignored by the client | the heartbeat mechanism |

**The four operational realities that decide whether SSE works for you:**

1. **Heartbeats are mandatory.** Load balancers, nginx, API gateways and corporate proxies all kill
   idle connections — 60 s is a common default. A `: heartbeat` comment every 15 s keeps the
   connection warm, and it also lets *you* detect a dead peer, because the write fails. StreamPulse
   uses 15 s (Phase 5).
2. **HTTP/1.1 caps you at ~6 connections per origin, per browser.** Each SSE stream permanently
   occupies one. Open the dashboard in seven tabs and the seventh hangs — and so does every other
   request to that origin. Over **HTTP/2** streams are multiplexed on one TCP connection (typically
   ~100 concurrent streams), so the limit effectively disappears. This is the single most important
   deployment note for SSE, and the fix is "terminate HTTP/2 at the edge", not "change protocol".
3. **Text only, UTF-8.** Binary means base64, which costs ~33% size. If you need binary frames, you
   wanted WebSocket.
4. **`EventSource` cannot set custom headers** — so no `Authorization: Bearer …`. Your options are a
   cookie (with `withCredentials`, plus CORS configured for credentials), a short-lived token in the
   query string (logged by every proxy — avoid, or make it single-use), or a `fetch`-based SSE client
   that parses the stream itself (which loses the built-in reconnect and `Last-Event-ID` handling, so
   you reimplement them). StreamPulse uses a cookie set at Keycloak login. This is the detail that
   proves you have actually shipped SSE rather than read about it.

**Code — the WebFlux endpoint (Phase 5):**

```java
@GetMapping(value = "/v1/stream/metrics", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
Flux<ServerSentEvent<MetricUpdate>> stream(
        @RequestParam String tenant,
        @RequestHeader(name = "Last-Event-ID", required = false) String lastEventId) {

    // 1. Replay the gap from a bounded Redis ring buffer, so a 3-second reconnect
    //    does not leave a hole in the chart. Bounded on purpose: if the client was
    //    away longer than the buffer, we send a `resync` event and it reloads.
    Flux<MetricUpdate> backlog = lastEventId == null
            ? Flux.empty()
            : ringBuffer.since(tenant, lastEventId);

    // 2. Then the live multicast (Q10).
    Flux<MetricUpdate> live = hub.subscribe(tenant);

    // 3. Heartbeat comments so proxies keep the connection open. Merged into the same
    //    Flux so a slow/stalled producer still produces traffic.
    Flux<ServerSentEvent<MetricUpdate>> beats = Flux.interval(Duration.ofSeconds(15))
            .map(i -> ServerSentEvent.<MetricUpdate>builder().comment("hb").build());

    return Flux.merge(
        Flux.concat(backlog, live).map(m -> ServerSentEvent.builder(m)
                .id(m.eventId())            // echoed back as Last-Event-ID
                .event("metric")
                .build()),
        beats);
}
```

**Follow-ups they'll ask:**
- *"What happens on reconnect if you have no buffer?"* — the client resumes and simply misses whatever
  happened while it was away. For a live gauge that self-corrects in one second, fine. For an alert
  feed, not fine — hence the ring buffer, and hence a `resync` event when the gap is too large.
- *"Does SSE work through HTTP/2 without changes?"* — yes; the format is identical, you just stop
  hitting the six-connection limit. `X-Accel-Buffering: no` and disabling response compression
  buffering still matter at the proxy.
- *"How do you test it?"* — `WebTestClient.returnResult(...).getResponseBody()` with `StepVerifier` for
  the Flux, plus an integration test that asserts the heartbeat arrives and that a reconnect with
  `Last-Event-ID` replays the gap.

**Pitfalls / red flags:** not knowing about the blank-line delimiter, the 6-connection limit, or the
missing-`Authorization`-header problem. Those three are what separate "I've used SSE" from "I've
deployed SSE".

**Where in StreamPulse:** Phase 5 (`GET /v1/stream/metrics`), Phase 7 (`useEventSource` hook with
jittered backoff).

---

## Q9. SSE vs WebSocket vs long-polling vs gRPC streaming — pick one and defend it.

**30-second answer:** I choose by direction and payload. If the data flows **one way, server to
client**, and it is text — dashboards, notifications, alert feeds, progress, live logs — SSE is the
right answer: it is plain HTTP, it works through every proxy and CDN, it reconnects and resumes for
free, and there is no separate protocol to secure or scale. WebSocket earns its complexity when you
need **bidirectional, low-latency, or binary** — chat, collaborative editing, gaming, a trading ladder.
gRPC streaming is for **service-to-service**, where you want schemas, deadlines and flow control, and
where the client is not a browser. Long-polling is the fallback for environments that break the other
three. StreamPulse is a telemetry dashboard: one direction, JSON, many subscribers. That's SSE.

**Deeper:**

| | SSE | WebSocket | Long-polling | gRPC streaming |
|---|---|---|---|---|
| Direction | server → client | full duplex | server → client (per request) | uni- or bidirectional |
| Protocol | plain HTTP | HTTP upgrade → `ws://` | plain HTTP | HTTP/2 |
| Payload | UTF-8 text only | text **or binary** | anything | protobuf (binary) |
| Auto-reconnect | **built in** (`EventSource`) | you write it | inherent (new request each time) | you write it |
| Resume from a position | **built in** (`Last-Event-ID`) | you design it | you design it | you design it |
| Proxy/CDN friendliness | excellent (it's a normal response) | needs upgrade support; many gateways need config | perfect | needs HTTP/2 end to end and L7 LB |
| Per-connection cost | 1 HTTP/1.1 connection (≈free over HTTP/2) | 1 TCP connection | a request per poll — highest overhead | 1 HTTP/2 stream |
| Browser support | native, no library | native, no library | native | **not directly** — needs gRPC-Web + a proxy |
| Custom auth headers | **no** (`EventSource` limitation) | via subprotocol/first message | yes | yes (metadata) |
| Backpressure | none in the protocol; you handle it | none in the protocol | natural (client asks) | **built in** (HTTP/2 flow control) |
| Compression | HTTP response compression (watch buffering) | per-message deflate extension | HTTP | HTTP/2 + per-message |
| Best for | dashboards, feeds, notifications, progress | chat, games, collaborative editing | legacy/hostile networks | service-to-service |

**The argument I'd actually make in the room:** "WebSocket is a *more capable* protocol, and for this
problem that capability is pure cost. I don't need client→server messages — the client already has a
REST API for that. I don't need binary. What I do need is reconnect-and-resume, which SSE gives me in
the protocol and WebSocket makes me build, and I need it to work through a corporate proxy on the
first try. If a later requirement adds a client→server channel with sub-100ms latency, that's the
moment to reconsider, and the client-side abstraction in the React hook means only one file changes."

That last clause matters: naming the *trigger* that would change your decision is the difference
between a choice and a preference.

**Follow-ups they'll ask:**
- *"Isn't a WebSocket cheaper than 'a whole HTTP connection'?"* — they are the same thing: one TCP
  connection each. Over HTTP/2 the SSE stream is *cheaper*, because it multiplexes onto a connection
  the browser already has.
- *"What about server load with 50,000 subscribers?"* — with blocking I/O that is 50,000 threads and
  you're dead; with WebFlux/Netty or with virtual threads it is 50,000 mostly-idle connections, which
  is a memory and file-descriptor question, not a thread question. The real constraint is usually
  outbound bandwidth and how many distinct payloads you must serialise — which is why you fan out from
  **one** serialised message (Q10).
- *"Why not just poll every second?"* — at 50,000 clients that is 50,000 req/s of request overhead,
  TLS state and log volume to deliver mostly "nothing changed", plus up to a second of latency. Polling
  is the right answer at small scale or low frequency, and I'd say so rather than pretending otherwise.

**Pitfalls / red flags:** "WebSocket, because it's real-time" — a preference, not a decision. Or not
knowing that browsers can't speak gRPC directly.

**Where in StreamPulse:** Phase 5 ADR "SSE vs WebSocket for the live dashboard"; BS-09 / file 10 covers
the gRPC side, where the client is a smart TV rather than a browser.

---

## Q10. One JVM, one Kafka listener, 5,000 subscribers. How do you fan out?

**30-second answer:** You consume once and multicast in memory. A Kafka listener writes each record
into a `Sinks.Many` multicast sink per tenant; every SSE request subscribes to that sink's `Flux`. The
key properties are: **one** consumer group offset regardless of subscriber count, **one**
deserialization and **one** serialization to JSON, and subscriber lifecycle decoupled from Kafka —
a browser closing a tab cancels its subscription and nothing else notices. The alternative people
reach for — one Kafka consumer per SSE connection — is a disaster: 5,000 consumers in a group with 12
partitions means 4,988 idle members and a rebalance storm every time a tab closes.

**Deeper:**

```
telemetry.agg.v1 (12 partitions)
        │
        ▼  ONE Spring Kafka listener (concurrency = 12), ONE consumer group
  ┌───────────────┐
  │ MetricHub     │   Map<TenantId, Sinks.Many<MetricUpdate>>
  │  emitNext(m)  │──┬──▶ Flux ──▶ SSE connection #1  (browser tab)
  └───────────────┘  ├──▶ Flux ──▶ SSE connection #2
                     ├──▶ … 4,998 more
                     └──▶ Flux ──▶ MCP server subscription (Phase 13)
```

**Choosing the sink is the whole question:**

| Sink | Semantics | Use for |
|---|---|---|
| `Sinks.many().multicast().onBackpressureBuffer()` | buffers when a subscriber is slow; **the buffer is shared**, so one slow subscriber can grow it | default multicast — but bound it |
| `Sinks.many().multicast().directBestEffort()` | **drops** for subscribers that can't keep up; fast ones are unaffected | live telemetry — exactly the semantic a dashboard wants |
| `Sinks.many().replay().limit(n)` / `.limit(Duration)` | new subscribers get the last n / last window | a late-joining tab that should see recent history immediately |
| `Sinks.many().unicast()` | exactly one subscriber, ever | per-connection pipelines, not fan-out |

`directBestEffort` deserves the emphasis: for a metrics dashboard, **dropping is correct**. A value
from four seconds ago has no business arriving now; the next window supersedes it. Buffering would
turn a slow client into a memory leak and would show it stale data anyway.

**Code:**

```java
@Component
class MetricHub {

    // One sink per tenant. multicast: every subscriber sees events emitted AFTER it subscribed.
    // directBestEffort: a subscriber that can't keep up misses updates instead of forcing us
    // to buffer. For live telemetry, stale-and-buffered is worse than fresh-and-gapped.
    private final Map<String, Sinks.Many<MetricUpdate>> sinks = new ConcurrentHashMap<>();

    private Sinks.Many<MetricUpdate> sinkFor(String tenant) {
        return sinks.computeIfAbsent(tenant,
                t -> Sinks.many().multicast().directBestEffort());
    }

    /** Called from the Kafka listener thread — ONE deserialization for all subscribers. */
    void publish(MetricUpdate m) {
        Sinks.EmitResult r = sinkFor(m.tenant()).tryEmitNext(m);
        if (r.isFailure()) {
            // FAIL_ZERO_SUBSCRIBER is normal and boring (nobody is watching this tenant).
            // FAIL_NON_SERIALIZED or FAIL_OVERFLOW is a real signal — alert on it.
            meter.counter("sse.emit.failed", "reason", r.name()).increment();
        }
    }

    Flux<MetricUpdate> subscribe(String tenant) {
        return sinkFor(tenant).asFlux()
                .onBackpressureLatest()                    // per-subscriber safety net (Q12)
                .doFinally(sig -> gauge.decrement());      // tab closed → subscription cancelled
    }
}
```

```java
@KafkaListener(topics = "telemetry.agg.v1", concurrency = "12")
void onAggregate(List<MetricUpdate> batch) {     // batch listener: one poll, many records
    batch.forEach(hub::publish);                 // pure in-memory fan-out; no blocking here
}
```

**The thread-safety caveat I'd raise unprompted:** `tryEmitNext` is not thread-safe against concurrent
emission — Reactor's contract requires serialized calls, and concurrent emitters get
`FAIL_NON_SERIALIZED`. With `concurrency = 12` there are twelve listener threads. Since each tenant's
records are keyed and therefore land on a fixed partition, and each partition is owned by exactly one
listener thread, **emissions for a given tenant are already serialized** — the partitioning does the
synchronisation for me. That is a real design payoff of choosing the Kafka key properly, and it is
worth saying out loud because it shows the two halves of the system connecting. If that invariant
didn't hold, I'd use `emitNext` with a busy-loop/retry handler or serialise per sink.

**Follow-ups they'll ask:**
- *"What if the tenant has no subscribers?"* — the sink still exists and emits into the void
  (`FAIL_ZERO_SUBSCRIBER`). With thousands of tenants, evict idle sinks on a timer, or you have a slow
  map leak. Worth mentioning; it's the kind of thing that only shows up after a week of uptime.
- *"Why not `@Async` + a `CopyOnWriteArrayList` of emitters?"* — that is what `SseEmitter` on MVC
  gives you, and it works. Reactor gives you the backpressure operators (`onBackpressureLatest`,
  `sample`) for free, which is the reason query-api is WebFlux (Phase 5).
- *"How does this behave on a rebalance?"* — partitions move to another instance, so the *source* of a
  tenant's events moves. Subscribers connected to the old instance stop receiving. That is exactly the
  multi-replica problem — Q11.

**Pitfalls / red flags:** one Kafka consumer per HTTP connection. It is the naive design, and knowing
why it fails (group size vs partition count, rebalance on every connect/disconnect) is the point of the
question.

**Where in StreamPulse:** Phase 5 (`MetricHub`, `Sinks.Many` per tenant), Phase 4 (batch listeners).

---

## Q11. Now run three replicas behind a load balancer. What breaks?

**30-second answer:** Fan-out breaks. Kafka partitions are spread across the three instances, so
instance A holds partitions 0–3 and only sees a subset of tenants' events — but the load balancer may
have put a `sportsnow` subscriber on instance C. The subscriber gets nothing, or a fraction. There are
three fixes: make every instance see every event (each instance uses its **own consumer group**, so all
of them consume all partitions — my choice for this workload); or route subscribers to the instance
that owns their data (sticky routing by tenant, which is correct but couples your LB to your
partitioning); or put a broker between them (Redis pub/sub) so the SSE tier is stateless. The first is
simplest and the volume is small, because a dashboard consumes **aggregates**, not raw events.

**Deeper:**

```
❌ BROKEN: one shared consumer group across replicas
   partitions 0-3 → instance A ──▶ tenants {streamflix, kidsplus}
   partitions 4-7 → instance B ──▶ tenants {sportsnow}
   partitions 8-11→ instance C ──▶ tenants {…}
   LB sends the sportsnow dashboard to instance A → it sees nothing. Silently.

✅ FIX 1: per-instance consumer group  (group.id = "sse-fanout-" + instanceId)
   every instance consumes ALL partitions → every instance can serve ANY tenant
   → the SSE tier is stateless; the LB can do whatever it likes
```

| Approach | How | Pros | Cons |
|---|---|---|---|
| **Per-instance consumer group** | `group.id = "sse-" + POD_NAME` | stateless tier, any LB, trivial | every instance reads the whole topic: N× broker egress; each instance needs the whole working set in memory |
| **Sticky routing by tenant** | consistent-hash the tenant to an instance at the ingress | each instance reads only its share | LB must know the hash; rebalances break affinity; a hot tenant is a hot pod |
| **Shared broker (Redis pub/sub)** | Kafka → one consumer group → Redis channel per tenant → all instances subscribe | true stateless fan-out, decoupled from partitioning | one more hop, one more dependency, Redis pub/sub is fire-and-forget (no replay) |
| **Sticky sessions** | LB cookie/IP affinity pinning a client to a pod | works with a shared group | breaks on deploy (every pod restart drops every stream), uneven load, and it is affinity, not correctness |

**Why per-instance groups is right *here*, stated as a calculation rather than a preference:** the SSE
tier consumes `telemetry.agg.v1` — windowed aggregates, not raw events. If that topic runs at, say, 2k
messages/s at ~300 bytes, that is ~600 KB/s; three instances each reading all of it is ~1.8 MB/s of
broker egress. That is nothing. If the SSE tier consumed `telemetry.raw.v1` at 50k events/s, the same
design would be 3× a much bigger number and I would move to the Redis-fan-out or sticky-routing
design. **Say the number; that's what makes it an engineering decision.**

The operational detail that comes with per-instance groups: those groups are **ephemeral**, one per
pod, and they accumulate in the broker's group metadata across every deploy. Set
`auto.offset.reset = latest` (a new SSE subscriber wants *now*, not the last hour) and make sure
`offsets.retention.minutes` cleans them up, or configure a periodic purge.

**Sticky sessions, and why I'd avoid them here.** They look like the easy answer and they are a trap:
a rolling deploy terminates every pod in turn, and with affinity each termination drops every stream
pinned to it — so a deploy becomes a thundering-herd reconnect. SSE reconnects automatically, which
softens it, but you have made deploys a visible event for users. With stateless replicas, a
reconnecting client lands anywhere and just works.

**Code:**

```yaml
# query-api — per-instance consumer group so every replica sees every partition.
spring:
  kafka:
    consumer:
      group-id: sse-fanout-${HOSTNAME}     # K8s injects the pod name; unique per replica
      auto-offset-reset: latest            # a new dashboard wants NOW, not history
      properties:
        # Ephemeral groups: don't keep committed offsets around forever.
        session.timeout.ms: 10000
```

**Follow-ups they'll ask:**
- *"What about `Last-Event-ID` resume across instances?"* — the ring buffer must be **shared**, so it
  lives in Redis (a per-tenant `ZSET`/stream trimmed to N entries or a time window), not in the JVM.
  Otherwise a reconnect that lands on a different pod cannot replay. This is the detail that makes the
  design actually work and it is easy to miss.
- *"How do you load-test this?"* — Gatling with N concurrent SSE connections, asserting per-connection
  event counts match what was produced, and watching heap and file descriptors. The interesting
  failure is not CPU, it's descriptors and socket buffers.
- *"Does this change with virtual threads on MVC instead of WebFlux?"* — the fan-out design is
  identical; what changes is that each connection holds a virtual thread parked on a write instead of
  an event-loop callback. That is fine for 5k connections, and the comparison is the Phase 5 benchmark
  ADR.

**Pitfalls / red flags:** proposing sticky sessions as the primary answer, or not spotting that a
shared consumer group silently gives some subscribers a partial stream. "Silently" is the dangerous
part — there is no error, just missing data.

**Where in StreamPulse:** Phase 5 (Redis ring buffer for `Last-Event-ID`), Phase 11 (three replicas,
rolling deploy drill).

---

## Q12. A dashboard client is on hotel wifi and can't keep up. What happens?

**30-second answer:** Without a strategy, the slow client's TCP window fills, the server's write buffer
backs up, and either memory grows until you OOM or — with a naive reactive chain — backpressure
propagates upstream and the **slow client slows down everyone**, including the Kafka listener. For
telemetry the correct policy is to **drop**, not to buffer: `onBackpressureLatest` keeps only the most
recent value per subscriber, or `sample(Duration)` emits at a fixed rate regardless of source rate. A
dashboard showing "18,422 viewers" does not want the six values it missed — it wants the current one.
Dropping is not a compromise here; it is the semantically correct behaviour, and I'd defend it on
those grounds rather than as a performance hack.

**Deeper:**

| Operator | Behaviour | Right for |
|---|---|---|
| `onBackpressureBuffer(n, strategy)` | queue up to n, then error / drop oldest / drop latest | events you must not lose (alerts) — with a **bound** |
| `onBackpressureLatest()` | keep only the newest unconsumed element | **gauges and counters** — the default for metrics |
| `onBackpressureDrop(cb)` | discard anything the subscriber isn't ready for | fire-hose telemetry where you also want to count the drops |
| `sample(Duration)` | emit the latest value every interval, regardless of input rate | rendering — a chart repainting at 4 Hz is plenty |
| `buffer(Duration)` / `bufferTimeout(n, d)` | batch into lists | one SSE frame per second containing many updates: fewer syscalls, less framing overhead |
| (no operator) | unbounded buffering | an OOM with your name on it |

**The distinction that makes this a senior answer:** different streams on the same connection deserve
different policies.

```
metric updates  → onBackpressureLatest / sample(250ms)   "show me now" — dropping is CORRECT
alert events    → onBackpressureBuffer(256, DROP_OLDEST) "don't lose these" — but still bounded,
                                                          because an unbounded alert buffer for a
                                                          dead client is just a slow memory leak
```

And when the alert buffer *does* overflow, you don't fail silently: emit a `resync` event telling the
client to re-fetch alerts over REST, and increment a counter. **Degrade to a known state, loudly.**

**Code:**

```java
Flux<ServerSentEvent<?>> stream(String tenant) {

    // Metrics: the chart repaints ~4×/s no matter how fast Kafka produces.
    // sample() also caps SERIALIZATION cost, which is the real CPU line item at 5k subscribers.
    Flux<ServerSentEvent<?>> metrics = hub.subscribe(tenant)
            .sample(Duration.ofMillis(250))
            .onBackpressureLatest()
            .map(m -> sse("metric", m));

    // Alerts: bounded buffer; on overflow tell the client to resync rather than lying to it.
    Flux<ServerSentEvent<?>> alerts = alertHub.subscribe(tenant)
            .onBackpressureBuffer(256, dropped -> meter.counter("sse.alert.dropped").increment(),
                                  BufferOverflowStrategy.DROP_OLDEST)
            .map(a -> sse("alert", a));

    return Flux.merge(metrics, alerts, heartbeats())
               .timeout(Duration.ofMinutes(30))      // recycle connections; forces a clean reconnect
               .doOnCancel(() -> activeStreams.decrementAndGet());
}
```

**How you know it is working** — the metrics I'd put on the Grafana panel (Phase 8):

| Metric | Why |
|---|---|
| `sse.connections.active` (gauge) | capacity planning; also spots reconnect storms after a deploy |
| `sse.events.dropped{stream}` | is the drop policy actually firing, and for whom |
| `sse.emit.failed{reason}` | `FAIL_ZERO_SUBSCRIBER` is noise; `FAIL_OVERFLOW` is a bug |
| end-to-end freshness (event ts → SSE write ts) | the actual user-facing SLI (BS-10 targets < 10 s) |
| JVM heap + direct buffer memory | the thing that dies if you got the buffering wrong |

**Follow-ups they'll ask:**
- *"Why not let backpressure propagate to Kafka?"* — because the sink is **multicast**: the Kafka
  listener has one upstream and 5,000 downstreams. Slowing the source to the speed of the slowest
  subscriber means one bad client degrades every other client *and* grows consumer lag *and* eventually
  triggers a rebalance. Backpressure must terminate at the per-subscriber boundary. This is the single
  most important sentence in this answer.
- *"When would you disconnect the client instead?"* — when the drop rate stays high for a sustained
  window: close with a `retry:` hint and let it reconnect (possibly to a less loaded pod). Better than
  serving a permanently degraded stream.
- *"How do you test a slow consumer?"* — `StepVerifier` with `thenAwait` and controlled `request(n)`,
  or an integration test with a client that reads the socket slowly and asserts the server's heap and
  connection stay stable.

**Pitfalls / red flags:** "Reactor handles backpressure automatically." Reactor gives you *operators*;
choosing the policy is your job, and the default (unbounded buffering) is the one that OOMs.

**Where in StreamPulse:** Phase 5 (`onBackpressureLatest`, `sample`, 15 s heartbeat, bounded ring
buffer), Phase 8 (the freshness SLI), BS-10 / REQ-SL-01.

---

## Q13. Put it together: describe StreamPulse's CDC → SSE pipeline end to end.

*(The applied question. Answer it as a walk along the data path, naming the guarantee at each hop.)*

**30-second answer:** An operator edits an alert rule in the admin UI. The control plane writes the
rule **and** an outbox row in one Postgres transaction — so there is no dual write and no window.
Debezium reads the WAL through a replication slot, and the Outbox Event Router unwraps the payload,
keys it by `aggregate_id` and publishes to `config.changes.v1` in **commit order**. Every service
consumes that topic to refresh its caches — including the alert engine, which is how rules reach it
without a database join, which is what made extracting it into its own service possible at all
(BS-08 / REQ-SF-02). When the engine fires, the alert goes to `alerts.v1`; query-api consumes it once
per replica, multicasts it in memory to every subscribed dashboard, and pushes it over SSE with an
event id the browser can resume from. Guarantees: exactly-once into Postgres, at-least-once
everywhere after that, and idempotent consumers keyed on a stable event id — so effectively-once.
End-to-end freshness is a measured SLI, not a hope.

**Deeper — the path, with the guarantee at each hop:**

```
[admin UI] ──PUT /v1/rules/r-42──▶ [control-plane]
                                       │  ONE transaction: UPDATE alert_rule + INSERT outbox
                                       ▼  ← EXACTLY ONCE (local ACID). REQ-DL-06.
                                  [Postgres WAL]
                                       │  logical decoding, replication slot, COMMIT ORDER
                                       ▼  ← the slot is the operational risk (Q4)
                              [Debezium / Kafka Connect]
                                       │  EventRouter: key=aggregate_id, value=payload
                                       ▼  ← AT LEAST ONCE (connector restarts re-emit)
                          config.changes.v1  (keyed ⇒ per-rule ordering)
                           ├──▶ query-api    → LRU cache invalidate      (idempotent)
                           ├──▶ sink-writer  → tenant config refresh     (idempotent)
                           └──▶ alert-engine → rule reload               (REQ-SF-02)
                                       │
                            evaluates telemetry.agg.v1 against the rule
                            alertId = hash(ruleId, windowStart)          ← REQ-SF-05: deterministic,
                                       ▼                                    so the BS-08 cutover
                                  alerts.v1                                 overlap cannot duplicate
                                       │
                       [query-api replica ×3, per-instance consumer group]
                                       │  ONE deserialization → Sinks.Many multicast per tenant
                                       ▼  ← fan-out is in-memory; Kafka doesn't know about clients
                     SSE: event: alert / id: … / data: {…}
                                       │  heartbeat every 15 s; Last-Event-ID replays from the
                                       ▼  shared Redis ring buffer on reconnect
                                  [browser dashboard]
```

**The decisions, and what each one bought:**

| Decision | Alternative rejected | Why |
|---|---|---|
| Outbox in the business transaction | publish after commit | removes the crash window entirely (Q1) |
| Domain event in `payload`, not the row image | table-level CDC | schema changes stop being a cross-team event (Q7) |
| Debezium over a polling relay | `@Scheduled` relay | commit-order delivery, no poll load, and CDC is the skill being demonstrated — the ADR records that polling was viable |
| `aggregate_id` as the Kafka key | round-robin | per-rule ordering, and it makes the `Sinks.Many` emission serialized for free (Q10) |
| Deterministic `alertId` | random UUID | makes the BS-08 cutover overlap safe without coordination (REQ-SF-05) |
| Per-instance consumer groups for SSE | shared group + sticky sessions | stateless tier; deploys don't become reconnect storms (Q11) |
| Drop-latest for metrics, bounded buffer for alerts | one policy for both | different data, different correct behaviour (Q12) |
| `trace_id` carried in the outbox row | nothing | the Tempo trace survives the CDC hop; otherwise observability has a hole |

**How I'd prove it works** (the artifacts, not the claims):
- `OutboxAtomicityIT` — rollback leaves neither row (REQ-DL-06).
- A Testcontainers pipeline test: change a rule → assert the event lands on `config.changes.v1` with
  the right key and headers → assert the consumer's cache is invalidated.
- `ShadowComparatorIT` (BS-08 / REQ-SF-03) — the old and new alert engines run in parallel and their
  outputs are diffed per window; the mismatch metric must be zero for an hour before cutover.
- Kill the Connect container mid-stream and assert no lost events and no duplicate side effects
  (the dedup counter goes up, the cache-invalidation count does not).
- A Grafana panel for replication-slot lag, outbox depth / oldest-pending age, consumer lag, and
  end-to-end freshness.

**Follow-ups they'll ask:**
- *"Where's the weakest link?"* — the replication slot. If Connect is down long enough, WAL grows and
  the database is at risk. Mitigations: alert on `retained_wal`, `max_slot_wal_keep_size` so the
  **slot** dies instead of the database, and a documented re-snapshot runbook. Naming your own weakest
  link is the most senior thing you can do in this answer.
- *"What if a consumer needs the full history?"* — that's what the log is for: reset the consumer group
  to the earliest offset and replay, as long as the topic is compacted or retained long enough.
  `config.changes.v1` is keyed by `aggregate_id` and **compacted**, so a full replay reconstructs
  current state. That is the payoff of choosing the key well.
- *"How do you bound the Last-Event-ID buffer?"* — a Redis structure trimmed to N entries or a time
  window per tenant. If the client's id is older than the window, send `resync` and let it re-fetch
  over REST. Bounded memory, explicit degradation.
- *"What would you do differently at 10× scale?"* — the SSE fan-out is the first thing to move: at
  50k subscribers I'd split the SSE tier from query-api, put Redis (or a dedicated fan-out service)
  between Kafka and the SSE nodes, and batch multiple updates into one frame per second.

> **Your story:** this is the file where your CDC→SSE experience goes, and it is the highest-value
> story you have for this role. Structure it the same way as the walk above: what was the *source of
> truth*, what carried the change (what CDC mechanism, over what transport), what did the last mile
> (SSE, WebSocket, polling), what was the failure that made you build it, and how did you verify no
> events were lost. Be precise about your own scope — "I built the X" vs "I worked on the team that
> built X" — because every sentence here invites three follow-ups. If you hit the replication-slot or
> connection-limit problems in real life, that detail is worth more than any diagram.

> **Your story:** if you have ever had to *debug* a live streaming connection — a proxy idling out a
> stream, a reconnect storm after a deploy, a slow client causing memory growth, a missed event
> traced back to a dual write — tell that one. Debugging stories land harder than architecture
> stories, because they cannot be memorised.

**Where in StreamPulse:** Phase 5 (SSE), Phase 6 (outbox, Debezium, alert engine), BS-08 /
REQ-SF-02, REQ-SF-05; Phase 7 (`useEventSource`); Phase 8 (freshness SLI).

---

## Key terms

| Term | One-line meaning |
|---|---|
| **Dual write** | Changing two systems without a shared transaction; the crash window between them causes permanent divergence. |
| **Transactional outbox** | Writing the business change and an event row in one local transaction, then relaying the row asynchronously. |
| **Relay / polling publisher** | The job that reads outbox rows and publishes them; at-least-once by design. |
| **WAL** | Postgres's write-ahead log — the durable, ordered record of every change. |
| **Logical decoding** | Turning physical WAL records into a per-transaction stream of row changes, in commit order. |
| **Replication slot** | Durable server-side bookmark that pins WAL until the consumer acks; the CDC disk-growth hazard. |
| **`REPLICA IDENTITY`** | What Postgres logs as the "before" image for updates/deletes — PK only by default. |
| **Snapshot vs streaming** | Debezium's initial consistent read of the table (`op: "r"`) vs subsequent WAL following. |
| **Outbox Event Router** | Debezium SMT that unwraps the outbox row into a clean keyed domain event on a routed topic. |
| **Effectively once** | At-least-once delivery + an idempotent consumer write; the honest end-to-end guarantee. |
| **SSE** | One long-lived HTTP response of `text/event-stream`; `event`/`data`/`id`/`retry` fields, blank-line delimited. |
| **`Last-Event-ID`** | Header the browser resends on reconnect, carrying the last `id:` it saw — protocol-level resume. |
| **Heartbeat comment** | A `: …` line that keeps proxies from idling out an otherwise silent stream. |
| **`Sinks.Many` multicast** | Reactor's in-JVM fan-out: one Kafka record, N subscribers, one deserialization. |
| **`onBackpressureLatest` / `sample`** | Drop-based backpressure policies — the semantically correct choice for live gauges. |
| **Per-instance consumer group** | Giving each replica its own `group.id` so every replica sees every partition. |

---

## Self-check (answer out loud, no notes)

1. Why doesn't wrapping the Kafka publish in a try/catch fix the dual-write problem? Name the two
   distinct failure cases.
2. A polling relay tracks progress with `WHERE created_at > :watermark`. Draw the interleaving that
   makes it skip an event.
3. Debezium has been down for six hours. What is happening on the Postgres server right now, and what
   three settings would have saved you?
4. Write out an SSE frame by hand — every field, and the delimiter.
5. Three replicas, one shared consumer group, SSE dashboards behind a round-robin LB. What does the
   user see, and what are your three options?
