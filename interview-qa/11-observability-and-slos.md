# 11 — Observability & SLOs: Micrometer, Tracing, SLI/SLO/SLA, Error Budgets

**Why this matters for an Event Hub role:** the JD says "Datadog/Grafana" and the prep doc says the
panel will ask *"what metrics define system health?"*. That question is a trap for people who answer
"CPU, memory, disk". For a telemetry pipeline the health metrics are **freshness, consumer lag, DLT
rate and ingest availability** — user-facing symptoms, not machine vitals. And the follow-up that
separates senior from mid is the one about **error budgets**: not "what is an SLO" but "how do you
turn 99.98% into a pager policy that doesn't wake you up for a blip".

The second reason this file matters: an Event Hub *is* an observability product. If you cannot
instrument your own pipeline, you cannot be trusted to build one for other teams.

**Scenario anchor — BS-10:** `sportsnow` signs a fictional 99.98% monthly availability contract on the
ingest API plus dashboard freshness < 10s, with service credits on breach. REQ-SL-01..06 require
per-tenant SLIs as Prometheus recording rules, the budget arithmetic written down, multi-window
burn-rate alerts, a deploy-gate freeze policy at < 25% budget remaining, and a per-tenant SLA report
endpoint. Phase 8 builds the observability stack; Phase 11 builds the Kubernetes mechanics that make
the number achievable.

---

## Q1. What are the three pillars of observability, and are they equally useful?

**30-second answer:** Metrics, logs and traces. They are absolutely **not** equal — they differ in
cost, cardinality tolerance and the question they answer. Metrics are cheap, aggregatable and
bounded-cardinality, so they answer *"is something wrong, and how bad?"* — that's what I alert on.
Traces answer *"where in the call graph is the latency or the error?"* — that's what I open **after**
the alert. Logs answer *"what exactly happened to this one record?"* — the highest-detail, highest-cost
signal, and the one I sample and structure most aggressively. The mistake I see is teams alerting on
log counts, which couples their paging to their most expensive and least reliable pipeline.

**Deeper:**

| | Metrics | Traces | Logs |
|---|---|---|---|
| Question answered | *Is it broken? How much?* | *Where is it broken?* | *What happened to this one thing?* |
| Cost per unit | very low (pre-aggregated time series) | medium (sampled spans) | high (raw text/JSON per event) |
| Cardinality tolerance | **low** — every tag combination is a new time series | high (trace/span IDs are the point) | high |
| Retention typical | long (months–years) | short (days) | medium (days–weeks), often tiered |
| Good for alerting | ✅ yes | ❌ no (sampled, so not a reliable count) | ⚠️ only via derived metrics |
| Good for root cause | ❌ no | ✅ yes | ✅ yes |

The fourth signal people forget: **events/profiles**. Continuous profiling (async-profiler, JFR,
Pyroscope) answers *"which code is burning the CPU"*, which neither metrics nor traces tell you. In
StreamPulse that's Phase 9.

The unifying idea is **correlation**: a metric anomaly should link to an exemplar trace, and a trace
should link to the logs of that request. Prometheus supports this with **exemplars** (a trace ID
attached to a histogram bucket sample), and Grafana links Prometheus → Tempo → Loki by `traceId`.
Without that linkage you have three dashboards and no story.

```
 alert fires (metric)
        │  exemplar traceId
        ▼
   trace in Tempo  ── shows the slow span is the DynamoDB write
        │  traceId
        ▼
   logs in Loki  ── "ProvisionedThroughputExceededException tenant=sportsnow"
```

**Follow-ups they'll ask:**
- *"Monitoring vs observability?"* — monitoring is checking known failure modes you predicted;
  observability is being able to ask new questions of the system without shipping new code. A
  dashboard with 12 fixed panels is monitoring. High-cardinality trace/log attributes you can group by
  arbitrarily is observability.
- *"Which would you build first on a new service?"* — RED metrics plus structured logs with a
  correlation ID. Tracing third, because it needs a collector and a backend to be useful.
- *"Where does profiling fit?"* — the fourth signal, for CPU/alloc attribution. Not part of the
  classic three.

**Pitfalls / red flags:** reciting "metrics, logs, traces" with no sense of cost or of which one you
alert on. Alerting on a log-derived count when the log pipeline itself can drop data.

**Where in StreamPulse:** Phase 8 — Micrometer → Prometheus, OTel agent → Collector → Tempo, JSON logs
→ Loki, all correlated by `traceId`.

---

## Q2. Walk me through Micrometer's meter types and when you use each.

**30-second answer:** Five that matter. A **Counter** is monotonically increasing — events ingested,
errors. A **Gauge** samples a current value — queue depth, consumer lag, cache size — and Micrometer
holds a weak reference to the object, so if you gauge a local variable it gets collected and reports
NaN. A **Timer** records duration plus a count, so it gives you throughput and latency from one meter.
A **DistributionSummary** is a Timer for non-time quantities — batch size, payload bytes. And a
**LongTaskTimer** measures tasks that are *still running*, which is the one people forget: a normal
Timer only records on completion, so a permanently hung S3 upload shows up in a LongTaskTimer and is
completely invisible in a Timer.

**Deeper:**

| Meter | Shape | StreamPulse use | Trap |
|---|---|---|---|
| `Counter` | monotonic double | `ingest.events.accepted`, `dlt.messages` | Never use for a value that can go down. |
| `Gauge` | instantaneous sample | `kafka.consumer.lag`, `s3.spool.bytes`, `lru.cache.size` | **Weak reference** — keep a strong ref to the gauged object or it reports NaN. |
| `Timer` | count + total time + max (+ optional histogram) | `ingest.request.duration`, `port.call` | Only records on *completion*. |
| `DistributionSummary` | count + total + max of a unitless value | `ingest.batch.size` (events per request), `kafka.record.bytes` | Same histogram/percentile considerations as Timer. |
| `LongTaskTimer` | how many tasks are in flight and how long the oldest has run | `s3.upload.active`, `kafka.streams.rebalance.active` | The only one that shows a *stuck* operation. |

Naming convention: Micrometer uses dot-separated lowercase names (`ingest.events.accepted`) and each
registry translates to its backend's convention — Prometheus turns it into
`ingest_events_accepted_total`. Write the Micrometer name; let the registry do the mapping. Don't
hand-write Prometheus-style names in Java.

**Code:**

```java
/**
 * StreamPulse ingest-gateway instrumentation (Phase 8, BS-10 REQ-SL-01).
 * Note every meter is created ONCE and reused — creating a meter per request is a
 * common performance bug (registry lookup + tag list allocation on the hot path).
 */
@Component
class IngestMetrics {

    private final Counter accepted;
    private final Counter rejectedQuota;
    private final Timer   publishLatency;
    private final DistributionSummary batchSize;
    private final LongTaskTimer activeUploads;

    IngestMetrics(MeterRegistry registry, InFlightTracker tracker) {
        // Counter: monotonic. Tag by LOW-cardinality dimensions only (see Q3).
        this.accepted = Counter.builder("ingest.events.accepted")
                .description("Telemetry events accepted into the pipeline")
                .baseUnit("events")
                .register(registry);

        this.rejectedQuota = Counter.builder("ingest.events.rejected")
                .tag("reason", "quota")          // bounded set: quota|schema|auth|overload
                .register(registry);

        // Timer: gives rate AND latency. Percentile histogram is an explicit opt-in (Q4).
        this.publishLatency = Timer.builder("ingest.publish.duration")
                .publishPercentileHistogram()     // exports Prometheus buckets, server-side aggregatable
                .register(registry);

        this.batchSize = DistributionSummary.builder("ingest.batch.size")
                .baseUnit("events")
                .register(registry);

        // LongTaskTimer: shows uploads that have STARTED but not finished. A plain Timer
        // would show nothing at all while S3 is hanging — the exact moment you need data.
        this.activeUploads = LongTaskTimer.builder("s3.upload.active")
                .register(registry);

        // Gauge: registry holds a WEAK reference to `tracker`. `tracker` is a Spring bean,
        // so it is strongly held elsewhere. Gauging a local object here would report NaN.
        Gauge.builder("ingest.inflight.batches", tracker, InFlightTracker::count)
                .register(registry);
    }

    void recordBatch(int events, Duration publishTime) {
        accepted.increment(events);
        batchSize.record(events);
        publishLatency.record(publishTime);
    }
}
```

**Follow-ups they'll ask:**
- *"Why did my gauge disappear?"* — the gauged object was garbage collected. Micrometer holds it
  weakly on purpose (so meters don't leak your objects); you must hold a strong reference.
- *"`@Timed` vs manual `Timer`?"* — `@Timed` is AOP-based and convenient, but it inherits the
  proxy limitations (no self-invocation, Spring beans only). I use manual timers on hot paths and in
  library code, `@Timed`/Observation for controller-level convenience.
- *"How do you time something that throws?"* — `Timer.Sample` in a try/finally, or
  `timer.record(() -> ...)`. Record the failure path too, tagged `outcome=failure`, otherwise your p99
  only measures successes and looks great during an outage.
- *"What is a `MeterFilter`?"* — a registry-level hook to rename, deny, or add common tags to meters.
  It's the enforcement point for the cardinality rules in Q3.

**Pitfalls / red flags:** creating meters inside the request path; using a Counter for something that
decreases; not knowing the weak-reference gauge behaviour, which is the single most common Micrometer
bug.

**Where in StreamPulse:** Phase 8 across all services; `libs/platform` registers the common tags
(`service`, `env`, `region`) via a `MeterFilter` so no service has to remember.

**Verify:** Micrometer builder method names (`publishPercentileHistogram`, `serviceLevelObjectives`)
against the Micrometer version you pin — the API is stable but check before quoting it.

---

## Q3. What is cardinality explosion, and how do you prevent it?

**30-second answer:** Every unique combination of tag values creates a **separate time series** in
Prometheus or a separate custom metric in Datadog. So one metric tagged `tenant` (3 values) ×
`endpoint` (10) × `status` (5) is 150 series — fine. Add `sessionId` and you have one series per
session, forever, and you've either DOSed your own Prometheus or turned a $2k/month Datadog bill into
$40k. The rule I apply without exception: **never tag by anything unbounded** — no `userId`, no
`sessionId`, no `eventId`, no raw URL path with IDs in it, no exception message. Unbounded identifiers
belong on **spans and logs**, which are built for high cardinality; metrics are built for aggregation.

**Deeper:**

The multiplication is the part people underestimate:

| Tag | Values | Running series count |
|---|---|---|
| `service` | 6 | 6 |
| `endpoint` | 10 | 60 |
| `status` | 5 | 300 |
| `tenant` | 3 | **900** — still fine |
| `deviceType` | 8 | 7,200 — acceptable, deliberate |
| `sessionId` | ~2,000,000/day | **14,400,000,000** — Prometheus falls over |

And it's worse than a one-off spike: Prometheus keeps a series in memory for a while after it stops
receiving samples (churn), so a high-cardinality label that *changes* every day (a deploy `version`
tag, a `podName` tag on a service that restarts constantly) is slow poison.

**The URL path trap** deserves its own mention because Spring gets it right and hand-rolled code gets
it wrong. Micrometer's `http.server.requests` tags by `uri` using the **route template**
(`/v1/titles/{id}`), not the resolved path (`/v1/titles/tt-9931`). If you build your own HTTP timer
from `request.getRequestURI()`, every title ID becomes a time series.

**Code — the enforcement, not just the intention:**

```java
/**
 * libs/platform: registry-level guardrails. Intentions in a wiki don't survive;
 * a MeterFilter does. (Phase 8)
 */
@Configuration
class MetricsGuardrailConfig {

    @Bean
    MeterFilter commonTags(@Value("${spring.application.name}") String service,
                           @Value("${streampulse.env}") String env) {
        // Applied to EVERY meter; services never write these tags by hand (DRY).
        return MeterFilter.commonTags(Tags.of("service", service, "env", env));
    }

    @Bean
    MeterFilter cardinalityCeiling() {
        // Hard ceiling per metric name. Beyond it, extra tag combinations are
        // collapsed into a single series rather than unbounded growth.
        // This is a safety net for a bug, NOT a substitute for tagging correctly.
        return MeterFilter.maximumAllowableTags(
                "ingest.events.accepted", "tenant", 100,
                MeterFilter.deny());
    }

    @Bean
    MeterFilter denyHighCardinalityTags() {
        // Fail loudly in tests, drop silently in prod — the ArchUnit-style guard.
        return MeterFilter.ignoreTags("sessionId", "eventId", "userId", "idempotencyKey");
    }
}
```

❌ **Broken** — this looks harmless in review and destroys the metrics backend:

```java
registry.counter("ingest.events",
        "tenant",    e.tenantId(),
        "sessionId", e.sessionId(),      // ← unbounded: one series PER VIEWING SESSION
        "title",     e.titleId())        // ← ~100k titles: also unbounded in practice
       .increment();
```

✅ **Fixed** — bounded tags on the metric, the identifiers on the span and the log:

```java
registry.counter("ingest.events",
        "tenant",     e.tenantId(),          // 3 tenants — bounded by contract
        "deviceType", e.deviceType(),        // enum, ~8 values
        "outcome",    "accepted")            // enum
       .increment();

// The identifiers go where high cardinality is CHEAP:
Span.current().setAttribute("session.id", e.sessionId())
              .setAttribute("title.id",  e.titleId());
log.atInfo().addKeyValue("sessionId", e.sessionId())
            .addKeyValue("titleId",  e.titleId())
            .log("event accepted");
```

**Follow-ups they'll ask:**
- *"What if you genuinely need per-title metrics?"* — three options, in order: (1) query the trace or
  log store instead, (2) keep a **top-N** only — e.g. a Redis ZSET of the busiest titles, exported as
  a bounded gauge set, (3) accept the cost for a deliberately capped allowlist of titles during a live
  event, and remove it afterwards. What I would not do is tag the main metric and hope.
- *"How do you find an existing explosion?"* — Prometheus `topk(20, count by (__name__)({__name__=~".+"}))`
  and `/api/v1/status/tsdb` which reports the highest-cardinality label pairs. Datadog has a metrics
  summary page showing custom metric counts per name. **Verify the exact query/endpoint** for your
  Prometheus version.
- *"Does this apply to Datadog the same way?"* — worse, commercially: Datadog bills on *custom metric*
  count, where a custom metric is roughly one tag-value combination. Cardinality is literally a line
  item on the invoice.

> **Your story:** you have on-call experience with Datadog and Grafana — if you ever had a bill
> conversation, a metrics-cleanup project, or a dashboard that stopped loading because of series
> count, this is the exact question to tell it on. Keep it to: what the tag was, how you found it,
> what you replaced it with.

**Pitfalls / red flags:** "we tag by user so we can debug individual users" — that person has not run
a metrics backend. Also: not distinguishing *metric* cardinality (fatal) from *trace/log* cardinality
(the whole point).

**Where in StreamPulse:** Phase 8; `libs/platform` `MeterFilter`s; the tenant tag is safe precisely
because the plan fixes three fictional tenants.

---

## Q4. Histograms vs percentiles in Micrometer — why can't you average p99s across instances?

**30-second answer:** Because **percentiles are not linearly aggregatable**. If instance A reports
p99 = 100ms and instance B reports p99 = 300ms, the fleet p99 is *not* 200ms — it could be anywhere
between 100 and 300 depending on the request distribution, and you cannot recover it from the two
numbers. Micrometer's `publishPercentiles(...)` computes percentiles **client-side, per JVM**, so what
you scrape is already collapsed and unmergeable. `publishPercentileHistogram()` instead exports
**bucket counts** — the raw material — and lets Prometheus compute the percentile server-side with
`histogram_quantile()` **after** summing buckets across every pod. With 20 pods behind a load
balancer, the second is the only one that answers "what is our p99".

**Deeper:**

```
publishPercentiles(0.99)          publishPercentileHistogram()
 ┌────────┐                        ┌────────┐
 │ pod A  │ p99 = 100ms  ─┐        │ pod A  │ le=50:900 le=100:980 le=250:999 ─┐
 ├────────┤               │        ├────────┤                                  │ sum
 │ pod B  │ p99 = 300ms  ─┤ ???    │ pod B  │ le=50:100 le=100:400 le=250:950 ─┤ buckets
 ├────────┤               │        ├────────┤                                  │
 │ pod C  │ p99 = 120ms  ─┘        │ pod C  │ ...                             ─┘
                                            ▼
      no valid fleet p99            histogram_quantile(0.99, sum(rate(...)) by (le))
                                            = a real, correct fleet p99
```

| | `publishPercentiles` | `publishPercentileHistogram` | `serviceLevelObjectives` |
|---|---|---|---|
| Computed where | in the JVM | in Prometheus/Grafana | in the JVM, counted server-side |
| Aggregatable across pods | ❌ no | ✅ yes | ✅ yes |
| Time series cost | ~1 per percentile | ~30–70 buckets per meter (bounded) | 1 per SLO boundary |
| Accuracy | exact-ish for that pod | approximate, bucket-bounded | exact for "under/over this threshold" |
| Best for | single-instance debugging | fleet latency dashboards | **SLO compliance** |

The third column is the underrated one and it is exactly right for BS-10. If the SLO is "99% of ingest
requests under 300ms", I don't need a p99 at all — I need **the count of requests under 300ms divided
by total requests**. `serviceLevelObjectives(Duration.ofMillis(300))` adds a bucket at exactly that
boundary, and the SLI becomes a simple ratio of two counters, which is cheap, exact and trivially
aggregatable. Deriving an SLI from `histogram_quantile` interpolation is doing extra work and
introducing bucket-interpolation error for no reason.

Also note the Prometheus histogram bucket semantics: buckets are **cumulative** (`le` = "less than or
equal"), and `histogram_quantile` **linearly interpolates within the bucket**, so a percentile that
falls in a wide bucket is approximate. If your SLO threshold sits between two bucket edges, the number
you report is partly an artefact of your bucket layout. Put a bucket edge exactly on the threshold.

**Code:**

```yaml
# application.yml — Phase 8 / BS-10 REQ-SL-01
management:
  metrics:
    distribution:
      # Export aggregatable buckets for the endpoints we have SLOs on.
      percentiles-histogram:
        http.server.requests: true
        ingest.publish.duration: true
      # Put a bucket edge EXACTLY on the SLO threshold so the SLI is a clean ratio,
      # not an interpolation between two bucket edges.
      slo:
        http.server.requests: 100ms, 300ms, 1s
      # Bound the bucket range so we don't emit 70 useless buckets up to 30s.
      minimum-expected-value:
        http.server.requests: 5ms
      maximum-expected-value:
        http.server.requests: 2s
```

```promql
# The SLI as a ratio of counters — exact, cheap, aggregatable (REQ-SL-01)
sum(rate(http_server_requests_seconds_bucket{uri="/v1/events:batch",le="0.3"}[5m])) by (tenant)
/
sum(rate(http_server_requests_seconds_count{uri="/v1/events:batch"}[5m])) by (tenant)
```

**Follow-ups they'll ask:**
- *"What about Prometheus native histograms?"* — a newer sparse-bucket histogram format that gives far
  better resolution at much lower series cost, with Micrometer/OTel support. I'd evaluate it, but I'd
  **verify its stability status and the Grafana/Prometheus versions required** before depending on it.
- *"Why not just use a summary?"* — a Prometheus `summary` has the same client-side percentile problem.
  Same reason.
- *"How do you pick buckets?"* — from the SLO thresholds first, then a rough log-spaced spread around
  the observed distribution. Buckets outside your operating range are pure cost.
- *"p99 vs p99.9?"* — p99.9 needs ~1000× the traffic to be statistically meaningful per window. On a
  low-traffic endpoint a p99.9 is one request and pure noise.

**Pitfalls / red flags:** proudly showing a Grafana panel that does `avg(p99)` across pods. Also:
turning on percentile histograms for *every* meter, which is a cardinality problem in the other
direction (buckets are series too).

**Where in StreamPulse:** BS-10 / REQ-SL-01; the `serviceLevelObjectives` bucket at 300ms is what the
SLA report endpoint (REQ-SL-06) reads.

**Verify:** the `management.metrics.distribution.*` property paths and whether your Boot version has
moved any of them under `management.observations.*`. Also verify native-histogram support status.

---

## Q5. How do Spring Boot Actuator health groups map to Kubernetes probes?

**30-second answer:** Actuator exposes `/actuator/health`, and Boot can split it into **groups** —
`liveness` and `readiness` are the two built-in ones, surfaced at
`/actuator/health/liveness` and `/actuator/health/readiness`. The critical design rule is what goes in
each: **liveness must check only "is this JVM still functional"**, with no dependency checks at all,
because a failing liveness probe means Kubernetes *restarts the pod* — and restarting your pod does
not fix Kafka being down. **Readiness** is where dependency checks belong, because a failing readiness
probe just removes the pod from the Service endpoints, which is the correct response to "I temporarily
can't serve". Getting this backwards is the classic cascading-restart outage: a shared dependency
blips, every pod fails liveness, the whole fleet restarts simultaneously, and now you have a thundering
herd on the dependency that was already struggling.

**Deeper:**

| | Liveness | Readiness | Startup |
|---|---|---|---|
| Failure action | **restart the container** | remove from Service endpoints | keep waiting, then restart |
| Should check | JVM alive, no deadlock, event loop responsive | downstream deps, warm caches, drain state | slow init progress |
| Should NOT check | **any external dependency** | — | — |
| Actuator group | `liveness` (`LivenessState`) | `readiness` (`ReadinessState`) | usually reuse readiness path |

Boot models this internally as `ApplicationAvailability` with two enums — `LivenessState`
(`CORRECT` / `BROKEN`) and `ReadinessState` (`ACCEPTING_TRAFFIC` / `REFUSING_TRAFFIC`) — and publishes
`AvailabilityChangeEvent`s. That's the hook BS-01's ordered drain uses: the first `SmartLifecycle`
phase to stop flips readiness to `REFUSING_TRAFFIC`, Kubernetes removes the endpoint, *then* the
listeners stop.

**Code:**

```yaml
# Phase 11 — Helm-templated application-k8s.yml (BS-01 REQ-LC-06, BS-10 REQ-SL-05)
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics
  endpoint:
    health:
      probes:
        enabled: true          # exposes /actuator/health/{liveness,readiness}
      show-details: when-authorized
      group:
        liveness:
          # DELIBERATELY minimal. livenessState only. No Kafka, no Redis, no DB.
          include: livenessState
        readiness:
          # Dependencies we genuinely cannot serve without.
          include: readinessState,kafkaStreams
          # NOT included: the partner webhook, the mongo optional profile — we
          # degrade on those (BS-04) rather than going unready.
```

```java
/**
 * A custom readiness indicator for sink-writer (Phase 4/8).
 * Reports DOWN if the local S3 spool (BS-04 REQ-RS-02 fallback) is near its 1 GB cap —
 * we'd rather stop taking traffic than start dropping events.
 */
@Component("spoolCapacity")
class SpoolCapacityHealthIndicator implements HealthIndicator {

    private final SpoolStore spool;

    @Override
    public Health health() {
        long used = spool.bytesUsed(), cap = spool.capacityBytes();
        double ratio = (double) used / cap;
        // Readiness DOWN → endpoint removed → LB stops sending us batches.
        // Liveness is untouched: restarting would LOSE the spool, the opposite of what we want.
        return (ratio > 0.9 ? Health.outOfService() : Health.up())
                .withDetail("usedBytes", used)
                .withDetail("ratio", ratio)
                .build();
    }
}
```

**Follow-ups they'll ask:**
- *"Why is `show-details` restricted?"* — the detail payload leaks internal hostnames, DB names and
  sometimes credentials-in-URLs. `never` or `when-authorized` in production.
- *"What about `/actuator/prometheus` security?"* — it's an internal endpoint. I put actuator on a
  separate management port (`management.server.port`) that the Ingress doesn't route, and scrape it
  from inside the cluster.
- *"Does Actuator health cost anything?"* — yes, if an indicator does real I/O and the probe runs every
  few seconds across hundreds of pods. Health indicators should be cached or cheap; a DB health
  indicator that runs a query every 2s on 50 pods is 25 qps of pure overhead.
- *"How does readiness interact with graceful shutdown?"* — directly: `server.shutdown=graceful` plus
  flipping readiness first is the whole of BS-01's drain (see file 13, Q on graceful shutdown).

**Pitfalls / red flags:** pointing liveness at `/actuator/health` (the full aggregate, including
dependency indicators). That single line is the cascading-restart outage.

**Where in StreamPulse:** BS-01 / REQ-LC-01, BS-10 / REQ-SL-05; Phase 11 Helm charts.

**Verify:** the `management.endpoint.health.group.*` and `probes.enabled` property paths for your Boot
version, and whether Boot auto-enables probes when it detects Kubernetes (it does detect the
environment, but confirm the exact condition).

---

## Q6. RED vs USE — which method applies to what?

**30-second answer:** **RED** — Rate, Errors, Duration — is for **request-driven services**: anything
with a caller waiting for a response. It's a *symptom* view, so it maps directly to SLIs and to what
users feel. **USE** — Utilization, Saturation, Errors — is for **resources**: CPU, memory, disk, thread
pools, connection pools, queues. It's a *cause* view, for diagnosis. The rule I use: **alert on RED,
diagnose with USE**. A saturated thread pool is only interesting because it caused latency; paging on
80% CPU gets you woken up for a system that is working perfectly.

**Deeper:**

| | RED | USE |
|---|---|---|
| Subject | services / endpoints | resources |
| Signals | Rate, Errors, Duration | Utilization, Saturation, Errors |
| Perspective | symptom (what the user sees) | cause (why) |
| Use for | SLIs, SLOs, paging alerts | dashboards, capacity planning, root cause |
| Origin | Tom Wilkinson / Weaveworks (verify attribution) | Brendan Gregg (verify attribution) |

The gap both methods leave for StreamPulse: **a streaming pipeline is not request-driven**. There is
no caller waiting on the Kafka Streams topology. So RED doesn't cleanly apply to the middle of the
pipe. The streaming analogue is a third set:

| Streaming signal | What it means | StreamPulse metric |
|---|---|---|
| **Throughput** | records in/out per second | `kafka.consumer.records.consumed.rate` |
| **Lag** | how far behind the consumer is, in records | `kafka.consumer.fetch.manager.records.lag.max` |
| **Freshness / end-to-end latency** | `now − event.timestamp` at delivery | `streampulse.freshness.seconds` (custom) |
| **Error/DLT rate** | share of records that failed terminally | `dlt.messages.total / records.consumed.total` |

Freshness is the one I'd lead with in an Event Hub interview, because it is the only one that is
genuinely **user-facing** for a pipeline. Lag can be zero while freshness is terrible (the producer is
slow), and lag can be high while freshness is fine (you're catching up on a backlog nobody is
watching). They are different questions.

```
 RED  ──▶ ingest-gateway  (HTTP: rate, errors, duration)  ← SLI lives here
          │
 stream ─▶ stream-processor (throughput, lag, freshness)  ← SLI lives here too
          │
 USE  ──▶ Kafka broker disk, RocksDB size, Hikari pool, ThreadPoolExecutor queue depth
                                                          ← diagnosis only, no pages
```

**Code:**

```java
/**
 * Freshness: the metric that actually matters to a dashboard user (BS-10 REQ-SL-01).
 * Recorded at the LAST hop before the user sees data, not at the consumer.
 */
void onAggregateDelivered(MetricAggregate agg) {
    Duration freshness = Duration.between(agg.eventTime(), Instant.now());
    freshnessTimer.record(freshness);        // has an SLO bucket at 10s (REQ-SL-01)
    // NOT tagged by title/session — see Q3.
}
```

**Follow-ups they'll ask:**
- *"What are the Four Golden Signals?"* — Google SRE's version: latency, traffic, errors, **saturation**.
  It's RED plus saturation, which I think is the better default because saturation is the leading
  indicator that predicts the other three.
- *"Give me a USE metric that actually paged you."* — a saturating metric with a hard ceiling is fair
  to page on *when the ceiling is unrecoverable*: disk 95% full, connection pool at max with a growing
  wait queue. Those don't self-heal.
- *"How do you measure saturation for a thread pool?"* — queue depth and rejection count, not CPU.
  Micrometer's `ExecutorServiceMetrics` binds `executor.queued`, `executor.active`,
  `executor.pool.size` — that's exactly the BS-02 / REQ-CC-04 bounded S3 executor.

**Pitfalls / red flags:** reciting RED/USE as a mnemonic with no opinion about which one you page on.
Also: describing lag and freshness as the same thing.

**Where in StreamPulse:** Phase 8 Grafana dashboards — RED per service, plus lag/freshness/DLT panels.

---

## Q7. How do you do logging properly in a distributed system?

**30-second answer:** Four things. **Structured** — JSON, not a formatted string, so fields are
queryable rather than regex-able. **Correlated** — every line carries a `traceId` and the business
keys (`tenantId`, `sessionId`) via MDC, so one query reconstructs a request across six services.
**Levelled with intent** — INFO is for things an operator would want to see, DEBUG is off in prod,
ERROR means "a human needs to look at this" and nothing else; a handled retry is not an ERROR.
And **sampled and scrubbed** — at 50k events/s you cannot log per event, so you log per batch plus a
sampled subset, and you never put PII or tokens in a log line because logs are the least
access-controlled store you have.

**Deeper:**

**Why MDC and what breaks it.** MDC is a `ThreadLocal<Map<String,String>>` in SLF4J. That has two
consequences that come up constantly in this codebase:

1. Handing work to another thread loses it. `CompletableFuture.supplyAsync` on the BS-02 panel
   executor runs on a pool thread with an empty MDC. Fixes: `MDC.getCopyOfContextMap()` captured
   before the fork and restored inside the task, or a `TaskDecorator` on the executor that does it for
   every submission (the DRY option — do it once in `libs/platform`).
2. Thread pools **reuse** threads, so a stale MDC from a previous request leaks into the next log line.
   Always clear in a `finally`, or use `MDC.putCloseable` in try-with-resources.

Micrometer Tracing / the OTel agent put `traceId` and `spanId` into MDC automatically, which is why
the correlation "just works" once tracing is wired — but the *business* keys are yours to add.

**Log levels, with an opinion:**

| Level | Prod default | Meaning I enforce |
|---|---|---|
| ERROR | on | A human must look. Pages or tickets come from here. A caught-and-retried failure is **not** ERROR. |
| WARN | on | Degraded but handled — circuit breaker opened, fell back to spool, retry exhausted. |
| INFO | on, low volume | Lifecycle and business milestones: startup config, rebalance, cutover flag flipped. Not per-request. |
| DEBUG | **off**, enable per-logger at runtime | Per-request detail. Actuator `/actuator/loggers` lets you turn it on for one logger for ten minutes without a deploy — this is the single most useful Actuator endpoint on-call. |
| TRACE | off | Wire-level. |

**Sampling.** Three techniques, in the order I'd apply them: (1) log per *batch*, not per event —
one line with `count=487`; (2) **deterministic sampling** on a hash of the trace ID so that if you log
a request, you log *all* of it across every service (random per-line sampling gives you useless
fragments); (3) **rate-limit the repeaters** — Logback's `DuplicateMessageFilter` or a
`TurboFilter`, so one poison message doesn't produce 2M identical ERROR lines and blow out the Loki
bill during an incident.

**PII.** In StreamPulse the `kidsplus` tenant has "strict data rules", which is the scenario hook. The
rules: no raw event payloads at INFO, no tokens/JWTs ever (log the `jti` or a hash, not the token),
IDs are opaque (a `sessionId` is fine, an email is not), and enforce it with a Logback masking
converter plus a test — because "don't log PII" as a code-review convention fails eventually.

**Code:**

```java
/**
 * libs/platform: one TaskDecorator so no service has to remember MDC propagation.
 * Applies to every @Async executor and the BS-02 panel executor (REQ-CC-01).
 */
public class MdcTaskDecorator implements TaskDecorator {
    @Override
    public Runnable decorate(Runnable task) {
        Map<String, String> caller = MDC.getCopyOfContextMap();   // captured on the REQUEST thread
        return () -> {
            Map<String, String> previous = MDC.getCopyOfContextMap();
            try {
                if (caller != null) MDC.setContextMap(caller);
                task.run();
            } finally {
                // Must restore, not just clear: pool threads are reused (see above).
                if (previous != null) MDC.setContextMap(previous); else MDC.clear();
            }
        };
    }
}
```

```java
// Structured, correlated, no PII, one line per batch — not per event.
try (var t = MDC.putCloseable("tenantId", batch.tenantId());
     var b = MDC.putCloseable("batchId",  batch.id())) {          // traceId added by the agent
    log.atInfo()
       .addKeyValue("eventCount", batch.size())
       .addKeyValue("bytes",      batch.bytes())
       .addKeyValue("outcome",    "accepted")
       .log("ingest batch accepted");     // ← message is a CONSTANT, so it groups in Loki
}
```

```xml
<!-- logback-spring.xml — JSON to stdout; the container runtime ships it to Loki. -->
<appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
  <encoder class="net.logstash.logback.encoder.LogstashEncoder">
    <includeMdcKeyName>traceId</includeMdcKeyName>
    <includeMdcKeyName>spanId</includeMdcKeyName>
    <includeMdcKeyName>tenantId</includeMdcKeyName>
  </encoder>
</appender>
```

**Follow-ups they'll ask:**
- *"Why is the log message a constant string?"* — so the log backend can group by message and you can
  count occurrences. Interpolating values into the message defeats grouping; values go in fields.
- *"Sync or async appender?"* — async (`AsyncAppender` / Logback's async with a bounded queue) so a
  slow log sink can't block request threads. But set `neverBlock` deliberately: blocking means log
  backpressure stalls your service; dropping means you lose lines during the incident you most need
  them. I choose dropping for INFO and blocking-with-a-short-queue for ERROR. That's a real trade-off,
  not a default.
- *"How do you handle logs during an incident when volume spikes 100×?"* — that's exactly when the
  rate limiter and the drop policy matter; design it before the incident.
- *"Boot 3.4 has built-in structured logging, right?"* — I believe recent Boot versions added
  `logging.structured.format.console` with ECS/Logstash/GELF formats, which would replace the
  `logstash-logback-encoder` dependency. **Verify the version and property name** before relying on it.

**Pitfalls / red flags:** `log.error("failed", e)` inside a retry loop (now every transient blip pages
someone), `e.printStackTrace()`, logging the full request body at INFO, and string-concatenated log
messages.

**Where in StreamPulse:** Phase 8 (JSON → Loki), BS-01 (MDC tenant tag via the AOP aspect in file 01
Q8), BS-02 (MDC across the `CompletableFuture` fan-out).

**Verify:** built-in Boot structured-logging property names and version; `logstash-logback-encoder`
coordinates.

---

## Q8. Explain OpenTelemetry: trace, span, context — and what W3C traceparent actually carries.

**30-second answer:** A **trace** is one end-to-end operation, identified by a 16-byte trace ID. A
**span** is one unit of work within it — an HTTP handler, a DB call — with its own 8-byte span ID, a
parent span ID, a start/end time, attributes, events and a status. **Context** is the ambient
(thread-local, or `Context`-object in async code) holder of the current span, and **propagation** is
serializing that context across a process boundary. The wire format everyone standardizes on is the
W3C `traceparent` header: `version-traceId-parentSpanId-flags`, where the flag bit says "this trace
was sampled". The part that trips people up is that propagation is **not automatic across
non-HTTP boundaries** — Kafka, SQS, a thread pool hand-off — and that's exactly where this role lives.

**Deeper:**

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             │  │                                │                │
             │  │                                │                └ flags: 01 = sampled
             │  │                                └ parent span id (8 bytes / 16 hex)
             │  └ trace id (16 bytes / 32 hex)
             └ version
```

There is a companion header, `tracestate`, which carries vendor-specific key/value data along the
trace. The pair is the W3C Trace Context standard; the older de-facto formats are B3 (Zipkin, `X-B3-*`)
and Jaeger's `uber-trace-id`. In a mixed estate you configure the propagator to inject/extract several
formats — OTel supports composite propagators. **Verify the exact config key name** (`OTEL_PROPAGATORS`
as an env var, as I recall) in current docs.

**Span shape, and the bits people skip:**

| Field | Why it matters |
|---|---|
| `name` | Must be **low cardinality** — `GET /v1/titles/{id}`, not the resolved path. Same rule as metric tags. |
| `kind` | `SERVER`, `CLIENT`, `PRODUCER`, `CONSUMER`, `INTERNAL`. Backends use it to build the service graph; a Kafka producer span **must** be `PRODUCER` or the topology is wrong. |
| attributes | High cardinality is fine and encouraged here — `session.id`, `title.id`, `tenant.id`. |
| events | Timestamped points inside a span — "retry attempt 2", "circuit breaker opened". |
| links | Connect a span to **other traces** — the mechanism for batch consumers (Q9). |
| status | `OK` / `ERROR` + description. Set it explicitly; an exception caught and handled should not be ERROR. |

**Semantic conventions** are the underrated part: OTel defines standard attribute names
(`http.request.method`, `messaging.system`, `db.system`). Using them is what makes a backend able to
draw a service map and auto-detect a slow DB without per-app config. Note that several of these were
renamed during OTel's stabilization (`http.method` → `http.request.method`, for example) —
**verify the current semantic convention names** rather than trusting memory.

**Code:**

```java
/**
 * Manual span around the BS-02 panel fan-out. The OTel agent instruments HTTP and Kafka
 * automatically, but a business-meaningful boundary like "assemble command center"
 * is one you add by hand — that's where the latency attribution actually helps.
 */
@Component
class CommandCenterAssembler {

    private final Tracer tracer;    // io.opentelemetry.api.trace.Tracer

    CommandCenterDto assemble(String titleId) {
        Span span = tracer.spanBuilder("commandcenter.assemble")
                .setSpanKind(SpanKind.INTERNAL)
                .setAttribute("title.id", titleId)          // high cardinality: fine on a span
                .startSpan();
        try (Scope scope = span.makeCurrent()) {            // makes it the CURRENT context
            var dto = fanOutPanels(titleId);                 // REQ-CC-01
            span.setAttribute("panels.degraded", dto.degradedCount());
            return dto;
        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR, "panel assembly failed");
            throw e;
        } finally {
            span.end();                                      // ALWAYS in finally, or the span leaks
        }
    }
}
```

**Follow-ups they'll ask:**
- *"What breaks context propagation inside one JVM?"* — any thread hand-off the instrumentation doesn't
  know about. OTel's agent instruments common executors, but for a hand-rolled `ThreadPoolExecutor`
  you wrap it with `Context.taskWrapping(executor)` — same problem, same shape, as the MDC decorator in
  Q7. **Verify the helper's exact name** in the OTel Java API version you use.
- *"Baggage vs attributes?"* — **baggage** propagates key/value data *across services* along the trace
  (so a downstream service knows the `tenantId`); attributes stay on one span. Baggage is powerful and
  dangerous: it goes on every wire hop, so keep it tiny, and never put anything sensitive in it because
  it will reach third parties.
- *"Micrometer Observation vs raw OTel?"* — Micrometer's Observation API is one instrumentation call
  that emits both a metric and a span, bridged to OTel via `micrometer-tracing-bridge-otel`. In a
  Spring shop that's the better abstraction; it stops you from instrumenting the same thing twice.

**Pitfalls / red flags:** thinking a trace ID in the logs *is* tracing (it's correlation, not a trace);
not ending spans in `finally`; high-cardinality span *names*.

**Where in StreamPulse:** Phase 8 — OTel Java agent → Collector → Tempo; `libs/platform` owns the
tracing auto-configuration (BS-01 REQ-LC-07).

**Verify:** OTel semantic convention attribute names (several were renamed at stabilization), the
propagator config key, and `Context.taskWrapping`.

---

## Q9. How do you propagate a trace across Kafka?

**30-second answer:** Through **message headers**. Kafka records carry arbitrary headers, so the
producer injects the W3C `traceparent` (and `tracestate`) into the record headers, and the consumer
extracts them and starts its span as a child — or, more correctly, as a **linked** span. The OTel Java
agent does this automatically for the standard Kafka clients, and Spring Kafka does it natively when
observations are enabled. Two subtleties matter. First, a Kafka hop is **asynchronous** — the consumer
may run hours later, so a naive parent-child relationship produces a trace with an enormous gap; the
convention is producer span `kind=PRODUCER`, consumer span `kind=CONSUMER`, joined by a **span link**
for batch consumption. Second, **batch listeners break the one-trace model entirely**: 500 records from
200 different traces arrive in one poll, and there is no single parent. The right shape there is one
span for the batch with N links, not one arbitrary parent.

**Deeper:**

```
ingest-gateway                        Kafka                     sink-writer
┌──────────────────┐                                       ┌────────────────────┐
│ SERVER span      │                                       │                    │
│  └ PRODUCER span │──record headers──────────────────────▶│ CONSUMER span      │
│      injects     │  traceparent: 00-4bf9...-00f0...-01    │  extracts + links  │
│      traceparent │                                       │   └ INTERNAL spans │
└──────────────────┘   (may be minutes/hours later)        └────────────────────┘
```

| Shape | When | Trace looks like |
|---|---|---|
| child span (parent = producer) | single-record processing, near-real-time | one continuous trace, ingest → sink |
| **span link** | batch listeners, replay, long delay | a batch span that references N origin traces |

The reason span links exist is precisely this case: "this work was *caused by* those N operations, but
it is not a nested sub-operation of any one of them".

**The sampling coupling is the senior detail.** The sampled flag travels in the `traceparent` flags
byte. If the producer sampled the trace out (flag `00`), the consumer's instrumentation will normally
respect that and also not record — so **your consumer-side traces inherit the producer's sampling
decision**. If ingest samples at 1% you get 1% of pipeline traces, and no amount of consumer config
changes that. That's usually correct (you want whole traces, not fragments), but you must know it,
because the failure mode is "I turned up sampling on sink-writer and still see nothing".

**Code:**

```java
/**
 * Spring Kafka propagates trace context natively when observation is enabled,
 * so in the common case you write NO propagation code (Phase 8).
 */
@Configuration
class KafkaObservabilityConfig {

    @Bean
    KafkaTemplate<String, TelemetryEvent> kafkaTemplate(ProducerFactory<String, TelemetryEvent> pf) {
        var template = new KafkaTemplate<>(pf);
        template.setObservationEnabled(true);   // injects traceparent into record headers
        return template;
    }

    @Bean
    ConcurrentKafkaListenerContainerFactory<String, TelemetryEvent> factory(
            ConsumerFactory<String, TelemetryEvent> cf) {
        var factory = new ConcurrentKafkaListenerContainerFactory<String, TelemetryEvent>();
        factory.setConsumerFactory(cf);
        factory.getContainerProperties().setObservationEnabled(true);  // extracts + starts CONSUMER span
        return factory;
    }
}
```

```java
/**
 * Manual form — worth knowing, because it is what you fall back to for a batch listener
 * where the automatic single-parent model is wrong (BS-03 REQ-KT-05 uses batch listeners).
 */
void onBatch(List<ConsumerRecord<String, TelemetryEvent>> records) {

    SpanBuilder batchSpan = tracer.spanBuilder("telemetry.raw.v1 process")
            .setSpanKind(SpanKind.CONSUMER)
            .setAttribute("messaging.system", "kafka")
            .setAttribute("messaging.batch.message_count", (long) records.size());

    // One LINK per origin trace: "caused by", not "nested inside".
    for (var record : records) {
        Context ctx = propagators.getTextMapPropagator()
                .extract(Context.root(), record.headers(), KAFKA_HEADER_GETTER);
        SpanContext origin = Span.fromContext(ctx).getSpanContext();
        if (origin.isValid()) {
            batchSpan.addLink(origin);
        }
    }

    Span span = batchSpan.startSpan();
    try (Scope s = span.makeCurrent()) {
        process(records);
    } finally {
        span.end();
    }
}

/** Adapter from Kafka Headers to OTel's TextMapGetter — the only glue code you need. */
private static final TextMapGetter<Headers> KAFKA_HEADER_GETTER = new TextMapGetter<>() {
    @Override public Iterable<String> keys(Headers headers) {
        return () -> StreamSupport.stream(headers.spliterator(), false)
                                  .map(Header::key).iterator();
    }
    @Override public String get(Headers headers, String key) {
        Header h = headers.lastHeader(key);
        return h == null ? null : new String(h.value(), StandardCharsets.UTF_8);
    }
};
```

**Follow-ups they'll ask:**
- *"Does this work for Kinesis / SQS / RabbitMQ?"* — same principle, different carrier. SQS has message
  attributes; RabbitMQ has AMQP headers (BS-07). Kinesis has **no** per-record header concept, so you
  embed the trace context **in the record payload** — which means your Avro/JSON envelope needs a
  `traceparent` field. That's a genuine design difference and a good thing to raise in the Kafka vs
  Kinesis ADR (Phase 10).
- *"What about Kafka Streams?"* — the topology does its own record forwarding; verify what the OTel
  agent instruments for Streams specifically, because it is less complete than the plain producer/
  consumer instrumentation. For repartition topics you may lose context unless headers are forwarded.
- *"How much do headers cost?"* — a traceparent is ~55 bytes per record. At 50k ev/s with small
  telemetry records that is real overhead, and it works directly against BS-03's "cut bytes by 30%"
  goal. Worth measuring; compression helps because the headers are highly repetitive.
- *"How do you trace the whole pipeline end to end for a demo?"* — that's exactly the Phase 8 interview
  proof: one trace from HTTP ingest → Kafka → Streams → sink → SSE delivery, visible in Tempo.

**Pitfalls / red flags:** "tracing works automatically" without knowing the carrier is headers, or
claiming a batch of 500 records belongs to one trace.

**Where in StreamPulse:** Phase 8 ("trace from HTTP ingest through Kafka headers to the consumer" is
listed as the phase deliverable); BS-09 REQ-GR-05 propagates W3C context over gRPC metadata too.

**Verify:** `setObservationEnabled` availability in your Spring Kafka version; OTel agent coverage for
Kafka Streams; `messaging.*` semantic convention attribute names.

---

## Q10. Head-based vs tail-based sampling, and agent vs manual instrumentation?

**30-second answer:** **Head sampling** decides at the start of the trace — cheap, no buffering,
consistent across services because the decision rides in the traceparent flag — but it's blind: you
throw away 99% of traces before you know which ones were slow or errored, so the interesting ones are
mostly gone. **Tail sampling** buffers all spans of a trace in the collector until it's complete, then
decides — so you can keep 100% of errors and slow traces and 1% of the boring ones. The cost is real:
the collector must hold every span for the trace duration, and it must see *all* spans of a trace,
which means a load-balancing collector layer that routes by trace ID. My default is head sampling at a
low rate plus tail sampling in the collector for errors and latency outliers — and, critically, a
**parent-based** sampler so a decision is never made twice mid-trace.

**Deeper:**

| | Head-based | Tail-based |
|---|---|---|
| Decision point | first service, at trace start | collector, after trace completes |
| Keeps all errors? | ❌ only by luck | ✅ by policy |
| Memory cost | ~0 | buffers every span for N seconds |
| Requires | nothing | trace-ID-aware routing to a collector, so all spans of a trace land together |
| Consistency | guaranteed via `traceparent` flags | guaranteed by seeing the whole trace |
| Config | SDK sampler | Collector `tail_sampling` processor |

Useful third option: **rate-limiting samplers** (N traces/sec per service) instead of a percentage,
which keeps your trace bill flat regardless of traffic spikes — relevant for BS-03's "live match
kickoff 10× spike", where a percentage sampler multiplies your trace volume by 10 at the worst moment.

**Agent vs manual:**

| | Java auto-instrumentation agent | Manual spans |
|---|---|---|
| How | `-javaagent:opentelemetry-javaagent.jar`, bytecode instrumentation at class load | SDK/Observation API calls in your code |
| Covers | servlet, HTTP clients, JDBC, Kafka clients, gRPC, Redis, AWS SDK, executors | your business boundaries |
| Code change | **zero** | yes |
| Startup cost | noticeable — bytecode rewriting adds to startup (measure it, it interacts with Lambda cold starts, file 12) | none |
| Blind to | your domain (`commandcenter.assemble`, `rule.evaluate`, `window.suppress`) | everything you didn't write |

They are complementary, and that's the answer: agent for the plumbing, a handful of hand-written spans
for the business operations that make a trace readable. A trace that is 40 JDBC spans and no domain
span tells you *where* time went but not *what* the system was doing.

**Code:**

```yaml
# otel-collector-config.yaml (Phase 8, compose profile `obs`)
processors:
  tail_sampling:
    decision_wait: 10s            # how long to buffer spans before deciding — must exceed p99 trace duration
    num_traces: 100000
    policies:
      - name: keep-all-errors
        type: status_code
        status_code: { status_codes: [ERROR] }
      - name: keep-slow-ingest            # BS-10: anything near the 300ms SLO threshold
        type: latency
        latency: { threshold_ms: 250 }
      - name: keep-sportsnow              # the enterprise-SLA tenant gets a higher rate
        type: string_attribute
        string_attribute: { key: tenant.id, values: [sportsnow] }
      - name: baseline
        type: probabilistic
        probabilistic: { sampling_percentage: 1 }
```

```properties
# JVM-side head sampling: parentbased_traceidratio means
#  - if there IS a parent decision, respect it (never split a trace)
#  - if we are the ROOT, sample at the ratio
OTEL_TRACES_SAMPLER=parentbased_traceidratio
OTEL_TRACES_SAMPLER_ARG=0.01
OTEL_SERVICE_NAME=ingest-gateway
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
```

**Follow-ups they'll ask:**
- *"Why `parentbased_*` rather than plain ratio?"* — a plain ratio sampler re-rolls the dice at every
  service, so you get fragments: service A sampled, B not, C sampled. Parent-based respects the
  upstream flag, so a trace is all-or-nothing.
- *"What's the gotcha with tail sampling behind a load balancer?"* — all spans of a trace must reach
  the *same* collector instance. You need a `loadbalancing` exporter tier routing by trace ID in front
  of the sampling collectors. Skipping that silently breaks tail sampling: it decides on partial traces.
- *"Does the agent slow things down?"* — the steady-state overhead is usually modest but non-zero
  (measure, don't quote a number); the more noticeable cost is **startup**, from class-load-time
  instrumentation. That is why it interacts badly with Lambda cold starts (file 12 Q2).
- *"Can you use the agent and the SDK together?"* — yes; the agent exposes the global
  `OpenTelemetry`/`Tracer` so manual spans join the same trace.

**Pitfalls / red flags:** "we sample at 100%" on a high-throughput pipeline (either untrue or
ruinously expensive), or proposing tail sampling with no mention of the routing requirement.

**Where in StreamPulse:** Phase 8 compose profile `obs`; the sampling policy is deliberately
tenant-aware because `sportsnow` has the SLA (BS-10).

**Verify:** Collector `tail_sampling` policy type names and the `loadbalancing` exporter config —
Collector config schema changes between releases. Verify `OTEL_TRACES_SAMPLER` values too.

---

## Q11. Prometheus pull vs Datadog push — compare them.

**30-second answer:** Prometheus **pulls**: it scrapes `/actuator/prometheus` on each target every N
seconds, which means Prometheus owns the schedule, the scrape itself is a health signal (`up` metric),
and the app holds only a current snapshot with no outbound network dependency. Datadog (and StatsD,
and OTLP push) **pushes**: the app or a local agent sends metrics out, which works where the collector
can't reach in — Lambda, batch jobs, anything behind NAT or ephemeral. The operational difference that
actually bites: with pull, **service discovery is your problem** (and solved, in Kubernetes, by the
Prometheus operator watching the API server); with push, **buffering and backpressure are your
problem** when the sink is down. In StreamPulse I use Prometheus as the default and keep a `datadog`
Spring profile, because Micrometer makes the registry a one-line swap — the instrumentation code
doesn't change.

**Deeper:**

| | Prometheus (pull) | Datadog / OTLP (push) |
|---|---|---|
| Schedule owned by | the monitoring system | the application |
| Target discovery | service discovery (K8s SD, Consul, file SD) | the app knows where to send |
| Scrape failure visible as | `up == 0` — a free liveness signal | absence of data (ambiguous: down? or network?) |
| Short-lived jobs | ❌ badly — needs a Pushgateway | ✅ naturally |
| Behind NAT / serverless | ❌ hard | ✅ fine |
| Cost model | your infra (storage, retention) | per-host + per-custom-metric — cardinality is billable |
| Data loss on outage | scrape gap; app keeps its counters, so counters resume correctly | buffered then dropped, unless the local agent persists |
| Cardinality blast radius | your Prometheus OOMs | your invoice |
| Long-term storage | Thanos / Mimir / Cortex | included |

The counter-resumption point is subtle and nice to say out loud: because Prometheus stores
**monotonic counters** and computes `rate()` at query time, a missed scrape loses resolution but not
correctness — the next scrape still shows the accumulated total. A push system sending *deltas* loses
the delta permanently if the send fails. This is why counters, not gauges, are the right primitive
for anything you bill or report on.

**Code:**

```java
// The whole point of Micrometer: the facade is the same, the backend is a dependency + profile.
// BS-01 REQ-LC-06 uses a `datadog` profile exactly for this.
```

```gradle
// build.gradle.kts
runtimeOnly("io.micrometer:micrometer-registry-prometheus")          // default
runtimeOnly("io.micrometer:micrometer-registry-datadog")             // activated by profile
```

```yaml
# application-datadog.yml  (BS-01 REQ-LC-06)
management:
  datadog:
    metrics:
      export:
        enabled: true
        api-key: ${DD_API_KEY}
        step: 30s            # push interval — the analogue of the scrape interval
        uri: https://api.datadoghq.com
---
# application.yml (default: prometheus)
management:
  prometheus:
    metrics:
      export:
        enabled: true
```

> **Your story:** you have real Datadog and Grafana on-call experience — this is the question to spend
> it on. Concrete things that land: what your dashboards were organized around (per-service RED? per
> customer?), whether you owned monitors, one alert you *deleted* because it was noise, and whether
> you ever dealt with custom-metric cost. "I've used Datadog" is a weak answer; "we cut monitor noise
> by X by moving from threshold alerts to burn-rate alerts" is a senior one.

**Follow-ups they'll ask:**
- *"When would you use a Pushgateway?"* — batch/cron jobs that exit before a scrape can happen. It is
  explicitly **not** for pushing service metrics: the Pushgateway becomes a stale-data singleton with
  no `up` signal. For StreamPulse's S3 compaction Lambda (BS-06 REQ-AW-01) a Lambda extension or the
  Datadog forwarder is the right pattern, not a Pushgateway.
- *"Where does OpenTelemetry fit?"* — OTLP is a push protocol, and the Collector can *also* expose a
  Prometheus endpoint for scraping. So the Collector is the seam that lets you push from the app and
  pull from Prometheus — which is how you avoid rewriting instrumentation when you change vendor.
- *"How do scrape interval and alert windows relate?"* — an alert on a `[5m]` rate with a 60s scrape
  interval has ~5 samples; too coarse a scrape makes short incidents invisible and makes `rate()` noisy.
  Rule of thumb: the range must cover at least 4 scrape intervals. **Verify the current Prometheus
  guidance** rather than quoting a number.

**Pitfalls / red flags:** presenting pull vs push as a religious preference rather than a set of
operational consequences; not knowing that the scrape itself is a health signal.

**Where in StreamPulse:** Phase 8; `datadog` profile in BS-01 REQ-LC-06 (needs your own Datadog trial
— the plan flags that).

---

## Q12. What does "dashboards as code" mean, and why do you care?

**30-second answer:** The dashboard JSON lives in the repo, is reviewed in a PR, and is applied by
CI or by Grafana's provisioning/sidecar rather than clicked together in the UI. I care for three
reasons: dashboards drift and rot when they're not versioned, a new service should inherit the
standard RED dashboard automatically instead of someone rebuilding it from memory, and — the real one —
**alert rules and recording rules must be reviewable**, because a bad alert threshold is a production
change. In StreamPulse the Prometheus recording rules that define the SLIs are the source of truth for
the SLA report, so they're tested in CI with `promtool test rules` (REQ-SL-01, and the acceptance test
list names it explicitly).

**Deeper:**

The practical stack:

| Layer | Tool | Lives in |
|---|---|---|
| Dashboard definition | Grafana JSON, or Grafonnet/jsonnet, or Terraform `grafana_dashboard` | `ops/grafana/dashboards/*.json` |
| Provisioning | Grafana file provisioning, or the sidecar reading ConfigMaps | `ops/grafana/provisioning/` |
| Recording rules (the SLIs) | Prometheus rules YAML | `ops/prometheus/rules/slo.yml` |
| Rule tests | `promtool test rules` in CI | `ops/prometheus/rules/slo_test.yml` |
| Alert routing | Alertmanager config | `ops/alertmanager/` |

**Recording rules** deserve a sentence of their own: they pre-compute an expensive query at scrape time
into a new series. For an SLO you want this, because the SLI expression is evaluated by every alert
rule, every dashboard panel and the SLA report endpoint — computing it once and naming it
`tenant:ingest_availability:ratio_rate5m` is both faster and, more importantly, gives you **one
definition** of what availability means. Three copies of an SLI expression is three subtly different
SLOs.

**Code:**

```yaml
# ops/prometheus/rules/slo.yml — BS-10 REQ-SL-01
groups:
  - name: streampulse-sli
    interval: 30s
    rules:
      # ── SLI 1: ingest availability, per tenant ────────────────────────────
      # "Good" = 2xx, PLUS 429s that are the tenant hitting their OWN contracted
      # quota (a correct response to an over-quota client is not our outage).
      # This judgement call is documented in docs/system-health.md — see Q13.
      - record: tenant:ingest_requests_good:rate5m
        expr: |
          sum by (tenant) (rate(http_server_requests_seconds_count{
            uri="/v1/events:batch", status=~"2.."}[5m]))
          +
          sum by (tenant) (rate(ingest_requests_throttled_total{
            reason="tenant_quota"}[5m]))

      - record: tenant:ingest_requests_valid:rate5m
        expr: |
          sum by (tenant) (rate(http_server_requests_seconds_count{
            uri="/v1/events:batch"}[5m]))
          - sum by (tenant) (rate(http_server_requests_seconds_count{
            uri="/v1/events:batch", status=~"4.."}[5m]))
          + sum by (tenant) (rate(ingest_requests_throttled_total{
            reason="tenant_quota"}[5m]))
          # 4xx caused by a malformed client payload is the CLIENT's bug, not our
          # unavailability — so it is excluded from the denominator entirely.

      - record: tenant:ingest_availability:ratio_rate5m
        expr: tenant:ingest_requests_good:rate5m / tenant:ingest_requests_valid:rate5m

      # ── SLI 2: freshness (BS-10: dashboard freshness < 10s) ───────────────
      - record: tenant:freshness_good:ratio_rate5m
        expr: |
          sum by (tenant) (rate(streampulse_freshness_seconds_bucket{le="10.0"}[5m]))
          /
          sum by (tenant) (rate(streampulse_freshness_seconds_count[5m]))
```

```yaml
# ops/prometheus/rules/slo_test.yml — run by `promtool test rules` in CI
# This is the bit that makes the SLO trustworthy: the definition has a unit test.
rule_files: [ slo.yml ]
evaluation_interval: 30s
tests:
  - interval: 30s
    input_series:
      - series: 'http_server_requests_seconds_count{uri="/v1/events:batch",status="200",tenant="sportsnow"}'
        values: '0+100x20'                       # 100 good requests per interval
      - series: 'http_server_requests_seconds_count{uri="/v1/events:batch",status="503",tenant="sportsnow"}'
        values: '0+1x20'                         # 1 server error per interval
    promql_expr_test:
      - expr: tenant:ingest_availability:ratio_rate5m
        eval_time: 5m
        exp_samples:
          - labels: 'tenant:ingest_availability:ratio_rate5m{tenant="sportsnow"}'
            value: 0.9900990099009901            # 100/101 — the SLI does what we claim
```

**Follow-ups they'll ask:**
- *"How do you stop 200 people making 200 dashboards?"* — a templated base dashboard (one per service
  *archetype*: HTTP service, stream processor) with `$service` and `$tenant` template variables, plus a
  review rule that a new panel on the shared dashboard needs a reason. Personal exploration happens in
  Explore, not in a saved dashboard.
- *"What about Grafana's own alerting vs Prometheus alerting rules?"* — both work; I prefer rules in
  Prometheus/Mimir so the alert definition lives next to the recording rule it depends on and is
  testable with `promtool`. Grafana alerting is friendlier for multi-datasource conditions.
- *"How do you handle a dashboard someone edited in the UI?"* — provisioned dashboards can be marked
  non-editable, which makes the drift impossible rather than merely discouraged.

**Pitfalls / red flags:** "our dashboards are in Grafana" as if that were an answer; no version control
on alert thresholds; three different definitions of "availability" in three panels.

**Where in StreamPulse:** Phase 8 — "Grafana dashboards-as-code: RED per service, Kafka consumer lag,
end-to-end freshness, JVM, DLT rate"; BS-10 acceptance test `promtool test rules`.

---

## Q13. SLI, SLO, SLA, error budget — define them, and tell me how you'd choose a good SLI.

**30-second answer:** An **SLI** is a *measurement* — a ratio of good events to valid events, like
"fraction of ingest requests that returned 2xx within 300ms". An **SLO** is an *internal target* on
that measurement over a window — "99.98% over 30 rolling days". An **SLA** is an *external contract*
built on an SLO, with **financial consequences** — service credits — and it is always set looser than
the SLO, because you want your own alarm to fire well before the customer's lawyer does. The **error
budget** is the inverse of the SLO: `1 − SLO` — the amount of unreliability you are *allowed* to spend,
which converts reliability from a moral argument into a quantity you can budget, spend on releases, and
run out of.

**Deeper:**

```
        SLI  ── measurement  ───▶ 99.993% this month
                    │
        SLO  ── internal target ─▶ 99.98%     ← alert here, freeze releases here
                    │   (gap = your safety margin)
        SLA  ── customer contract ▶ 99.9%     ← service credits here
```

| | SLI | SLO | SLA |
|---|---|---|---|
| Is a | metric | target | contract |
| Owned by | engineering | engineering + product | legal/sales |
| Consequence of missing | none, it's a number | error budget spent, releases freeze | **money** |
| Should be | few, user-centric | achievable and tight | looser than the SLO |

**Choosing a good SLI — the actual senior content.** Three tests:

1. **Does the user feel it?** If the SLI goes down, does a human notice? CPU utilization fails this
   test. "Fraction of dashboard updates delivered within 10s" passes.
2. **Is it a ratio of good events to valid events?** This shape is what makes it aggregatable,
   comparable across traffic levels, and directly convertible into a budget.
3. **Is it measured as close to the user as possible?** A server-side latency SLI misses load-balancer
   queuing, TLS, and the client's network. Server-side is usually the pragmatic choice (it's the part
   you control), but say out loud that you know what it excludes.

**Request-based vs window-based availability** — the distinction people miss:

| | Request-based | Window-based |
|---|---|---|
| Definition | good requests / valid requests over the period | good *minutes* / total minutes, where a minute is "good" if its error rate is under a threshold |
| Budget unit | requests | **minutes** — which is how SLAs are usually written |
| Bias | a low-traffic outage barely dents the budget | a 1-minute total outage at 3am costs the same as at peak |
| Good for | engineering SLOs | contract language, incident accounting |

The 8.64 minutes in BS-10's contract is a *window-based* framing; the Prometheus ratio in Q12 is
*request-based*. You need both and should say so: report the request-based SLI for engineering, and
translate to minutes for the SLA report (REQ-SL-06).

**The 429 judgement call (REQ-SL-01 explicitly asks for this to be justified).** StreamPulse returns
429 + `Retry-After` in two very different situations:

| Situation | Is it a "good" request? | Reasoning |
|---|---|---|
| Tenant exceeded their **contracted quota** (Bucket4j, BS-04 REQ-RS-04) | ✅ **good** | The system worked exactly as specified. Counting this as our unavailability means a tenant hammering their own quota could breach our SLA — and worse, it creates an incentive to remove quotas. |
| Ingest shed load because our **in-flight semaphore was saturated** (BS-02 backpressure) | ❌ **bad** | This is us failing to serve work we agreed to accept. It is unavailability wearing a 429 costume. |

The mechanism that makes this decidable is that they must be **different metrics**, not one 429
counter: `ingest_requests_throttled_total{reason="tenant_quota"}` vs `{reason="overload"}`. If you only
have the HTTP status code you cannot tell them apart, and the SLI is unfixable. That's a design
decision you make on day one, and it is a genuinely good thing to volunteer in the interview.

Same logic for **4xx**: a malformed payload is the client's bug and belongs in *neither* numerator nor
denominator — it's not a valid request. Leaving 400s in the denominator means a buggy client release
silently improves or degrades your SLI.

**Code:**

```java
/**
 * BS-10 REQ-SL-01: the two 429s are DIFFERENT metrics, because they mean
 * different things for the SLI. One is contract-conformant, one is an outage.
 */
@ExceptionHandler(QuotaExceededException.class)
ResponseEntity<ProblemDetail> quota(QuotaExceededException e) {
    // Tenant used their own contracted quota → GOOD request for SLI purposes.
    registry.counter("ingest.requests.throttled", "reason", "tenant_quota").increment();
    return retryAfter(e.retryAfter(), "Tenant quota exceeded");
}

@ExceptionHandler(BackpressureRejectedException.class)
ResponseEntity<ProblemDetail> overload(BackpressureRejectedException e) {
    // WE could not accept work we promised to accept → BAD request, burns budget.
    registry.counter("ingest.requests.throttled", "reason", "overload").increment();
    return retryAfter(Duration.ofSeconds(1), "Service overloaded");
}
```

**Follow-ups they'll ask:**
- *"How many SLOs should a service have?"* — few. Two or three. Every SLO you add dilutes attention and
  adds a way to be "in breach" for something nobody cares about. StreamPulse has exactly two:
  availability and freshness.
- *"Rolling window or calendar month?"* — rolling (e.g. 30 days) for engineering, because a calendar
  month resets the budget at midnight on the 1st and creates a perverse incentive to ship risky changes
  on the 31st. Calendar for the SLA report, because that's how the contract is written. Again: both.
- *"Why is 100% never the target?"* — because the cost curve is exponential and because your users'
  own reliability (ISP, device, client app) is lower, so the marginal nine is invisible to them. A 100%
  SLO also means zero error budget, which means you can never ship.
- *"Who should own the SLO?"* — the team that can change the system. An SLO imposed by a manager with
  no engineering input gets gamed.

**Pitfalls / red flags:** using SLA and SLO interchangeably. Choosing CPU as an SLI. Not being able to
say what "good" means for a request.

**Where in StreamPulse:** BS-10 / REQ-SL-01, REQ-SL-06; `docs/system-health.md`.

---

## Q14. Show me the error-budget arithmetic, and how you turn it into alerts and a release policy.

**30-second answer:** 99.98% over a 30-day month means 0.02% of 43,200 minutes, which is **8.64 minutes**
of allowed unavailability per month. That number is the whole point: it's small enough that a single
bad deploy can eat the month, which is why the policy has to be "deploy safely" rather than "deploy
less". I turn it into alerting with **multi-window, multi-burn-rate** alerts — page when we're burning
budget fast enough to exhaust it quickly, with a short second window to confirm the burn is still
happening, so a 30-second blip doesn't page. And I tie it to releases: below 25% budget remaining, the
CD pipeline blocks non-fix deploys until the budget recovers.

**Deeper:**

### The arithmetic — showing the working

A 30-day month is `30 × 24 × 60 = 43,200 minutes`.

| SLO | Unavailability allowed | Minutes / 30 days | Minutes / week | Minutes / day |
|---|---|---|---|---|
| 99.9% | 0.1% | `0.001 × 43,200` = **43.2 min** | `0.001 × 10,080` = 10.08 min | 1.44 min |
| 99.95% | 0.05% | `0.0005 × 43,200` = **21.6 min** | 5.04 min | 0.72 min |
| **99.98%** (BS-10) | **0.02%** | `0.0002 × 43,200` = **8.64 min** | `0.0002 × 10,080` = **2.016 min** | **0.288 min ≈ 17.3 s** |
| 99.99% | 0.01% | `0.0001 × 43,200` = **4.32 min** | 1.008 min | 0.144 min |

The **request-based** equivalent (REQ-SL-02 asks for both): at a StreamPulse design-target load of
50,000 events/s arriving in batches of 500, that's 100 requests/s → `100 × 86,400 × 30 = 259,200,000`
requests in the month. The budget is `0.0002 × 259,200,000 = 51,840` failed requests. Stating it both
ways is useful because "8.64 minutes" sounds tiny and terrifying while "51,840 requests" sounds
manageable — and the second is the one you can actually measure per-tenant.

*(The 50k ev/s is a StreamPulse **design target** chosen in the plan, not a measured number; Phase 9
replaces it with real Gatling data. The minute arithmetic above, by contrast, is exact.)*

### Burn rate

**Burn rate** = how fast you are consuming budget relative to consuming it evenly across the window.
Burn rate 1 means you will exactly exhaust the budget at the end of the month. Burn rate 14.4 means
you'll exhaust it in `720 h / 14.4 = 50 hours`.

The conversion to an error rate is the bit to be able to do live:

```
burn_rate = observed_error_ratio / (1 − SLO)

For SLO = 99.98%:   1 − SLO = 0.0002
  burn rate  1   →  error ratio 0.0002  (0.02%)
  burn rate  6   →  error ratio 0.0012  (0.12%)
  burn rate 14.4 →  error ratio 0.00288 (0.288%)
```

And the budget consumed by a burn of rate B sustained for time T over a 30-day (720-hour) window:

```
budget_consumed = B × T / 720h

  14.4 × 1h  / 720 =  2%  of the month's budget
   6   × 6h  / 720 =  5%
   3   × 24h / 720 = 10%
   1   × 72h / 720 = 10%
```

### Multi-window, multi-burn-rate alerting

The pattern is from the **Google SRE Workbook** (chapter on alerting on SLOs). Two windows per alert:
a **long** window that defines the burn condition, and a **short** window (conventionally 1/12 of the
long one) that must *also* be burning, so the alert **resets quickly** once the incident ends instead
of staying firing for the rest of the long window.

| Severity | Long window | Short window | Burn rate | Budget consumed if sustained | Why |
|---|---|---|---|---|---|
| Page | 1 h | 5 m | 14.4× | 2% | fast burn — respond now |
| Page | 6 h | 30 m | 6× | 5% | sustained medium burn |
| Ticket | 1 d | 2 h | 3× | 10% | slow leak, fix this week |
| Ticket | 3 d | 6 h | 1× | 10% | very slow leak |

> **I am recalling these factors from the SRE Workbook and I am not fully certain of the exact
> recommended numbers or window pairings. Verify them in the Workbook (Google SRE Workbook,
> "Alerting on SLOs") before quoting them in an interview or committing them to
> `ops/prometheus/rules/`.** The *structure* — two windows, several burn rates, fast burns page and
> slow burns ticket — is the part I would defend; the specific constants are tuning.

**Why alert on symptoms, not causes.** A cause-based alert ("Kafka broker 2 is down", "CPU > 80%")
fires whether or not users are affected, so it generates pages for non-events and trains people to
ignore the pager. A symptom-based alert fires exactly when the SLI degrades, so every page
corresponds to real user impact. The practical rule: **pages come from SLO burn; everything else is a
dashboard or a ticket**. Cause-based signals are still valuable — they're what you look at *after* the
symptom alert fires, and they're why USE metrics exist (Q6). The one legitimate exception is a
**predictive** cause alert for something that is unrecoverable and has a hard deadline: disk filling at
a rate that hits 100% in 4 hours, or a certificate expiring. Those aren't symptoms yet, but by the
time they are, it's too late.

**Code:**

```yaml
# ops/prometheus/rules/slo-alerts.yml — BS-10 REQ-SL-03
groups:
  - name: streampulse-slo-burn
    rules:
      - alert: IngestErrorBudgetFastBurn
        # Uses the RECORDING RULE from Q12 so there is exactly one definition of "availability".
        # 1h long window AND 5m short window must BOTH exceed 14.4x burn.
        expr: |
          (
            (1 - tenant:ingest_availability:ratio_rate1h) > (14.4 * 0.0002)
            and
            (1 - tenant:ingest_availability:ratio_rate5m) > (14.4 * 0.0002)
          )
        for: 2m
        labels:
          severity: page
          slo: ingest_availability
        annotations:
          summary: "{{ $labels.tenant }} burning ingest error budget at >14.4x"
          description: "At this rate the 30-day budget (8.64 min) is gone in ~50h."
          runbook: "https://github.com/.../docs/runbook.md#ingest-availability"

      - alert: IngestErrorBudgetSlowBurn
        expr: |
          (
            (1 - tenant:ingest_availability:ratio_rate6h) > (6 * 0.0002)
            and
            (1 - tenant:ingest_availability:ratio_rate30m) > (6 * 0.0002)
          )
        for: 15m
        labels: { severity: page, slo: ingest_availability }
```

```yaml
# .github/workflows/deploy.yml — BS-10 REQ-SL-04, the freeze gate
- name: Error budget gate
  run: |
    # Remaining budget = 1 - (consumed / allowed). Query the 30-day SLI.
    SLI=$(curl -sG "$PROM/api/v1/query" \
          --data-urlencode 'query=tenant:ingest_availability:ratio_rate30d{tenant="sportsnow"}' \
          | jq -r '.data.result[0].value[1]')
    # consumed fraction = (1 - SLI) / 0.0002
    CONSUMED=$(echo "(1 - $SLI) / 0.0002" | bc -l)
    REMAINING=$(echo "1 - $CONSUMED" | bc -l)
    echo "Error budget remaining: $REMAINING"
    if (( $(echo "$REMAINING < 0.25" | bc -l) )) && [[ "${{ github.event.pull_request.labels }}" != *"slo-override"* ]]; then
      echo "::error::Budget below 25% — non-fix releases are frozen (REQ-SL-04)."
      exit 1
    fi
```

**Follow-ups they'll ask:**
- *"Isn't a release freeze just a way to stop shipping?"* — the opposite, and this is the
  argument to make: the budget makes reliability *negotiable with data*. Above budget, product wins
  arguments and we ship faster. Below budget, reliability work is automatically prioritized without
  anyone having to win a political fight. The freeze is the forcing function, and there's always a
  labelled override for a genuine emergency fix (which is itself an audit trail).
- *"What if the budget is never used?"* — that's a signal too: you're over-invested in reliability and
  could be shipping faster or running cheaper. An unused budget is a wasted budget.
- *"How do you handle a single huge incident that eats the whole month?"* — the budget is spent; you
  say so honestly, run the postmortem, and the freeze does its job. Some orgs allow an
  "exclusion" for an upstream-provider outage, but that must be a pre-agreed policy in the SLA
  document, not a retroactive argument.
- *"Why `for: 2m` on an alert that already has a 5-minute window?"* — belt and braces against a
  single evaluation-cycle spike; keep it short, because on a 14.4× burn you are spending budget at
  2%/hour while you wait.
- *"What's the alternative to burn-rate alerts?"* — a fixed threshold ("error rate > 1%"), which is
  either too sensitive at low traffic or too slow at high traffic, and has no relationship to whether
  you'll actually breach. Burn rate normalizes by the SLO.

**Pitfalls / red flags:** quoting "three nines is 43 minutes" without being able to derive it; alerting
on a single window (either flaps or is slow); no policy attached to the budget, which makes it
decoration.

**Where in StreamPulse:** BS-10 / REQ-SL-02, REQ-SL-03, REQ-SL-04; `docs/system-health.md`; the
acceptance tests `promtool test rules` and the deploy-gate workflow dry run.

**Verify:** the exact burn-rate factors and window pairings in the Google SRE Workbook. Also verify
`promtool test rules` syntax for your Prometheus version.

---

## Q15. In StreamPulse, what metrics define system health, and how would you prove 99.98% is achievable?

*(The applied question — and, per the prep doc, one the panel is likely to ask almost verbatim.)*

**30-second answer:** Four user-facing metrics, in the order I'd put them on the dashboard.
**Freshness** — the time from event timestamp to SSE delivery, with an SLO bucket at 10 seconds; this
is the only metric that tells me whether the product is doing its job. **Ingest availability** — good
requests over valid requests, per tenant, with the 429 distinction I described. **Consumer lag** —
per consumer group and partition, because it's the leading indicator that freshness is about to break.
And **DLT rate** — the share of records that failed terminally, because a pipeline that is fast and
available while silently dropping 2% of events is not healthy. Everything else — CPU, heap, GC pause,
RocksDB size, Hikari pool — is on the dashboard for diagnosis and pages nobody. Then to prove 99.98% is
achievable I don't argue, I run the chaos test: kill one of three pods every two minutes under load and
measure what fraction of the monthly budget it consumes. The NFR in the plan says under 10%.

**Deeper:**

### The health model, top to bottom

```
USER-FACING (SLIs — these page)
  freshness_seconds                 event.ts → SSE delivery,  SLO bucket at 10s
  ingest_availability               good/valid per tenant,    SLO 99.98%
  dlt_rate                          terminal failures / consumed
  alert_delivery_latency            rule fired → partner notified (BS-07)

LEADING INDICATORS (these ticket, and they feed the dashboards)
  kafka_consumer_lag                per group/partition — predicts freshness breach
  ingest_inflight_batches           the BS-02 semaphore — predicts 429 overload
  circuit_breaker_state             BS-04 — predicts degraded writes
  s3_spool_bytes                    BS-04 fallback depth — predicts data at risk
  rebalance_rate                    predicts lag spikes

RESOURCE / USE (diagnosis only — no pages)
  jvm_gc_pause_seconds, jvm_memory_used, virtual thread counts
  hikaricp_connections_pending, executor_queued (BS-02 S3 pool)
  rocksdb_estimate_num_keys, state store size
  broker disk, network bytes (BS-03's cost proxy)
```

### Why freshness is the headline metric, and why it's subtle to measure

Freshness is `deliveryTime − event.eventTime`, which spans four hops and two clocks. The trap:
`eventTime` comes from a **client device**, and client clocks are wrong — sometimes by hours. So a raw
freshness metric is polluted by clock skew, and a device with a clock set to 1970 produces a freshness
of 56 years and destroys your histogram. Three mitigations, and having an answer here is what makes
this a senior answer rather than "we measure end-to-end latency":

1. Ingest stamps a **server-side `receivedAt`** on every event. Report *two* freshness metrics:
   `receivedAt → delivery` (what we control, the one with the SLO) and `eventTime → receivedAt`
   (client-side lag, an informational metric about SDK/network health, deliberately **not** in the SLO).
2. Clamp and count: events whose `eventTime` is outside a sane window are counted in
   `ingest_clock_skew_total{direction=future|past}` and excluded from the SLI histogram.
3. Measure at the **last hop** — the SSE emit — not at the sink write, because the SSE fan-out
   (`Sinks.Many` with `onBackpressureLatest`, Phase 5) is itself a place latency can hide.

### The chaos proof (the part that converts a claim into evidence)

`PodKillChaosTest` (BS-10 acceptance tests) on a `kind` cluster: steady Gatling load, kill 1 of 3 pods
every 2 minutes, run for 30 minutes, and measure the *actual* SLI. If a pod kill costs, say, 3 seconds
of 503s, then 15 kills cost 45 seconds — about 8.7% of the 8.64-minute budget. That's a real number I
can put in a doc and defend, and it is the difference between "we designed for 99.98%" and "here is
what one disruption costs us in budget minutes".

### What actually keeps it there (REQ-SL-05, each item mapped to what it absorbs)

| Mechanism | Failure it absorbs | Where |
|---|---|---|
| `maxUnavailable: 0` rolling update | capacity dip during deploy | Phase 11 Helm |
| PodDisruptionBudget `minAvailable: 2` | voluntary disruption (node drain, cluster upgrade) | Phase 11 |
| Readiness gate + ordered drain | in-flight requests killed mid-deploy | BS-01 REQ-LC-01 |
| Topology spread across zones | single-AZ/node failure | Phase 11 |
| HPA/KEDA headroom on consumer lag | traffic spike (the "live match kickoff" 10×) | Phase 11 |
| Retries + idempotency end to end | transient dependency failure | BS-04 REQ-RS-01, BS-03 REQ-KT-05 |
| Circuit breaker + local spool | 3-minute S3 outage without data loss | BS-04 REQ-RS-02 |
| Bulkhead per tenant | one tenant's partner outage blocking others | BS-04 REQ-RS-05 |

The connective point to make out loud: **the SLO is not achieved by monitoring, it is achieved by the
mechanisms above; monitoring only tells you whether they worked.** People who conflate the two propose
better dashboards as a fix for reliability problems.

**Follow-ups they'll ask:**
- *"What single metric would you put on a wall display?"* — freshness p99 per tenant. If freshness is
  good, the pipeline is delivering; almost every other failure eventually shows up there.
- *"Lag is 2 million records. Is that bad?"* — I can't tell yet, and that's the right first answer.
  Two million records at 100k/s of drain capacity is 20 seconds of lag and self-healing; at 5k/s it's
  400 seconds and growing. What I actually alert on is **lag in time** (lag / drain rate) or, better,
  freshness directly — because the record count is meaningless without throughput.
- *"DLT rate is 0. Good?"* — suspicious. Either the pipeline is genuinely clean, or the DLT path is
  broken, or errors are being swallowed. I'd assert on it: a synthetic poison message in the smoke test
  that must land in the DLT proves the path works (the plan has a poison-message test in Phase 4).
- *"How do you report this to the account manager?"* — `GET /v1/sla/{tenant}?month=` (REQ-SL-06):
  SLI values, budget consumed in both minutes and requests, and the incident list from alert history.
  A per-tenant Grafana SLO dashboard for the same data.
- *"What would you add if you had one more week?"* — synthetic probes. Every metric above is derived
  from real traffic, so at 3am with no traffic I'd have no signal at all. A synthetic client posting a
  known event every 30 seconds and asserting it appears on the SSE stream within 10s gives a
  traffic-independent availability and freshness signal — and it's the only thing that catches "the
  whole pipeline is down so there's no data to alert on."

> **Your story:** the NetApp 99.98% line on your resume is the natural bridge here — but present the
> StreamPulse version as *"I rebuilt this in Java to prove I can define and defend the number, not
> just quote it."* If you personally owned monitors or an on-call rotation in Datadog/Grafana, the
> strongest thing you can say is a specific alert you tuned or deleted and the noise reduction it
> bought. Keep "I built" for what you wrote yourself.

**Pitfalls / red flags:** answering "CPU, memory, disk, network" — that is a sysadmin answer to a
product question. Also: naming lag without naming freshness, and having no way to prove the SLO is
achievable other than assertion.

**Where in StreamPulse:** BS-10 end to end; REQ-SL-01..06; Phase 8 (`docs/system-health.md`, Grafana
dashboards-as-code) and Phase 11 (the mechanisms); acceptance tests `promtool test rules`,
`PodKillChaosTest`, `SlaReportIT`, deploy-gate dry run.

---

## Key terms

| Term | One-line meaning |
|---|---|
| **RED** | Rate, Errors, Duration — the symptom view for request-driven services. |
| **USE** | Utilization, Saturation, Errors — the cause view for resources. |
| **Four Golden Signals** | Latency, traffic, errors, saturation (Google SRE's variant of RED). |
| **Cardinality explosion** | Unbounded tag values creating one time series per value; kills Prometheus, inflates Datadog bills. |
| **`LongTaskTimer`** | The only Micrometer meter that shows an operation that has started but not finished. |
| **`publishPercentileHistogram`** | Exports aggregatable buckets so Prometheus computes fleet-wide percentiles. |
| **`serviceLevelObjectives`** | Adds a bucket edge exactly on an SLO threshold, turning the SLI into a clean counter ratio. |
| **Exemplar** | A trace ID attached to a histogram sample, linking a metric spike to one real trace. |
| **MDC** | SLF4J's thread-local log context; lost across thread hand-offs unless propagated. |
| **W3C `traceparent`** | `version-traceId-spanId-flags`; the standard trace propagation header. |
| **Span link** | "Caused by these N traces" — the correct shape for batch consumers. |
| **Baggage** | Key/value data propagated *across services* with the trace; keep it tiny, never sensitive. |
| **Head vs tail sampling** | Decide at trace start (cheap, blind) vs after completion in the collector (keeps errors, needs trace-ID routing). |
| **Recording rule** | A pre-computed Prometheus series; the single source of truth for an SLI definition. |
| **SLI / SLO / SLA** | Measurement / internal target / external contract with money attached. |
| **Error budget** | `1 − SLO`; the allowed unreliability, spendable on releases. |
| **Burn rate** | `observed_error_ratio / (1 − SLO)`; 1× exhausts the budget exactly at window end. |
| **Multi-window multi-burn-rate** | Long window defines the burn, short window confirms it's ongoing; fast burns page, slow burns ticket. |
| **Freshness** | `deliveryTime − eventTime`; the user-facing latency metric for a pipeline, distinct from lag. |

---

## Self-check (answer out loud, no notes)

1. Why can't you average p99 across pods, and what do you export instead?
2. Derive the monthly error budget for 99.98%, then for 99.99%, showing the arithmetic.
3. A 429 is returned. Is it a good request or a bad one for the SLI — and what makes that question
   answerable at all?
4. How does a trace get from ingest-gateway to sink-writer across Kafka, and why is a batch listener
   a special case?
5. Name the four user-facing health metrics for StreamPulse and say which one you'd put on a wall
   display and why.
