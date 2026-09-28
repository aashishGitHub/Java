# 05 — Resilience4j: Circuit Breaker, Retry, Rate Limiter, Bulkhead, TimeLimiter

**Why this matters for an Event Hub role:** an ingest pipeline is a machine made almost entirely of
*other people's dependencies* — DynamoDB, S3, Redis, a partner webhook, a schema registry. The
interesting question is never "do you know what a circuit breaker is"; it's **what happens to the rest
of the system when one of those is slow**. Slow is worse than down: a dependency that returns in 30
seconds instead of failing in 30 milliseconds will consume every thread you have and take out
functionality that has nothing to do with it. Resilience4j is how you bound that blast radius, and the
senior signal is being able to name the config values, the *order* the decorators compose in, and the
cases where a fallback is the wrong answer.

**Scenario anchor — BS-04 (REQ-RS-01…09):** during a big match, four things go wrong at once.
(a) DynamoDB throttles writes. (b) S3 returns 503s for three minutes. (c) A partner webhook — a
broadcaster's ops pager — allows only 50 req/s and gets slow. (d) Those slow partner calls exhaust
threads and stall alert evaluation **for every tenant**, including `kidsplus`, who has nothing to do
with the partner. The NFRs: zero raw events lost during the S3 outage (spool-to-disk fallback +
replay), and `kidsplus` alert latency unaffected (< 2s p99) while `sportsnow`'s partner is down.
Phases 2, 4, 6, 9.

> **Version note:** Resilience4j 2.x is the Java 17+ / Spring Boot 3 line
> (`io.github.resilience4j:resilience4j-spring-boot3`); 1.7.x is the Boot 2 line. Every default value
> below is flagged for verification — the library documents them, but they have changed across
> versions and a wrong default stated confidently is worse than "I'd look it up".
>
> **Verify (applies to the whole file):** all default config values, the exact Micrometer metric
> names, and the default aspect order. I flag the specific ones again where they matter.

---

## Q1. You have five Resilience4j modules available. Map each one to the failure it actually guards against.

**30-second answer:** They are not interchangeable and they don't overlap much. **TimeLimiter** bounds
how long a single call can take — it's the foundation, because without a timeout every other pattern
is defeated by a call that simply never returns. **Retry** handles *transient* failure: the call
would succeed if you tried again. **CircuitBreaker** handles *sustained* failure: stop calling a
dependency that is clearly broken, so you fail fast instead of piling up. **Bulkhead** bounds
*concurrency*, so one slow dependency can't consume every thread and take down unrelated work — that's
the isolation pattern. **RateLimiter** bounds *call rate*, which is a contract concern, not a failure
concern: it's how you honour "50 req/s" to a partner. The mental model I use: TimeLimiter and
Bulkhead protect **me** from **them**; RateLimiter protects **them** from **me**; CircuitBreaker and
Retry decide **whether to call at all**.

**Deeper:** REQ-RS-09 asks for exactly this table in `docs/resilience.md`:

| Pattern | Guards against | Bounds | BS-04 instance | What the user sees |
|---|---|---|---|---|
| **TimeLimiter** | a call that hangs forever | **time per call** | every outbound call, 2s default (REQ-RS-06) | a fast error instead of a spinner |
| **Retry** | transient blips (throttling, a 503, a connection reset) | **attempts** | DynamoDB throttling (REQ-RS-01) | nothing — it self-heals |
| **CircuitBreaker** | sustained failure; cascading pile-up | **whether we call at all** | S3 503s for 3 min (REQ-RS-02) | degraded mode: spooled to disk, replayed later |
| **Bulkhead** | one slow dependency eating all threads | **concurrency** | per-tenant partner isolation (REQ-RS-05) | `kidsplus` alerts unaffected by `sportsnow` |
| **RateLimiter** | *us* exceeding a contracted rate | **calls per period** | partner at 50 req/s (REQ-RS-03) | delayed (queued) delivery, not a 429 from the partner |

**The ordering of importance**, which is a good thing to volunteer: if I could only have one, it's the
**timeout**. A system with timeouts and no circuit breakers degrades; a system with circuit breakers
and no timeouts doesn't — the breaker's own failure-rate window never fills because the calls never
return to be counted. That's why REQ-RS-06 is an ArchUnit rule ("no `HttpClient` without a timeout"),
not a code-review convention.

**Code:** the full BS-04 wiring, so the rest of the file has a reference point:

```yaml
# ingest-gateway / sink-writer / alert-engine — BS-04.
# Each instance name maps to exactly one dependency. Never share an instance across
# dependencies: a shared breaker means S3's outage opens the circuit for DynamoDB too.
resilience4j:
  timelimiter:
    instances:
      s3: { timeoutDuration: 2s }
      dynamodb: { timeoutDuration: 1s }
      partnerWebhook: { timeoutDuration: 3s }
  retry:
    instances:
      dynamodb: { maxAttempts: 5 }         # detail in Q5
  circuitbreaker:
    instances:
      s3: { slidingWindowType: COUNT_BASED } # detail in Q2/Q3
  bulkhead:
    instances:
      queryApiPanels: { maxConcurrentCalls: 16 }
  thread-pool-bulkhead:
    instances:
      partner-sportsnow: { coreThreadPoolSize: 2, maxThreadPoolSize: 4, queueCapacity: 20 }
  ratelimiter:
    instances:
      partnerWebhook: { limitForPeriod: 17, limitRefreshPeriod: 1s, timeoutDuration: 0 }
```

**Follow-ups they'll ask:**
- *"Which one would you add to a legacy service first?"* — timeouts, then bulkheads. Circuit breakers
  are more visible but do less if the first two are missing.
- *"Resilience4j vs Hystrix?"* — Hystrix is in maintenance mode; Resilience4j is the functional
  successor (Vavr-influenced, decorator-based, no thread pool forced on you, first-class Micrometer
  integration). Also: Hystrix forced thread isolation; R4j lets you choose semaphore or thread-pool
  bulkhead, which matters a lot with virtual threads (Q14).
- *"Isn't this what a service mesh does?"* — a mesh (Envoy/Istio) gives you timeouts, retries, outlier
  detection and rate limiting at the network layer, without code. What it can't do is
  application-semantic decisions: "retry this exception but not that one", "fall back to a disk
  spool", "isolate per *tenant*". I'd use both, and I'd be careful not to stack mesh retries on top of
  application retries (Q6's amplification problem).

**Pitfalls / red flags:** treating all five as "resilience stuff" without distinguishing what each
bounds.

**Where in StreamPulse:** BS-04 / REQ-RS-09, `docs/resilience.md`.

---

## Q2. Walk me through the circuit breaker state machine — all the states.

**30-second answer:** Three operational states plus two manual-override states. **CLOSED** is normal:
calls pass through and results are recorded in a sliding window. When the failure rate (or slow-call
rate) over that window exceeds the threshold — *and* a minimum number of calls has been recorded —
the breaker trips to **OPEN**. In OPEN, calls are **not attempted at all**: they fail immediately with
`CallNotPermittedException`, which is the point — you fail in microseconds instead of burning a
2-second timeout per request. After `waitDurationInOpenState` the breaker moves to **HALF_OPEN** and
allows a limited number of probe calls through; if their failure rate is below the threshold it goes
back to CLOSED, otherwise straight back to OPEN. On top of that, **DISABLED** (always allow, record
nothing) and **FORCED_OPEN** (always reject) are manual states for operators — a kill switch you can
flip without a deploy.

**Deeper:**

```
                   failure rate ≥ threshold
                   (and calls ≥ minimumNumberOfCalls)
        ┌──────────────────────────────────────────────┐
        │                                              ▼
   ┌─────────┐                                   ┌──────────┐
   │ CLOSED  │                                   │   OPEN   │  ← calls rejected immediately
   │ record  │                                   │  fail    │    (CallNotPermittedException)
   │ results │                                   │  fast    │
   └─────────┘                                   └──────────┘
        ▲                                              │
        │ probe failure rate < threshold               │ waitDurationInOpenState elapses
        │                                              │ (the transition happens on the NEXT CALL
        │                                              │  unless automaticTransitionFrom
        │                                              │  OpenToHalfOpenEnabled=true)
        │              ┌──────────────┐                │
        └──────────────│  HALF_OPEN   │◀───────────────┘
                       │ allow N probe│
                       │ calls only   │──── probe failure rate ≥ threshold ──▶ back to OPEN
                       └──────────────┘

  manual/operator states (not reached automatically):
    DISABLED     — always allow, record nothing  (kill the breaker without a deploy)
    FORCED_OPEN  — always reject                 (take a dependency out of rotation)
```

**The detail most candidates miss:** the OPEN → HALF_OPEN transition is, by default, **lazy** — it
happens when a call arrives after the wait duration has elapsed, not on a timer. On a low-traffic
endpoint that means the breaker can sit in OPEN long after it could have recovered, because nothing
triggered the check. `automaticTransitionFromOpenToHalfOpenEnabled=true` adds a scheduler thread that
does it eagerly. The trade is one more thread and a probe call you didn't ask for; for
StreamPulse's S3 breaker I enable it, because REQ-RS-02's spool-replay job wants the breaker to close
as soon as S3 is healthy, not when the next batch happens to arrive.

**In HALF_OPEN**, `permittedNumberOfCallsInHalfOpenState` calls are allowed and the breaker waits for
**all of them to complete** before deciding — so with a slow dependency, HALF_OPEN can last a while.
`maxWaitDurationInHalfOpenState` (0 = wait indefinitely) bounds that.

**`CallNotPermittedException`** is what a rejected call throws — note it is *not* the original
exception. Code that catches `S3Exception` to build a fallback will silently not catch this, and your
fallback never runs while the breaker is open. Catch it explicitly, or use `fallbackMethod` with a
`Throwable` parameter, which catches everything.

**Code:** the state-transition test REQ-RS-02 requires — this is the test that proves you understand
the machine rather than the vocabulary:

```java
/**
 * BS-04 / REQ-RS-02 — CircuitBreakerStateTransitionTest.
 * Drives CLOSED → OPEN → HALF_OPEN → CLOSED with a controllable stub, asserting the
 * state at each step. No Spring context, no S3, ~5ms.
 */
@Test
void drivesTheFullStateMachine() {
    var config = CircuitBreakerConfig.custom()
            .slidingWindowType(CircuitBreakerConfig.SlidingWindowType.COUNT_BASED)
            .slidingWindowSize(20)                    // last 20 calls
            .minimumNumberOfCalls(10)                 // don't judge on fewer than 10
            .failureRateThreshold(50.0f)              // ≥ 50% failures → OPEN
            .waitDurationInOpenState(Duration.ofSeconds(30))
            .permittedNumberOfCallsInHalfOpenState(5)
            .recordExceptions(S3Exception.class, IOException.class)
            .ignoreExceptions(InvalidObjectKeyException.class)   // our bug, not S3's — don't trip
            .build();

    var cb = CircuitBreaker.of("s3", config);
    var events = new ArrayList<CircuitBreaker.StateTransition>();
    cb.getEventPublisher().onStateTransition(e -> events.add(e.getStateTransition()));

    // --- CLOSED: 9 failures is below minimumNumberOfCalls → still CLOSED even at 100% failure
    for (int i = 0; i < 9; i++) failOnce(cb);
    assertThat(cb.getState()).isEqualTo(CircuitBreaker.State.CLOSED);

    // --- the 10th failure crosses minimumNumberOfCalls at 100% > 50% → OPEN
    failOnce(cb);
    assertThat(cb.getState()).isEqualTo(CircuitBreaker.State.OPEN);

    // --- OPEN: the call is never attempted. Assert on the EXCEPTION TYPE, not just "it threw".
    assertThatThrownBy(() -> cb.executeSupplier(() -> { throw new AssertionError("must not run"); }))
            .isInstanceOf(CallNotPermittedException.class);

    // --- force the wait to elapse without sleeping 30s in a unit test
    cb.transitionToHalfOpenState();
    assertThat(cb.getState()).isEqualTo(CircuitBreaker.State.HALF_OPEN);

    // --- 5 successful probes → CLOSED
    for (int i = 0; i < 5; i++) cb.executeSupplier(() -> "ok");
    assertThat(cb.getState()).isEqualTo(CircuitBreaker.State.CLOSED);

    assertThat(events).containsExactly(
            CircuitBreaker.StateTransition.CLOSED_TO_OPEN,
            CircuitBreaker.StateTransition.OPEN_TO_HALF_OPEN,
            CircuitBreaker.StateTransition.HALF_OPEN_TO_CLOSED);
}
```

**Follow-ups they'll ask:**
- *"What happens to calls that were already in flight when it opens?"* — nothing; they complete (or
  time out) normally. The breaker only gates *new* calls. This is why you still need a TimeLimiter.
- *"Is the breaker state shared across instances?"* — **no.** Resilience4j is entirely in-process, so
  10 pods have 10 independent breakers. Each learns about the outage separately. That's usually fine
  (each pod protects its own threads), but it means "the breaker is open" is a per-pod statement, and
  your Grafana panel must show state *per instance* (REQ-RS-08), not a single line.
- *"What if the dependency is only failing for one tenant?"* — then a single breaker per dependency is
  too coarse: `sportsnow`'s failures trip the breaker for `kidsplus`. You either key the breaker
  instance per tenant (`CircuitBreakerRegistry.circuitBreaker("partner-" + tenantId)`) or you use a
  bulkhead per tenant instead. BS-04 does the latter (REQ-RS-05) because the failure there is
  *slowness*, not errors.

**Pitfalls / red flags:** describing only three states, or not knowing that OPEN→HALF_OPEN is lazy by
default.

**Where in StreamPulse:** BS-04 / REQ-RS-02, `CircuitBreakerStateTransitionTest`, Phase 4 + 9.

**Verify:** the `CircuitBreaker.State` enum's full member list (I believe it also includes
`METRICS_ONLY`), and every default value quoted anywhere in this file.

---

## Q3. Count-based vs time-based sliding window — and what do `minimumNumberOfCalls` and the slow-call settings do?

**30-second answer:** A **count-based** window records the outcomes of the last N *calls*; a
**time-based** window records the outcomes of all calls in the last N *seconds*, bucketed by second.
The choice follows the traffic shape: count-based on a steady high-traffic path, because N calls is a
meaningful sample; time-based on bursty or low-traffic paths, because a count-based window there can
hold results from twenty minutes ago and trip on ancient history. **`minimumNumberOfCalls`** is the
statistical floor — it stops the breaker tripping on "1 of 1 calls failed = 100%", which would make
every cold start flap. And the **slow-call** settings are the ones that actually matter in production:
`slowCallDurationThreshold` defines "slow", and `slowCallRateThreshold` trips the breaker when too
many calls are slow *even though they all succeeded*. Slow is the failure mode that takes systems
down; a breaker that only counts errors will sit CLOSED while every thread in the pool is parked.

**Deeper:**

| | `COUNT_BASED` | `TIME_BASED` |
|---|---|---|
| `slidingWindowSize` means | number of calls | number of **seconds** |
| Memory | a ring buffer of N outcomes | one aggregation bucket per second |
| Good for | steady, high-throughput calls | bursty or low-traffic calls |
| Failure mode | on a quiet endpoint, the window spans a long wall-clock period — stale data trips it | on a quiet endpoint, few calls per second → `minimumNumberOfCalls` protects you |
| StreamPulse | S3 PUTs from sink-writer (constant traffic) → count-based, size 20 | partner webhook (spiky per alert) → time-based, 60s |

**`minimumNumberOfCalls` interacts with both.** In a count-based window it's "don't compute a rate
until we have at least this many results". In a time-based window it's "don't compute a rate until at
least this many calls happened *within the window*" — which means on a truly quiet endpoint the
breaker may **never** open, because it never gathers a quorum. That's a deliberate trade, and if it's
wrong for you the answer is a smaller `minimumNumberOfCalls`, not a bigger window.

**Slow calls — the part I'd spend the most time on:**

```
Dependency behaviour        errors?  breaker with only failureRateThreshold   with slowCallRateThreshold
─────────────────────────────────────────────────────────────────────────────────────────────────────
S3 returns 503 instantly     yes     ✅ opens                                  ✅ opens
S3 takes 30s then succeeds   no      ❌ stays CLOSED — every thread parked      ✅ opens on slow rate
S3 takes 30s then times out  yes*    ✅ opens, but only AFTER the timeout       ✅ opens on slow rate
                                        fires on enough calls                     BEFORE the timeouts
```

(*only if a TimeLimiter is present to convert the hang into a `TimeoutException` the breaker records.)

Row 2 is why `slowCallRateThreshold` exists, and row 3 is why it's *better* than relying on timeouts:
the slow-call counter trips on a call that took 2.1s against a 2s threshold, whereas a timeout-based
trip requires you to actually burn the full timeout on enough calls to fill the window. The slow-call
detector sees the problem earlier and cheaper.

Note that `slowCallDurationThreshold` should be set **relative to the TimeLimiter timeout**, not
independently — typically somewhat below it. If slow-threshold ≥ timeout, every slow call becomes a
timeout (recorded as a failure) before it's ever counted as slow, and the slow-call rate is dead
config.

**Code:**

```yaml
resilience4j:
  circuitbreaker:
    configs:
      default:                                  # shared base — DRY, per REQ-RS-09
        registerHealthIndicator: true           # exposes /actuator/health/circuitBreakers
        automaticTransitionFromOpenToHalfOpenEnabled: true
        recordExceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
        ignoreExceptions:
          - com.streampulse.platform.ValidationException   # our bug → never trips a breaker
    instances:
      s3:                                        # REQ-RS-02, exactly as the requirement specifies
        baseConfig: default
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 20                    # last 20 PUTs
        minimumNumberOfCalls: 10                 # never judge on < 10
        failureRateThreshold: 50                 # ≥ 50% errors → OPEN
        slowCallDurationThreshold: 2s            # > 2s counts as a failure (REQ-RS-02)
        slowCallRateThreshold: 50                # ≥ 50% slow → OPEN, even with zero errors
        waitDurationInOpenState: 30s
        permittedNumberOfCallsInHalfOpenState: 5
      partnerWebhook:
        baseConfig: default
        slidingWindowType: TIME_BASED            # alerts are bursty; "last 60 seconds" is meaningful
        slidingWindowSize: 60                    # SECONDS here, not calls
        minimumNumberOfCalls: 20
        failureRateThreshold: 60                 # partners are flaky; be less twitchy
        slowCallDurationThreshold: 2500ms        # < the 3s TimeLimiter, so slow is detected first
        slowCallRateThreshold: 70
        waitDurationInOpenState: 60s
```

**Follow-ups they'll ask:**
- *"How do you pick `failureRateThreshold`?"* — from the dependency's normal error rate. If S3
  normally errors at 0.1%, then 50% is not "half broken", it's "catastrophically broken" and the
  breaker will only trip on real outages. If a dependency normally errors at 20%, a 50% threshold is
  reasonable but 25% would flap. You need the baseline first — which means you need the metrics
  before you need the breaker.
- *"What goes in `ignoreExceptions`?"* — anything that is *our* fault or a legitimate business
  outcome: a 400 from validation, a 404 for a missing object, a conditional-write failure. Those are
  not evidence the dependency is unhealthy, and counting them means a burst of bad client input opens
  your breaker and takes out good traffic too. This is the single most common misconfiguration I'd
  look for in a review.
- *"Does a call rejected by the breaker count in the window?"* — no. Rejected calls are counted
  separately (`onCallNotPermitted`) and don't pollute the failure-rate statistics.

**Pitfalls / red flags:** configuring only `failureRateThreshold` and having no story for slow calls.

**Where in StreamPulse:** BS-04 / REQ-RS-02, Phase 9 Toxiproxy chaos (inject latency, assert the
slow-call rate trips the breaker before any timeout fires).

**Verify:** defaults for `slidingWindowSize`, `minimumNumberOfCalls`, `failureRateThreshold`,
`slowCallRateThreshold`, `slowCallDurationThreshold`, `waitDurationInOpenState`,
`permittedNumberOfCallsInHalfOpenState` and `automaticTransitionFromOpenToHalfOpenEnabled` in the
Resilience4j version you pin.

---

## Q4. When does the fallback run, and when is a fallback *worse* than failing?

**30-second answer:** With the Spring Boot starter, `fallbackMethod` on the annotation runs when the
decorated call throws — including `CallNotPermittedException` from an open breaker,
`RequestNotPermitted` from a rate limiter, `BulkheadFullException`, and a `TimeoutException`. The
method must have the **same signature plus a trailing `Throwable`** parameter, and you can overload it
per exception type for different behaviour. The design question is harder than the mechanics: a
fallback is only correct when there is a **genuinely acceptable degraded answer**. Returning cached or
partial data with a "stale" marker is good. Returning an empty list that the caller can't distinguish
from "there really are no alerts" is **worse than an error**, because it converts a loud failure into
a silent wrong answer. My rule: a fallback must either preserve the work (spool it, queue it) or
visibly mark the response as degraded. If it can do neither, fail.

**Deeper:**

| Fallback | Verdict | Why |
|---|---|---|
| S3 PUT fails → **write the batch to a local disk spool**, replay when the breaker closes | ✅ **best** — work is preserved, not lost | REQ-RS-02 |
| Panel query times out → return the panel with `status: DEGRADED` | ✅ good — the UI can show a stale badge | BS-02 / REQ-CC-02 |
| Rate limiter has no permit → **enqueue to RabbitMQ** for delayed delivery | ✅ good — deferred, not dropped | REQ-RS-03 |
| Metrics read fails → return the last cached value **with its timestamp** | ✅ acceptable if the age is visible | |
| Alert rule lookup fails → return an empty rule set | ❌ **dangerous** — silently stops all alerting | |
| Write to DynamoDB fails → return HTTP 200 and drop the event | ❌ **the worst case** — silent data loss | |
| Any fallback that swallows the exception and logs at DEBUG | ❌ you've built an invisible outage | |

Row 5 is the one worth saying out loud in an interview because it's counter-intuitive: for a *read*
path, "fail open" (return nothing, carry on) is often fine. For anything that governs **safety,
authorisation, or alerting**, failing open is how you get a silent outage — and nobody pages you,
because from the outside everything returned 200.

**The mechanics that bite:**
1. The fallback must be in the **same bean** (the aspect resolves it by reflection on the target
   class), and it must be `public`/accessible with a matching signature. A typo in the method name is
   a runtime failure, not a compile error — so **test your fallbacks**.
2. Fallbacks are subject to the same self-invocation rule as every Spring proxy: calling an annotated
   method from inside the same bean skips the aspect entirely, silently (file 01, Q9). Same mechanism,
   same silence.
3. The fallback itself is **not** decorated. If your fallback calls something that can hang, you've
   just moved the problem.

**Code:**

```java
/**
 * BS-04 / REQ-RS-02 — the spool-to-disk fallback. This is the fallback done right:
 * the work is PRESERVED, the Kafka offsets are only committed after the spool write is
 * durable (fsync), and a background replayer drains the spool when the breaker closes.
 * Nothing is lost; delivery is just deferred.
 */
@Service
public class S3RawWriter {

    private static final Logger log = LoggerFactory.getLogger(S3RawWriter.class);

    private final S3Client s3;
    private final DiskSpool spool;              // bounded at 1 GB (REQ-RS-02)
    private final MeterRegistry meters;

    @CircuitBreaker(name = "s3", fallbackMethod = "spool")
    @Retry(name = "s3")                          // ordering matters — see Q10
    @TimeLimiter(name = "s3")
    public CompletableFuture<PutResult> put(RawBatch batch) {
        return CompletableFuture.supplyAsync(() -> s3.putObject(request(batch), body(batch)))
                                .thenApply(PutResult::of);
    }

    /**
     * Signature = the original parameters + a trailing Throwable. Overload per exception
     * type when you want different behaviour; the most specific match wins.
     */
    private CompletableFuture<PutResult> spool(RawBatch batch, CallNotPermittedException e) {
        // Breaker is OPEN: we didn't even try. Cheap, expected, log at INFO not ERROR —
        // an ERROR per record during a 3-minute outage is its own incident.
        meters.counter("s3.spooled", "reason", "circuit_open").increment();
        return CompletableFuture.completedFuture(spool.writeAndFsync(batch));
    }

    private CompletableFuture<PutResult> spool(RawBatch batch, Throwable e) {
        // Everything else (timeout, IOException after retries are exhausted).
        log.warn("S3 put failed after retries, spooling batch {}", batch.id(), e);
        meters.counter("s3.spooled", "reason", "exhausted").increment();
        return CompletableFuture.completedFuture(spool.writeAndFsync(batch));
    }
}

/**
 * ❌ The anti-pattern, for contrast — this is what a weak answer implements.
 * It "handles" the failure by throwing the data away and telling the caller everything is fine.
 */
// @CircuitBreaker(name = "dynamodb", fallbackMethod = "ignore")
// private void ignore(MetricAggregate agg, Throwable e) { log.debug("dynamo down", e); }
```

The replayer that makes the fallback honest — a fallback without a drain is just a slower leak:

```java
/**
 * REQ-RS-02. Listens to the breaker's own events rather than polling: when s3 transitions
 * to CLOSED, drain the spool. The bound (1 GB) is enforced on write; if the spool is full
 * we STOP COMMITTING KAFKA OFFSETS (BS-01 / REQ-LC-02) so the events are redelivered
 * instead of dropped. Backpressure all the way to the log.
 */
@Component
class SpoolReplayer {

    SpoolReplayer(CircuitBreakerRegistry registry, DiskSpool spool, S3Client s3) {
        registry.circuitBreaker("s3").getEventPublisher()
                .onStateTransition(event -> {
                    if (event.getStateTransition().getToState() == CircuitBreaker.State.CLOSED) {
                        spool.drainTo(s3);
                    }
                });
    }
}
```

**Follow-ups they'll ask:**
- *"How do you test a fallback?"* — deterministically, by forcing the state:
  `registry.circuitBreaker("s3").transitionToOpenState()` in the test, then assert the fallback path
  ran and the spool file exists. Don't test it by hammering the real dependency until it breaks.
- *"What if the fallback also fails?"* — the exception propagates to the caller. Which is correct:
  you've exhausted your options and the caller must know. The spool's own failure is therefore the
  thing you alert on hardest.
- *"Does `fallbackMethod` catch `Error`s?"* — the parameter is typed `Throwable`, so a matching
  overload catches `Error` too. Usually you do **not** want to swallow an `OutOfMemoryError` into a
  fallback; type the parameter as `Exception` unless you specifically mean otherwise.

**Pitfalls / red flags:** a fallback returning an empty collection for an alerting or authorisation
path. Or having a fallback and no metric on how often it fires — you can't tell degraded from healthy.

**Where in StreamPulse:** BS-04 / REQ-RS-02, `S3OutageSpoolReplayIT` (Toxiproxy/moto fault injection:
3-minute outage, assert zero raw events lost).

---

## Q5. How do you configure Retry, and why does jitter matter?

**30-second answer:** Retry is `maxAttempts` (which **includes** the first call — `maxAttempts=3`
means one call and two retries), a wait strategy, and — the part that actually matters — an explicit
list of which exceptions to retry. The wait strategy should be **exponential backoff with
randomisation**: `IntervalFunction.ofExponentialRandomBackoff(initialInterval, multiplier,
randomizationFactor)`. Exponential because a dependency that's overloaded needs progressively more
breathing room; **randomised (jitter) because without it every client that failed at the same instant
retries at the same instant** — a thundering herd that re-creates the exact load spike that caused the
failure, in a self-sustaining loop. Jitter spreads those retries across a window, which is the
difference between a dependency recovering and a dependency being held down by its own clients. In
StreamPulse, DynamoDB throttling gets 5 attempts at 100ms base × 2 with jitter (REQ-RS-01), and
validation errors get zero retries, ever.

**Deeper:**

**The thundering herd, drawn:**

```
WITHOUT jitter — 500 pods, exponential backoff 100ms / 200ms / 400ms
  t=0     500 calls ─▶ DynamoDB throttles all 500
  t=100ms 500 calls ─▶ throttles all 500   ← identical spike, identical result
  t=300ms 500 calls ─▶ throttles all 500
  t=700ms 500 calls ─▶ ...                  the dependency NEVER gets a quiet moment

WITH randomizationFactor=0.5 — each wait is uniform in [w×0.5, w×1.5]
  t=0        500 calls ─▶ throttled
  t=50-150   500 calls spread over 100ms ─▶ ~5 concurrent instead of 500 ─▶ most succeed
  the dependency drains its queue and recovers
```

That is the whole argument for jitter, and it is worth being able to draw.

**Which exceptions to retry — the classification that matters more than the intervals:**

| Category | Example | Retry? | Why |
|---|---|---|---|
| Throttling / backpressure | `ProvisionedThroughputExceededException`, HTTP 429, 503 | ✅ **yes**, with backoff | it will succeed later by definition |
| Transient network | connection reset, `SocketTimeoutException` on a **read** | ✅ yes | |
| Timeout on a **non-idempotent write** | a POST that may have succeeded | ⚠️ **only with an idempotency key** | otherwise you double-charge / double-page |
| Validation / 4xx | `ValidationException`, 400, 404, 422 | ❌ **never** | the input is wrong; retrying is pure waste and pure latency |
| Auth | 401, 403 | ❌ no (except a one-shot token refresh) | |
| Conditional-write conflict | DynamoDB `ConditionalCheckFailedException` | ❌ no | that's idempotency working as designed (Kafka file, Q7) |
| `CallNotPermittedException` | breaker is open | ❌ no — see Q10 | retrying a rejection defeats the breaker |

**Retry amplification** is the failure mode people don't see coming. Retries **multiply** through
layers:

```
browser retries 3×
  └─ API gateway retries 3×
      └─ query-api @Retry(3)
          └─ AWS SDK's own internal retry (default ~3, and it is ON BY DEFAULT)
              └─ 3 × 3 × 3 × 3 = 81 requests to DynamoDB for one user click
```

The AWS SDK layer is the one that gets missed: `software.amazon.awssdk` has a built-in retry policy,
so wrapping an SDK call in `@Retry` without configuring the SDK gives you the product of the two. In
StreamPulse I set the SDK's retry policy to none (or to a single attempt) where Resilience4j owns
retries, so there is exactly **one** retry layer and I can reason about the total. That's a concrete,
specific answer that shows you've actually integrated these.

**Retry budgets** are the systemic control: rather than "each call may retry 3×", you cap retries at a
percentage of total traffic (say 10%), so a broad outage can't triple your load on the dependency.
Resilience4j does not ship a retry-budget primitive as far as I know — **verify this** — so in
StreamPulse I approximate it by putting the retry *inside* a circuit breaker (Q10): once the failure
rate is high enough, the breaker opens and retries stop happening at all. That's a coarse budget, and
saying so honestly is better than claiming a feature that isn't there.

**Code:**

```yaml
resilience4j:
  retry:
    instances:
      dynamodb:                                  # REQ-RS-01
        maxAttempts: 5                           # INCLUDES the first call → 4 retries
        waitDuration: 100ms                      # base
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2          # 100, 200, 400, 800 ms
        exponentialMaxWaitDuration: 5s           # ceiling, so attempt 10 isn't 51 seconds
        enableRandomizedWait: true               # ← THE JITTER. Without this, thundering herd.
        randomizedWaitFactor: 0.5                # each wait uniform in [w×0.5, w×1.5]
        retryExceptions:
          - software.amazon.awssdk.services.dynamodb.model.ProvisionedThroughputExceededException
          - java.io.IOException
        ignoreExceptions:
          - software.amazon.awssdk.services.dynamodb.model.ConditionalCheckFailedException
          - com.streampulse.platform.ValidationException
```

```java
/**
 * The programmatic equivalent, which is what the RetryJitterTest asserts against.
 * IntervalFunction.ofExponentialRandomBackoff(initial, multiplier, randomizationFactor)
 * — verify the exact overload signature in your R4j version before quoting it.
 */
var retryConfig = RetryConfig.custom()
        .maxAttempts(5)
        .intervalFunction(IntervalFunction.ofExponentialRandomBackoff(
                Duration.ofMillis(100), 2.0, 0.5))
        .retryOnException(t -> t instanceof ProvisionedThroughputExceededException
                            || t instanceof IOException)
        .failAfterMaxAttempts(true)   // throw MaxRetriesExceededException rather than returning
                                      // the last (bad) result — important when you retry on RESULT
        .build();
```

```java
/**
 * REQ-RS-01 — RetryJitterTest. Asserts the intervals fall within the randomized bounds.
 * This is a better test than "it retried 5 times", because it proves the jitter is actually on.
 */
@Test
void waitsAreExponentialAndJittered() {
    var fn = IntervalFunction.ofExponentialRandomBackoff(Duration.ofMillis(100), 2.0, 0.5);

    // attempt 1 → base 100ms, so expect [50, 150)
    assertThat(fn.apply(1)).isBetween(50L, 150L);
    // attempt 2 → base 200ms, so expect [100, 300)
    assertThat(fn.apply(2)).isBetween(100L, 300L);
    // and the values must actually VARY — a fixed value would pass the range check above
    var samples = IntStream.range(0, 50).mapToObj(i -> fn.apply(3)).distinct().count();
    assertThat(samples).isGreaterThan(1);
}
```

**Follow-ups they'll ask:**
- *"Full jitter vs equal jitter vs decorrelated jitter?"* — these are the variants from AWS's
  published guidance on backoff and jitter: full jitter picks uniformly in `[0, backoff]`, equal
  jitter in `[backoff/2, backoff]`, decorrelated jitter derives the next wait from the previous one.
  Resilience4j's `randomizedWaitFactor` is an equal-jitter-style band around the computed backoff.
  The important point is *any* jitter beats none; the variant is a second-order tuning choice.
- *"Does retrying a write risk duplicates?"* — yes, whenever the failure was a timeout rather than a
  definite rejection: the first attempt may have succeeded. That's exactly why every retryable write
  in StreamPulse carries an idempotency key or a conditional expression (Kafka file, Q7). **Retry and
  idempotency are the same design decision**, and answering them together is the senior move.
  The concrete version that lands well: **Kafka's producer can deduplicate a retry and Kinesis
  cannot.** `enable.idempotence=true` gives the producer an id and a per-partition sequence number,
  so the broker drops a duplicate from a retried batch. Kinesis `PutRecords` has no equivalent, so a
  retry after a partial success genuinely duplicates records and the consumer has to absorb it. Same
  pattern, different guarantee, and it changes what you must build downstream.
- *"What should you NOT retry?"* — the list matters more than the backoff. Three categories:
  **(1)** anything that will fail identically — a validation error, a record too large;
  **(2)** a rejection from your own resilience layer (`CallNotPermittedException`,
  `BulkheadFullException`, `RequestNotPermitted`) — retrying a decision that will not change for
  30 seconds is a busy-loop; **(3)** and the one people miss, **slow failures**. The rule is not
  "is this error transient?" but "is it transient **and cheap**?". Every attempt holds whatever
  resource the call occupies, so retrying a 5s timeout three times holds it for 15 seconds —
  multiplying consumption of exactly the resource you are trying to protect, during the incident.
  Fast failures are worth riding over; slow ones belong to the caller, whose waiting costs you
  nothing.
- *"How do you know retries are helping?"* — `resilience4j.retry.calls` is tagged by kind
  (successful-without-retry / successful-with-retry / failed-with-retry / failed-without-retry).
  If `failed_with_retry` dominates, your retries are pure latency and load — turn them down.
  **Verify** the exact tag values for your version.

**Pitfalls / red flags:** "we retry 3 times with a 1-second delay" with no jitter, no exception
classification, and no awareness of the SDK's own retries underneath.

**Where in StreamPulse:** BS-04 / REQ-RS-01 — **built and verified**.
`ResilientEventPublisher.retryPolicy(...)` holds the exclusion list (policy lives with the publisher,
not in Spring config, so the tests run the exact policy production runs). The jitter test asserts the
observed intervals are *not all identical*, which is precisely the property lost by switching to
`ofExponentialBackoff` — confirmed by mutating it and watching the test go red.

**Verify:** `IntervalFunction` factory method signatures, the `resilience4j.retry.calls` tag values,
and whether your R4j version has any retry-budget feature (I don't believe it does).

---

## Q6. RateLimiter: how is it configured, and what's the problem with a per-instance limiter?

**30-second answer:** Three settings: `limitForPeriod` (how many permits), `limitRefreshPeriod` (how
often the permit count resets — it's a fixed-window refill, not a leaky bucket), and
`timeoutDuration` (how long a caller waits for a permit before giving up with `RequestNotPermitted`).
The catch, and it's the whole point of REQ-RS-03: **Resilience4j's RateLimiter is in-process.** Ten
pods with `limitForPeriod=50` do not enforce 50 req/s — they enforce 500. So either you divide by the
replica count (`50 / replicas`), which is brittle because it's wrong the moment the HPA scales or a
pod is unhealthy, or you use a genuinely distributed limiter. In StreamPulse the outbound partner
limit uses R4j with a divided limit plus a safety margin *and* honours the partner's 429s, while the
inbound **per-tenant ingest quota** uses **Bucket4j backed by Redis**, because that is a
business-contract quota that must be cluster-wide and exact-ish.

**Deeper:**

| | Resilience4j `RateLimiter` | Bucket4j + Redis |
|---|---|---|
| Scope | one JVM | the whole cluster |
| Cost per call | a few nanoseconds, in-memory | a Redis round-trip (~sub-millisecond, but a network hop) |
| Failure mode | none — it's local | Redis down → you must choose fail-open or fail-closed |
| Algorithm | fixed-window permit refill | token bucket (supports burst + sustained rate) |
| Protects | the callee, best-effort | a business quota, enforceably |
| StreamPulse | outbound partner calls (REQ-RS-03) | per-tenant ingest quota (REQ-RS-04) |

**The division problem, concretely.** REQ-RS-03 says `limitForPeriod = 50 / replicas`. With 3
replicas that's 16.67 → you must round *down* to 16 (rounding up means 51 req/s and a contract
breach), so you're already leaving 2 req/s on the table. Then:

- The HPA scales to 6 pods. Now you're doing 96 req/s against a 50 req/s contract, until someone
  updates the config. **This is a config that silently breaks when you scale** — which is why it has
  to be documented in an ADR, as REQ-RS-03 requires.
- One pod is unhealthy and not serving. Now you're at 32 req/s — under-using the contract, invisible.

Ways out, in increasing order of correctness and cost:
1. Divide, round down, add margin, and **alert on the partner's 429 rate** as the real detector. This
   is what StreamPulse does, and the honesty is in admitting it's approximate.
2. Derive the limit at runtime from the observed replica count (Kubernetes API / a shared counter in
   Redis). Better, but now a rolling deploy transiently double-counts.
3. A distributed limiter: Bucket4j + Redis (Lua-scripted, so the check-and-decrement is atomic), or a
   dedicated rate-limiting service. Exact, at the cost of a network hop per call and a new dependency
   on the request path.
4. Don't rate-limit at all — let the partner's 429 be the signal, and treat it as a retryable
   throttle. Viable for low volumes, terrible for a contract with penalties.

**`timeoutDuration=0` vs `> 0`** is a meaningful choice. With `0`, a caller with no permit fails
*immediately* with `RequestNotPermitted` — which for BS-04 is exactly right, because the fallback is
"enqueue to RabbitMQ for delayed delivery" (REQ-RS-03 → BS-07), and blocking a thread to wait for a
permit would defeat the bulkhead sitting next to it. With `timeoutDuration > 0` the caller **blocks**,
which turns a rate limiter into a source of thread exhaustion under load — the very thing the bulkhead
exists to prevent. As a rule: on a thread-per-request server, prefer `timeoutDuration=0` and handle
the rejection.

**Code:**

```yaml
resilience4j:
  ratelimiter:
    instances:
      partnerWebhook:                     # REQ-RS-03
        # 50 req/s contracted ÷ 3 replicas = 16.67 → 16, deliberately rounded DOWN.
        # ⚠ This value is coupled to the replica count. See ADR-0xx. It is set from an env
        #   var derived from the Deployment's replica count, not hard-coded, and there is an
        #   alert on the partner's observed 429 rate as the real safety net.
        limitForPeriod: 16
        limitRefreshPeriod: 1s
        timeoutDuration: 0                # fail fast → fallback enqueues to RabbitMQ (BS-07)
        registerHealthIndicator: false
```

```java
/**
 * REQ-RS-04 — the inbound, distributed, per-tenant quota. Different problem, different tool.
 * This one MUST be cluster-wide (a tenant's contracted 10k ev/s is 10k across all pods),
 * and it must survive a pod restart, so it lives in Redis.
 */
@Component
public class TenantQuotaGate {

    private final ProxyManager<String> buckets;   // Bucket4j Redis-backed proxy manager

    public boolean tryConsume(String tenantId, int events) {
        var bucket = buckets.builder().build(tenantId, () -> configFor(tenantId));
        return bucket.tryConsume(events);         // atomic in Redis (Lua)
    }

    private BucketConfiguration configFor(String tenantId) {
        var quota = tenantConfig.quotaFor(tenantId);      // refreshed via config.changes.v1 (CDC)
        return BucketConfiguration.builder()
                // Token bucket = sustained rate PLUS a burst allowance, which is what a real
                // quota needs: a live-match kickoff is a legitimate burst, not abuse.
                .addLimit(Bandwidth.classic(quota.burst(),
                          Refill.greedy(quota.perSecond(), Duration.ofSeconds(1))))
                .build();
    }
}
```

```java
// Ingest gateway: exceeding the quota is a 429 with Retry-After, NOT a 500 and NOT a silent drop.
// The client is supposed to slow down; tell it how long to wait.
if (!quotaGate.tryConsume(tenantId, batch.size())) {
    throw new QuotaExceededException(Duration.ofSeconds(1));   // → ProblemDetail 429 + Retry-After
}
```

**Follow-ups they'll ask:**
- *"Fixed window vs token bucket vs sliding window?"* — a fixed window (R4j's model) allows a
  double-rate burst at the boundary: 50 at t=0.99s and 50 at t=1.01s is 100 in 20ms. A token bucket
  smooths that and supports a configured burst explicitly. A sliding-window log is exact but stores a
  timestamp per call. For a partner contract, token bucket is usually the honest model.
- *"What if Redis is down?"* — a decision you must make explicitly and write down. For the *inbound*
  ingest quota I fail **open** (accept the events) because dropping customer telemetry to protect a
  quota is the wrong trade, and I alert loudly. For something protecting a paid downstream, failing
  **closed** may be right. The wrong answer is not having decided.
- *"Where should rate limiting live — app, gateway, or mesh?"* — the coarse limit belongs at the edge
  (API gateway / WAF) where it's cheapest, because that traffic never touches your JVM. The
  *per-tenant business quota* belongs in the app, because only the app knows the tenant's contract
  after JWT validation. Both, at different granularities.

**Pitfalls / red flags:** configuring `limitForPeriod: 50` on a 10-pod deployment and claiming you
honour a 50 req/s contract.

**Where in StreamPulse:** BS-04 / REQ-RS-03 + REQ-RS-04, `PartnerRateLimitIT` (WireMock partner
asserts ≤ 50 rps observed across all instances), Phase 2 + 6.

**Verify:** R4j RateLimiter defaults (`limitForPeriod`, `limitRefreshPeriod` — I recall the refresh
default being an unusually small value like 500ns, `timeoutDuration`), and Bucket4j's current
Redis/`ProxyManager` API names.

---

## Q7. Bulkhead: semaphore vs thread pool. When do you use each?

**30-second answer:** A bulkhead caps **concurrency** so one dependency can't consume every thread.
`SemaphoreBulkhead` is a counter: the *calling* thread runs the work, but only `maxConcurrentCalls` of
them may be inside at once; extras wait up to `maxWaitDuration` and then get `BulkheadFullException`.
`ThreadPoolBulkhead` is a bounded thread pool with a bounded queue: the call is **handed to a
different thread**, so the caller returns immediately with a `CompletableFuture`. Semaphore is
cheaper, keeps the call on the caller's thread (so `ThreadLocal`s, MDC and security context survive),
and is the right default — especially on reactive stacks and with virtual threads. Thread-pool
bulkhead is for when you need to **isolate a genuinely blocking call off the caller's thread** and get
real timeout-and-abandon semantics. BS-04 / REQ-RS-05 uses a `ThreadPoolBulkhead` **per tenant** for
partner webhooks, precisely so `sportsnow`'s slow partner cannot consume the threads that `kidsplus`
alerts need.

**Deeper:**

```
SEMAPHORE BULKHEAD (maxConcurrentCalls = 16)
  caller thread ──▶ tryAcquire() ──▶ [ runs the call ON THE CALLER'S THREAD ] ──▶ release()
                        │
                        └─ full ──▶ wait up to maxWaitDuration ──▶ BulkheadFullException
  ✅ no thread hand-off, no context loss, ~zero overhead
  ❌ cannot abandon a call: if the callee hangs, the caller's thread hangs with it
     (so you MUST pair it with a TimeLimiter or a client-level socket timeout)

THREAD-POOL BULKHEAD (core 2, max 4, queue 20)
  caller thread ──▶ submit ──▶ [ queue(20) ] ──▶ [ pool threads 2..4 ] ──▶ CompletableFuture
        │                            │
        └─ returns immediately       └─ queue full & pool at max ──▶ BulkheadFullException
  ✅ true isolation: the caller thread is never blocked by the dependency
  ✅ works with TimeLimiter's cancelRunningFuture
  ❌ context hand-off: MDC / SecurityContext / request scope do NOT propagate automatically
  ❌ a real thread pool per bulkhead — costly if you create one per tenant at high cardinality
```

| | `SemaphoreBulkhead` | `ThreadPoolBulkhead` |
|---|---|---|
| Config | `maxConcurrentCalls`, `maxWaitDuration` | `coreThreadPoolSize`, `maxThreadPoolSize`, `queueCapacity`, `keepAliveDuration` |
| Runs on | caller's thread | its own pool |
| Return type | anything | must be `CompletableFuture` when used via the annotation |
| Reactive support | ✅ (`BulkheadOperator`) | ❌ **not supported** with reactive types |
| Context propagation | automatic | manual (`ContextPropagator`) |
| Use for | the default; reactive; virtual threads | isolating a blocking call; per-tenant isolation |

**REQ-RS-05's design, and the trade I'd name unprompted:** a thread-pool bulkhead per tenant gives
hard isolation, but it's a pool per tenant. With 3 fictional tenants that's fine. With 5,000 tenants
it's 5,000 pools and you've built a thread-count problem to solve a thread-count problem. The scalable
version is a **semaphore bulkhead per tenant** (a counter costs bytes, not threads) on top of one
shared virtual-thread executor — which is exactly what BS-02 / REQ-CC-05 prototypes. I'd present the
thread-pool version as correct for a small, bounded tenant set and name the crossover point.

Note also that a bulkhead's rejection is **fast and cheap**, which is the entire value: 
`BulkheadFullException` in microseconds instead of a request queued behind a 30-second call. Load
shedding is a feature, not a failure — and the fallback decides what "shed" means (here: enqueue to
RabbitMQ, so the alert is delayed, not dropped).

**Code:**

```yaml
resilience4j:
  thread-pool-bulkhead:
    instances:
      # REQ-RS-05 — one bulkhead PER TENANT. This is what makes the NFR achievable:
      # "kidsplus alert latency < 2s p99 while sportsnow's partner is down".
      partner-sportsnow:
        coreThreadPoolSize: 2
        maxThreadPoolSize: 4
        queueCapacity: 20          # 4 threads × 3s timeout ⇒ worst-case drain ≈ 18s; queue
                                   # deeper than that just adds latency to doomed work
      partner-kidsplus:
        coreThreadPoolSize: 2
        maxThreadPoolSize: 4
        queueCapacity: 20
  bulkhead:
    instances:
      # Semaphore bulkhead on the reactive query-api path (REQ-RS-05, second half):
      # WebFlux has a tiny event-loop pool; a thread-pool bulkhead there would mean
      # blocking hand-offs on a non-blocking stack. A counter is the right tool.
      queryApiPanels:
        maxConcurrentCalls: 16
        maxWaitDuration: 0         # fail fast → panel marked DEGRADED (BS-02 / REQ-CC-02)
```

```java
/**
 * REQ-RS-05. The bulkhead NAME is derived per tenant, which is what creates the isolation.
 * A single shared "partner" bulkhead would let sportsnow's 4 stuck threads starve kidsplus.
 */
@Service
public class PartnerNotifier {

    private final ThreadPoolBulkheadRegistry bulkheads;
    private final TimeLimiterRegistry timeLimiters;
    private final RabbitTemplate rabbit;

    public CompletableFuture<Void> notify(String tenantId, AlertEvent alert) {
        var bulkhead = bulkheads.bulkhead("partner-" + tenantId);     // ← per-tenant isolation
        var timeLimiter = timeLimiters.timeLimiter("partnerWebhook");

        try {
            var decorated = Decorators.ofSupplier(() -> partnerClient.deliver(alert))
                    .withThreadPoolBulkhead(bulkhead)
                    .withTimeLimiter(timeLimiter, scheduler)
                    .withCircuitBreaker(breakers.circuitBreaker("partner-" + tenantId))
                    .get();
            return decorated.toCompletableFuture().thenApply(r -> null);

        } catch (BulkheadFullException e) {
            // Load shed — but NOT drop. The alert goes to RabbitMQ for delayed delivery
            // (BS-07 / REQ-MQ-04's TTL retry queue). Fast rejection, preserved work.
            meters.counter("partner.shed", "tenant", tenantId).increment();
            rabbit.convertAndSend("alerts.x", routingKey(alert), alert);
            return CompletableFuture.completedFuture(null);
        }
    }
}
```

```java
/**
 * REQ-RS-05 — TenantBulkheadIsolationIT. The test that proves the NFR.
 */
@Test
void slowSportsnowPartnerDoesNotDelayKidsplus() {
    wireMock.stubFor(post("/sportsnow").willReturn(aResponse().withFixedDelay(30_000)));
    wireMock.stubFor(post("/kidsplus").willReturn(ok().withFixedDelay(20)));

    // Saturate sportsnow's bulkhead: 4 threads + 20 queue = 24 in flight, the 25th is rejected.
    IntStream.range(0, 40).forEach(i -> notifier.notify("sportsnow", alert()));

    var start = System.nanoTime();
    notifier.notify("kidsplus", alert()).join();
    var elapsedMs = Duration.ofNanos(System.nanoTime() - start).toMillis();

    assertThat(elapsedMs).isLessThan(2_000);          // NFR: kidsplus p99 < 2s. Unaffected.
}
```

**Follow-ups they'll ask:**
- *"How do you size a bulkhead?"* — from Little's Law: `concurrency = arrival rate × latency`. If the
  partner handles 50 req/s at 100ms, ~5 concurrent calls saturate it, so a bulkhead of 4–8 is
  generous. Sizing far above that just means you queue work the dependency can't absorb. And when the
  dependency slows to 3s, that same bulkhead is what stops the arrival rate translating into 150
  blocked threads.
- *"Bulkhead vs just a smaller thread pool?"* — a bulkhead is a *named, per-dependency* bound with
  metrics and a defined rejection path. A smaller pool is a global bound that one bad dependency can
  still fully occupy. The isolation is the point.
- *"How does this interact with `@Async` and `CompletableFuture`?"* — a `ThreadPoolBulkhead` **is**
  the executor, so you don't also wrap it in `supplyAsync` on the common pool (BS-02's point about
  `supplyAsync` defaulting to `ForkJoinPool.commonPool()` — one blocking dependency there stalls
  parallel streams everywhere in the JVM).

**Pitfalls / red flags:** using a `ThreadPoolBulkhead` on a WebFlux path (it isn't supported for
reactive types and would block the event loop on hand-off anyway), or one shared bulkhead for all
tenants while claiming per-tenant isolation.

**Where in StreamPulse:** BS-04 / REQ-RS-05, `TenantBulkheadIsolationIT`, BS-02 / REQ-CC-05.

**Verify:** `ThreadPoolBulkhead` defaults (`coreThreadPoolSize`, `maxThreadPoolSize`, `queueCapacity`,
`keepAliveDuration`) and `SemaphoreBulkhead` defaults (`maxConcurrentCalls`, `maxWaitDuration`), plus
current reactive-support status for `ThreadPoolBulkhead`.

---

## Q8. TimeLimiter: why is a timeout the foundation, and what are its limits?

**30-second answer:** Because **every other pattern assumes calls return**. Without a timeout, a slow
dependency doesn't trip the breaker (the failure window never fills), doesn't release the bulkhead
permit, and doesn't free the thread — it just accumulates. `TimeLimiter` wraps a `Future`/
`CompletableFuture` and throws `TimeoutException` after `timeoutDuration`, optionally cancelling the
underlying future (`cancelRunningFuture`, on by default). The important limitation to state honestly:
**cancelling a `Future` does not stop a blocking socket read.** `Future.cancel(true)` interrupts the
thread, and a thread blocked in a legacy blocking I/O call may not respond to interruption at all. So
the TimeLimiter bounds *how long the caller waits*, not necessarily *how long the work runs*. That's
why you also set socket/connect/read timeouts on the HTTP and SDK clients themselves — belt and
braces, and REQ-RS-06 enforces it with an ArchUnit rule.

**Deeper:**

The three layers of timeout, all of which you need:

| Layer | Bounds | Example | Without it |
|---|---|---|---|
| **Client/transport** | connect + read on the socket | `HttpClient.Builder.connectTimeout`, SDK `apiCallTimeout`, JDBC `socketTimeout` | a TCP connection to a black-holed host hangs until the OS gives up (minutes) |
| **TimeLimiter** | how long *this caller* waits | `resilience4j.timelimiter.instances.s3.timeoutDuration` | the caller is coupled to the callee's worst case |
| **End-to-end budget** | the whole request | the panel budget in BS-02 / REQ-CC-02 | the sum of per-call timeouts exceeds the user's patience |

The third row is the one seniors get right and juniors miss: **the sum of your timeouts must fit
inside your SLO**. If the Command Center has a 250ms p95 budget (BS-02) and calls five panels, you
cannot give each panel a 2-second timeout and call it a design — the per-panel budget is 150ms and the
page is assembled from whatever returned (REQ-CC-02). Timeout values are derived *downwards* from the
SLO, not *upwards* from what feels safe.

And: **timeout must exceed the retry budget** at the layer above it, or the outer timeout fires
mid-retry and you've paid for retries you never got to use. Retry total (Q5: 100+200+400+800 ≈ 1.5s
plus jitter, plus 4 × per-call latency) has to fit inside whatever bounds it.

**`cancelRunningFuture`** deserves a sentence: with `true` (the default, I believe — verify), the
TimeLimiter calls `cancel(true)` on the future, which interrupts the running thread. That's effective
for code that checks interruption or is blocked in an interruptible operation, and useless for a
thread blocked in a non-interruptible native read. With a `ThreadPoolBulkhead` underneath, cancelling
at least frees the *slot* logically — but the thread may remain occupied, so the pool can still
saturate. Honest answer: timeouts limit waiting, transport timeouts limit working.

**Code:**

```yaml
resilience4j:
  timelimiter:
    instances:
      s3:             { timeoutDuration: 2s,  cancelRunningFuture: true }
      dynamodb:       { timeoutDuration: 1s,  cancelRunningFuture: true }
      partnerWebhook: { timeoutDuration: 3s,  cancelRunningFuture: true }
```

```java
/**
 * REQ-RS-06 — the transport-layer timeouts, which the TimeLimiter does NOT replace.
 * Note apiCallAttemptTimeout vs apiCallTimeout: per-attempt vs whole-call-including-SDK-retries.
 */
@Bean
S3Client s3Client(S3Properties props) {
    return S3Client.builder()
            .httpClientBuilder(ApacheHttpClient.builder()
                    .connectionTimeout(Duration.ofMillis(500))
                    .socketTimeout(Duration.ofSeconds(2)))
            .overrideConfiguration(c -> c
                    .apiCallAttemptTimeout(Duration.ofSeconds(2))
                    .apiCallTimeout(Duration.ofSeconds(4))
                    // Resilience4j owns retries (Q5). Disable the SDK's, or you get the
                    // PRODUCT of the two retry policies (retry amplification).
                    .retryPolicy(RetryPolicy.none()))
            .build();
}
```

```java
/**
 * REQ-RS-06 — the ArchUnit rule. A convention nobody enforces is a convention nobody follows.
 */
@ArchTest
static final ArchRule every_http_client_has_a_timeout =
    noClasses().that().resideInAPackage("..adapter.out..")
        .should().callMethod(HttpClient.class, "newHttpClient")
        .because("HttpClient.newHttpClient() has no timeouts; use the builder with "
               + "connectTimeout, and a TimeLimiter on the call (REQ-RS-06)");
```

**Follow-ups they'll ask:**
- *"Why not `Mono.timeout()` on a reactive stack?"* — you should. `TimeLimiter` is designed around
  `Future`s; on WebFlux, `Mono.timeout(Duration)` is idiomatic, composes with the rest of the
  operators, and cancels the subscription properly (which on a non-blocking client actually does abort
  the request). The R4j reactor module provides operators for the other four patterns, but for
  timeouts use Reactor's own.
- *"How do you pick the value?"* — from the dependency's observed p99, plus headroom, capped by your
  SLO budget. A timeout at the p50 causes needless failures; a timeout at 10× the p99 isn't a timeout.
  You need the latency histogram first, which is another reason observability precedes resilience.
- *"What about virtual threads?"* — a blocking call on a virtual thread parks the *virtual* thread and
  releases the carrier, so a hung dependency no longer consumes an OS thread. That changes the
  *cost* of a hang dramatically, but not the *correctness*: you still need a timeout, because
  unbounded virtual threads still hold memory, connections and downstream resources (BS-02 /
  REQ-CC-05's point about still needing a semaphore).

**Pitfalls / red flags:** "we have a circuit breaker" with no timeouts anywhere. That breaker will
never trip on the failure mode that matters.

**Where in StreamPulse:** BS-04 / REQ-RS-06 (ArchUnit rule: no `HttpClient` without a timeout), all
outbound adapters.

**Verify:** `cancelRunningFuture` default, and the AWS SDK v2 override-configuration method names
(`apiCallTimeout` / `apiCallAttemptTimeout` / `retryPolicy` vs `retryStrategy` — this API changed).

---

## Q9. How do you observe all this? Metrics, events, and what you put on a dashboard.

**30-second answer:** Three layers. **Metrics** via the Micrometer binder — breaker state as a gauge,
call counts by outcome, retry calls tagged by kind, bulkhead available-concurrency, rate-limiter
available-permits. **Events**: every module has an `EventPublisher` you can subscribe to
(`onStateTransition`, `onError`, `onCallNotPermitted`, `onSlowCallRateExceeded`), which is how you get
a *log line with a reason* rather than just a counter going up. **Actuator endpoints**
(`/actuator/circuitbreakers`, `/actuator/circuitbreakerevents`, and the equivalents for the other
modules) for live inspection during an incident. On the dashboard, REQ-RS-08 asks for breaker state
over time per instance, with an alert if any breaker stays OPEN for more than 5 minutes — because an
OPEN breaker is *supposed* to be transient, and one that isn't is an outage nobody declared.

**Deeper:**

The panels I'd actually build (Phase 8), and why each one:

| Panel | Metric | What it tells you |
|---|---|---|
| Breaker state over time, **per instance and per pod** | `resilience4j_circuitbreaker_state` | is a dependency degraded, and is it all pods or one? |
| Breaker failure rate vs threshold | `resilience4j_circuitbreaker_failure_rate` | how close to tripping — the leading indicator |
| Calls by outcome | `resilience4j_circuitbreaker_calls{kind=successful\|failed\|not_permitted}` | `not_permitted` is the load you *shed* |
| Retry calls by kind | `resilience4j_retry_calls{kind=...}` | are retries helping or just adding load? |
| Bulkhead available concurrency | `resilience4j_bulkhead_available_concurrent_calls` | how close to shedding |
| Rate limiter available permits / waiting threads | `resilience4j_ratelimiter_available_permissions` | are we actually hitting the partner's limit? |
| **Fallback rate** (your own counter) | `s3.spooled`, `partner.shed` | the user-visible degradation rate |

That last row is mine, not the library's, and it's the most important one: library metrics tell you
the *mechanism* fired; only your own counter tells you the *business impact*. "The breaker opened" is
interesting; "we spooled 412 batches to disk and 3 tenants saw stale dashboards" is the incident.

**Events vs metrics.** A counter tells you *that* something happened; the event tells you *why*.
Subscribing to state transitions and logging them at WARN with the instance name gives on-call a
single searchable line — and it's how the `SpoolReplayer` in Q4 knows to drain.

**Code:**

```java
/**
 * REQ-RS-08. One listener for every registered breaker, including ones created later.
 * The registry's own event publisher fires when a breaker is ADDED — that's how you
 * instrument dynamically-created per-tenant breakers without a restart.
 */
@Configuration
class ResilienceObservability {

    private static final Logger log = LoggerFactory.getLogger(ResilienceObservability.class);

    ResilienceObservability(CircuitBreakerRegistry registry, MeterRegistry meters) {

        Consumer<CircuitBreaker> instrument = cb ->
            cb.getEventPublisher()
              .onStateTransition(e -> {
                  // One WARN line per transition — searchable, low volume, high signal.
                  log.warn("circuit-breaker '{}' {} → {}",
                           e.getCircuitBreakerName(),
                           e.getStateTransition().getFromState(),
                           e.getStateTransition().getToState());
                  meters.counter("resilience.breaker.transition",
                                 "name", e.getCircuitBreakerName(),
                                 "to",   e.getStateTransition().getToState().name())
                        .increment();
              })
              .onCallNotPermitted(e ->
                  // This is SHED LOAD, not an error. Counting it separately is what lets you
                  // tell "we're failing" apart from "we're deliberately not trying".
                  meters.counter("resilience.breaker.rejected",
                                 "name", e.getCircuitBreakerName()).increment())
              .onSlowCallRateExceeded(e ->
                  log.warn("circuit-breaker '{}' slow-call rate {}%",
                           e.getCircuitBreakerName(), e.getSlowCallRate()));

        registry.getAllCircuitBreakers().forEach(instrument);
        registry.getEventPublisher().onEntryAdded(e -> instrument.accept(e.getAddedEntry()));
    }
}
```

```yaml
# Phase 8 — REQ-RS-08. An OPEN breaker is meant to be transient.
groups:
  - name: streampulse-resilience
    rules:
      - alert: CircuitBreakerStuckOpen
        expr: resilience4j_circuitbreaker_state{state="open"} == 1
        for: 5m                      # transient OPEN is the system working; 5m is an outage
        labels:   { severity: page }
        annotations:
          summary: "Breaker {{ $labels.name }} OPEN > 5m on {{ $labels.pod }}"
      - alert: RetriesNotHelping
        # If most retried calls still fail, retries are pure added load — turn them down.
        expr: rate(resilience4j_retry_calls{kind="failed_with_retry"}[10m])
              > rate(resilience4j_retry_calls{kind="successful_with_retry"}[10m])
        for: 15m
      - alert: SpoolNearCapacity
        expr: streampulse_s3_spool_bytes / streampulse_s3_spool_limit_bytes > 0.8
        for: 2m                      # the fallback itself is bounded — know before it's full
```

**Follow-ups they'll ask:**
- *"How do you make a breaker's state visible in health checks?"* — `registerHealthIndicator: true`
  adds it to `/actuator/health`. Be careful: by default a breaker contributing `DOWN` can make your
  **readiness** probe fail, and Kubernetes will then remove a pod that is actually working fine in
  degraded mode. Map it into the *liveness/readiness groups* deliberately — usually a degraded
  dependency should not take the pod out of rotation.
- *"How do you know a fallback is firing in prod?"* — your own counter, plus the trace: I tag the span
  with `resilience.fallback=true` so a Tempo trace shows the degraded path explicitly (Phase 8).
- *"Can you change config at runtime?"* — yes; the registries accept new configs, and
  `transitionToForcedOpenState()` / `transitionToDisabledState()` give you an operator kill switch
  without a deploy. Worth exposing behind an admin endpoint with an audit trail.

**Pitfalls / red flags:** having the patterns but no metrics, so nobody can tell whether the system is
healthy or quietly running on fallbacks.

**Where in StreamPulse:** BS-04 / REQ-RS-08, Phase 8 Grafana dashboards.

**Verify:** the exact Micrometer metric names and tag values for your R4j + Spring Boot version (the
Prometheus-rendered names differ from the Micrometer names by the usual `.`→`_` mangling).

---

## Q10. In what order do the decorators compose, and why must Retry wrap CircuitBreaker?

**30-second answer:** The documented nesting for the Spring Boot starter is
`Retry( CircuitBreaker( RateLimiter( TimeLimiter( Bulkhead( call ))))) ` — Retry outermost, Bulkhead
innermost. The ordering that matters most is **Retry outside CircuitBreaker**, and the reason is
that each retry *attempt* then passes through the breaker and is recorded individually. So a
dependency that is failing outright trips the breaker N× sooner, where N is `maxAttempts` — which is
what you want, because the whole job of a breaker is to notice quickly. The price is that once the
breaker opens, the retry immediately receives `CallNotPermittedException`, which **must** be in
`ignoreExceptions` or you spin against a decision that will not change for 30 seconds. Bulkhead
innermost is right because the concurrency permit should be held only for the actual call, not for
the retry sleeps.

**Deeper:**

```
        ┌── Retry ────────────────────────────────────────────────┐
        │  ┌── CircuitBreaker ────────────────────────────────┐   │
        │  │  ┌── RateLimiter ───────────────────────────┐    │   │
        │  │  │  ┌── TimeLimiter ────────────────────┐   │    │   │
        │  │  │  │  ┌── Bulkhead ──────────────┐     │   │    │   │
 call ──┼──┼──┼──┼──┤     the actual call      │     │   │    │   │
        │  │  │  │  └──────────────────────────┘     │   │    │   │
        │  │  │  └───────────────────────────────────┘   │    │   │
        │  │  └──────────────────────────────────────────┘    │   │
        │  └──────────────────────────────────────────────────┘   │
        └─────────────────────────────────────────────────────────┘
```

Reading it from the inside out — what each layer's position buys you:

| Layer | Position | Consequence of that position |
|---|---|---|
| **Bulkhead** | innermost | the concurrency permit is held only while the call is actually running — not during retry sleeps, not while waiting for a rate-limit permit |
| **TimeLimiter** | inside the rate limiter | the timeout clock covers the call, not the time spent queueing for a permit |
| **RateLimiter** | inside the breaker | when the breaker is OPEN, we don't burn permits on calls we're not going to make |
| **CircuitBreaker** | inside Retry | sees **every attempt**, so it trips `maxAttempts`× sooner on a dead dependency — and can abort a retry sequence already in progress |
| **Retry** | outermost | a retry is a retry of the *whole* protected operation, including its breaker check |

**Which way the counting actually goes** — get this the right way round, because the inverted
version is a common and confident-sounding mistake:

```
✅ Retry( CircuitBreaker( call ) )        — the default
   each ATTEMPT passes through the breaker → 1 request x 3 attempts = 3 recorded calls
   → a dead dependency fills the window 3x faster and trips 3x sooner.

❌ CircuitBreaker( Retry( call ) )        — breaker outside, retry inside
   the breaker wraps the whole retry loop → 1 request = 1 recorded call
   → the breaker only learns the FINAL outcome, and cannot intervene mid-sequence.
```

**This is measured, not reasoned.** In StreamPulse the two lines were swapped and the suite re-run.
With `slidingWindowSize=4`, `minimumNumberOfCalls=4`, `maxAttempts=3` and a delegate that always
fails, two batches are enough to open the breaker in the correct order and not enough in the
inverted one:

```
THE ORDERING PROOF: the breaker counts every ATTEMPT, not every batch FAILED
    expected: OPEN
     but was: CLOSED
```

The second thing the correct order buys you: **once the breaker opens mid-retry-sequence, the
remaining attempts are rejected instantly** instead of hammering the dead dependency, so the retry
loop terminates fast rather than sleeping through its full backoff schedule. The breaker gets to
*stop* the retry storm. In the inverted order the retry loop runs to completion inside a breaker
that has no chance to intervene until the whole sequence is already over.

If someone pushes back that retry-outside makes `slidingWindowSize` "mean attempts, not requests" —
they are right, and that is the deliberate trade, not an accident. You are choosing to measure the
dependency's health in calls made to it rather than in requests received from your users. For a
breaker, calls made to it is the more useful unit.

Either way, the essential companion config is:

```yaml
resilience4j:
  retry:
    instances:
      s3:
        ignoreExceptions:
          # DO NOT retry a rejection. The breaker is telling you the dependency is down;
          # retrying that decision is a busy-loop that burns CPU and delays the fallback.
          - io.github.resilience4j.circuitbreaker.CallNotPermittedException
          - io.github.resilience4j.ratelimiter.RequestNotPermitted
          - io.github.resilience4j.bulkhead.BulkheadFullException
```

**How the order is actually implemented:** the Spring Boot starter's aspects are ordinary Spring AOP
aspects (file 01, Q8), ordered by `Ordered`/`@Order`, and each module's order is configurable:
`resilience4j.retry.retryAspectOrder`, `resilience4j.circuitbreaker.circuitBreakerAspectOrder`, and
the equivalents. **I would not quote the numeric defaults from memory** — and note that Resilience4j's
convention for which direction the number means "outer" is not the same intuition as plain Spring
`@Order`. Say the *nesting*, which is documented and stable; verify the *numbers* if you need to
change them.

When you decorate functionally instead of with annotations, **you control the order explicitly** and
there's no ambiguity — which is one good reason to prefer the functional form for anything subtle:

```java
/**
 * BS-04 / REQ-RS-07. The functional form makes the ordering explicit and testable.
 * Read it inside-out: bulkhead wraps the call, then timeLimiter, then rateLimiter,
 * then circuitBreaker, then retry outermost.
 */
var decorated = Decorators.ofSupplier(() -> s3.putObject(req, body))
        .withBulkhead(bulkheads.bulkhead("s3"))               // innermost
        .withCircuitBreaker(breakers.circuitBreaker("s3"))
        .withRetry(retries.retry("s3"))                        // outermost
        .withFallback(List.of(CallNotPermittedException.class, IOException.class),
                      throwable -> spool.writeAndFsync(batch))
        .decorate();
```

**Follow-ups they'll ask:**
- *"Where should the TimeLimiter go relative to Retry?"* — inside, always. A timeout per *attempt* is
  what you want; a timeout around the whole retry sequence is a different control (an overall budget)
  and if you want both you need two timers. Watch the arithmetic: `maxAttempts × (timeout + backoff)`
  is your true worst case, and it must fit the caller's SLO.
- *"Does the order matter if only two are present?"* — yes, and the two-pattern case is the common
  one. Retry + CircuitBreaker is exactly the pair discussed above.
- *"How would you test the order?"* — with counting stubs: a fake dependency that records invocation
  counts, plus assertions on breaker state after a known sequence of failures. REQ-RS-07 requires
  the order be "documented **and tested**". The part worth saying out loud: a test that passes in
  *both* orderings is not testing the ordering. In StreamPulse the way this was established was to
  swap the two decorator lines and confirm the suite goes red — and nine of the ten tests still
  passed, so only the one written for that property had any value at all.
- *"Should you ever retry a timeout?"* — usually no, and this is the sharper version of the question.
  The rule is not "is this error transient?" but "is it transient **and cheap**?". Every attempt
  holds whatever resource the call occupies — a bulkhead permit, a connection, an in-flight slot —
  so retrying a 5s timeout three times holds it for 15 seconds, multiplying consumption of exactly
  the resource you are trying to protect, during the incident. Fast failures are worth riding over;
  slow ones belong to the caller, whose waiting costs you nothing.

**Pitfalls / red flags:** knowing the order as a memorised string without being able to explain what
any position buys. Or retrying `CallNotPermittedException`.

**Where in StreamPulse:** BS-04 / REQ-RS-07 — **built and verified**.
`services/ingest-gateway/.../adapter/out/resilience/ResilientEventPublisher.java` composes
`Retry( CircuitBreaker( TimeLimiter( publish ) ) )` functionally; `ResilientEventPublisherTest` drives
the full state machine, and both orderings were confirmed by mutation. ADR-0009 records the reasoning
and the retry exclusion list.

**Verify:** the default numeric aspect orders (`retryAspectOrder`, `circuitBreakerAspectOrder`,
`rateLimiterAspectOrder`, `timeLimiterAspectOrder`, `bulkheadAspectOrder`) and which direction the
number means "outermost" — I am confident about the documented *nesting*, not the numbers.

---

## Q11. Annotations versus functional decoration — when do you use which?

**30-second answer:** Annotations (`@CircuitBreaker`, `@Retry`, …) are declarative, configured in
YAML, and right for the common case: a whole method that represents one call to one dependency.
Functional decoration (`Decorators.ofSupplier(...).withCircuitBreaker(...)...`) is right whenever the
instance name is **dynamic** (per tenant — REQ-RS-05), when you need **explicit ordering** (REQ-RS-07),
when you're decorating something that isn't a Spring bean method, or when you want to unit-test the
decoration without a Spring context. The annotations also inherit every Spring AOP limitation:
**self-invocation silently skips them**, they don't work on `private` or `final` methods, and the
bean must be proxied. That last point is worth stating unprompted because it's the same footgun as
`@Transactional`.

**Deeper:**

| | Annotation | Functional |
|---|---|---|
| Config source | YAML, per named instance | code |
| Instance name | must be a constant (annotation attribute) | **can be computed at runtime** |
| Ordering | the starter's default aspect order | explicit in the builder |
| Works outside Spring | no | yes |
| Self-invocation | ❌ silently bypassed | ✅ unaffected |
| Testability | needs a context (or `ApplicationContextRunner`) | plain unit test |
| Readability | high for the simple case | explicit but verbose |

The self-invocation trap, restated for this file because it causes *silent* loss of protection:

```java
// ❌ BROKEN — the @Retry on put() is never applied.
@Service
class S3Writer {
    public void writeAll(List<RawBatch> batches) {
        batches.forEach(this::put);       // direct call on `this` → no proxy → no aspect
    }
    @Retry(name = "s3")
    public void put(RawBatch b) { s3.putObject(...); }
}
```

No error, no warning — the retries just never happen, and you find out during an S3 blip. Same
mechanism as file 01, Q9: the proxy is created in `postProcessAfterInitialization` and an internal
call never touches it. The fixes are the same: extract to another bean, or use the functional form.

**Code — the dynamic-name case, which the annotation simply cannot express:**

```java
/**
 * REQ-RS-05. The breaker/bulkhead instance name depends on the TENANT, which is only known
 * at runtime. No annotation can express this; the registry lookup is the whole point.
 * Registries create-on-first-use from the named config (falling back to "default"),
 * so tenants onboarded via config.changes.v1 (CDC) get protection with no deploy.
 */
@Service
public class PartnerCallGateway {

    private final CircuitBreakerRegistry breakers;
    private final RetryRegistry retries;
    private final ThreadPoolBulkheadRegistry bulkheads;

    public CompletableFuture<DeliveryResult> call(String tenantId, AlertEvent alert) {

        String instance = "partner-" + tenantId;      // ← dynamic

        var supplier = Decorators.ofSupplier(() -> client.deliver(alert))
                .withThreadPoolBulkhead(bulkheads.bulkhead(instance))
                .withCircuitBreaker(breakers.circuitBreaker(instance))
                .withRetry(retries.retry("partnerWebhook"), scheduler)
                .withFallback(List.of(CallNotPermittedException.class,
                                      BulkheadFullException.class,
                                      RequestNotPermitted.class),
                              t -> queueForLaterDelivery(alert));

        return supplier.get().toCompletableFuture();
    }
}
```

```java
/**
 * The unit test this enables — no Spring, no WireMock, no clock skew. This is the
 * strongest practical argument for the functional form on anything non-trivial.
 */
@Test
void openBreakerRoutesToTheQueue() {
    var breaker = CircuitBreaker.ofDefaults("partner-sportsnow");
    breaker.transitionToOpenState();                       // deterministic, no hammering

    var result = gateway.call("sportsnow", alert()).join();

    assertThat(result.status()).isEqualTo(QUEUED);
    verify(rabbit).convertAndSend(eq("alerts.x"), anyString(), any(AlertEvent.class));
}
```

**Follow-ups they'll ask:**
- *"Can you mix them?"* — yes, and StreamPulse does: annotations for the fixed dependencies (s3,
  dynamodb), functional for the per-tenant partner path. Consistency within a module matters more than
  a global rule.
- *"How are registries configured?"* — the Spring Boot starter builds `CircuitBreakerRegistry` etc.
  from your YAML, including the `configs.default`/`configs.shared` inheritance via `baseConfig`.
  A registry lookup for an unknown name creates an instance from the default config, which is what
  makes the dynamic per-tenant pattern work.
- *"What about `@Bulkhead(type = THREADPOOL)`?"* — it exists, but the annotated method must return a
  `CompletableFuture`, and the thread hand-off loses MDC/SecurityContext unless you configure a
  `ContextPropagator`. Another reason the functional form is clearer for that case.

**Pitfalls / red flags:** annotating a private method, or a method called only from inside its own
class, and believing it's protected.

**Where in StreamPulse:** BS-04 throughout; the ArchUnit rule from file 01 (no field injection) has a
sibling here: no Resilience4j annotation on a non-public method.

---

## Q12. How does Resilience4j behave with reactive types and with virtual threads?

**30-second answer:** For reactive code you use the `resilience4j-reactor` module's operators via
`transformDeferred` — `CircuitBreakerOperator.of(cb)`, `RetryOperator`, `RateLimiterOperator`,
`BulkheadOperator`. Two caveats: `ThreadPoolBulkhead` is **not supported** with reactive types (a
thread hand-off is exactly what a reactive stack exists to avoid), so you use the semaphore bulkhead;
and for timeouts you use Reactor's own `Mono.timeout()` rather than `TimeLimiter`, because it
integrates with cancellation properly. For **virtual threads**, the relevant shift is that blocking is
cheap again: a call parked on a virtual thread releases its carrier, so thread exhaustion — the thing
`ThreadPoolBulkhead` was invented for — is much less of a concern. That makes the plain blocking style
with a **semaphore** bulkhead the natural fit, and it's why BS-02 / REQ-CC-05 pairs
`newVirtualThreadPerTaskExecutor()` with a `Semaphore`: unbounded virtual threads still need a bound,
just not a *thread-count* bound — a **downstream-concurrency** bound.

**Deeper:**

| | Blocking (platform threads) | Reactive (WebFlux) | Virtual threads |
|---|---|---|---|
| Cost of a blocked call | one OS thread (~1 MB stack) | none (no thread blocks) | ~a few hundred bytes; carrier released |
| Bulkhead of choice | `ThreadPoolBulkhead` for isolation | `SemaphoreBulkhead` (`BulkheadOperator`) | `SemaphoreBulkhead` |
| Timeout | `TimeLimiter` | `Mono.timeout()` | `TimeLimiter`, or just a client socket timeout |
| Why you still need a bulkhead | thread exhaustion | event-loop and downstream protection | **downstream** protection — you can still send 50k concurrent requests to a service that handles 50 |
| StreamPulse | sink-writer (Phase 4) | query-api (Phase 5) | ingest-gateway (Phase 2) |

That fourth row is the key insight and the one interviewers reward: **virtual threads remove the
thread-count constraint, not the need for a concurrency limit.** Without a bound, an ingest spike
creates 50,000 virtual threads all hitting DynamoDB simultaneously — you have simply moved the
queue from your thread pool (where it was visible and bounded) to the dependency (where it becomes
throttling and timeouts). The bulkhead's job was never really "save threads"; it was always "don't
send more concurrent work to a dependency than it can absorb". Virtual threads change the
implementation, not the requirement.

**Pinning** is the virtual-thread caveat to mention: a virtual thread that blocks inside a
`synchronized` block pins its carrier thread, defeating the model. Resilience4j's own internals use
locks in places, and some client libraries use `synchronized` heavily. The JDK's pinning behaviour
around `synchronized` has been improving across recent releases, so the practical answer is: run with
`-Djdk.tracePinnedThreads` (Phase 9's JVM lab, REQ) and look, rather than assume.
**Verify** both the current pinning behaviour for your JDK and whether your R4j version does anything
Loom-specific.

**Code:**

```java
/**
 * Phase 5 — query-api (WebFlux). Note: NO ThreadPoolBulkhead (unsupported on reactive types),
 * and Reactor's own timeout rather than TimeLimiter.
 */
public Mono<PanelData> loadTrendPanel(String tenantId, String titleId) {
    return dynamoReactive.query(tenantId, titleId)
            .transformDeferred(BulkheadOperator.of(bulkheads.bulkhead("queryApiPanels")))
            .transformDeferred(CircuitBreakerOperator.of(breakers.circuitBreaker("dynamodb")))
            .transformDeferred(RetryOperator.of(retries.retry("dynamodb")))
            // Reactor's timeout: cancels the subscription, which a non-blocking client
            // turns into an actual aborted request. TimeLimiter cannot do that here.
            .timeout(Duration.ofMillis(150))
            // BS-02 / REQ-CC-02: a degraded panel, explicitly marked. Not an empty list.
            .onErrorResume(e -> Mono.just(PanelData.degraded("trend", e.getClass().getSimpleName())));
}
```

```java
/**
 * BS-02 / REQ-CC-05 — the virtual-thread form. The Semaphore IS the bulkhead.
 * The ADR's argument: an unbounded virtual-thread executor protects your JVM's threads
 * and destroys your dependency. Bound the DOWNSTREAM concurrency, not the thread count.
 */
@Bean
ExecutorService s3UploadExecutor() {
    return Executors.newVirtualThreadPerTaskExecutor();   // cheap threads, unbounded
}

@Component
class VirtualThreadS3Offload {

    private final Semaphore downstreamLimit = new Semaphore(16);   // ← the real bound

    void upload(RawBatch batch) throws InterruptedException {
        // tryAcquire with a timeout, NOT acquire() — an unbounded wait re-creates the
        // problem in a different place (now you have 50k parked virtual threads).
        if (!downstreamLimit.tryAcquire(50, TimeUnit.MILLISECONDS)) {
            spool.writeAndFsync(batch);         // shed to the same fallback as the breaker (Q4)
            return;
        }
        try {
            executor.submit(() -> s3.putObject(request(batch), body(batch)));
        } finally {
            downstreamLimit.release();
        }
    }
}
```

**Follow-ups they'll ask:**
- *"Does `@CircuitBreaker` work on a method returning `Mono`?"* — the Spring Boot starter has reactive
  support for the annotations, but I prefer the explicit operator form on reactive code because the
  ordering and the subscription semantics are visible. **Verify** the annotation's reactive support in
  your R4j version.
- *"Where does the retry happen on a reactive stream?"* — `RetryOperator` resubscribes, which means
  your `Mono` must be **cold and re-subscribable**. Retrying a hot source, or one with a side effect
  performed outside the subscription, does the wrong thing. This is a genuinely subtle bug.
- *"Would you migrate WebFlux code to virtual threads?"* — only where the reactive style was adopted
  *purely* for thread efficiency rather than for streaming/backpressure semantics. SSE fan-out
  (Phase 5) genuinely wants Reactor's operators; a blocking-to-blocking service does not. That's the
  Phase 5 benchmark ADR: same endpoint on MVC+virtual threads vs WebFlux, with my own numbers.

**Pitfalls / red flags:** using `ThreadPoolBulkhead` on a reactive path, or claiming virtual threads
make bulkheads unnecessary.

**Where in StreamPulse:** Phase 5 (reactive query-api), Phase 2 (virtual-thread ingest), BS-02 /
REQ-CC-05, Phase 9 (the comparison).

**Verify:** `ThreadPoolBulkhead` reactive support status, annotation support for reactive return
types, and current virtual-thread pinning behaviour for your JDK.

---

## Q13. A dependency is degraded. How do you decide between retry, breaker, shed, and queue?

**30-second answer:** By asking two questions in order. **(1) Is the work durable elsewhere?** If the
input came from Kafka and I haven't committed the offset, the cheapest correct response is to **stop
consuming** — pause the listener, let lag build, resume when the dependency recovers. Lag is
recoverable; a DLQ flood is manual work. **(2) If I must accept the work now, can I defer it?** Then
queue it (RabbitMQ delay queue) or spool it (disk + replay). Only if neither is possible do I shed —
and shedding must be visible (429 + Retry-After, or a `DEGRADED` marker), never a silent drop. Retry
handles the seconds-scale blip; the breaker handles the minutes-scale outage by stopping the retries
from making it worse. The thing I try hardest to avoid is turning a dependency outage into
**permanent data loss**, because that's the only failure you can't undo after the incident.

**Deeper:** the decision table (this is the `docs/resilience.md` deliverable, REQ-RS-09):

| Situation | Source of work | Response | Why |
|---|---|---|---|
| DynamoDB throttles for ~2s | Kafka (uncommitted) | **Retry** with jittered backoff | it will succeed; nothing else needs to change |
| S3 down for 3 min | Kafka (uncommitted) | **Breaker → spool to disk → replay**, commit only after fsync | preserves every event (REQ-RS-02 NFR: zero loss) |
| S3 down for 3 min, spool full | Kafka (uncommitted) | **Stop committing offsets; pause the container** | backpressure to the log — lag is recoverable |
| Partner webhook slow | Kafka (alerts.v1) | **Bulkhead per tenant → shed → RabbitMQ delay queue** | isolate the blast radius; defer, don't drop |
| Partner at its rate limit | Kafka | **RateLimiter → queue** | a contract, not a failure |
| Ingest spike beyond quota | HTTP (client holds it) | **429 + Retry-After** | the client can retry; we must not buffer unboundedly |
| Redis (dedup cache) down | HTTP | **Fail open**, rely on the sink's conditional write | correctness is defended at the sink, not the cache |
| Control-plane Postgres down | HTTP admin API | **Fail closed** (503) | writes cannot be faked; a 200 here would be a lie |

**The pattern behind the table:** *who is holding the work?* If a durable log holds it, apply
backpressure — that's the cheapest and safest response and it's uniquely available to event-driven
systems. If a *client* holds it, tell them to come back (429). If *you* hold it in memory, you have a
deadline: persist it, hand it off, or lose it — and "lose it" must be a deliberate, metered decision,
not an implicit consequence of an unbounded queue overflowing.

This is also the honest answer to "what's the limit of Resilience4j?" — it gives you the *mechanism*
for each response, but the decision above is architecture, not library configuration. The library
can't tell you that spooling to disk beats dropping; you have to know the domain.

**Code:** the pause/resume response, which is the one people don't think of:

```java
/**
 * BS-04 + BS-01. When the DynamoDB breaker opens, DLT-ing 50k records would be the wrong
 * response: they aren't poison, the dependency is down. Pause the container instead —
 * lag grows (visible, alertable, recoverable), nothing is lost, and we resume automatically.
 */
@Component
class BreakerDrivenConsumerControl {

    BreakerDrivenConsumerControl(CircuitBreakerRegistry registry,
                                 KafkaListenerEndpointRegistry containers) {

        registry.circuitBreaker("dynamodb").getEventPublisher()
                .onStateTransition(e -> {
                    var container = containers.getListenerContainer("aggregate-sink");
                    switch (e.getStateTransition().getToState()) {
                        // pause(), not stop(): keeps the partition assignment and keeps
                        // heartbeating, so NO rebalance is triggered (Kafka file, Q3).
                        case OPEN      -> container.pause();
                        case CLOSED    -> container.resume();
                        case HALF_OPEN -> container.resume();   // let probes flow
                        default        -> { }
                    }
                });
    }
}
```

**Follow-ups they'll ask:**
- *"How long can you pause before it's a problem?"* — until retention expires on the topic. With a
  3-day retention (Kafka file, Q1) you have enormous headroom, which is precisely the property that
  makes "just stop consuming" viable in a log-based system and impossible in a request/response one.
  Alert on the lag *trend* so you know how much time you actually have.
- *"What if the outage is longer than the spool and the retention?"* — then you're in a genuine
  capacity/DR conversation, and the answer is documented ahead of time (BS-06's RPO/RTO targets), not
  improvised at 3am.
- *"How do you test all this?"* — Toxiproxy in front of Kafka/Redis/Postgres and moto fault injection
  for S3/DynamoDB (Phase 9). The acceptance tests are named in BS-04: `S3OutageSpoolReplayIT`,
  `PartnerRateLimitIT`, `TenantBulkheadIsolationIT`, `CircuitBreakerStateTransitionTest`,
  `RetryJitterTest`.

**Pitfalls / red flags:** reaching for a DLQ as the response to a *dependency* outage. A DLQ is for
poison messages, not for "the database is down".

**Where in StreamPulse:** BS-04 / REQ-RS-09 (`docs/resilience.md`), Phase 9 chaos suite.

---

## Q14. Walk me through BS-04: peak night, four dependencies degrade at once. What happens?

*(The applied question — knowledge into evidence.)*

**30-second answer:** Four independent failures, four different patterns, and crucially **no shared
blast radius**. (a) DynamoDB throttles: `@Retry` with exponential backoff **plus jitter** absorbs it;
5 attempts over ~1.5s; nobody is paged because it self-heals, and `resilience4j.retry.calls` shows it
happened. (b) S3 returns 503s for three minutes: after 10 failures in a 20-call count-based window the
breaker opens, and the fallback **spools batches to a bounded local disk buffer, fsync'd** — and
critically, the Kafka offsets are only committed after that fsync, so if the spool fills we stop
committing and let lag build rather than lose data. `automaticTransitionFromOpenToHalfOpen` closes the
breaker as soon as S3 recovers, and a replayer drains the spool. Zero raw events lost. (c) The partner
webhook allows 50 req/s: a `RateLimiter` sized at `50/replicas` with `timeoutDuration=0` fails fast,
and the fallback enqueues to RabbitMQ for delayed delivery — we never exceed the contract. (d) The
slow partner calls are confined by a **per-tenant `ThreadPoolBulkhead`** (core 2, max 4, queue 20), so
`sportsnow`'s stuck threads are `sportsnow`'s; `kidsplus` alert latency stays under 2s. Every one of
those is wrapped in a TimeLimiter, because none of it works if calls never return.

**The failure-mode table I'd draw:**

| | Without the pattern | With it |
|---|---|---|
| (a) DynamoDB throttles 2s | writes fail, records to DLT, manual replay | retried with jitter, invisible to users |
| (a') …and no jitter | 500 pods retry in lockstep, sustaining the throttle | staggered, dependency recovers |
| (b) S3 503 for 3 min | every batch fails, 3 min × 50k ev/s to a DLT or dropped | breaker opens in <1s of calls, spool + replay, **0 lost** |
| (b') …and no slow-call threshold | S3 *slow* rather than failing → threads park, breaker never trips | `slowCallRateThreshold` trips on 2s+ calls |
| (c) Partner rate limit | we exceed 50 req/s, partner 429s or bans us, contract breach | limiter + queue, contract honoured |
| (d) Slow partner | alert evaluation stalls **for all tenants** | per-tenant bulkhead; `kidsplus` < 2s p99 |
| all | one breaker for "AWS" opens on S3 and blocks DynamoDB too | one named instance per dependency |

**The sequence for (b), which is the one worth narrating:**

```
batch ready
  └─ TimeLimiter(2s) ─ CircuitBreaker(s3) CLOSED ─ PUT ─▶ 503
     ... 10 failures recorded in a 20-call window, ≥ minimumNumberOfCalls, rate 100% ≥ 50%
  └─ breaker → OPEN
     └─ next call: CallNotPermittedException in microseconds (no 2s timeout burned)
        └─ fallback: spool.writeAndFsync(batch)          ← work PRESERVED
           └─ Kafka offset committed ONLY after fsync    ← BS-01 / REQ-LC-02
              └─ if spool ≥ 1 GB: do NOT commit, pause container  ← backpressure to the log
  ... 30s waitDurationInOpenState, automaticTransition → HALF_OPEN
  └─ 5 permitted probe PUTs succeed → CLOSED
     └─ SpoolReplayer (subscribed to the state-transition event) drains the spool to S3, deletes files
```

**What I'd measure to prove it works:** `S3OutageSpoolReplayIT` counts events accepted at ingest
versus distinct `eventId`s landed in S3 after a scripted 3-minute moto/Toxiproxy outage — the same
shape of proof as BS-01's `RollingRestartChaosIT`. `TenantBulkheadIsolationIT` asserts the `kidsplus`
latency bound while `sportsnow` is saturated. `PartnerRateLimitIT` asserts the observed rate at a
WireMock partner is ≤ 50 rps. `RetryJitterTest` asserts intervals fall inside the randomised bounds —
the test that proves jitter is actually enabled rather than configured and ignored.

**Follow-ups they'll ask:**
- *"What's the cost of all this?"* — complexity, and a real risk of *hiding* problems. Six patterns
  and three fallbacks mean the system can run degraded for a long time without anyone noticing, which
  is why REQ-RS-08's fallback-rate metrics and the "breaker OPEN > 5 min" alert are not optional
  extras — they're what keeps graceful degradation from becoming silent degradation.
- *"What would you drop if you had to simplify?"* — the per-tenant breakers (the bulkheads already
  give isolation) and the rate limiter if the partner's own 429s were an acceptable signal. I'd never
  drop the timeouts or the spool fallback; those are the ones defending against irreversible loss.
- *"What's the weakest part of this design?"* — the rate limiter's coupling to the replica count
  (Q6). It is correct only while the replica count matches the configured divisor, and nothing in the
  system enforces that. The honest mitigations are an ADR that says so, an alert on the partner's
  observed 429 rate, and a plan to move to Bucket4j+Redis if the contract gets penalties attached.
- *"Where does the alert engine's CPU cost fit?"* — that's BS-08's extraction story; resilience
  patterns bound the *dependency* blast radius, not a CPU-bound noisy neighbour. Different problem,
  different fix (a separate service with its own scaling).

> **Your story:** if you've built circuit breakers, retry/backoff or bulkheads at Couchbase,
> JustAnswer or NetApp, this is where it goes — in the same shape: which dependency, what the failure
> looked like, which pattern, and how you proved it worked. Keep "I built" for what you personally
> wrote. BS-04 is a **fictional design scenario** for a portfolio project — present it as *"I built
> this to prove the patterns end to end"*, never as an outage you lived through.

**Where in StreamPulse:** BS-04 / REQ-RS-01…09 end to end; `docs/resilience.md`; Phases 2, 4, 6, 9.

---

## Key terms

| Term | One-line meaning |
|---|---|
| **CLOSED / OPEN / HALF_OPEN** | Normal / reject everything / allow N probes. The three operational breaker states. |
| **DISABLED / FORCED_OPEN** | Operator overrides: always allow (record nothing) / always reject. A kill switch without a deploy. |
| **`CallNotPermittedException`** | What a rejected call throws when the breaker is OPEN. Never retry it. |
| **Sliding window** | The sample the breaker judges on: last N *calls* (count-based) or last N *seconds* (time-based). |
| **`minimumNumberOfCalls`** | The statistical floor that stops "1 of 1 failed = 100%" from tripping the breaker. |
| **Slow-call rate** | Trips the breaker on calls that are *slow but successful* — the failure mode that actually takes systems down. |
| **Jitter** | Randomising retry waits so clients don't retry in lockstep and re-create the spike. |
| **Retry amplification** | Retries multiplying through layers (browser × gateway × app × SDK). Own exactly one layer. |
| **Retry budget** | Capping retries as a share of total traffic so an outage can't multiply your load. |
| **`RequestNotPermitted`** | What the RateLimiter throws when no permit is available within `timeoutDuration`. |
| **Semaphore bulkhead** | A concurrency counter; the call runs on the caller's thread. Cheap, context-preserving, the default. |
| **Thread-pool bulkhead** | A bounded pool + queue; true isolation off the caller's thread. Not supported on reactive types. |
| **`BulkheadFullException`** | Fast load shedding. A feature — provided the fallback preserves or visibly degrades the work. |
| **TimeLimiter** | Bounds how long the *caller waits*. Does not necessarily stop the work — that needs socket timeouts too. |
| **Decorator order** | `Retry( CircuitBreaker( RateLimiter( TimeLimiter( Bulkhead( call )))))` — documented nesting. |
| **Fallback** | A degraded answer. Must preserve the work or visibly mark it degraded, or it's a silent outage. |
| **Effectively degraded** | Running on fallbacks. Fine — as long as a metric says so and someone is watching it. |

---

## Self-check (answer out loud, no notes)

1. A dependency is slow but never errors. Which breaker setting saves you, and what happens without
   it?
2. Why must Retry sit outside CircuitBreaker, and which exception must be in `ignoreExceptions`?
3. You have 6 pods and a partner contract of 50 req/s. What do you configure, and what breaks when the
   HPA scales to 12?
4. Name two fallbacks that are better than failing, and two that are worse.
5. Virtual threads make blocking cheap. Do you still need a bulkhead? Why?
