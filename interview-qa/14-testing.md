# 14 — Testing at the Senior Bar: JUnit 5, Mockito, Slices, Testcontainers, Contracts, Load

**Why this matters for an Event Hub role:** seniors are trusted because their code is *provably*
correct against real dependencies, not because they wrote a lot of mocks. Testcontainers in particular
is a strong signal for someone moving from .NET — it's standard in the Java ecosystem and .NET folks
often arrive without it. The other senior marker is knowing what a test **proves**: a green
concurrency test proves almost nothing unless you can say why, and 90% line coverage says nothing at
all about whether your assertions are any good.

**Scenario anchor:** StreamPulse's plan names specific tests as the acceptance criteria for each
business scenario — `ShutdownOrderIT`, `RollingRestartChaosIT`, `QuotaTransferDeadlockTest`,
`CircuitBreakerStateTransitionTest`, `TenantTreeQueryCountTest`, `ProtocolParityTest`,
`ShadowComparatorIT`. Q14 goes through what each one actually proves. Being able to say *"this test
exists because that failure mode exists"* is the whole point.

---

## Q1. What does your test pyramid look like, and has Testcontainers changed it?

**30-second answer:** The classic pyramid says lots of unit tests, fewer integration, very few E2E —
and the reason was economics: integration tests used to mean a shared database, so they were slow,
flaky and environment-dependent. Testcontainers changed that economics. A real Postgres or Kafka in
Docker starts in seconds, is isolated per run, and needs no shared environment. So my pyramid is
wider in the middle than it used to be: I still unit-test domain logic exhaustively, but I no longer
mock the database to test a repository — I run the real one. What I did **not** change is the top:
full end-to-end tests are still few, because they're still slow and still tell you the least about
*where* something broke.

**Deeper:**

```
        ╱ E2E ╲            few — Playwright against the compose stack, nightly
      ╱ contract ╲         a handful — consumer/provider boundaries
    ╱ integration  ╲       MORE than the classic pyramid — Testcontainers made these cheap
  ╱   slice tests    ╲     @WebMvcTest / @DataJpaTest — fast, one layer at a time
╱      unit tests      ╲   the bulk — domain logic, plain `new`, no Spring
```

The thing that makes this work in StreamPulse is hexagonal architecture (file 01 Q10): the domain has
no Spring or Kafka imports, so the bottom layer is genuinely instant. If your business logic can only
be tested by booting the app, your pyramid is a rectangle no matter what you intended.

Separate the two suites in the build so the fast one can run on every save:

```kotlin
// build-logic/src/main/kotlin/test-conventions.gradle.kts
testing {
    suites {
        val test by getting(JvmTestSuite::class) { useJUnitJupiter() }   // fast, no Docker
        register<JvmTestSuite>("integrationTest") {                      // needs Docker
            useJUnitJupiter()
            targets.all { testTask.configure { shouldRunAfter(test) } }
        }
    }
}
```

**Follow-ups they'll ask:**
- *"What if Docker isn't available in CI?"* — it usually is on GitHub-hosted runners. If it truly isn't,
  that's an argument for fixing CI, not for mocking the database.
- *"Isn't the middle layer slow?"* — container startup is the cost, and the singleton-container pattern
  (Q6) amortizes it across the whole suite. StreamPulse pays it once, not once per test class.
- *"What do you NOT test?"* — framework behaviour. I don't test that `@Transactional` rolls back; I test
  that *my* audit record survives a rollback (file 01 Q9), which is my logic depending on it.

**Pitfalls / red flags:** reciting the pyramid as dogma without the economics argument. The interesting
answer is *why the shape changed*.

**Where in StreamPulse:** every phase; `./gradlew test` (no Docker) vs `./gradlew integrationTest`.

---

## Q2. Which JUnit 5 features do you actually use?

**30-second answer:** Parameterized tests for anything with a table of cases — they turn ten
copy-pasted tests into one with ten rows, and the failure message tells you which row. `@Nested` to
group tests by the state they share, so the class reads like a specification. Lifecycle annotations
with `@TestInstance(PER_CLASS)` when setup is expensive. Extensions instead of inheritance for
cross-cutting setup. And `assertThatThrownBy`-style exception assertions — I use AssertJ rather than
raw JUnit assertions because the failure messages are dramatically better, and a test's failure message
is the only part of it you read under pressure.

**Deeper:**

```java
class RetryIntervalTest {

    /** REQ-RS-01: exponential backoff with jitter. One test, five cases, precise failure output. */
    @ParameterizedTest(name = "attempt {0} → interval within [{1}, {2}] ms")
    @CsvSource({
        "1,  100,  200",
        "2,  200,  400",
        "3,  400,  800",
        "4,  800, 1600",
        "5, 1600, 3200"
    })
    void backoffStaysWithinRandomizedBounds(int attempt, long lowerMs, long upperMs) {
        var fn = IntervalFunction.ofExponentialRandomBackoff(
                Duration.ofMillis(100), 2.0, 0.5);        // base, multiplier, randomization factor
        assertThat(fn.apply(attempt)).isBetween(lowerMs, upperMs);
    }

    @Nested
    @DisplayName("when the circuit breaker is OPEN")
    class WhenOpen {
        @BeforeEach void openIt() { breaker.transitionToOpenState(); }

        @Test void callsAreRejectedWithoutInvokingTheSupplier() { /* ... */ }
        @Test void theFallbackSpoolsToDisk()                    { /* ... */ }
    }
}
```

`@DisplayName` earns its place when you tie tests to requirement IDs — StreamPulse uses
`@DisplayName("REQ-LC-02: unflushed buffer does not commit offsets")`, so a failing build names the
requirement that broke, not just a method name.

**Follow-ups they'll ask:**
- *"`@ParameterizedTest` sources?"* — `@ValueSource`, `@CsvSource`, `@EnumSource`, `@MethodSource` for
  complex objects, `@ArgumentsSource` for custom providers.
- *"How do you share expensive setup?"* — `@TestInstance(PER_CLASS)` with `@BeforeAll` non-static, or a
  JUnit `Extension`, or a static singleton container (Q6).
- *"Parallel execution?"* — JUnit 5 supports it via configuration, but it will expose every piece of
  shared mutable state in your test suite. Worth doing, expect a cleanup phase first.

**Verify:** JUnit 5's parallel-execution property names (`junit.jupiter.execution.parallel.*`) against
current docs before quoting them.

---

## Q3. Mock, stub, spy, fake — define them, and tell me when you'd use a fake instead of a mock.

**30-second answer:** A **stub** returns canned answers. A **mock** is a stub you also assert
interactions on. A **spy** wraps a real object so most calls go through but some are stubbed. A **fake**
is a working lightweight implementation — an in-memory map standing in for a repository. My rule is:
mock at the boundary you own, fake anything with real behaviour worth preserving, and never mock a type
you don't own. Mocking someone else's library encodes your *belief* about how it behaves; if the belief
is wrong, the test passes and production fails. That's the worst possible test outcome.

**Deeper:**

| Double | Returns canned data | Asserts interactions | Has real behaviour | Use when |
|---|---|---|---|---|
| Stub | ✅ | ❌ | ❌ | you need a value to proceed |
| Mock | ✅ | ✅ | ❌ | the *interaction* is the thing under test |
| Spy | partly | ✅ | mostly | legacy code you can't restructure (a smell) |
| Fake | ✅ | ❌ | ✅ | the collaborator has behaviour you rely on |

"Never mock what you don't own" in practice: don't mock `KafkaTemplate` and assert `send()` was called.
That test passes whether or not the record is serializable, whether or not the topic exists, and whether
or not your partition key does what you think. Use Testcontainers Kafka and assert on the record that
actually landed. StreamPulse does exactly this — `ProtocolParityTest` asserts on **real Kafka records**,
which is why it's meaningful evidence.

**Code — a fake that outperforms a mock:**

```java
/** In-memory IdempotencyStore. Real behaviour: second call with the same key returns false. */
class InMemoryIdempotencyStore implements IdempotencyStore {
    private final Set<String> seen = ConcurrentHashMap.newKeySet();

    @Override public boolean claim(String eventId, Duration ttl) {
        return seen.add(eventId);       // genuinely idempotent — a mock would have to be told this
    }
}
```

With a mock you'd write `when(store.claim(any(), any())).thenReturn(true)` and your dedup logic would
never actually be exercised. The fake makes the duplicate case work for free, including under
concurrency.

**Follow-ups they'll ask:**
- *"When is `verify()` right?"* — when the call *is* the outcome: "the DLQ publisher was invoked once",
  "the audit service was called before validation". When there's a state assertion available, prefer it.
- *"`@Mock` vs `@InjectMocks`?"* — `@InjectMocks` uses reflection to guess wiring; I prefer constructing
  the object with `new` and passing mocks explicitly. It's the same benefit as constructor injection
  (file 01 Q1): explicit beats magic.
- *"Mocking statics and finals?"* — possible with `mockito-inline`, but needing it is usually a sign
  there's a seam missing. Wrap the static in a thin interface instead.

**Pitfalls / red flags:** a test class with eight mocks and six `verify()` calls. That test asserts the
implementation, so it fails on every refactor and catches no bugs — it's a change-detector, not a test.

---

## Q4. What Mockito details trip people up?

**30-second answer:** Three. **Mixing matchers and raw values** — if any argument uses a matcher, all
of them must, or you get `InvalidUseOfMatchersException`. **`ArgumentCaptor`**, which is how you assert
on an object you don't construct — indispensable for checking what got published to Kafka or written to
the DLQ. And **`verify(mock, never())`**, which is how you prove a negative, like "the fallback was
*not* invoked while the breaker was closed". The subtle one is that stubbing a `void` method needs the
inverted syntax, `doThrow().when(mock).method()`, because you can't pass a void call to `when()`.

**Code:**

```java
@Test
void poisonMessageGoesToDltWithTheOriginalHeaders() {
    var captor = ArgumentCaptor.forClass(ProducerRecord.class);

    consumer.handle(malformedRecord());

    verify(dltPublisher).publish(captor.capture());
    ProducerRecord<?, ?> sent = captor.getValue();
    assertThat(sent.topic()).isEqualTo("telemetry.raw.v1.dlt");
    assertThat(sent.headers().lastHeader("x-original-offset")).isNotNull();

    verify(kafkaProducer, never()).send(any());     // proving the negative: it was NOT reprocessed
}

// void method stubbing — note the inverted form
doThrow(new S3Exception("503")).when(s3Client).putObject(any(), any());
```

**Follow-ups they'll ask:**
- *"`when/thenReturn` vs `doReturn/when`?"* — `doReturn` avoids actually invoking the real method,
  which matters on spies and on methods that would throw during stubbing.
- *"What is strict stubbing?"* — JUnit 5's Mockito extension fails the test on unused stubs by default.
  Keep it on; an unused stub means either a dead test path or a stub that's silently not matching.
- *"How do you test a `CompletableFuture`-returning collaborator?"* — return
  `CompletableFuture.completedFuture(x)` or `failedFuture(ex)`; for timing behaviour, return a future
  you complete manually so the test controls the schedule instead of sleeping.

---

## Q5. What are Spring test slices, and why can inconsistent test configuration slow your whole suite?

**30-second answer:** Slices boot a *subset* of the context: `@WebMvcTest` gives you the MVC layer and
your controller with no database, `@DataJpaTest` gives you JPA and a transaction with no web layer,
`@WebFluxTest` the reactive equivalent. They're fast and they fail for one reason at a time. The
performance trap is the **context cache**: Spring caches a booted context and reuses it across test
classes that request an *identical* configuration. Every variation — a different `@MockitoBean`, an
extra property, a different active profile — is a cache miss and a fresh context boot. A suite with
thirty slightly-different configurations boots thirty contexts, and people conclude "Spring tests are
slow" when the real problem is configuration sprawl.

**Deeper:**

| Annotation | Boots | Use for |
|---|---|---|
| `@WebMvcTest(X.class)` | MVC infra + that controller | routing, validation, error mapping, serialization |
| `@WebFluxTest` | WebFlux + controller | same, reactive; pairs with `WebTestClient` |
| `@DataJpaTest` | JPA, repositories, tx (rolls back by default) | queries, mappings, N+1 counting |
| `@JsonTest` | Jackson only | serialization contracts |
| `@SpringBootTest` | everything | wiring, and only when you need it |

The practical rule: **standardize your test configuration into a small number of shared base classes**,
so the cache hits. StreamPulse has one `@SpringBootTest` base per service plus the slices, not a
bespoke configuration per test.

**Code:**

```java
@WebMvcTest(IngestController.class)
class IngestControllerTest {

    @Autowired MockMvc mvc;
    @MockitoBean IngestUseCase ingest;      // the use case is mocked; only the web layer is real

    @Test
    void rejectsBatchOver500WithProblemDetail() throws Exception {
        mvc.perform(post("/v1/events:batch")
                        .contentType(APPLICATION_JSON)
                        .content(batchOf(501)))
           .andExpect(status().isBadRequest())
           .andExpect(content().contentTypeCompatibleWith("application/problem+json"))
           .andExpect(jsonPath("$.title").value("Validation failed"));

        verifyNoInteractions(ingest);       // the request never reached the domain
    }
}
```

**Follow-ups they'll ask:**
- *"`@SpringBootTest(webEnvironment=...)`?"* — `MOCK` (default, no server), `RANDOM_PORT` (real server,
  use `TestRestTemplate`/`WebTestClient`), `DEFINED_PORT`, `NONE`. Use `RANDOM_PORT` in CI so parallel
  builds don't collide.
- *"Why does `@DataJpaTest` hide N+1 problems?"* — it runs inside an open transaction, so lazy loading
  always succeeds; in production the session may already be closed. Test the query *count*
  (`TenantTreeQueryCountTest`), not just that the data comes back.
- *"How do you inspect the cache?"* — enable `logging.level.org.springframework.test.context.cache=DEBUG`
  and it reports hits, misses and evictions.

**Verify:** `@MockitoBean` replaced `@MockBean` in recent Spring Framework/Boot versions — confirm which
annotation your pinned version wants. Also confirm the context-cache logging category name.

**Where in StreamPulse:** every service; `TenantTreeQueryCountTest` (BS-05 / REQ-DL-03).

---

## Q6. How do you use Testcontainers without making the build slow?

**30-second answer:** Three techniques. **Singleton containers** — a static container started once in a
base class and shared by the whole suite, rather than one per test class; Testcontainers' Ryuk sidecar
cleans up at JVM exit. **Reuse** across local runs during development, so you're not restarting Postgres
every time you hit run. And **`@ServiceConnection`**, which wires the container's dynamic host and port
into Spring's properties automatically, so you stop hand-writing `@DynamicPropertySource` blocks. The
combination means my integration suite pays container startup roughly once.

**Code:**

```java
/** Shared by every integration test in the module — started once per JVM. */
public abstract class AbstractIntegrationTest {

    @Container
    @ServiceConnection                              // Boot wires spring.datasource.* automatically
    static final PostgreSQLContainer<?> POSTGRES =
            new PostgreSQLContainer<>("postgres:16-alpine");

    @Container
    @ServiceConnection
    static final KafkaContainer KAFKA =
            new KafkaContainer(DockerImageName.parse("apache/kafka:latest"));  // pin a real tag

    static {
        POSTGRES.start();                           // static init = started once, reused, never stopped
        KAFKA.start();                              // Ryuk removes them when the JVM exits
    }
}
```

Two cautions worth volunteering: **pin image tags**, because `latest` makes your build
non-reproducible and one day breaks CI for reasons unrelated to your change; and remember that with a
shared container, **tests are no longer isolated by default** — you need either per-test schemas/topics
or disciplined cleanup, or you get order-dependent failures.

**Follow-ups they'll ask:**
- *"Testcontainers vs an embedded/in-memory database?"* — H2 is not Postgres. Every `JSONB` column,
  every `FOR UPDATE SKIP LOCKED`, every window function and every MVCC behaviour you rely on is either
  absent or different. Testing against a different database than you deploy is testing something else.
- *"How does this work in CI?"* — it needs a Docker daemon; GitHub-hosted runners provide one.
- *"What's Ryuk?"* — the sidecar container that reaps containers when the JVM that created them exits,
  so an abandoned build doesn't leave containers running.

**Verify:** `@ServiceConnection` (Boot 3.1+) support for each container type, the `KafkaContainer`
class name and image coordinates for your Testcontainers version, and the reuse flag
(`testcontainers.reuse.enable`).

**Where in StreamPulse:** every `*IT` test; Phase 0 sets up the `integrationTest` source set.

---

## Q7. How do you test Kafka code?

**30-second answer:** It depends which part. For a **Kafka Streams topology** I use `TopologyTestDriver`
— no broker at all, synchronous, millisecond-fast, and crucially it lets me control **time**, so I can
test out-of-order events, late arrivals past the grace period and window closing deterministically.
That's impossible against a real broker. For **producers and consumers** I use Testcontainers Kafka,
because the things that break there are serialization, partition assignment, offset commit and DLT
routing — all of which a mock would hide. The rule that keeps the suite sane: never `Thread.sleep` for
an async result, use Awaitility.

**Code:**

```java
/** BS-03 / Phase 3: late-arriving events, tested deterministically with no broker. */
@Test
void eventArrivingAfterGracePeriodIsDroppedNotCounted() {
    var input  = driver.createInputTopic("telemetry.raw.v1", keySerde.serializer(), valueSerde.serializer());
    var output = driver.createOutputTopic("telemetry.agg.v1", keySerde.deserializer(), aggSerde.deserializer());

    Instant windowStart = Instant.parse("2026-01-01T00:00:00Z");

    input.pipeInput("sess-1", heartbeat(), windowStart);                       // inside the window
    input.pipeInput("sess-2", heartbeat(), windowStart.plusSeconds(15));       // closes the 10s window
    input.pipeInput("sess-3", heartbeat(), windowStart.plusSeconds(1));        // LATE: 14s after close

    var results = output.readKeyValuesToList();
    assertThat(countFor(results, windowStart)).isEqualTo(1);   // sess-3 excluded, grace period exceeded
}
```

```java
// Async assertions against a real broker — never Thread.sleep.
await().atMost(Duration.ofSeconds(10))
       .untilAsserted(() -> assertThat(dltConsumer.poll()).hasSize(1));
```

**Follow-ups they'll ask:**
- *"`EmbeddedKafka` vs Testcontainers?"* — `EmbeddedKafka` (Spring Kafka test) starts faster and needs no
  Docker, but it's not the same broker you deploy. I'd use it for a quick listener wiring test and
  Testcontainers for anything whose correctness depends on real broker behaviour.
- *"How do you test consumer rebalancing?"* — start a second consumer in the same group and assert
  partition reassignment and that no records were double-processed. This is how you validate the
  idempotent-consumer claim rather than asserting it in a design doc.
- *"How do you test a DLT?"* — feed a poison record, await its arrival on the `.dlt` topic, and assert
  the original headers survived (`DlqRetryIT` in BS-07 does the RabbitMQ equivalent).

**Verify:** `TopologyTestDriver`'s current API (`createInputTopic`/`createOutputTopic` replaced older
`pipeInput`/`readOutput` forms) for your Kafka Streams version.

**Where in StreamPulse:** Phase 3 (topology), Phase 4 (`@RetryableTopic` + DLT), BS-03.

---

## Q8. How do you test concurrent code, and what does a passing concurrency test actually prove?

**30-second answer:** Honestly? Usually less than people think. A concurrency bug is a *schedule*, and
a normal test exercises one arbitrary schedule — passing means "this bug didn't happen this time." So I
do three things to raise the odds: force genuine simultaneity with a `CountDownLatch` so threads start
at the same instant rather than sequentially; run the critical section in a loop, because interleavings
are sampled, not enumerated; and assert an **invariant** rather than an outcome — total quota conserved,
no lost increments, no deadlock — because invariants hold under every schedule. For the deadlock case
specifically I can do better than sampling: `ThreadMXBean.findDeadlockedThreads()` detects it
deterministically.

**Code — REQ-CC-07, the quota-transfer deadlock lab:**

```java
@Test
void crossedTransfersDeadlockWithNaiveLockingAndDoNotWithOrderedLocking() throws Exception {
    var barrier = new CyclicBarrier(2);          // force the two threads to collide, not queue
    var pool = Executors.newFixedThreadPool(2);

    // v1 locks `from` then `to` → thread A holds streamflix, wants sportsnow; B holds the reverse.
    pool.submit(() -> { barrier.await(); naive.transfer("streamflix", "sportsnow", 100); return null; });
    pool.submit(() -> { barrier.await(); naive.transfer("sportsnow", "streamflix", 100); return null; });

    // Deterministic detection — no sleeping, no guessing.
    await().atMost(Duration.ofSeconds(5)).until(() ->
            ManagementFactory.getThreadMXBean().findDeadlockedThreads() != null);

    // v2 orders locks by tenantId and uses tryLock(timeout): 10k crossed transfers, no deadlock,
    // and the invariant holds.
    long before = ordered.totalQuota();
    runCrossedTransfers(ordered, 10_000);
    assertThat(ordered.totalQuota()).isEqualTo(before);      // conservation: true under ANY schedule
    assertThat(ManagementFactory.getThreadMXBean().findDeadlockedThreads()).isNull();
}
```

**Follow-ups they'll ask:**
- *"So how do you get real confidence?"* — invariant assertions plus stress iterations, code review of
  the locking protocol, and for lock-free/memory-model questions, **jcstress**, which actually
  enumerates interleavings under the JMM. Most teams don't need jcstress; knowing it exists and when
  it's warranted is the senior signal.
- *"Why is `Thread.sleep` wrong here?"* — it makes the test slow *and* flaky: too short and it fails
  spuriously, too long and the suite crawls. Latches and Awaitility express the actual condition.
- *"Can you test a race in CI reliably?"* — a *reproduction* usually can't be made reliable; that's why
  I convert it into an invariant test plus the fix, and I keep the failing v1 in a lab test as
  documentation of the bug.

**Pitfalls / red flags:** "I wrote a test with 100 threads and it passed, so it's thread-safe."

**Where in StreamPulse:** BS-02 / REQ-CC-07, REQ-CC-08; also `TenantContextConcurrencyTest` (BS-01) and
`QuotaAllocationConcurrencyIT` (BS-05).

**Verify:** jcstress usage and current coordinates before recommending it as a build dependency.

---

## Q9. What problem does contract testing solve that integration tests don't?

**30-second answer:** Scale and coupling. To catch a breaking change with integration tests you'd have
to run the consumer against the real provider — which means every consumer needs the provider deployed,
and the test matrix grows with every new consumer. Consumer-driven contract testing inverts it: each
consumer publishes what it actually *uses* from the provider's API, and the provider verifies all those
contracts in its own pipeline, alone. The provider learns it broke someone **before merging**, with no
shared environment. It's the cheapest way to make "don't break your consumers" mechanical.

**Deeper:**

| | Integration test | Contract test |
|---|---|---|
| Needs both sides deployed | ✅ | ❌ |
| Catches breaking change | at integration time | at provider build time |
| Scales with N consumers | poorly (N×M) | linearly |
| Verifies business behaviour | ✅ | ❌ — only the contract shape |

They're complementary, not alternatives: contract tests verify the *interface*, integration tests verify
the *behaviour*.

Worth noting for this role: **gRPC and Avro get much of this for free** (file 10 Q3). The `.proto` with
a CI breaking-change check, and Schema Registry with BACKWARD compatibility, are contract enforcement
built into the format. Contract testing frameworks matter most for JSON/REST, where nothing stops you
from deleting a field.

**Follow-ups they'll ask:**
- *"Pact vs Spring Cloud Contract?"* — Pact is consumer-driven with a broker and is polyglot; Spring
  Cloud Contract is provider-driven (the provider writes contracts, stubs are generated for consumers)
  and is JVM-centric. Pick by whether your consumers are in other languages.
- *"Where would you use it in StreamPulse?"* — at the `query-api` ↔ `web` boundary, because the React app
  is a separate build. Internal Kafka boundaries are covered by Avro compatibility instead.

**Where in StreamPulse:** Phase 1 (schemas), Phase 5 (query-api ↔ web), Phase 7.

---

## Q10. How do you load-test properly, and what's the open vs closed workload distinction?

**30-second answer:** A **closed** model has a fixed number of virtual users, each waiting for a response
before sending the next request — so if the system slows down, the load automatically decreases. An
**open** model injects at a fixed *arrival rate* regardless of how the system is coping. Real internet
traffic is open: users arrive whether or not you're struggling. So a closed-model test can never
reproduce a saturation collapse — it self-throttles exactly when things get interesting. For BS-03's
"live match kickoff" spike I use an open model, because the entire point is what happens when arrivals
exceed capacity. And I always discard a warmup period, because the JVM's first thousand requests are
interpreted, not JIT-compiled (file 15).

**Deeper:**

```
CLOSED (constant users)          OPEN (constant arrival rate)
50 users, each waits for a reply  1000 req/s injected regardless
system slows → load drops         system slows → queue grows → collapse visible
↳ models a fixed worker pool      ↳ models real user traffic
```

The measurement discipline matters as much as the model:

- Discard the first N seconds (JVM warmup, connection pool fill, cache priming).
- Report **percentiles, not averages** — and remember client-side percentiles can't be averaged across
  load-generator instances (file 11).
- Watch for **coordinated omission**: if your generator waits for a slow response before sending the
  next request, it silently under-samples exactly the slow period. Open-model injection is the defence.
- Record the *system* metrics alongside the client ones — consumer lag, GC pauses, pool saturation.
  Client latency alone tells you there's a problem, not where.

**Follow-ups they'll ask:**
- *"Gatling vs k6?"* — Gatling has a Java DSL, which keeps the load test in the same repo and language;
  k6 is JavaScript and often nicer for ops-owned tests. StreamPulse uses Gatling for the Java DSL.
- *"What do you assert?"* — assertions in the load test itself (p99 under X, error rate under Y) so it
  can fail a pipeline, not just produce a pretty report nobody reads.
- *"How does this feed the rest of the work?"* — Phase 9's numbers set the Phase 11 Kubernetes resource
  requests and the BS-10 SLO targets. The load test is where the plan's *design targets* get replaced by
  measured values.

**Pitfalls / red flags:** quoting a throughput number with no mention of warmup, percentiles or workload
model.

**Where in StreamPulse:** Phase 9; BS-03 / REQ-KT-02; `tools/load-gen`.

**Verify:** Gatling's current Java DSL injection-profile API names (`constantUsersPerSec`,
`rampUsersPerSec` etc.) before quoting them.

---

## Q11. Is code coverage a useful metric?

**30-second answer:** Line coverage tells you which lines *executed*, not whether anything was
*asserted*. You can get 90% coverage with zero assertions. So I treat it as a one-directional signal:
low coverage reliably indicates a problem, high coverage proves nothing. If I want to know whether the
tests are any good, **mutation testing** answers that directly — it deliberately mutates the code
(flips a conditional, changes a boundary) and checks whether a test fails. A surviving mutant is a line
that is covered but not actually verified. It's slow, so I run it on the critical modules, not the
whole build.

**Deeper:**

```java
// 100% line coverage. Zero value.
@Test
void testCalculate() {
    calculator.calculate(input);      // no assertion at all — the line ran, nothing was checked
}
```

Where I'd *insist* on high coverage: the domain module, where the logic lives and tests are cheap.
Where I'd not chase it: adapters and generated code, where an integration test gives better evidence per
line than a unit test would.

**Follow-ups they'll ask:**
- *"What coverage target would you set?"* — I'd set it per-module and treat it as a ratchet (never
  decrease) rather than a number to hit. A global 80% gate encourages assertion-free tests on the
  easy code.
- *"Branch vs line coverage?"* — branch is more informative (it catches the untested `else`), still
  doesn't check assertions.

**Verify:** PIT (pitest) as the standard JVM mutation-testing tool and its current Gradle plugin
coordinates.

---

## Q12. A test fails once a week in CI. How do you handle it?

**30-second answer:** First, never re-run and move on — a flaky test is either a real race or a broken
test, and both are information you're throwing away. I quarantine it so it stops blocking everyone,
but with a ticket and an owner, because an un-owned quarantine is just deletion with extra steps. Then
I look at the usual causes in order: time and timezone assumptions, `Thread.sleep` instead of a real
condition, shared state between tests, test-order dependence, and genuine concurrency bugs in the
*production* code — which is the case you most want to find, because that one is a real defect that
happens to be visible in CI first.

**Deeper:**

| Cause | Signature | Fix |
|---|---|---|
| `Thread.sleep` for async | fails under CI load, passes locally | Awaitility with a real condition |
| Shared state | fails only in a particular order | isolate; run with random order to expose it |
| Time / timezone | fails at midnight, month end, or DST | inject a `Clock`, never call `Instant.now()` directly |
| Port collision | fails in parallel builds | `RANDOM_PORT`, container-assigned ports |
| Real race in production code | rare, non-deterministic, no test smell | the valuable one — see Q8 |

The `Clock` point deserves emphasis because it's cheap and permanent:

```java
// ❌ untestable and flaky at window boundaries
if (Instant.now().isAfter(windowEnd.plus(grace))) { ... }

// ✅ inject java.time.Clock; tests use Clock.fixed(...) and control time exactly
if (clock.instant().isAfter(windowEnd.plus(grace))) { ... }
```

**Follow-ups they'll ask:**
- *"How do you find order dependence?"* — run the suite in random order (JUnit 5 supports order
  configuration) and see if it reproduces.
- *"Is retrying a test ever acceptable?"* — for a genuinely external dependency in an E2E suite,
  grudgingly. Never in unit or integration tests, because there you own everything that could vary.

**Verify:** JUnit 5's test-order configuration property names.

---

## Q13. What does each of StreamPulse's named tests actually prove?

*(The applied question — this is where a test suite becomes an argument rather than a checkbox.)*

**30-second answer:** Each acceptance test in the plan exists because a specific failure mode exists,
and I'd describe them that way rather than as coverage. The pattern is consistent: a business scenario
states a failure, a requirement states the guarantee, and a named test is the evidence. The ones I'd
lead with are `RollingRestartChaosIT`, because "zero event loss across 20 restarts" is a claim with a
number behind it, and `ProtocolParityTest`, because it proves a second protocol didn't fork the
business logic — which is exactly the thing a reviewer would worry about.

| Test | Scenario | What it proves | What it would catch |
|---|---|---|---|
| `ShutdownOrderIT` | BS-01 / REQ-LC-01 | `SmartLifecycle` phases stop in the intended order | someone adds a bean with a wrong phase |
| `RollingRestartChaosIT` | BS-01 / REQ-LC-02 | 0 events lost over 20 restarts under load | a flush that commits offsets before the write is durable |
| `ConfigValidationTest` | BS-01 / REQ-LC-03 | missing config fails startup, not the first request | a new property added without validation |
| `TenantContextConcurrencyTest` | BS-01 / REQ-LC-04 | no cross-tenant bleed between concurrent requests | a tenant value cached on a singleton |
| `PlatformAutoConfigurationTest` | BS-01 / REQ-LC-07 | the starter's defaults apply *and* can be overridden | a missing `@ConditionalOnMissingBean` |
| `CommandCenterTimeoutTest` | BS-02 / REQ-CC-02 | one slow panel degrades, doesn't fail the page | a missing `orTimeout` on a new panel |
| `QuotaTransferDeadlockTest` | BS-02 / REQ-CC-07 | ordered locking survives 10k crossed transfers | reintroducing the naive lock order |
| `S3OffloadBackpressureIT` | BS-02 / REQ-CC-04 | consumer lag stays bounded when S3 is slow | swapping the bounded pool for an unbounded one |
| `SlowConsumerRebalanceIT` | BS-03 / REQ-KT-04 | `max.poll.records` × processing time < poll interval | a slower handler added without re-tuning |
| `CircuitBreakerStateTransitionTest` | BS-04 / REQ-RS-02 | CLOSED→OPEN→HALF_OPEN→CLOSED with real configs | a threshold change that makes the breaker never open |
| `S3OutageSpoolReplayIT` | BS-04 / REQ-RS-02 | a 3-minute outage loses nothing; the spool replays | a fallback that drops instead of spooling |
| `TenantBulkheadIsolationIT` | BS-04 / REQ-RS-05 | one tenant's outage doesn't delay another's alerts | shared thread pool creeping back in |
| `TenantTreeQueryCountTest` | BS-05 / REQ-DL-03 | ≤3 SQL statements regardless of tenant count | an N+1 reintroduced by a new lazy association |
| `AuditRequiresNewTest` | BS-05 / REQ-DL-04 | the audit row survives the rule rollback | the audit call becoming a self-invocation |
| `OutboxAtomicityIT` | BS-05 / REQ-DL-06 | rollback leaves neither entity nor outbox row | an outbox write moved outside the transaction |
| `PartnerRoutingIT` | BS-07 / REQ-MQ-02 | only matching queues receive an alert | a binding pattern typo |
| `DlqRetryIT` | BS-07 / REQ-MQ-04 | 3 retries → DLQ → replay works | a retry loop with no terminal state |
| `ShadowComparatorIT` | BS-08 / REQ-SF-03 | the extracted service matches the monolith exactly | a behaviour drift during extraction |
| `CutoverFlagIT` | BS-08 / REQ-SF-04 | flipping mid-stream produces no gaps or duplicates | a non-deterministic `alertId` |
| `ProtocolParityTest` | BS-09 / REQ-GR-02 | REST and gRPC produce identical Kafka records | logic duplicated into the gRPC adapter |
| `GrpcDeadlineCancellationTest` | BS-09 / REQ-GR-03 | the server actually stops work on cancellation | a handler ignoring `Context.isCancelled()` |

**Follow-ups they'll ask:**
- *"Which of these would you write first?"* — the ones encoding a *correctness* guarantee rather than
  an optimization: `OutboxAtomicityIT`, `AuditRequiresNewTest`, `ProtocolParityTest`,
  `RollingRestartChaosIT`. A performance regression is recoverable; silent data loss isn't.
- *"That's a lot of integration tests — is that not slow?"* — the `*IT` suffix separates them into their
  own source set and they share singleton containers. Unit tests run on every save; integration tests
  run on push.
- *"What's missing from this list?"* — property-based testing of the aggregation logic, and chaos
  testing beyond the pod-kill drill. I'd say so rather than pretend the suite is complete.

> **Your story:** if you introduced Testcontainers, contract tests, or a load-testing practice at
> Couchbase or JustAnswer, this is where it goes — what the team could not verify before, and what
> became provable afterwards. Keep "I built" for what you personally wrote.

**Where in StreamPulse:** all phases; CI runs unit + integration on every push, Playwright E2E nightly.

---

## Key terms

| Term | One-line meaning |
|---|---|
| **Test slice** | A Spring test that boots only one layer (`@WebMvcTest`, `@DataJpaTest`). |
| **Context cache** | Spring's reuse of booted contexts across tests with identical configuration. |
| **Fake** | A working lightweight implementation (in-memory store), as opposed to a mock. |
| **Change-detector test** | A test coupled to implementation detail; fails on refactor, catches no bugs. |
| **Singleton container** | One Testcontainers instance started per JVM and shared by the whole suite. |
| **Ryuk** | The Testcontainers sidecar that reaps containers when the creating JVM exits. |
| **`TopologyTestDriver`** | Kafka Streams test harness with no broker and full control over time. |
| **Awaitility** | Polls until a condition holds, replacing `Thread.sleep` for async assertions. |
| **Coordinated omission** | Under-sampling slow periods because the generator waits for responses. |
| **Open vs closed model** | Fixed arrival rate vs fixed virtual users — only open reproduces saturation. |
| **Mutation testing** | Mutates code to check whether any test fails; measures assertion quality. |
| **Consumer-driven contract** | The consumer publishes what it uses; the provider verifies it alone in CI. |

---

## Self-check (answer out loud, no notes)

1. Why did Testcontainers change the shape of the test pyramid, and what part of the shape did it not
   change?
2. Give the case where a fake is strictly better than a mock, and explain what the mock would hide.
3. Your Spring test suite takes 8 minutes and boots 30 contexts. What is the likely cause and the fix?
4. What does a passing concurrency test prove, and what three techniques raise that value?
5. Pick three StreamPulse tests and state, for each, the specific production failure it would catch.
