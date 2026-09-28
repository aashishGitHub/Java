# 12 — AWS & Terraform: Lambda, VPC, Streaming Services, DynamoDB, S3, Cross-Region, IaC

**Why this matters for an Event Hub role:** the JD asks for Terraform and AWS, and the system is a
streaming pipeline — so the interview will not be "what is S3". It will be **"Kinesis or MSK, and
why"**, **"how do you model this in DynamoDB"**, **"your Lambda gets invoked twice, now what"**, and
**"walk me through the VPC"**. Those are all *decision* questions, and the senior signal is being able
to state the constraint that drives the decision (per-shard read limits, partition-key cardinality,
NAT gateway cost, at-least-once delivery) rather than listing features.

Your AWS SA-Associate gives you the breadth. What this file adds is the **operational depth and the
trade-off framing**, plus the honest story about what StreamPulse can and cannot prove locally.

**Scenario anchor — BS-06:** sink-writer writes many small S3 objects (one per flush), making
Athena-style queries slow and expensive; security requires services reach S3/DynamoDB only over
private networking; and `sportsnow` has an enterprise contract requiring a documented, drilled
regional failover with RPO ≤ 1 min and RTO ≤ 15 min (both **targets chosen for the project**).
REQ-AW-01..07 cover the compaction Lambda, the VPC module, MirrorMaker 2 cross-region simulation, the
failover drill and the Terraform module/CI discipline. Phase 10.

> **A standing caution for this whole file:** AWS service quotas, limits and feature availability
> change constantly, and my knowledge has a cutoff. **Every number below is approximate and must be
> checked against the current AWS Service Quotas page or service documentation before you quote it in
> an interview.** Saying "roughly 1 MB/s per shard, but I'd check the current quota" is a *stronger*
> answer than a confident wrong number — it signals you've actually operated this.

---

## Q1. Explain Lambda's execution model.

**30-second answer:** Lambda runs your handler inside a **micro-VM execution environment** that it
creates on demand. The lifecycle is: download/unpack the code, start the runtime, run your
**initialization** code (static blocks, constructor, Spring context refresh), then invoke the handler.
The environment is then **frozen** and reused for subsequent invocations — so init runs once and the
handler runs many times, which is why anything expensive belongs in init and why **static state leaks
between invocations of different requests**. Concurrency is the model's defining property: one
environment handles exactly **one invocation at a time**, so 500 concurrent requests means 500
environments. There's no in-process concurrency to tune, which is a very different mental model from a
Spring Boot service with a thread pool.

**Deeper:**

```
  ┌─ COLD START ──────────────────────────────────────────┐
  │ 1. Download code / image layers                        │  ← billed? (init is billed on
  │ 2. Start runtime (JVM)                                 │     standard Lambda; verify)
  │ 3. INIT phase: static init, DI container, connections  │
  ├─ WARM ─────────────────────────────────────────────────┤
  │ 4. INVOKE handler                                      │  ← billed for duration
  │    ...freeze...                                        │  ← background threads are SUSPENDED
  │ 4. INVOKE handler (reused environment)                 │
  │    ...freeze...                                        │
  ├─ SHUTDOWN ─────────────────────────────────────────────┤
  │ 5. Environment reclaimed after idle (minutes, no SLA)  │
  └────────────────────────────────────────────────────────┘
```

Consequences that come up in interviews:

| Property | Consequence |
|---|---|
| One invocation per environment at a time | No request-level thread contention. But also: no connection *sharing* across concurrent requests — 500 concurrent Lambdas = 500 DB connections. This is why RDS Proxy exists. |
| Environment is **frozen**, not stopped, between invocations | A background thread or async task started during an invocation is **paused** mid-flight and may resume much later, or never. Never fire-and-forget in Lambda. |
| `/tmp` persists across invocations in the same environment | Usable as a cache — and a source of cross-request data leakage if you're careless. |
| Static/singleton state persists | Great for a connection pool or an SDK client. Dangerous for anything request-scoped. |
| CPU scales with configured **memory** | "Memory" is really a compute-size dial. A CPU-bound Java function is often *cheaper* at higher memory because it finishes proportionally faster. |

Approximate limits worth knowing (**verify all of these**): max timeout ~15 minutes; memory roughly
128 MB → ~10 GB; `/tmp` 512 MB by default with a larger configurable option; synchronous payload
~6 MB, asynchronous ~256 KB; default account concurrency ~1,000 (raisable).

**Code:**

```java
/**
 * BS-06 REQ-AW-01 — S3 hourly compaction as a Spring Cloud Function.
 * The Spring context is built ONCE per execution environment (init phase),
 * not per invocation. That is the single biggest cold-start lever you control.
 */
@SpringBootApplication
public class CompactionApplication {

    public static void main(String[] args) { SpringApplication.run(CompactionApplication.class, args); }

    /**
     * Spring Cloud Function exposes this bean as the handler.
     * Everything it depends on (S3 client, config) is a singleton created at INIT.
     */
    @Bean
    public Function<SQSEvent, CompactionResult> compactHour(CompactionService service) {
        return event -> service.compact(event);
    }
}

@Service
class CompactionService {

    // Created during INIT, reused across every invocation in this environment.
    // Creating the SDK client inside the handler would add ~hundreds of ms to EVERY call.
    private final S3Client s3;

    CompactionService(S3Client s3) { this.s3 = s3; }

    CompactionResult compact(SQSEvent event) { /* merge part-files → hour.ndjson.gz + manifest */ }
}
```

**Follow-ups they'll ask:**
- *"Is init time billed?"* — for standard on-demand Lambda, init has historically been billed as part
  of the invocation. **Verify current billing behaviour**, especially for SnapStart and provisioned
  concurrency, which have different rules.
- *"How do you share connections across invocations?"* — you can't share across *concurrent*
  environments. Within one environment, keep the client static. Across environments, use a proxy layer
  (RDS Proxy for Postgres) or a stateless data API (DynamoDB, which is HTTP and needs no pool).
- *"What happens to a thread you started?"* — it is frozen with the environment. It may resume on the
  next invocation, seconds or hours later, in a completely different logical request. Treat any
  background work as unreliable.

**Pitfalls / red flags:** treating Lambda like a servlet container; starting async work and returning;
building SDK clients inside the handler.

**Where in StreamPulse:** BS-06 / REQ-AW-01, REQ-AW-02; Phase 10.

**Verify:** all limits above; init billing; `/tmp` sizing options.

---

## Q2. Why are Java Lambda cold starts bad, and what do you do about it?

**30-second answer:** Java's cold start is dominated by **JVM startup plus classloading plus framework
initialization** — for a Spring Boot function that's hundreds of classes, reflection-driven component
scanning and proxy generation, and it can be seconds. It's also worsened by JIT: the first few hundred
invocations run interpreted or in C1, so even a "warm" function is slow at first. The levers, in the
order I'd apply them: shrink the work (trim dependencies, turn off component scanning in favour of
explicit `@Bean` registration, use a plain `RequestHandler` rather than a full Spring context if the
function is small); **SnapStart**, which snapshots the initialized JVM and restores from it; and
**provisioned concurrency**, which keeps N environments permanently initialized. And the honest senior
answer underneath all of that: if cold start latency actually matters for this workload, Lambda may be
the wrong runtime, and an always-on container is the simpler fix.

**Deeper:**

What actually costs time in a Java cold start:

| Contributor | Typical weight | Mitigation |
|---|---|---|
| JVM process start | fixed, moderate | unavoidable on standard Lambda; SnapStart skips it |
| Classloading + verification | **large**, scales with dependency count | fewer dependencies; AppCDS/class-data sharing |
| Spring context refresh (scanning, reflection, proxies) | **large** | `@Import` explicit configs instead of `@ComponentScan`; functional bean registration; `spring.main.lazy-initialization` (careful — moves cost to first use) |
| SDK client construction + first TLS handshake | moderate | construct in init; use the SDK's HTTP client that starts fastest (`UrlConnectionHttpClient` for the AWS SDK v2 is a common Lambda choice) |
| JIT warmup | affects the first N *invocations*, not just the first | `-XX:TieredStopAtLevel=1` trades peak throughput for faster warmup — good for short-lived functions |

Also: the **OTel Java agent** (file 11 Q10) rewrites bytecode at class load, which is directly at odds
with cold-start optimization. On Lambda you'd use the Lambda-specific OTel layer or manual
instrumentation, not the full agent.

**SnapStart** is the interesting one. The mechanism: AWS runs your init phase once at *publish* time,
takes a **Firecracker microVM snapshot** of the fully initialized JVM, and restores from that snapshot
on invocation — so the JVM start and the Spring context refresh are already done. Reported improvements
are large. But it changes your programming model in ways you must be able to name:

| SnapStart consequence | What you must do |
|---|---|
| The snapshot is taken **once** and restored many times | Anything unique-per-environment captured at init is now **shared across every restore** — most dangerously a seeded `Random`/`SecureRandom` state, or a cached "instance ID". |
| Network connections in the snapshot are **stale** on restore | Re-establish connections after restore. |
| Ephemeral credentials in the snapshot may be expired | Refresh on restore. |
| Hooks | The **CRaC** API (`org.crac.Resource` with `beforeCheckpoint` / `afterRestore`) is the supported way to run code around the snapshot. Spring Boot 3.2+ has CRaC integration. |

**I am not certain of SnapStart's current runtime coverage, architecture support, or its
compatibility with provisioned concurrency / EFS / VPC configurations — these have all changed since
launch. Verify against current AWS docs before relying on any of it.**

**Provisioned concurrency** is the blunt instrument: you pay to keep N environments initialized and
warm. It works, it costs money whether or not you use it, and it does not eliminate cold starts above
N — it just moves the cliff.

**Code:**

```java
/**
 * CRaC hooks for SnapStart (BS-06 REQ-AW-02). The pattern to know:
 * tear down anything environment-specific BEFORE the snapshot,
 * rebuild it AFTER restore.
 */
@Component
class SnapStartLifecycle implements org.crac.Resource {

    private final ConnectionHolder connections;
    private volatile SecureRandom random;

    SnapStartLifecycle(ConnectionHolder connections) {
        this.connections = connections;
        org.crac.Core.getGlobalContext().register(this);   // register for checkpoint callbacks
    }

    @Override
    public void beforeCheckpoint(org.crac.Context<? extends org.crac.Resource> ctx) {
        // Connections in the snapshot would be dead on restore. Close them now.
        connections.closeAll();
    }

    @Override
    public void afterRestore(org.crac.Context<? extends org.crac.Resource> ctx) {
        connections.reopen();
        // CRITICAL: reseed. Every restored environment shares the snapshot's RNG state,
        // so a cached SecureRandom would produce identical sequences across environments.
        this.random = new SecureRandom();
    }
}
```

```hcl
# Terraform — provisioned concurrency on the published version (Phase 10)
resource "aws_lambda_provisioned_concurrency_config" "compact" {
  function_name                     = aws_lambda_function.compact_hour.function_name
  qualifier                         = aws_lambda_function.compact_hour.version  # NOT $LATEST
  provisioned_concurrent_executions = 2
}
```

**Follow-ups they'll ask:**
- *"GraalVM native image instead?"* — a native image starts in milliseconds and uses far less memory,
  which essentially solves the cold start. The costs are real: reflection/proxies need explicit
  configuration (Spring's AOT processing generates most of it), build times are long, and peak
  throughput can be lower than a warmed JIT. For a short-lived compaction function it's a good fit; for
  a long-running service it usually isn't.
- *"Does provisioned concurrency remove cold starts?"* — only up to the provisioned count. Beyond it
  you spill to on-demand and get cold starts at exactly the moment traffic spiked.
- *"How do you *measure* cold start?"* — the `Init Duration` field in the CloudWatch `REPORT` log line,
  and the `initDuration` in Lambda Insights. BS-06 REQ-AW-02 explicitly requires cold start to be
  measured, not assumed.
- *"Is the compaction function even latency-sensitive?"* — no, and that's the answer that shows
  judgement: it runs on an S3 event after an hour closes, so a 3-second cold start is irrelevant. I'd
  measure and document it, and *not* pay for provisioned concurrency.

**Pitfalls / red flags:** proposing SnapStart without mentioning the uniqueness/connection caveats;
treating provisioned concurrency as a cure rather than a cost.

**Where in StreamPulse:** BS-06 / REQ-AW-02 ("cold-start is measured, and SnapStart is discussed");
Phase 10.

**Verify:** SnapStart runtime/architecture support and compatibility constraints; CRaC API coordinates;
Spring Boot's CRaC support version; `-XX:TieredStopAtLevel` effect on your workload.

---

## Q3. Which Lambda event sources are push and which are poll, and why does it matter?

**30-second answer:** **Push** sources invoke Lambda directly — API Gateway, ALB, S3 notifications,
SNS, EventBridge. **Poll** sources are read *by the Lambda service* on your behalf through an **event
source mapping** — SQS, Kinesis, DynamoDB Streams, MSK and self-managed Kafka. The distinction matters
for three reasons: error handling (a poll source retries by *not advancing*, so a poison record on a
Kinesis shard **blocks that shard** until it expires, whereas SQS moves on and eventually DLQs);
batching and concurrency (poll sources give you batch size, batch window and — for stream sources —
concurrency tied to shard count); and ordering (stream sources preserve per-shard order, queues
generally don't). Getting this wrong produces the classic "one bad record stopped our entire pipeline"
incident.

**Deeper:**

| Source | Model | Invocation | Retry behaviour | Ordering | Concurrency driver |
|---|---|---|---|---|---|
| API Gateway / ALB / Function URL | push | **synchronous** | caller retries | n/a | request rate |
| S3 event notification | push | asynchronous | Lambda service retries (approx. twice), then on-failure destination/DLQ | none | event rate |
| SNS | push | asynchronous | as above | none (FIFO topics differ) | event rate |
| EventBridge | push | asynchronous | retries over ~24h with backoff | none | event rate |
| **SQS** | poll | ESM | message returns to queue after visibility timeout; `maxReceiveCount` → DLQ | none (FIFO: per message group) | scales with queue depth, bounded by max concurrency |
| **Kinesis Data Streams** | poll | ESM | **retries the same batch** until success or record expiry — *blocks the shard* | per shard | 1 concurrent invoke per shard (more with parallelization factor) |
| **DynamoDB Streams** | poll | ESM | same as Kinesis | per partition key | per shard |
| **MSK / self-managed Kafka** | poll | ESM | consumer-group semantics | per partition | per partition |

**The poison-pill problem on stream sources** is the one to be able to solve out loud. Because a
Kinesis/DynamoDB Streams ESM retries the *same batch*, a single unprocessable record stalls the shard
until the record's retention expires — which could be 24 hours. The mitigations, all ESM settings:

- `BisectBatchOnFunctionError` — on failure, split the batch in half and retry, isolating the bad
  record quickly.
- `MaximumRetryAttempts` and `MaximumRecordAgeInSeconds` — bound how long you'll keep trying.
- `DestinationConfig` / `OnFailure` — send metadata about the failed batch to an SQS queue or SNS topic
  (note: for stream sources the destination receives *metadata about* the failure, not the records
  themselves, so you need the sequence numbers to go re-read them).
- `FunctionResponseTypes: ["ReportBatchItemFailures"]` — the handler returns the sequence numbers it
  failed on, and the ESM only retries those. This is the best option and the one I'd reach for.

**Verify the exact ESM parameter names and semantics** — they've been extended repeatedly.

**Code:**

```java
/**
 * Partial batch failure reporting — the fix for "one bad record blocks the shard".
 * BS-06 uses SQS in front of the compaction Lambda, but the same pattern applies
 * to the Kinesis comparison adapter (Phase 10).
 */
public SQSBatchResponse handleRequest(SQSEvent event, Context ctx) {
    List<SQSBatchResponse.BatchItemFailure> failures = new ArrayList<>();
    for (SQSEvent.SQSMessage msg : event.getRecords()) {
        try {
            compactionService.compact(msg);
        } catch (RetryableException e) {
            // Report ONLY this message as failed. The rest are deleted from the queue.
            // Without this, a single failure returns the WHOLE batch to the queue and
            // you reprocess (and re-charge for) everything that already succeeded.
            failures.add(new SQSBatchResponse.BatchItemFailure(msg.getMessageId()));
        } catch (PermanentException e) {
            // Non-retryable: log, emit a metric, and ACK it so it doesn't spin forever.
            log.error("permanently failed, dropping to audit log", e);
            auditStore.record(msg);
        }
    }
    return new SQSBatchResponse(failures);
}
```

```hcl
# Event source mapping with the guardrails (Phase 10)
resource "aws_lambda_event_source_mapping" "compaction" {
  event_source_arn                   = aws_sqs_queue.compaction.arn
  function_name                      = aws_lambda_function.compact_hour.arn
  batch_size                         = 10
  maximum_batching_window_in_seconds = 20   # trade latency for fewer, fuller invocations
  function_response_types            = ["ReportBatchItemFailures"]
}
```

**Follow-ups they'll ask:**
- *"Why SQS between S3 and Lambda instead of S3 → Lambda directly?"* (REQ-AW-01 does exactly this) —
  four reasons: SQS gives a real retry policy and a DLQ, it buffers a burst of thousands of S3 events
  so Lambda concurrency doesn't spike, it decouples the compaction rate from the arrival rate, and it
  lets me batch many object-created events into one compaction run. S3 → Lambda directly has weak
  retry semantics and no buffering.
- *"How does SQS Lambda scaling work?"* — the service polls and scales the number of concurrent
  pollers up as the queue grows, subject to your reserved concurrency. **Verify the current scaling
  step behaviour** — it has changed.
- *"What's the parallelization factor on Kinesis?"* — it lets more than one Lambda process a single
  shard concurrently, partitioned by partition key so per-key ordering is preserved. Useful when a
  shard's records are slow to process; useless if you need strict per-shard ordering.

**Pitfalls / red flags:** not knowing that stream-source retries block the shard; using S3 → Lambda
directly for anything that must not be lost.

**Where in StreamPulse:** BS-06 / REQ-AW-01 (S3 event → SQS → Lambda); Phase 10 Kinesis adapter.

**Verify:** ESM parameter names, SQS scaling behaviour, retry counts for async push sources.

---

## Q4. Your Lambda gets invoked twice for the same event. What do you do?

**30-second answer:** Expect it and design for it — **almost every Lambda trigger is at-least-once**.
S3 notifications, SQS standard queues, Kinesis after an ESM retry, EventBridge retries: all can deliver
the same logical event more than once. So the handler must be **idempotent**, and there are two ways to
get there. The better one is to make the operation **naturally idempotent** — write to a deterministic
S3 key so a re-run overwrites with identical bytes, or use a DynamoDB conditional write keyed by the
event ID. The fallback is an explicit **idempotency table** in DynamoDB with the event ID as the
partition key, a conditional put to claim it, a status field, and a TTL. In StreamPulse's compaction
function the natural route works: the output key is derived from the hour, the merge is deterministically
ordered, and re-running produces byte-identical output — which is exactly what REQ-AW-01 specifies.

**Deeper:**

Why duplicates happen, by source:

| Source | Duplicate cause |
|---|---|
| SQS standard | at-least-once by design; also visibility timeout expiring while you're still processing |
| S3 notifications | rare but documented as possible |
| Kinesis / DynamoDB Streams | ESM retries the batch; records that already succeeded are replayed |
| EventBridge / SNS | retry on a transient invoke failure where the function actually succeeded |
| Your own retry logic | a timeout on the *response* path — the write happened, you just didn't hear about it |

That last row is the general principle and worth stating: **a timeout tells you nothing about whether
the work happened.** Idempotency is the only correct answer to a network that can lose replies.

**The idempotency-table pattern, in full:**

```
1. Compute a deterministic idempotency key: hash(eventId) or hash(bucket, key, etag)
2. Conditional PutItem: PK=key, status=IN_PROGRESS, ttl=now+24h
       ConditionExpression: attribute_not_exists(PK)
   ├─ succeeded  → we own it → do the work → update status=DONE, store the result
   └─ failed (ConditionalCheckFailed) → someone else has it
        ├─ status=DONE        → return the stored result (true idempotency, not just dedup)
        └─ status=IN_PROGRESS → either return 409/retry-later, or wait — a deliberate choice
```

The subtlety most people miss: storing the **result**, not just the fact of completion. If the second
invocation only sees "already done" and returns nothing, the caller gets a different response for the
same request — which is dedup, not idempotency. AWS publishes a Powertools idempotency utility that
implements exactly this; **verify the current Java Powertools module name and API** if you use it
rather than hand-rolling.

**Code:**

```java
/**
 * BS-06 REQ-AW-01: "The function is idempotent: re-running produces the same output
 * (deterministic ordering, overwrite by key)."
 *
 * This is the BETTER kind of idempotency — no extra table, no extra failure mode.
 * The state lives in the output itself.
 */
CompactionResult compact(String tenant, LocalDate date, int hour) {

    // 1. Deterministic INPUT set: everything under the closed hour's prefix.
    List<S3Object> parts = s3.listObjectsV2(r -> r
                    .bucket(raw).prefix("raw/tenant=%s/dt=%s/hour=%02d/".formatted(tenant, date, hour)))
            .contents().stream()
            // 2. Deterministic ORDER. Without this, two runs merge in different
            //    orders and produce different bytes — breaking idempotency silently.
            .sorted(Comparator.comparing(S3Object::key))
            .toList();

    // 3. Deterministic OUTPUT key. A second run overwrites with identical content.
    String outKey = "compacted/tenant=%s/dt=%s/hour=%02d.ndjson.gz".formatted(tenant, date, hour);

    byte[] merged = gzipMerge(parts);
    s3.putObject(b -> b.bucket(archive).key(outKey), RequestBody.fromBytes(merged));

    // 4. Manifest written AFTER the data, so a manifest implies durable data.
    //    (Ordering matters: readers use the manifest as the commit marker.)
    s3.putObject(b -> b.bucket(archive).key(outKey + ".manifest.json"),
                 RequestBody.fromString(manifestJson(parts, merged.length)));

    return new CompactionResult(outKey, parts.size(), merged.length);
}
```

```java
/**
 * The explicit fallback, for operations that are NOT naturally idempotent
 * (e.g. "charge this tenant", "send this partner webhook" — BS-07).
 */
boolean claim(String idempotencyKey) {
    try {
        dynamo.putItem(r -> r.tableName("idempotency")
                .item(Map.of(
                        "pk",     AttributeValue.fromS(idempotencyKey),
                        "status", AttributeValue.fromS("IN_PROGRESS"),
                        "ttl",    AttributeValue.fromN(String.valueOf(epochPlus(Duration.ofHours(24))))))
                // The whole mechanism is this one line: an atomic compare-and-set.
                .conditionExpression("attribute_not_exists(pk)"));
        return true;                                   // we own it
    } catch (ConditionalCheckFailedException e) {
        return false;                                  // someone else got there first
    }
}
```

**Follow-ups they'll ask:**
- *"What TTL do you put on the idempotency record?"* — longer than the maximum possible redelivery
  window for the source. For SQS with a 4-day retention and a DLQ, 24 hours is too short; size it from
  the source's retention, not from a round number.
- *"What if the function crashes between claiming and finishing?"* — the record sits in `IN_PROGRESS`
  forever and blocks all retries. You need either a separate `lockExpiry` attribute that lets a later
  invocation steal an abandoned claim, or a short TTL on `IN_PROGRESS` records specifically. This is
  the flaw in naive implementations and a great thing to raise yourself.
- *"Does SQS FIFO give you exactly-once?"* — FIFO gives exactly-once *processing* within the
  deduplication window (approximately 5 minutes — **verify**) using a deduplication ID. Outside that
  window, or across the Lambda retry boundary, you're back to needing idempotency. "Exactly-once" in
  any distributed system is really "at-least-once delivery plus idempotent processing".

**Pitfalls / red flags:** "we use FIFO queues so duplicates can't happen"; an idempotency check that is
a read-then-write instead of a conditional write (a race, not a lock).

**Where in StreamPulse:** BS-06 / REQ-AW-01; the same pattern as the Redis `SET NX EX` dedup in ingest
(Phase 2) and the DynamoDB conditional writes in sink-writer (Phase 4).

**Verify:** SQS FIFO dedup window; Powertools idempotency module name/API.

---

## Q5. When is Lambda the wrong answer?

**30-second answer:** When the workload is **long-running, stateful, latency-sensitive at p99,
steady-and-high-volume, or needs a persistent connection**. Concretely: anything past the ~15-minute
timeout; a Kafka Streams topology, because it holds local RocksDB state and participates in
rebalancing; anything serving SSE or WebSocket connections, which are long-lived by definition;
anything where a cold start would land in your latency SLO; and, unglamorously, **steady high-volume
traffic**, where Lambda's per-invocation pricing is usually more expensive than a right-sized container
that's busy 24/7. Lambda's sweet spot is bursty, short, stateless, event-driven work — which is exactly
what the StreamPulse compaction function is, and exactly what the rest of the pipeline is not.

**Deeper:**

| Workload | Lambda? | Why |
|---|---|---|
| S3 hourly compaction (BS-06) | ✅ yes | Bursty, short, stateless, event-driven, latency-insensitive. |
| Ingest gateway | ❌ no | Steady high volume, latency SLO of 300ms p99, JWT validation cache, Kafka producer with batching/linger — all want a warm process. |
| Kafka Streams processor | ❌ no | Local state stores, consumer group membership, rebalancing. Fundamentally a long-lived stateful process. |
| SSE query-api | ❌ no | Long-lived connections; Lambda's model is request/response. |
| Alert notification fan-out to partners | ⚠️ maybe | Bursty and stateless, so it fits — but BS-04's per-tenant bulkheads and per-partner rate limiters are easier to express in one long-lived process than across N independent invocations. A per-instance rate limiter is meaningless when "instances" are created on demand. |
| Nightly billing rollup | ⚠️ depends | Fine if it fits in 15 minutes; otherwise Step Functions, ECS task, or Batch. |

The **rate limiter point** generalizes and is a genuinely senior observation: any pattern that depends
on *per-instance* state — a local circuit breaker, a semaphore bulkhead, a connection pool, an
in-memory cache, a local token bucket — degrades or breaks under Lambda's "arbitrary number of
independent environments" model. Those all have to become distributed (Redis/DynamoDB-backed), which
adds a network hop and a dependency. That's a real architectural cost, not a detail.

**The cost shape**, stated carefully: Lambda bills per request plus GB-seconds, with no idle charge;
a container bills for wall-clock time whether busy or not. So Lambda wins decisively on spiky/low-duty
workloads and loses on high-duty ones. Where exactly the crossover sits depends on your memory setting,
duration and request rate — **model it with the AWS pricing calculator for your actual numbers rather
than quoting a rule of thumb.**

**Follow-ups they'll ask:**
- *"What would you use instead?"* — ECS Fargate or EKS for long-running services (StreamPulse uses
  Kubernetes, Phase 11); Step Functions for orchestration longer than 15 minutes; AWS Batch for heavy
  batch compute.
- *"Lambda inside a VPC — any downside?"* — see Q7. Historically a big cold-start penalty; now
  mitigated by Hyperplane ENIs, but you still inherit subnet IP exhaustion at high concurrency and you
  need VPC endpoints or NAT for AWS API access. **Verify the current VPC cold-start behaviour.**
- *"Could you run Spring Boot on Lambda for the whole app?"* — you can (Spring Cloud Function, or the
  AWS serverless-java-container adapter) and it's a legitimate migration path, but you're paying Java
  cold-start cost to run a framework designed for a long-lived process. I'd only do it to lift an
  existing app, not as a greenfield choice.

**Pitfalls / red flags:** "serverless is cheaper" as an unqualified claim; proposing Lambda for
anything holding state or a connection.

**Where in StreamPulse:** BS-06 / REQ-AW-01; the plan deliberately confines Lambda to compaction and
keeps everything else on Kubernetes (Phase 11) — that contrast *is* the answer.

---

## Q6. Walk me through your VPC design for this system.

**30-second answer:** Three AZs, a `/16` VPC split into per-AZ public and private subnets with room to
grow. **Public subnets hold only load balancers and NAT gateways**; every compute workload — EKS nodes,
MSK brokers, Lambda ENIs — is in private subnets with no route to an internet gateway. Outbound
internet for private subnets goes through NAT, and that is a real cost driver, not a footnote: NAT
gateways are billed hourly *per AZ* plus per GB processed, so a chatty service can generate a NAT bill
that dwarfs its compute. The single biggest lever against that is **VPC endpoints** — gateway endpoints
for S3 and DynamoDB, which are free and remove that traffic from NAT entirely. Security groups do the
instance-level, stateful filtering; NACLs are the subnet-level, stateless backstop I use sparingly.

**Deeper:**

### CIDR planning

Plan for growth and for **non-overlap**, because the day you need VPC peering, Transit Gateway, or a
VPN to a corporate network, overlapping CIDRs are unfixable without renumbering.

```
VPC             10.40.0.0/16        (65,536 addresses — one VPC per env per region)
├─ public-a     10.40.0.0/20        ALB, NAT gateway
├─ public-b     10.40.16.0/20
├─ public-c     10.40.32.0/20
├─ private-a    10.40.64.0/19       EKS nodes, MSK, Lambda ENIs  ← generous: pods need IPs
├─ private-b    10.40.96.0/19
├─ private-c    10.40.128.0/19
└─ (reserved)   10.40.160.0/19      data subnets / future use
```

Two sizing notes that matter in practice:
- AWS reserves **5 IP addresses per subnet** (network, VPC router, DNS, future use, broadcast), so a
  `/28` gives you 11 usable, not 16.
- With the **EKS VPC CNI**, every pod gets a real VPC IP from the node's subnet. A cluster with
  hundreds of pods per node exhausts a small subnet fast. This is the number one reason to make private
  subnets much bigger than they "need" to be. (Prefix delegation and custom networking change the
  arithmetic — **verify the current CNI IP allocation behaviour.**)

### Public vs private, and the NAT cost driver

| | Public subnet | Private subnet |
|---|---|---|
| Route to IGW | yes (`0.0.0.0/0` → IGW) | no |
| Outbound internet | direct | via NAT gateway in a public subnet |
| Should contain | ALB/NLB, NAT gateways, bastion (if any) | **everything else** |

NAT gateway cost has two components — an **hourly charge per gateway** and a **per-GB data processing
charge**. Three consequences:

1. One NAT per AZ is the resilient design (a single NAT is a cross-AZ dependency *and* incurs cross-AZ
   data charges). Three AZs = 3× the hourly charge. In a dev environment I'd deliberately run **one**
   NAT and accept the reduced resilience, because dev doesn't need AZ-independent egress — that's a
   defensible cost decision to volunteer.
2. Per-GB processing means **high-volume traffic to AWS services through NAT is pure waste**. A
   sink-writer pushing terabytes to S3 through a NAT gateway is paying a data-processing fee for
   traffic that never needed to leave AWS's network.
3. That's what makes **gateway endpoints** (Q7) a cost decision as much as a security one.

**Code:**

```hcl
# infra/modules/vpc/main.tf — BS-06 REQ-AW-03 (Phase 10)
# Applied against moto at the API level; docs/network.md states plainly that real
# network isolation cannot be verified locally (see Q14).

locals {
  azs = ["${var.region}a", "${var.region}b", "${var.region}c"]
}

resource "aws_vpc" "this" {
  cidr_block           = var.cidr          # 10.40.0.0/16
  enable_dns_support   = true
  enable_dns_hostnames = true              # required for interface endpoints' private DNS
  tags                 = { Name = "${var.env}-streampulse" }
}

# Private subnets: /19 each, deliberately oversized for EKS pod IPs.
resource "aws_subnet" "private" {
  for_each          = toset(local.azs)
  vpc_id            = aws_vpc.this.id
  availability_zone = each.value
  cidr_block        = cidrsubnet(var.cidr, 3, index(local.azs, each.value) + 2)
  tags = {
    Name                              = "${var.env}-private-${each.value}"
    "kubernetes.io/role/internal-elb" = "1"   # EKS discovers internal LB subnets by tag
  }
}

# One NAT per AZ in prod for AZ-independent egress; one total in dev to cut cost.
resource "aws_nat_gateway" "this" {
  for_each      = var.single_nat ? toset([local.azs[0]]) : toset(local.azs)
  allocation_id = aws_eip.nat[each.value].id
  subnet_id     = aws_subnet.public[each.value].id
}
```

**Follow-ups they'll ask:**
- *"How many AZs?"* — three for production. Two survives one AZ loss but leaves you at 50% capacity;
  three leaves you at 67%, and quorum-based systems (Kafka/MSK with RF=3, etcd) genuinely need three.
- *"IPv6?"* — an egress-only internet gateway replaces NAT for IPv6 and has no hourly/per-GB charge,
  which makes it a real cost lever. The catch is that not everything you talk to is dual-stack.
- *"How do you connect to on-prem?"* — Site-to-Site VPN for simple/cheap, Direct Connect for
  consistent bandwidth and latency, Transit Gateway as the hub when you have many VPCs. Non-overlapping
  CIDRs are a prerequisite for all of them.
- *"Where do MSK brokers go?"* — private subnets, one per AZ, with a security group that only accepts
  9092/9094 from the application security groups. REQ-AW-03 specifies exactly that.

**Pitfalls / red flags:** putting application instances in public subnets "with a security group";
a single NAT gateway in production; `/24` private subnets under EKS.

**Where in StreamPulse:** BS-06 / REQ-AW-03; `infra/modules/vpc`; `docs/network.md`.

**Verify:** NAT gateway pricing components; EKS VPC CNI IP allocation behaviour and prefix delegation.

---

## Q7. VPC endpoints, security groups vs NACLs, and Lambda-in-VPC.

**30-second answer:** **Gateway endpoints** exist only for **S3 and DynamoDB**: they're a route-table
entry, they cost nothing, and they keep that traffic off the NAT gateway entirely — so they're the
first thing I add. **Interface endpoints** (PrivateLink) are for everything else — SQS, Kinesis, KMS,
Secrets Manager, ECR, STS — and they're an ENI in your subnet with a real hourly and per-GB cost, so
you add the ones you need rather than all of them. **Security groups** are stateful, attached to ENIs,
allow-only, and can reference other security groups — which is how I express "ingest may reach MSK"
without hardcoding IPs. **NACLs** are stateless, subnet-level, allow *and* deny, and evaluated in rule
order; because they're stateless you must open the ephemeral port range for return traffic, which is
the classic NACL bug. I use security groups for real access control and NACLs only as a coarse
backstop.

**Deeper:**

| | Gateway endpoint | Interface endpoint (PrivateLink) |
|---|---|---|
| Services | **S3, DynamoDB only** | most AWS services, plus partner/your own services |
| Mechanism | route table entry (prefix list) | ENI with a private IP in your subnet |
| Cost | **free** | hourly per endpoint per AZ + per GB |
| DNS | traffic routed by destination prefix; the public endpoint name still resolves publicly | private DNS overrides the service hostname to the ENI IP |
| Cross-region / on-prem | no | yes (reachable over VPN/DX) |
| Policy control | endpoint policy | endpoint policy + security group on the ENI |

**Security groups vs NACLs:**

| | Security group | NACL |
|---|---|---|
| Attached to | ENI (instance, Lambda, endpoint, RDS) | subnet |
| Stateful? | **yes** — return traffic is automatically allowed | **no** — you must allow the return path explicitly |
| Rules | allow only | allow **and** deny |
| Evaluation | all rules evaluated, permissive union | **in rule-number order**, first match wins |
| Can reference | other security groups ✅ | CIDRs only |
| Use for | real access control | coarse subnet-level deny (block a bad CIDR), defence in depth |

The security-group-referencing-a-security-group idiom is the one to show:

```hcl
# REQ-AW-03: "ingest SG → MSK SG on 9092/9094 only"
resource "aws_security_group_rule" "ingest_to_msk" {
  type                     = "ingress"
  security_group_id        = aws_security_group.msk.id
  source_security_group_id = aws_security_group.ingest.id   # ← not a CIDR
  from_port                = 9092
  to_port                  = 9094
  protocol                 = "tcp"
  description              = "ingest-gateway producers only"
}
```

This survives autoscaling, IP churn and subnet changes, which a CIDR rule does not.

**Lambda in a VPC.** What it means: Lambda attaches ENIs in your subnets so the function can reach
private resources (RDS, an internal ALB, MSK). What it costs you:

| Implication | Detail |
|---|---|
| Cold start | Historically a multi-second penalty for ENI creation. AWS re-architected this with shared Hyperplane ENIs and the penalty is now much smaller. **Verify the current behaviour** — this is a frequently-outdated interview answer. |
| **No internet access by default** | A VPC Lambda has no public IP. To reach the public internet (or a public AWS API endpoint) it needs a NAT gateway or a VPC endpoint. This surprises people constantly: "my Lambda can't call the S3 API" usually means "it's in a VPC with no S3 endpoint and no NAT". |
| IP consumption | ENIs consume subnet IPs; high concurrency in a small subnet exhausts it. |
| Security group | The function needs one, and it must allow the *egress* it needs. |

The rule: **don't put a Lambda in a VPC unless it needs to reach something private.** A function that
only talks to S3, DynamoDB and SQS should stay outside the VPC and talk to the public service
endpoints over AWS's network — simpler, cheaper and faster. StreamPulse's compaction function only
touches S3 and SQS, so the design-only note in `docs/network.md` is that it would stay out of the VPC
unless the security requirement in BS-06 forces it in — at which point it needs a gateway endpoint for
S3 and an interface endpoint for SQS.

**Code:**

```hcl
# REQ-AW-03: gateway endpoints first — free, and they take the biggest traffic off NAT.
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = aws_vpc.this.id
  service_name      = "com.amazonaws.${var.region}.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = [for rt in aws_route_table.private : rt.id]

  # Endpoint policy: defence in depth. Even with a permissive IAM role,
  # traffic through THIS endpoint can only reach our buckets.
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = "*"
      Action    = ["s3:GetObject", "s3:PutObject", "s3:ListBucket"]
      Resource  = [aws_s3_bucket.raw.arn, "${aws_s3_bucket.raw.arn}/*",
                   aws_s3_bucket.archive.arn, "${aws_s3_bucket.archive.arn}/*"]
    }]
  })
}

resource "aws_vpc_endpoint" "dynamodb" {
  vpc_id            = aws_vpc.this.id
  service_name      = "com.amazonaws.${var.region}.dynamodb"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = [for rt in aws_route_table.private : rt.id]
}

# Interface endpoints: only the ones we actually use. Each has an hourly cost per AZ.
resource "aws_vpc_endpoint" "interface" {
  for_each            = toset(["sqs", "kinesis-streams", "sts", "secretsmanager"])
  vpc_id              = aws_vpc.this.id
  service_name        = "com.amazonaws.${var.region}.${each.value}"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = [for s in aws_subnet.private : s.id]
  security_group_ids  = [aws_security_group.endpoints.id]
  private_dns_enabled = true    # so the normal SDK hostname resolves to the ENI
}
```

**Follow-ups they'll ask:**
- *"How do you verify traffic actually went through the endpoint?"* — VPC Flow Logs, and by removing
  the NAT route and seeing that it still works. The second is the only test I'd trust.
- *"Can you restrict a bucket to only be reachable via your endpoint?"* — yes: a bucket policy with a
  `aws:SourceVpce` condition. That's how you make "no public internet access to this data" actually
  enforceable rather than aspirational.
- *"Why are NACLs stateless a problem?"* — outbound HTTPS from a private subnet leaves on port 443 and
  returns on an **ephemeral** port. If your NACL only allows inbound 443 you've blocked every reply.
  This is the single most common NACL mistake.
- *"What's `aws:PrincipalOrgID` for?"* — restricting a resource policy to principals in your AWS
  Organization; useful on S3 bucket policies and endpoint policies as a blast-radius control.

**Pitfalls / red flags:** "I'd add interface endpoints for everything" (expensive, and unnecessary for
S3/DynamoDB); not knowing security groups are stateful; not knowing a VPC Lambda loses internet access.

**Where in StreamPulse:** BS-06 / REQ-AW-03; `docs/network.md`.

**Verify:** current Lambda-in-VPC cold-start behaviour; interface endpoint pricing; the exact service
names for endpoints in your region.

---

## Q8. Kinesis Data Streams vs MSK vs SQS vs SNS vs EventBridge — how do you choose?

**30-second answer:** Ask two questions first: **do I need a replayable log, or a queue?** and **do I
need ordering?** A log (Kinesis, MSK/Kafka) keeps records for a retention period so multiple
independent consumers can read the same data at their own pace and replay it — that's what StreamPulse
needs, because the stream processor, the sink writer and the alert engine all consume the same raw
telemetry. A queue (SQS) deletes a message once it's processed, gives you no replay and no ordering on
standard queues, but scales effortlessly and has per-message retry and DLQ semantics. SNS is pub/sub
fan-out with no storage. EventBridge is a routing bus with content-based filtering and schema handling,
optimized for cross-service and cross-account events, not for throughput. For the StreamPulse hot path
it's Kafka/MSK because of partitions, consumer groups, replay and the Kafka Streams ecosystem; Kinesis
is the **comparison adapter** in Phase 10 precisely so the ADR can argue the trade-off with code
behind it.

**Deeper:**

### The decision table

| | Kinesis Data Streams | MSK (managed Kafka) | SQS | SNS | EventBridge |
|---|---|---|---|---|---|
| Model | log (shards) | log (partitions) | queue | pub/sub fan-out | event bus + rules |
| Replay | ✅ within retention | ✅ within retention | ❌ | ❌ | via **archive/replay** feature |
| Ordering | per shard | per partition | ❌ standard / per message group in FIFO | ❌ (FIFO topics differ) | ❌ |
| Multiple independent consumers | ✅ (shared 2 MB/s, or enhanced fan-out) | ✅ consumer groups | ❌ one consumer per message | ✅ per subscriber | ✅ per rule/target |
| Unit of parallelism | **shard** | **partition** | none (scale consumers freely) | n/a | n/a |
| Scaling | resharding (split/merge) — an operation | add partitions (can't reduce) | automatic | automatic | automatic |
| Retention | default ~24h, extendable (up to ~365 days) | configurable, disk-bound | up to ~14 days | none | archive is separate |
| Max payload | ~1 MB per record | broker-configurable (~1 MB default) | 256 KB (larger via S3 extended client) | 256 KB | ~256 KB |
| Cost shape | per shard-hour + per PUT payload unit (+ EFO per consumer-shard-hour) | per broker-hour + storage (you run the cluster) | per request | per publish + per delivery | per event published |
| Ops burden | low | **high** (brokers, ZK/KRaft, upgrades, rebalancing) | none | none | none |
| Ecosystem | AWS SDK / KCL | the entire Kafka ecosystem: Streams, Connect, Schema Registry | — | — | — |

**All numbers above are approximate and change. Verify against current AWS service quotas.**

### Shards vs partitions — the comparison the ADR needs (REQ in Phase 10)

They're conceptually the same thing: an ordered, independently-consumable slice of the stream, keyed by
a hash of the partition key. The operational differences:

| | Kinesis shard | Kafka partition |
|---|---|---|
| Write limit | ~1 MB/s **or** ~1,000 records/s | broker/disk bound, typically much higher |
| Read limit | ~2 MB/s **shared across all consumers**; `GetRecords` ~5 calls/s | consumers read independently at broker throughput |
| Dedicated per-consumer throughput | **enhanced fan-out**: ~2 MB/s *per consumer* per shard, push-based, extra cost | native — every consumer group reads independently |
| Change the count | split/merge shards (resharding), which changes the hash key ranges | add partitions (never reduce); changes key→partition mapping for new data |
| Consumer coordination | KCL with a DynamoDB lease table | Kafka consumer group protocol |

The **~2 MB/s shared read limit** is the one that decides architectures and the one to lead with.
StreamPulse has three independent consumers of `telemetry.raw.v1` (stream-processor, sink-writer, alert
engine). On Kinesis without enhanced fan-out, those three share 2 MB/s per shard — so a third consumer
doesn't just cost money, it *takes throughput away from the other two*. That's an architectural
constraint that doesn't exist in Kafka, and it's the strongest single argument in the ADR.

The other Kinesis gotcha: `GetRecords` is limited to roughly 5 calls per second per shard, so with
several polling consumers you hit **`ProvisionedThroughputExceededException`** on the read side long
before you hit the byte limit. Enhanced fan-out (a push/HTTP2 subscription) solves it and costs per
consumer-shard-hour.

**When I'd actually pick Kinesis over MSK:** small team with no Kafka operational experience; modest
and predictable throughput; deep integration with Lambda/Firehose/Analytics; you want zero broker
ops. **When MSK:** you need Kafka Streams or Connect (StreamPulse needs both — Debezium is a Kafka
Connect connector), you need many independent consumers, you need >7-day retention as a normal thing,
or you already have Kafka expertise. There's also **MSK Serverless**, which removes capacity planning
at a different cost shape — **verify its current limits and pricing**.

### SNS + SQS fan-out

The classic AWS pattern worth naming: publish once to an SNS topic, subscribe N SQS queues, each
consumer owns its queue. That gives you fan-out *plus* per-consumer buffering, retry and DLQ — the
closest SQS-world approximation to Kafka consumer groups. It still doesn't give you replay: once a
message is consumed from its queue it's gone. SNS **filter policies** let each subscriber receive only
matching messages, which is the SQS-world analogue of BS-07's RabbitMQ topic-exchange routing.

**Code:**

```java
/**
 * Phase 10: the Kinesis adapter exists as a Strategy behind the SAME port as Kafka,
 * selected by Spring profile (BS-01 REQ-LC-06). This is the hexagonal payoff —
 * swapping the broker does not touch business code, so the ADR can be written
 * from measurement rather than opinion.
 */
@Component
@Profile("kinesis")
class KinesisEventPublisher implements EventPublisher {

    private final KinesisAsyncClient kinesis;
    private final String streamName;

    @Override
    public CompletableFuture<Void> publish(TelemetryEvent e) {
        return kinesis.putRecord(r -> r
                    .streamName(streamName)
                    // Same key as Kafka: sessionId, so per-session ordering holds.
                    // Same hot-partition risk, too — a whale session skews one shard.
                    .partitionKey(e.sessionId())
                    .data(SdkBytes.fromByteArray(avro.serialize(e))))
                // NOTE: unlike Kafka's idempotent producer (enable.idempotence=true),
                // Kinesis has no producer-side dedup. A retried PutRecord CAN duplicate,
                // so the consumer MUST be idempotent. This asymmetry goes in the ADR.
                .thenApply(r -> null);
    }
}
```

**Follow-ups they'll ask:**
- *"How do you pick a partition key?"* — high cardinality, even distribution, and matching your
  ordering requirement. StreamPulse uses `sessionId`: per-session ordering is the only ordering the
  aggregations need, and sessions are numerous enough to spread evenly. `tenantId` would be a disaster
  — three tenants, one at 40% of traffic, so one partition gets 40% of the load. That's the
  hot-partition answer in one sentence.
- *"How do you handle a hot key anyway?"* — composite key (`sessionId#bucket`) if you can tolerate the
  reduced ordering scope, or route the whale separately. You cannot fix a hot key by adding partitions.
- *"Does SQS FIFO scale?"* — FIFO has substantially lower throughput than standard (historically ~300
  TPS, or ~3,000 with batching, and there's a high-throughput FIFO mode now — **verify current
  numbers**). Ordering is per *message group ID*, so throughput scales with group count, not total.
- *"Where does Firehose fit?"* — Kinesis Data Firehose is the managed "stream → S3/Redshift/OpenSearch"
  delivery pipe with built-in buffering, compression and format conversion. It would replace a big part
  of StreamPulse's sink-writer for the raw→S3 path, and arguably should — the reason to hand-write it
  here is that the *point* of the project is to demonstrate Spring Kafka consumer mechanics.

**Pitfalls / red flags:** "Kinesis is AWS's Kafka" with no mention of the shared read limit; not
knowing that Kafka partitions can't be reduced; picking `tenantId` as a partition key.

**Where in StreamPulse:** Phase 10 — the Kinesis adapter + "ADR: Kafka vs Kinesis (shards vs
partitions, retention, ordering, cost model)"; BS-06 REQ-AW-04 (MSK replication).

**Verify:** every quota in this answer. Kinesis per-shard read/write limits, `GetRecords` call rate,
SQS FIFO throughput, retention maximums, MSK Serverless limits.

---

## Q9. How do you design a DynamoDB table for this workload?

**30-second answer:** Start from the **access patterns**, never from an ER diagram — the schema is a
function of the queries, which is the inverse of how you'd approach Postgres. The partition key
determines physical placement and therefore throughput distribution, so it must be high cardinality and
evenly accessed; the sort key gives you range queries within a partition. For StreamPulse's aggregate
store the patterns are "give me metric M for tenant T over a time range" and "give me the top titles
for tenant T", so the table is `PK = TENANT#<t>#METRIC#<m>`, `SK = WINDOW#<iso8601>` — which makes the
time-range query a single `Query` with a `between` on the sort key, and the ISO-8601 timestamp sorts
lexicographically, which is why that format is chosen. The hot-partition risk is real and named:
`sportsnow` during a live final is one tenant driving a large share of writes, so I'd shard the key
suffix if measurement showed a problem.

**Deeper:**

### Partition key design and hot partitions

DynamoDB hashes the partition key to place the item. Every item with the same partition key lives in
one **item collection**, and an item collection has hard throughput ceilings — roughly **3,000 RCU and
1,000 WCU per partition** (**approximate, verify**). Exceed it and you get throttling *even if the
table has plenty of provisioned capacity*, because the capacity is per-partition, not just per-table.

**Adaptive capacity** mitigates this: DynamoDB automatically shifts capacity toward hot partitions and
can isolate a very hot item collection. It helps a lot, and it is not a licence to design a hot key —
it handles imbalance, not a single key receiving more than one partition's worth of traffic.

| Symptom | Cause | Fix |
|---|---|---|
| `ProvisionedThroughputExceededException` on a table with headroom | hot partition | increase key cardinality |
| One tenant dominating | low-cardinality PK | **write sharding**: `TENANT#sportsnow#METRIC#viewers#<0..N>` and scatter-gather on read |
| Time-series with a date PK | every write goes to today's partition | put the high-cardinality dimension in the PK and time in the SK — which is what StreamPulse does |

Write sharding is the standard answer and has a real cost: a read now has to query N shards and merge,
turning one `Query` into N. Choose N from measurement, and only shard the keys that need it.

### Single-table design

The idea: put multiple entity types in one table with generic key attributes (`PK`, `SK`), so that
related entities share a partition and can be fetched in **one query** rather than N. It genuinely
reduces round trips and is the right answer for tightly-coupled access patterns.

My honest position, which is what makes this a senior answer rather than a recital: **single-table
design is a real optimization with a real cost.** The cost is that the schema is unreadable without a
documented access-pattern table, refactoring is painful because the keys encode the query plan, and
new access patterns often need a new GSI or a migration. I'd use it where the entities are genuinely
queried together, and I'd cheerfully use separate tables where they aren't. "Always single-table" is
cargo cult.

For StreamPulse the plan (Phase 4) already specifies `PK=TENANT#t#METRIC#m`, `SK=WINDOW#ts` with a GSI
for title — that's a single-table shape for the aggregate store, and the alert history deliberately
goes elsewhere (MongoDB under the optional profile, BS-05 REQ-DL-08) because its access pattern is
different. That contrast is worth pointing out.

### The access-pattern table (do this before writing any key)

| # | Access pattern | Key | Index |
|---|---|---|---|
| 1 | Metric M for tenant T, time range | `PK=TENANT#t#METRIC#m`, `SK between WINDOW#from and WINDOW#to` | base table |
| 2 | Latest value of metric M for tenant T | same, `ScanIndexForward=false, Limit=1` | base table |
| 3 | All metrics for a title, time range | `GSI1PK=TITLE#x`, `GSI1SK=WINDOW#ts` | GSI1 |
| 4 | Idempotency claim by event ID | `PK=EVENT#<id>` | base table (separate table in practice) |

If a pattern isn't in this table, the design doesn't support it — and that's the point. Making the gap
visible before you ship is the whole value of the exercise.

**Code:**

```java
/**
 * Phase 4 sink-writer → DynamoDB. Three things to notice:
 *  1. ISO-8601 in the sort key so lexicographic order == chronological order.
 *  2. A conditional write for idempotency — safe to replay the whole Kafka batch.
 *  3. TTL for automatic expiry of hot-tier data (free deletes, no scan-and-delete job).
 */
void writeAggregate(MetricAggregate agg) {
    dynamo.putItem(r -> r
        .tableName("streampulse-metrics")
        .item(Map.of(
            "PK",       s("TENANT#%s#METRIC#%s".formatted(agg.tenantId(), agg.metric())),
            "SK",       s("WINDOW#%s".formatted(agg.windowStart())),      // 2026-09-28T14:30:00Z
            "GSI1PK",   s("TITLE#%s".formatted(agg.titleId())),           // access pattern 3
            "GSI1SK",   s("WINDOW#%s".formatted(agg.windowStart())),
            "value",    n(agg.value()),
            "producedAt", n(agg.producedAtEpochMs()),
            "ttl",      n(agg.windowStart().plus(30, DAYS).getEpochSecond())))
        // IDEMPOTENCY: only write if this is new, or if we have a NEWER computation
        // of the same window. Kafka redelivery (BS-01 REQ-LC-02) replays batches;
        // this makes that replay a no-op instead of a correctness problem.
        .conditionExpression("attribute_not_exists(PK) OR producedAt < :ts")
        .expressionAttributeValues(Map.of(":ts", n(agg.producedAtEpochMs()))));
}
```

**Follow-ups they'll ask:**
- *"Why not just use a scan?"* — a `Scan` reads the whole table and is billed for it. It's acceptable
  for a one-off migration or a tiny table and nothing else. If you find yourself needing a scan in a
  request path, your key design is wrong.
- *"What's the item size limit?"* — approximately 400 KB (**verify**). For anything larger, store the
  payload in S3 and keep a pointer in DynamoDB. StreamPulse's raw events go to S3 for exactly this
  reason; DynamoDB holds aggregates, which are small.
- *"How do you do a 'get everything for tenant T' query?"* — you don't, unless you designed for it.
  That's the honest limitation: DynamoDB gives you fast, predictable access to patterns you planned,
  and nothing else. If ad-hoc querying is a requirement, that's an argument for Postgres (or for
  exporting to S3 and querying with Athena — which is what the BS-06 compaction is for).

**Pitfalls / red flags:** designing the table like a relational schema and adding indexes later;
choosing a low-cardinality partition key; "single-table design" as dogma.

**Where in StreamPulse:** Phase 4; the ADR "Postgres vs DynamoDB vs Mongo for time-series aggregates
(access patterns first)"; BS-05 REQ-DL-08.

**Verify:** per-partition throughput limits, item size limit, adaptive capacity behaviour.

---

## Q10. DynamoDB: LSI vs GSI, capacity modes, transactions, Streams, global tables.

**30-second answer:** A **GSI** has its own partition and sort key, its own capacity, is **eventually
consistent only**, and can be created or dropped at any time — that's the one you reach for. An **LSI**
shares the base table's partition key with a different sort key, shares the table's capacity, supports
**strongly consistent reads**, and — the killer constraint — **must be created with the table and can
never be added**. It also caps an item collection at approximately 10 GB. So LSIs are rare in practice.
On capacity: **on-demand** for unpredictable or spiky traffic with no capacity planning, **provisioned
+ auto-scaling** when the load is predictable enough that the lower per-request price wins.
**Conditional writes** are the atomic compare-and-set that makes idempotent consumers possible.
**Streams** give you a change log for CDC. **Global tables** give multi-region multi-active writes with
**last-writer-wins** conflict resolution — which is a real semantic you have to design around, not a
detail.

**Deeper:**

| | LSI | GSI |
|---|---|---|
| Partition key | **same as base table** | any attribute |
| Sort key | different | any attribute |
| Created | **only at table creation** | any time, backfilled online |
| Capacity | shares the table's | **its own** |
| Consistency | strong reads available | **eventually consistent only** |
| Item collection size limit | ~10 GB per partition key | none |
| Projection | KEYS_ONLY / INCLUDE / ALL | KEYS_ONLY / INCLUDE / ALL |

Two GSI traps worth naming:

1. **A GSI can throttle the base table.** If a GSI's write capacity can't keep up with the base table's
   write rate, writes to the *base table* get throttled. So an under-provisioned GSI takes down your
   writes. (Behaviour differs between capacity modes — **verify current semantics.**)
2. **Projection is a cost decision.** `ALL` duplicates every attribute into the index — double storage,
   double write cost. `KEYS_ONLY` is cheap but forces a second read against the base table for the
   attributes. Project exactly the attributes the query needs.

**Capacity modes:**

| | On-demand | Provisioned + auto-scaling |
|---|---|---|
| Pricing | per request, higher unit price | per hour of provisioned capacity, lower unit price |
| Planning | none | forecast + auto-scaling policy |
| Spikes | absorbs instantly (subject to previous peak scaling behaviour — **verify**) | auto-scaling reacts on a delay of minutes, so a sharp spike throttles |
| Best for | new/unknown workloads, spiky (the "live match kickoff" 10× spike in BS-03) | steady, predictable, high volume |

For StreamPulse's aggregate writes I'd start on-demand (unknown load, spiky by nature), measure in
Phase 9, and only move to provisioned if the steady-state volume makes the price difference material.
That's the honest sequence: on-demand is a cost of *not knowing*, and you pay it until you know.

**Conditional writes** are the primitive behind idempotency and optimistic locking (see the Q9 code and
the Q4 claim). `TransactWriteItems` extends that to multiple items **atomically across tables** —
all-or-nothing, with a condition per item. The costs: it consumes roughly **double** the capacity of
the equivalent non-transactional writes, it's limited to around 100 items per transaction
(**verify — this limit was raised from 25**), and it will fail outright on a conflict rather than
waiting, so the caller must retry. Use it where atomicity is genuinely required (a quota transfer —
which is exactly BS-02 REQ-CC-07's problem shape); don't use it as a default.

**DynamoDB Streams**: an ordered change log per partition key, ~24h retention, with configurable view
types (`KEYS_ONLY`, `NEW_IMAGE`, `OLD_IMAGE`, `NEW_AND_OLD_IMAGES`). Consumed by Lambda (a poll source,
Q3) or KCL. It's the DynamoDB analogue of the Debezium CDC pipeline StreamPulse builds on Postgres
(Phase 6) — and it's a much cleaner story than dual-writing, for the same reason: the change log *is*
the transaction.

**Global tables**: multi-region, **multi-active** — you can write in any region and DynamoDB replicates
asynchronously. Conflict resolution is **last-writer-wins by timestamp**. That's the sentence to think
hard about:

- Two regions update the same item within the replication window → one update is **silently discarded**.
- There is no merge, no vector clock, no application callback.
- So global tables are safe when writes are **naturally partitioned by region** (each tenant is served
  from one region) and dangerous when the same item is genuinely written from two regions.

Replication lag is typically sub-second but is **not** guaranteed, and that lag is your RPO for
DynamoDB data (Q12).

**Code:**

```java
/**
 * TransactWriteItems for BS-02 REQ-CC-07's quota transfer.
 * The Java-side version of that lab uses in-process locks; this is what the
 * same invariant looks like when the state lives in DynamoDB — and it is a
 * much better answer than distributed locking.
 */
void transferQuota(String from, String to, long amount) {
    dynamo.transactWriteItems(r -> r.transactItems(
        // Debit — conditional on sufficient balance. Atomicity means we cannot
        // debit without crediting, so the total is conserved (REQ-CC-08's invariant).
        TransactWriteItem.builder().update(u -> u
                .tableName("quota")
                .key(Map.of("tenant", s(from)))
                .updateExpression("SET remaining = remaining - :amt")
                .conditionExpression("remaining >= :amt")
                .expressionAttributeValues(Map.of(":amt", n(amount)))).build(),
        // Credit
        TransactWriteItem.builder().update(u -> u
                .tableName("quota")
                .key(Map.of("tenant", s(to)))
                .updateExpression("SET remaining = remaining + :amt")
                .expressionAttributeValues(Map.of(":amt", n(amount)))).build()));
    // No lock ordering needed, so no deadlock possible — the v1 bug in REQ-CC-07
    // cannot be expressed here. Cost: ~2x WCU, and TransactionCanceledException
    // on conflict, which the caller must retry with backoff.
}
```

**Follow-ups they'll ask:**
- *"Strongly vs eventually consistent reads?"* — eventually consistent is the default and costs half
  as much; strongly consistent reads the leader replica, costs double, is unavailable on GSIs, and has
  slightly higher latency. Default to eventual and opt in where a read-your-own-write matters.
- *"What's DAX?"* — a managed write-through cache in front of DynamoDB for microsecond reads. It's a
  fit for a read-heavy hot key; it's also another cluster to run, and StreamPulse already has Redis in
  front for the hot tier, so it would be redundant here.
- *"How do you do a global secondary index backfill safely?"* — creating a GSI on a large table is an
  online operation but consumes write capacity and takes time; on provisioned capacity, that can
  throttle production writes. Create it during low traffic, or temporarily raise capacity.
- *"How would you migrate a key schema?"* — you can't change keys in place. Create a new table,
  dual-write, backfill, cut reads over, stop dual-writing. Which is the same strangler-fig shape as
  BS-08, and worth saying so.

**Pitfalls / red flags:** thinking a GSI is strongly consistent; using transactions everywhere; treating
global tables' last-writer-wins as a solved problem.

**Where in StreamPulse:** Phase 4 (single-table aggregates, conditional writes), BS-06 REQ-AW-07
(global tables as the production cross-region approach, design-only).

**Verify:** transaction item limit, capacity-mode spike behaviour, GSI throttling semantics, Streams
retention.

---

## Q11. S3: the small-files problem, prefixes, storage classes, events, consistency.

**30-second answer:** The **small-files problem** is BS-06's whole reason for existing: sink-writer
flushes on a size-or-time trigger, so a low-volume tenant produces thousands of tiny objects per hour.
Every object costs a `GET` request when you query it, and an Athena or Spark scan over 10,000 small
files is dominated by per-object request latency, not by bytes — so the query is slow *and* the request
bill is large. The fix is **compaction**: merge an hour's part-files into one larger object once the
hour is closed, which is exactly REQ-AW-01. On **prefixes**: S3 scales request rate per partitioned
prefix (roughly 3,500 writes and 5,500 reads per second per prefix, **verify current numbers**), and
partitioning is automatic now, so the old "add a random hash prefix" advice is largely obsolete —
though a prefix scheme that puts all traffic on one path can still concentrate load. And S3 has had
**strong read-after-write consistency** for all operations since late 2020, which retires a whole
generation of workarounds.

**Deeper:**

### The small-files economics

| | 10,000 × 100 KB objects | 1 × 1 GB object |
|---|---|---|
| Bytes stored | ~1 GB | ~1 GB |
| GET requests to read it all | 10,000 | 1 |
| Athena/Spark scan time | dominated by request round-trips | dominated by throughput |
| Storage-class transition eligibility | tiny objects may cost more to transition than they save | fine |

There's a subtlety worth knowing: some storage classes have a **minimum billable object size** and a
**minimum storage duration**, so archiving many tiny objects to a colder class can cost *more* than
leaving them in Standard. That makes compaction a prerequisite for a sensible lifecycle policy, not
just a query optimization. **Verify the current minimums per class.**

### Prefix layout

StreamPulse uses `tenant=/dt=/hour=` (Phase 4), which is **Hive-style partitioning** — Athena and Spark
recognize it and can do **partition pruning**, so `WHERE dt='2026-09-28'` reads only that prefix
instead of the whole bucket. That's usually a bigger win than the raw request-rate concern.

```
s3://streampulse-raw/
  raw/tenant=sportsnow/dt=2026-09-28/hour=14/part-0000.ndjson.gz     ← many small (the problem)
  ...
s3://streampulse-archive/
  compacted/tenant=sportsnow/dt=2026-09-28/hour=14.ndjson.gz          ← one per hour (the fix)
  compacted/tenant=sportsnow/dt=2026-09-28/hour=14.ndjson.gz.manifest.json
```

The plan lists **Parquet as a stretch goal**, and it's worth explaining why: Parquet is columnar, so a
query selecting 3 of 20 fields reads roughly 3/20 of the bytes, and it carries per-column statistics
that let the engine skip row groups entirely. For analytics it's a much bigger win than compaction
alone. The trade-off is that Parquet is a batch-oriented format — you can't append a row — so it pairs
naturally with a compaction step and not with a streaming writer.

### Storage classes

| Class | Use | Notes |
|---|---|---|
| Standard | hot, frequently accessed | the default |
| Intelligent-Tiering | unknown/changing access patterns | small monitoring fee per object; automatic movement, no retrieval fee for the main tiers |
| Standard-IA / One Zone-IA | known-infrequent | retrieval fee; minimum duration; One Zone has a single-AZ durability profile |
| Glacier Instant / Flexible / Deep Archive | archive | increasing retrieval latency (ms → minutes/hours → hours) and decreasing cost |
| Express One Zone | very low latency, single AZ | newer; **verify availability and pricing** |

For StreamPulse the lifecycle policy writes itself: raw part-files expire after compaction (they're
redundant), compacted files go Standard → Standard-IA at 30 days → Glacier at 90, driven by the actual
query pattern (dashboards query the last few days; billing queries the last month; audits query years).

### Event notifications and consistency

Event notification targets: SQS, SNS, Lambda, and EventBridge. StreamPulse routes **S3 → SQS → Lambda**
(REQ-AW-01) for the retry/DLQ/buffering reasons in Q3. Note that notifications are **at-least-once**
and are **not guaranteed ordered**, which is the other half of why the compaction function must be
idempotent.

**Strong read-after-write consistency** (since Dec 2020) applies to PUTs of new objects, overwrites and
deletes, and to list operations. What it does **not** give you: atomicity across multiple objects. The
compaction function writes the data object and *then* the manifest precisely because there's no
transaction — the manifest is the commit marker, and readers that require completeness read the
manifest first.

**Code:**

```hcl
# BS-06 REQ-AW-01 wiring: S3 → SQS → Lambda, plus lifecycle (Phase 10)
resource "aws_s3_bucket_notification" "raw" {
  bucket = aws_s3_bucket.raw.id
  queue {
    queue_arn     = aws_sqs_queue.compaction.arn
    events        = ["s3:ObjectCreated:*"]
    filter_prefix = "raw/"
    filter_suffix = ".ndjson.gz"
  }
}

resource "aws_s3_bucket_lifecycle_configuration" "archive" {
  bucket = aws_s3_bucket.archive.id

  rule {
    id     = "compacted-tiering"
    status = "Enabled"
    filter { prefix = "compacted/" }
    transition { days = 30  storage_class = "STANDARD_IA" }
    transition { days = 90  storage_class = "GLACIER_IR" }
    expiration { days = 2555 }                       # ~7y retention policy
  }

  rule {
    # Raw part-files are redundant once compacted; keep a grace window for re-runs.
    id     = "expire-raw-parts"
    status = "Enabled"
    filter { prefix = "raw/" }
    expiration { days = 7 }
  }

  rule {
    # Never forget this one: failed multipart uploads accumulate invisibly
    # and are billed as storage forever.
    id     = "abort-incomplete-multipart"
    status = "Enabled"
    abort_incomplete_multipart_upload { days_after_initiation = 3 }
  }
}
```

**Follow-ups they'll ask:**
- *"How does sink-writer decide when to flush?"* — size-or-time, whichever comes first, which is
  exactly what produces small files for low-volume tenants. Raising the time trigger improves file size
  and *worsens* freshness — a direct trade-off, and the reason compaction exists as a separate step
  rather than as "just buffer longer".
- *"Multipart upload?"* — required above ~5 GB and a good idea well below that for parallelism and
  resumability. And always set the abort-incomplete lifecycle rule (above), or you pay for the debris.
- *"How do you make the bucket private properly?"* — Block Public Access at the account level, bucket
  policy denying non-TLS (`aws:SecureTransport`), SSE-KMS with a CMK, and a `aws:SourceVpce` condition
  if it should only be reachable through the endpoint (Q7).
- *"What about S3 Tables / Iceberg?"* — table formats over object storage that add ACID commits,
  schema evolution and compaction as a managed concern. That's the modern answer to the small-files
  problem for analytics. **Verify the current offerings and their status** — this area moves fast.

**Pitfalls / red flags:** still quoting the old eventual-consistency behaviour; recommending random
hash prefixes as if it were 2015; no lifecycle rule for incomplete multipart uploads.

**Where in StreamPulse:** BS-06 / REQ-AW-01; Phase 4 (the S3 NDJSON writer that creates the problem),
Phase 10 (the Lambda that fixes it).

**Verify:** per-prefix request rates, storage class minimum sizes/durations, Express One Zone /
S3 Tables availability.

---

## Q12. Cross-region: define RPO and RTO, and tell me what makes a failover credible.

**30-second answer:** **RPO — Recovery Point Objective — is how much data you can afford to lose**,
measured in time: an RPO of 1 minute means that after a disaster you may have lost up to a minute of
writes. **RTO — Recovery Time Objective — is how long you can afford to be down** before service is
restored. They're independent: you can have a 0-second RPO with a 4-hour RTO (synchronous replication,
manual failover) or a 5-minute RPO with a 60-second RTO (async replication, automated failover). Both
are **business requirements**, not technical ones, and the architecture follows from them. What makes a
failover credible is not the design — it's the **drill**: a documented, scheduled exercise where you
actually fail over, measure the real RPO and RTO, and find the thing nobody thought of. A DR plan that
has never been executed is a document, not a capability.

**Deeper:**

```
        disaster
           │
  ◀────────┤────────▶
    RPO    │   RTO
 (data loss│(downtime
  window)  │ window)
           │
last good  │        service
 replica   │        restored
```

| Strategy | RPO | RTO | Cost | StreamPulse fit |
|---|---|---|---|---|
| Backup & restore | hours | hours–days | lowest | not acceptable for `sportsnow` |
| Pilot light | minutes | ~tens of minutes | low | possible |
| Warm standby (scaled-down full stack running) | seconds–minutes | minutes | medium | **the fit for RPO ≤1min / RTO ≤15min** |
| Active-active (multi-region serving) | near-zero | near-zero | highest | overkill for the stated targets |

BS-06's targets (RPO ≤ 1 min, RTO ≤ 15 min — **project targets, not industry facts**) point at warm
standby: region B runs a scaled-down stack continuously, data replicates asynchronously, and failover
is a DNS/config change plus a scale-up.

### Active-passive vs active-active

| | Active-passive | Active-active |
|---|---|---|
| Writes | one region | both regions |
| Conflict resolution | **not needed** | **required** — LWW, CRDTs, or partition by key |
| Cost | standby capacity | full capacity ×2, always |
| Failover | a switch (DNS, config) | none — you just lose capacity |
| Hidden cost | the standby path is **rarely exercised**, so it rots | operational complexity everywhere |
| Data consistency | simple | the hard part |

The "standby rots" point is the practical one. A passive region that never serves traffic accumulates
config drift, expired certificates, stale AMIs and IAM policies that nobody noticed were wrong. The
mitigation is either regular drills or deliberately serving a small share of real traffic from the
standby — which turns it into a weak active-active and is usually the better engineering choice.

### Per-datastore replication, and what each gives you

| Data | Mechanism | Replication semantics | RPO driver |
|---|---|---|---|
| Kafka topics | **MSK Replicator** or **MirrorMaker 2** | async; offsets are *translated*, not identical | replication lag |
| DynamoDB | **global tables** | async multi-active, **last-writer-wins** | replication lag (typically sub-second, not guaranteed) |
| S3 | **Cross-Region Replication (CRR)** | async, per-object; **only replicates objects created after CRR is enabled** unless you run batch replication | replication lag |
| Postgres (RDS/Aurora) | cross-region read replica / Aurora Global Database | async | replica lag |
| Secrets/config | Secrets Manager replication, or Terraform applied to both | — | your pipeline |

**The MirrorMaker 2 offset problem** is the specific hard part for StreamPulse and worth being precise
about: MM2 replicates records, but a record at offset 1,000 in region A is **not** at offset 1,000 in
region B — the partitions have independent offset sequences. MM2 emits `checkpoint` records that map
consumer-group offsets between clusters, and consumers use `RemoteClusterUtils` (or the
`MirrorCheckpointConnector`'s offset-sync topic) to translate. The translation is **approximate** —
it maps to a *safe* offset, which in practice means you may **reprocess** some records after failover.
Which is fine, precisely because StreamPulse's consumers are idempotent (BS-01 REQ-LC-02, BS-03
REQ-KT-05). That's the connective point: **at-least-once plus idempotency is what makes regional
failover survivable at all.** **Verify MM2's current checkpoint/offset-sync behaviour** — it has
changed across Kafka versions, and there's also MSK Replicator as the managed alternative.

### What makes a drill credible

A drill that only proves the happy path is theatre. The checklist I'd want:

1. **Scheduled and repeated** — quarterly, on the calendar, not "when we get time".
2. **Measures, not asserts** — record actual RPO (data lag at cutover) and actual RTO (time to first
   successful request), and write them next to the targets.
3. **Includes the failback** — failing over is half the exercise; getting back without losing the
   writes that happened in region B is the half people skip, and it's harder.
4. **Includes the people** — who declares the disaster? Who has the IAM permission to do it at 3am? Is
   the runbook accessible if the wiki is in the failed region?
5. **Breaks something unexpected** — a drill where you announce the time and warm everything up
   measures your preparation, not your resilience.
6. **Produces action items** — a drill that found nothing was probably too gentle.

StreamPulse's version (REQ-AW-05) is honest about being a simulation: two local Kafka clusters with MM2
under compose profile `multiregion`, stop `region-a`, point query-api at `region-b` via a profile flag,
measure data lag as an RPO proxy and time-to-serve as an RTO proxy, and write the numbers into
`docs/dr-drill.md`. It cannot prove anything about real AWS regional failure, and the doc should say so.

**Code:**

```properties
# MirrorMaker 2 — BS-06 REQ-AW-04 (compose profile `multiregion`, Phase 10)
clusters = region-a, region-b
region-a.bootstrap.servers = kafka-a:9092
region-b.bootstrap.servers = kafka-b:9092

# Active-passive: replicate A → B only.
region-a->region-b.enabled = true
region-a->region-b.topics  = telemetry\.agg\.v1|config\.changes\.v1

# Replicate consumer group offsets so a failed-over consumer resumes near where it was.
# "near", not "at" — see the offset translation note above.
region-a->region-b.emit.checkpoints.enabled = true
region-a->region-b.sync.group.offsets.enabled = true
emit.checkpoints.interval.seconds = 5     # smaller interval → smaller RPO on offsets

replication.factor = 1                    # local demo only; 3 in any real cluster
```

**Follow-ups they'll ask:**
- *"What's your actual RPO with async replication?"* — the replication lag at the moment of failure,
  which you should be *measuring continuously* as a metric, not estimating. `MaxOffsetLag` for MM2,
  `ReplicationLatency` for DynamoDB global tables, replica lag for RDS. If you can't graph it, you
  don't know your RPO. That's the answer that lands.
- *"Can you get RPO = 0?"* — only with synchronous cross-region replication, which means every write
  pays the inter-region round-trip latency (tens of milliseconds) — and if the remote region is
  unreachable, you either block writes or you weren't synchronous. That's a CAP trade-off stated
  concretely, and for a 50k events/s telemetry pipeline it's not a sane choice.
- *"How does DNS failover work?"* — Route 53 health checks with failover routing, or a weighted policy
  you shift. The gotcha is **TTL**: clients cache DNS, so a 300-second TTL puts a floor under your RTO.
  Lower TTLs before a planned failover; keep them permanently low if RTO matters, and accept the extra
  query volume.
- *"What about the control plane?"* — Terraform state, CI/CD, secrets, container registry. If your
  deployment pipeline lives only in the failed region, you can fail over the data plane and then be
  unable to deploy a fix. This is the thing drills find.

**Pitfalls / red flags:** using RPO and RTO interchangeably; "we have a DR plan" with no drill;
proposing active-active without a word about write conflicts.

**Where in StreamPulse:** BS-06 / REQ-AW-04, REQ-AW-05, REQ-AW-07; `docs/dr-drill.md`; Phase 10.

**Verify:** MM2 checkpoint/offset-translation behaviour for your Kafka version; MSK Replicator as the
managed alternative; DynamoDB global table replication lag SLAs (if any).

---

## Q13. Terraform: state, modules, CI, drift, environments.

**30-second answer:** Terraform's **state file** is the map from your configuration to real resource
IDs — it's the whole product, and it's why the first thing I set up is a **remote backend with
locking** (S3 for the state, with a lock mechanism) so two engineers or two CI jobs can't apply
simultaneously and corrupt it. State contains secrets in plaintext, so the bucket is encrypted,
versioned and tightly IAM-scoped. In CI I split **plan and apply**: the plan runs on the PR and is
posted as a comment for review, `terraform plan -out=tfplan` saves the exact plan, and apply runs that
saved plan on merge — so what was reviewed is what gets applied. **Modules** are the reuse unit and
they should be small and composable rather than one god-module per environment. For environments I use
**separate directories with separate state**, not workspaces — because workspaces share one
configuration, which breaks down the moment prod needs something dev doesn't.

**Deeper:**

### State

| Concern | Answer |
|---|---|
| What's in it | resource IDs, attributes, dependency graph, **and secrets in plaintext** |
| Where | S3 (versioned, encrypted, block public access) or Terraform Cloud/Enterprise |
| Locking | historically a DynamoDB lock table; recent Terraform versions support **S3-native locking** (`use_lockfile`) — **verify the version and the current recommendation** |
| Why locking | concurrent applies produce a corrupted state and orphaned resources |
| Secrets | treat the state bucket as a secret store; better still, don't put secrets in Terraform at all — reference them from Secrets Manager/SSM by ARN |
| Blast radius | **split state**: network, data, and application layers in separate states, so an app change can't plan a VPC destroy |

That last row is the one that saves you. One state file for an entire environment means every `apply`
has the power to destroy everything, and a bad refresh can propose exactly that. Splitting state by
lifecycle (things that change hourly vs things that change yearly) is the single highest-value
structural decision.

### Workspaces vs directories

| | Workspaces | Directory per environment |
|---|---|---|
| Config | **one** configuration, N states | N configurations (usually thin, calling shared modules) |
| Divergence between envs | awkward — `count = terraform.workspace == "prod" ? 3 : 1` everywhere | natural |
| Risk of applying to the wrong env | **high** — one forgotten `terraform workspace select` | low — you're in the directory |
| Good for | short-lived identical copies (per-PR ephemeral envs, feature branches) | **long-lived environments** |

```
infra/
  modules/                # reusable, versioned, no environment knowledge
    vpc/  kinesis/  dynamodb/  s3/  sqs/  iam/
  envs/
    local/  main.tf  backend.tf  terraform.tfvars   # moto/DynamoDB Local endpoints
    dev/    main.tf  backend.tf  terraform.tfvars
    prod/   main.tf  backend.tf  terraform.tfvars   # plan-only in this project
```

This is REQ-AW-06's "modules + remote-state pattern, per-env tfvars".

### Drift

Drift is reality diverging from state — someone clicked in the console, or a service changed a default.
Detection: `terraform plan -refresh-only` (or a scheduled `plan` in CI that fails if the diff is
non-empty). Handling:

- **Adopt the change** — `terraform apply -refresh-only` to accept it into state, then codify it.
- **Revert it** — a normal `apply` puts reality back.
- **Prevent it** — IAM that denies console writes in prod. This is the real fix; detection is a
  consolation prize.

Related tools worth naming: `terraform import` (and, in newer versions, declarative `import` blocks) to
bring an existing resource under management; `moved` blocks to rename resources without destroy/create;
`removed` blocks to drop a resource from state without destroying it. **Verify which of these exist in
your Terraform version** — they're recent additions.

**Code:**

```hcl
# infra/envs/dev/backend.tf — REQ-AW-06
terraform {
  required_version = "~> 1.9"
  backend "s3" {
    bucket         = "streampulse-tfstate-dev"
    key            = "app/terraform.tfstate"    # per-layer key: network/, data/, app/
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "streampulse-tflock"       # verify: S3-native locking may replace this
  }
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }   # PIN the major version
  }
}
```

```yaml
# .github/workflows/terraform.yml — plan on PR, apply the SAVED plan on merge (REQ-AW-06)
jobs:
  plan:
    steps:
      - run: terraform init -backend-config=envs/${{ matrix.env }}/backend.tf
      - run: terraform fmt -check -recursive        # style, fails fast
      - run: terraform validate                     # syntax + provider schema
      - run: tflint --recursive                     # lint rules (REQ-AW-06 names tflint)
      - run: tfsec .                                # or checkov — security policy
      # -detailed-exitcode: 0 = no change, 1 = error, 2 = changes present
      - run: terraform plan -out=tfplan -detailed-exitcode
      - uses: actions/upload-artifact@v4            # the plan is the reviewed artifact
        with: { name: tfplan-${{ matrix.env }}, path: tfplan }

  apply:
    needs: plan
    if: github.ref == 'refs/heads/main'
    environment: ${{ matrix.env }}                  # GitHub environment = manual approval gate
    steps:
      - uses: actions/download-artifact@v4
        with: { name: tfplan-${{ matrix.env }} }
      # Apply the EXACT plan that was reviewed. Re-planning here would apply
      # something nobody looked at.
      - run: terraform apply -auto-approve tfplan
```

**Follow-ups they'll ask:**
- *"Why pin the provider version?"* — provider upgrades change resource schemas and defaults, and an
  unpinned provider means your Friday apply is not the same Terraform as your Monday plan. Pin, and
  upgrade deliberately with a plan you read.
- *"`count` vs `for_each`?"* — `for_each` almost always. `count` indexes resources by position, so
  removing the *first* item in a list re-creates everything after it. `for_each` keys by a stable
  string, so removals are surgical. This has caused real production destroys.
- *"How do you handle secrets?"* — don't put them in Terraform. Create the Secrets Manager secret
  *container* in Terraform and populate the value out of band, or reference an existing secret by ARN.
  A secret passed as a Terraform variable ends up in the state file and often in CI logs.
- *"Terraform vs CDK/Pulumi/CloudFormation?"* — Terraform for multi-cloud and the provider ecosystem
  (and because the declarative HCL is genuinely easier to review in a PR than generated code); CDK when
  the team is strongly TypeScript/Java and wants abstraction; CloudFormation when you need
  AWS-native features like StackSets or Service Catalog. The review-a-plan workflow is the thing I'd
  refuse to give up.
- *"What about `terraform destroy` in CI?"* — only for ephemeral per-PR environments, and even then
  behind a guard. Never wired to prod.

**Pitfalls / red flags:** local state; no locking; `terraform apply` directly from a laptop against
prod; one giant state file; using `count` for named resources.

**Where in StreamPulse:** BS-06 / REQ-AW-06; Phase 10; `infra/modules/{kinesis,dynamodb,s3,sqs,iam}`
applied against moto, with `msk`/`vpc`/`eks` as `validate`/`plan`-only.

**Verify:** S3-native state locking availability and version; `import`/`moved`/`removed` block support;
current AWS provider major version.

---

## Q14. How does StreamPulse test AWS locally, and what can't it prove?

*(The applied question — and the one where honesty is the whole point.)*

**30-second answer:** Terraform's AWS provider lets you override the **service endpoints**, so I point
S3, SQS and Kinesis at a local **moto server** and DynamoDB at **DynamoDB Local**, disable the
credential/account checks, and run a real `terraform apply` against them. The same trick works for the
application: the AWS SDK v2 takes an `endpointOverride`, selected by the `local` Spring profile
(BS-01 REQ-LC-06). That gets me real Terraform HCL, a real plan/apply cycle, real SDK calls and real
integration tests, on a Mac, for free — which is the whole reason the plan chose this over paid
LocalStack. What it explicitly **cannot** prove is anything below the API layer: IAM policy evaluation,
VPC routing and security-group enforcement, cross-AZ behaviour, throttling and quota behaviour, or
real service latency. Those are **API-level mocks**, and `docs/network.md` says so in plain language
rather than implying the network design was tested.

**Deeper:**

### The mechanism

```hcl
# infra/envs/local/provider.tf — the endpoint-override pattern (reused from your Go repos)
provider "aws" {
  region                      = "us-east-1"
  access_key                  = "test"           # moto accepts anything
  secret_key                  = "test"

  # These three skips are mandatory: moto cannot answer STS/IMDS/account-id calls.
  skip_credentials_validation = true
  skip_metadata_api_check     = true
  skip_requesting_account_id  = true
  s3_use_path_style           = true             # moto serves path-style, not virtual-host

  endpoints {
    s3       = "http://localhost:5000"           # moto server
    sqs      = "http://localhost:5000"
    kinesis  = "http://localhost:5000"
    iam      = "http://localhost:5000"
    dynamodb = "http://localhost:8000"           # DynamoDB Local (official AWS image)
  }
}
```

```java
/**
 * The application side, selected by the `local` profile (BS-01 REQ-LC-06).
 * `aws` profile omits endpointOverride entirely and uses the default resolver
 * plus the IAM role credential chain — so the SAME artifact runs in both.
 */
@Configuration
@Profile("local")
class LocalAwsConfig {

    @Bean
    S3Client s3Client(@Value("${streampulse.aws.s3-endpoint}") URI endpoint) {
        return S3Client.builder()
                .endpointOverride(endpoint)
                .credentialsProvider(StaticCredentialsProvider.create(
                        AwsBasicCredentials.create("test", "test")))
                .region(Region.US_EAST_1)
                // moto serves path-style URLs; virtual-host style would try to
                // resolve bucket.localhost and fail.
                .serviceConfiguration(S3Configuration.builder().pathStyleAccessEnabled(true).build())
                .build();
    }
}
```

```java
/**
 * The integration test (BS-06 acceptance test `CompactionLambdaIT`).
 * Testcontainers starts moto; the function runs in-process.
 */
@Testcontainers
class CompactionLambdaIT {

    @Container
    static final GenericContainer<?> moto = new GenericContainer<>("motoserver/moto:latest")
            .withExposedPorts(5000);

    @Test
    void compactionIsIdempotent() {
        seedParts(3);                                  // 3 small part-files in raw/
        var first  = compaction.compact("sportsnow", DATE, 14);
        var second = compaction.compact("sportsnow", DATE, 14);   // re-run

        // REQ-AW-01: "re-running produces the same output"
        assertThat(second.key()).isEqualTo(first.key());
        assertThat(sha256(readObject(first.key()))).isEqualTo(sha256(readObject(second.key())));
        assertThat(listObjects("compacted/")).hasSize(2);          // data + manifest, not 4
    }
}
```

### The honest limitations table

This is the part to volunteer in an interview, because claiming more than you can prove is what gets
caught on the third follow-up.

| Aspect | Locally provable? | What I'd say |
|---|---|---|
| Terraform HCL correctness, module structure, `plan`/`apply` cycle | ✅ yes | Real Terraform, real state, real dependency graph. |
| SDK usage, retries, pagination, conditional writes | ✅ yes | Real SDK against a real-enough API. |
| DynamoDB query/index semantics | ✅ mostly (DynamoDB Local is the official implementation) | Good fidelity; capacity/throttling behaviour is not simulated. |
| S3 object lifecycle of *my* code (keys, manifests, idempotency) | ✅ yes | That's what `CompactionLambdaIT` proves. |
| **IAM policy evaluation** | ❌ no | moto generally accepts any credential. An IAM policy that would deny in production passes here. |
| **VPC routing, security groups, NACLs, endpoints** | ❌ no | Created as API objects; **nothing enforces them**. REQ-AW-03 says this explicitly. |
| **Throttling, quotas, backpressure from the service** | ❌ no | The BS-04 DynamoDB-throttling scenario is simulated with Toxiproxy/stubs, not by real throttling. |
| **Real latency, cross-AZ, regional failure** | ❌ no | The MM2 drill is two local clusters — a *mechanism* test, not a regional failure test. |
| **Lambda runtime behaviour (cold start, freeze/thaw)** | ⚠️ partially | The plan flags that moto-Lambda needs Docker-in-Docker and may not work on Mac, with `sam local invoke` as the fallback. Neither reproduces real cold-start numbers. |
| **S3 event notification delivery semantics** | ⚠️ partially | moto fires events; it doesn't reproduce at-least-once delivery or ordering behaviour. |

### How I'd phrase this in the room

> "I built the Terraform modules and ran a real apply against moto and DynamoDB Local using provider
> endpoint overrides, so the HCL, the module boundaries and the SDK integration are genuinely tested
> and the integration tests run in CI on a free runner. What that does **not** test is IAM evaluation
> or network isolation — those are API-level mocks. So for the VPC design I'd want a real account and
> a policy-simulator run before I'd claim it's correct, and `docs/network.md` says exactly that. I'd
> rather tell you the boundary than have you find it."

That framing is worth more than a claim of full-fidelity local AWS, because every experienced
interviewer knows local AWS emulation has boundaries, and the question is whether you know where they
are.

**Follow-ups they'll ask:**
- *"Why not LocalStack?"* — the paid tiers cover more services and higher fidelity, but the project
  constraint was a free local stack, and moto plus DynamoDB Local covers S3, SQS, Kinesis, IAM objects
  and DynamoDB, which is the surface this project uses. It's a deliberate scope decision, not an
  oversight.
- *"How would you test IAM for real?"* — the IAM policy simulator, plus a dedicated sandbox AWS account
  where the CI role has *only* the policy under test and a suite that asserts the denies as well as the
  allows. Testing that something is denied is the half people skip.
- *"How do you keep local and real config from diverging?"* — one set of Terraform modules, different
  `tfvars` and provider blocks per environment, plus `terraform validate`/`plan` against the real
  provider schema in CI even for the environments I never apply. The schema validation catches "this
  argument doesn't exist in real AWS" without needing an account.
- *"What would you do first with a real AWS account?"* — apply the VPC module for real and verify
  isolation by removing the NAT route and confirming S3 still works through the gateway endpoint while
  the public internet doesn't. That single test validates most of REQ-AW-03.

> **Your story:** your Go repos (`ticket-booking`, `offline-sync-app`) already used this
> moto/Terraform-endpoint-override approach with EventBridge and SQS — so this is a technique you have
> actually applied more than once, in more than one language. That's a genuine, defensible claim and
> a good one to make: *"I've used this pattern in Go and reused it in Java; here's what it's good for
> and here's where it stops."* And the AWS SA-Associate is the right place to say you have the breadth
> but that the depth came from building.

**Pitfalls / red flags:** presenting local emulation as equivalent to AWS; claiming the VPC design is
"tested"; not knowing that moto doesn't evaluate IAM.

**Where in StreamPulse:** BS-06 / REQ-AW-02, REQ-AW-03, REQ-AW-06; Phase 10; `docs/network.md`,
`docs/dr-drill.md`; acceptance tests `CompactHourFunctionTest`, `CompactionLambdaIT`,
`terraform validate/plan` in CI, `MirrorMakerFailoverIT`.

**Verify:** `s3_use_path_style` vs the older `s3_force_path_style` argument name for your AWS provider
version; moto's current service coverage; whether moto Lambda works on Apple Silicon.

---

## Key terms

| Term | One-line meaning |
|---|---|
| **Execution environment** | Lambda's reusable micro-VM; init runs once, handler runs many times, frozen in between. |
| **Cold start** | The init phase cost, dominated for Java by JVM start + classloading + framework init. |
| **SnapStart** | Snapshot-and-restore of the initialized JVM; requires CRaC hooks for connections and RNG state. |
| **Event source mapping (ESM)** | The Lambda-service poller for SQS/Kinesis/DynamoDB Streams/Kafka. |
| **`ReportBatchItemFailures`** | Handler returns only the failed record IDs, so the ESM retries just those. |
| **Gateway endpoint** | Free, route-table-based VPC endpoint — **S3 and DynamoDB only**. |
| **Interface endpoint (PrivateLink)** | An ENI in your subnet for most other services; hourly + per-GB cost. |
| **Security group vs NACL** | Stateful, ENI-level, allow-only vs stateless, subnet-level, ordered allow/deny. |
| **Shard / partition** | The unit of ordering and parallelism in Kinesis / Kafka. |
| **Enhanced fan-out** | Kinesis push subscription giving each consumer its own per-shard read throughput. |
| **Hot partition** | A DynamoDB/Kinesis key receiving disproportionate traffic; throttles despite table headroom. |
| **Single-table design** | Multiple entity types in one DynamoDB table so related items share a partition. |
| **LSI vs GSI** | Same PK/different SK, table-creation-only, strongly consistent vs own keys/capacity, eventually consistent. |
| **Conditional write** | Atomic compare-and-set; the primitive behind idempotent consumers and optimistic locking. |
| **Global tables** | DynamoDB multi-region multi-active with **last-writer-wins** conflict resolution. |
| **Small-files problem** | Many tiny S3 objects make analytics request-bound; fixed by compaction. |
| **RPO / RTO** | How much data you can lose / how long you can be down. |
| **MirrorMaker 2 offset translation** | Maps consumer offsets between clusters approximately — expect reprocessing after failover. |
| **Terraform state** | The map from config to real resource IDs; remote, locked, encrypted, split by lifecycle. |
| **Provider endpoint override** | Pointing the AWS provider/SDK at moto or DynamoDB Local — API-level fidelity only. |

---

## Self-check (answer out loud, no notes)

1. Your Kinesis stream has 3 consumers. What breaks, and what does it cost to fix?
2. A Lambda triggered by SQS processes a batch of 10; record 7 fails. What happens by default, and
   what do you change?
3. Why is a gateway endpoint for S3 the first thing you add to a VPC, and what does it save?
4. Define RPO and RTO, then say what makes a failover drill credible rather than theatre.
5. What exactly does StreamPulse's moto/DynamoDB-Local setup prove, and what does it not?
