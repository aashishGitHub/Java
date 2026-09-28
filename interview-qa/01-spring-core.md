# 01 — Spring Core: IoC, Bean Lifecycle, Scopes, Profiles, Auto-configuration, AOP

**Why this matters for an Event Hub role:** juniors *use* Spring; seniors explain the machinery and
know when a feature is the wrong tool. Two questions separate the levels: *"what actually happens
between `new` and your bean being ready?"* and *"how does auto-configuration decide?"* In StreamPulse
this is not trivia — **BS-01 (zero-loss rolling deploys)** is solved entirely with lifecycle
callbacks, scopes and profiles.

**Scenario anchor — BS-01:** a pod killed mid-batch loses ~2k buffered events and leaves a half-written
S3 file; a missing Kafka property is only discovered at the first request. The fix is an ordered
shutdown (`SmartLifecycle`), fail-fast validated config, and profile-selected adapters.

---

## Q1. What is inversion of control, and why does Spring prefer constructor injection?

**30-second answer:** IoC means the framework owns object construction and wiring, not my code — I
declare dependencies, the container supplies them. I use constructor injection because it makes
dependencies explicit and mandatory, lets me mark fields `final` so the object is immutable and
thread-safe after construction, and means I can instantiate the class in a plain unit test with
`new` — no Spring context, no reflection. Field injection hides dependencies, allows a
half-constructed object, and forces either a container or reflection in tests.

**Deeper:**

Flow of control at startup:

```
ApplicationContext.refresh()
  └─ BeanFactoryPostProcessors run        (can modify BEAN DEFINITIONS — e.g. property placeholders)
  └─ for each singleton bean definition:
       ├─ resolve constructor args (recursively creating dependencies)
       ├─ instantiate
       ├─ populate @Autowired fields/setters
       ├─ *Aware callbacks
       ├─ BeanPostProcessor.postProcessBeforeInitialization   ← proxies/annotation scanning hook
       ├─ @PostConstruct
       ├─ InitializingBean.afterPropertiesSet()
       ├─ @Bean(initMethod=...)
       └─ BeanPostProcessor.postProcessAfterInitialization    ← AOP proxy is usually created HERE
```

The practical consequence of the last line: **your bean is not a proxy inside its own
`@PostConstruct`** — the proxy is wrapped around it afterwards. That is why `@Transactional` or
`@Async` work on a call from outside but not on a call you make to yourself (Q10).

A second consequence: **constructor injection makes an unresolvable circular dependency fail at
startup**, because A cannot be constructed without B and vice versa. Field/setter injection hides the
cycle by creating an incomplete object first. Since Spring Boot 2.6 circular references are
rejected by default and re-enabling them needs `spring.main.allow-circular-references=true` — treat
a cycle as a design smell, not a config flag.

**Code:**

```java
// StreamPulse ingest-gateway — dependencies are explicit, final, and testable with plain `new`.
@Service
public class IngestService {

    private final EventPublisher publisher;        // port: Kafka or Kinesis adapter (BS-09/Phase 10)
    private final IdempotencyStore idempotency;    // Redis-backed dedup
    private final IngestProperties props;          // validated config (Q6)

    // No @Autowired needed: a single constructor is autowired implicitly since Spring 4.3.
    public IngestService(EventPublisher publisher,
                         IdempotencyStore idempotency,
                         IngestProperties props) {
        this.publisher = publisher;
        this.idempotency = idempotency;
        this.props = props;
    }
}

// Unit test: no Spring context, no @SpringBootTest, ~5ms.
var service = new IngestService(new FakePublisher(), new InMemoryIdempotencyStore(), props);
```

**Follow-ups they'll ask:**
- *"When would you use setter injection?"* — genuinely optional collaborators, or a mutable
  dependency you need to swap after construction. Rare.
- *"How do you handle two beans of the same type?"* — `@Primary` for the default, `@Qualifier` at the
  injection point, or inject `List<T>`/`Map<String,T>` when you actually want all of them (I use the
  `Map<String, RuleEvaluator>` form for the Strategy pattern in the alert engine, BS-08).
- *"Does constructor injection hurt with many dependencies?"* — it doesn't hurt, it *reveals*. Seven
  constructor args is the class telling you it has too many responsibilities.

**Pitfalls / red flags:** saying "constructor injection is best practice" and stopping there. The
senior answer names the three concrete mechanisms: final fields, fail-fast cycles, no-container tests.

**Where in StreamPulse:** every service; enforced by an ArchUnit rule banning `@Autowired` on fields.

**Verify:** the `allow-circular-references` default changed in Spring Boot 2.6 — confirm the behaviour
in the Boot version you pin.

---

## Q2. Walk me through the full bean lifecycle, including shutdown.

**30-second answer:** Startup is: instantiate, inject, `*Aware` callbacks,
`BeanPostProcessor.postProcessBeforeInitialization`, `@PostConstruct`,
`InitializingBean.afterPropertiesSet`, custom init method, then
`postProcessAfterInitialization` — which is where AOP proxies get wrapped on. Shutdown is the mirror:
`@PreDestroy`, `DisposableBean.destroy`, custom destroy method. But for anything that needs *ordering*
across beans — stop accepting traffic before you stop the consumers, flush buffers before you close
the producer — `@PreDestroy` is the wrong tool, because its order between unrelated beans is not
something I want to rely on. That's what `SmartLifecycle` phases are for.

**Deeper:**

`BeanFactoryPostProcessor` vs `BeanPostProcessor` is a favourite follow-up:

| | Operates on | Runs | Typical use |
|---|---|---|---|
| `BeanFactoryPostProcessor` | bean **definitions** (metadata) | once, before any bean is instantiated | `PropertySourcesPlaceholderConfigurer` resolving `${...}` |
| `BeanPostProcessor` | bean **instances** | around every bean's initialization | `AutowiredAnnotationBeanPostProcessor`, AOP proxy creation, `@Async` |

`SmartLifecycle` adds a separate, explicitly ordered axis on top:

- `getPhase()` — **lower phases start first, and stop last**. So high phase = "outermost" = the
  traffic-facing component.
- `start()` / `stop(Runnable callback)` — the callback form is asynchronous: the container waits up to
  `timeoutPerShutdownPhase` for you to call it.
- `isAutoStartup()` — whether the container starts it automatically.

**Code:** BS-01's ordered drain (REQ-LC-01):

```java
/**
 * Shutdown order is the reverse of phase order, so we assign phases so that the
 * traffic-facing components (highest phase) stop FIRST:
 *   phase 300  HTTP/gRPC connector   → stop first  (refuse new work)
 *   phase 200  Kafka listeners       → then        (commit offsets, stop pulling)
 *   phase 100  S3 buffer flusher     → then        (drain what's in memory)
 *   phase   0  Kafka producer        → last        (nothing left to send)
 */
@Component
class S3BufferFlusher implements SmartLifecycle {

    private final S3BatchBuffer buffer;
    private final Duration flushTimeout;      // streampulse.shutdown.flush-timeout, default 20s
    private volatile boolean running;

    @Override public int getPhase()          { return 100; }
    @Override public boolean isRunning()     { return running; }
    @Override public void start()            { running = true; }

    @Override
    public void stop(Runnable callback) {
        running = false;
        try {
            // REQ-LC-02: flush within the timeout, or DON'T commit the offsets —
            // re-delivery (at-least-once) is correct; silent loss is not.
            buffer.flushAll(flushTimeout);
        } finally {
            callback.run();   // tell the container this phase is done
        }
    }

    @Override public void stop() { stop(() -> {}); }
}
```

Paired config:

```yaml
server:
  shutdown: graceful          # let in-flight HTTP requests finish
spring:
  lifecycle:
    timeout-per-shutdown-phase: 25s
```

…and in Kubernetes, `terminationGracePeriodSeconds` must exceed that total (BS-01 targets <30s of
drain inside a 45s grace period).

**Follow-ups they'll ask:**
- *"`@PostConstruct` vs `InitializingBean` vs `@Bean(initMethod)`?"* — same slot, different coupling.
  `@PostConstruct` (a JSR-250/Jakarta annotation) is the default; `InitializingBean` couples your class
  to Spring; `initMethod` is for third-party classes you can't annotate.
- *"Why not do the flush in `@PreDestroy`?"* — because I need *ordering relative to other beans*, and
  a timeout, and an async completion callback. `@PreDestroy` gives me none of those.
- *"What happens if the JVM is SIGKILLed?"* — nothing runs. That's exactly why the design is
  at-least-once with idempotent consumers, not "flush on shutdown and hope".

**Pitfalls / red flags:** reciting the lifecycle list but not knowing *where the proxy is created*, or
claiming `@PreDestroy` ordering is guaranteed across unrelated beans.

**Where in StreamPulse:** BS-01 / REQ-LC-01, REQ-LC-02; verified by `ShutdownOrderIT` (records the
callback order) and `RollingRestartChaosIT` (20 restarts under load, asserts 0 lost events).

**Verify:** the exact `SmartLifecycle` phase semantics and `timeout-per-shutdown-phase` key against
your Spring version's docs before quoting them.

---

## Q3. What bean scopes exist, and how does a request-scoped bean get injected into a singleton?

**30-second answer:** The core scopes are singleton (default, one per container) and prototype (a new
instance per lookup); web-aware contexts add request, session and application. The interesting part is
injecting a *shorter-lived* bean into a *longer-lived* one: you can't inject the instance, because at
singleton-construction time there is no request. Spring solves it with a **scoped proxy** — the
singleton gets a CGLIB proxy that, on every method call, looks up the real instance for the *current*
request from a thread-local holder. So the singleton holds one proxy forever, but each call resolves
to the right per-request object.

**Deeper:**

| Scope | Lifetime | Destruction callbacks run? |
|---|---|---|
| `singleton` | container | yes |
| `prototype` | per injection/lookup | **no** — the container hands it over and forgets it |
| `request` | one HTTP request | yes |
| `session` | HTTP session | yes |
| `application` | ServletContext | yes |

That prototype row matters: if a prototype bean holds a resource, *you* must close it — Spring will
not call `@PreDestroy`.

**Code:** REQ-LC-04, the per-request tenant context:

```java
@Component
@RequestScope   // shorthand for @Scope(value = "request", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class TenantContext {
    private String tenantId;      // populated by a filter from the JWT "tenant" claim
    public String tenantId() { return tenantId; }
    void set(String id) { this.tenantId = id; }
}

@Service
public class QuotaService {
    private final TenantContext tenant;   // a PROXY, injected once, at startup

    QuotaService(TenantContext tenant) { this.tenant = tenant; }

    public boolean allow(int events) {
        // Each call resolves the real request-scoped instance behind the proxy.
        return buckets.get(tenant.tenantId()).tryConsume(events);
    }
}
```

**Two traps worth naming out loud:**

1. **Virtual threads and `@Async` still work** with request scope (the request attributes are
   thread-bound via `RequestContextHolder`), but if you hand work to *another* thread — a
   `CompletableFuture` on a separate executor, as in BS-02 — the request scope is **not** propagated
   and the proxy throws. The fix is to capture the values you need *before* forking:

```java
String tenantId = tenant.tenantId();                 // capture on the request thread
CompletableFuture.supplyAsync(() -> load(tenantId), panelExecutor);   // pass the value, not the proxy
```

2. `proxyMode = TARGET_CLASS` uses CGLIB, so the class needs a non-private constructor and cannot be
   `final`.

**Follow-ups they'll ask:**
- *"Are singletons thread-safe?"* — Spring guarantees *one instance*, not thread safety. A singleton
  with mutable state is a bug; keep state in request scope, a `ThreadLocal`, or a concurrent structure.
- *"Alternative to a scoped proxy?"* — inject `ObjectProvider<TenantContext>` and call `getObject()`
  at use time. Same idea, explicit rather than magic.
- *"Why does a prototype in a singleton give you the same instance every time?"* — because injection
  happens once. You need a scoped proxy, `ObjectProvider`, or lookup-method injection.

**Pitfalls / red flags:** "singleton means one per JVM" (it's one *per ApplicationContext*), or not
knowing why the proxy is needed at all.

**Where in StreamPulse:** BS-01 / REQ-LC-04 (`TenantContextConcurrencyTest` fires two concurrent
requests and asserts no cross-tenant bleed), REQ-LC-05 (prototype `BatchAccumulator` per listener).

---

## Q4. How does Spring Boot auto-configuration actually work?

**30-second answer:** At startup `@EnableAutoConfiguration` reads a list of candidate configuration
classes from `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` on the
classpath — that's the modern replacement for the old `spring.factories` entry. Each candidate is
guarded by `@Conditional` annotations: `@ConditionalOnClass` ("is the driver on the classpath?"),
`@ConditionalOnMissingBean` ("did the app already define its own?"), `@ConditionalOnProperty`. The
conditions are evaluated and most candidates are discarded. The crucial rule is
`@ConditionalOnMissingBean`: **your bean always wins**, which is why Boot is opinionated but never
locks you in. Ordering between auto-configs is declared with `@AutoConfiguration(before=…, after=…)`.

**Deeper:**

Why does user config beat auto-config, when the file is loaded at startup? Because auto-configuration
classes are processed **last** — they are deferred imports (`DeferredImportSelector`), so by the time
their `@ConditionalOnMissingBean` is evaluated, your `@Configuration` beans are already registered.

Debugging it is a concrete senior answer: run with `--debug` (or `logging.level.org.springframework.boot.autoconfigure=DEBUG`)
and Boot prints the **condition evaluation report** — the positive matches, and, more usefully, the
negative matches with the reason each one was skipped. Or hit `/actuator/conditions` on a running app.

**Code:** your own starter (`libs/platform`, REQ-LC-07):

```java
// src/main/java/.../IdempotencyAutoConfiguration.java
@AutoConfiguration(after = RedisAutoConfiguration.class)     // we need Redis beans to exist first
@ConditionalOnClass(RedisTemplate.class)                     // only if Redis is on the classpath
@ConditionalOnProperty(prefix = "streampulse.idempotency",
                       name = "enabled", matchIfMissing = true)
@EnableConfigurationProperties(IdempotencyProperties.class)
public class IdempotencyAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean            // ← the app can replace this with its own implementation
    IdempotencyStore idempotencyStore(StringRedisTemplate redis, IdempotencyProperties props) {
        return new RedisIdempotencyStore(redis, props.ttl());
    }
}
```

```
# src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.streampulse.platform.idempotency.IdempotencyAutoConfiguration
```

Test it **both ways** without booting an app — this is the bit that impresses:

```java
class IdempotencyAutoConfigurationTest {

    private final ApplicationContextRunner runner = new ApplicationContextRunner()
            .withConfiguration(AutoConfigurations.of(IdempotencyAutoConfiguration.class));

    @Test
    void providesRedisStoreByDefault() {
        runner.withBean(StringRedisTemplate.class, () -> mock(StringRedisTemplate.class))
              .run(ctx -> assertThat(ctx).hasSingleBean(RedisIdempotencyStore.class));
    }

    @Test
    void backsOffWhenApplicationDefinesItsOwn() {          // proves @ConditionalOnMissingBean
        runner.withBean(IdempotencyStore.class, InMemoryIdempotencyStore::new)
              .run(ctx -> assertThat(ctx).hasSingleBean(IdempotencyStore.class)
                                         .doesNotHaveBean(RedisIdempotencyStore.class));
    }

    @Test
    void disabledByProperty() {
        runner.withPropertyValues("streampulse.idempotency.enabled=false")
              .run(ctx -> assertThat(ctx).doesNotHaveBean(IdempotencyStore.class));
    }
}
```

**Follow-ups they'll ask:**
- *"How do you exclude one?"* — `@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)`
  or `spring.autoconfigure.exclude`. Classic case: a service that has a JDBC driver on the classpath
  transitively but no database.
- *"Starter vs auto-configuration?"* — a *starter* is a dependency-only POM that pulls in libraries; the
  *auto-configuration* is the code with the conditions. They're usually shipped as separate artifacts.
- *"Why did my bean not get created?"* — condition report, `/actuator/conditions`.

**Pitfalls / red flags:** saying "Spring Boot scans the classpath and configures things by magic".
Name the imports file, the conditions and the deferred ordering.

**Where in StreamPulse:** `libs/platform` starter (tracing, ProblemDetail error model, idempotency),
REQ-LC-07. It's also the answer to "have you ever built a library other teams consume?"

**Verify:** the `AutoConfiguration.imports` path replaced `spring.factories` in Boot 2.7/3.x — confirm
for your pinned version. `@AutoConfiguration` is Boot 2.7+; earlier code uses `@Configuration` +
`@AutoConfigureAfter`.

---

## Q5. Profiles vs `@ConditionalOnProperty` — when do you use which?

**30-second answer:** A profile is an *environment* switch — "this whole set of beans belongs to local
development, or to AWS". A conditional property is a *feature* switch — "is this capability turned on,
independent of environment". My rule: if the answer to "which one?" depends on *where* the app runs,
it's a profile; if it depends on *what the app should do*, it's a property. And neither should ever
appear as an `if (env.equals("prod"))` inside business logic — that's the thing both mechanisms exist
to prevent.

**Deeper:**

StreamPulse (REQ-LC-06) uses profiles for adapter selection:

| Profile | What changes |
|---|---|
| `local` | moto/DynamoDB Local endpoints, cognito-free auth stub, verbose logging |
| `aws` | real AWS endpoints, IAM role credentials |
| `k8s` | Actuator probe groups, graceful shutdown timings |
| `datadog` | Micrometer Datadog registry instead of Prometheus |

…and properties for features: `streampulse.idempotency.enabled`, `alerts.engine=module|service`
(the BS-08 strangler-fig cutover flag — deliberately a *property*, not a profile, because you flip it
within the same environment).

Profile groups are the underrated feature — one activation implies several:

```yaml
spring:
  profiles:
    group:
      local: [ local-aws, local-auth, verbose-logging ]
```

**Code:**

```java
public interface EventPublisher {                       // port (hexagonal)
    CompletableFuture<Void> publish(TelemetryEvent e);
}

@Component
@Profile("!kinesis")                                     // default everywhere
class KafkaEventPublisher implements EventPublisher { /* ... */ }

@Component
@Profile("kinesis")                                      // Phase 10 comparison adapter
class KinesisEventPublisher implements EventPublisher { /* ... */ }
```

Config resolution order matters too, and is worth being able to sketch: command-line args >
`SPRING_APPLICATION_JSON` > OS environment variables > profile-specific
`application-{profile}.yml` > plain `application.yml` > `@PropertySource` > defaults. In Kubernetes
this is why a ConfigMap-as-env-vars entry silently overrides the value baked into your jar — usually
what you want, occasionally a two-hour debugging session.

**Follow-ups they'll ask:**
- *"How do you keep secrets out of config?"* — never in the repo; environment/Secrets Manager/Vault,
  injected as env vars or via Spring Cloud Config. In StreamPulse, `local` uses a `.env` file that is
  gitignored, and `aws` resolves from Secrets Manager.
- *"Can you activate a profile programmatically?"* — yes (`SpringApplication.setAdditionalProfiles`),
  but doing it conditionally at runtime is a smell.
- *"How do you test profile-specific wiring?"* — `@ActiveProfiles("aws")` on a slice test, or
  `ApplicationContextRunner.withPropertyValues(...)` with no context at all.

**Pitfalls / red flags:** a `prod` profile that changes business behaviour, not just infrastructure
wiring. Then your tests no longer test production.

**Where in StreamPulse:** BS-01 / REQ-LC-06; BS-08 / REQ-SF-04 (the cutover flag).

---

## Q6. How do you make configuration fail fast instead of at the first request?

**30-second answer:** Bind config to a typed `@ConfigurationProperties` record with Jakarta validation
annotations and `@Validated`. Binding happens at startup, so a missing or out-of-range value throws
before the app is ready — the pod never passes its readiness probe and the rolling update halts
instead of serving errors. Loose `@Value("${...}")` strings scattered across classes give you the
opposite: the failure surfaces on the first request that happens to touch that code path.

**Deeper:** this is REQ-LC-03 in BS-01, and the *why* is a deployment story, not a style preference:
a fail-fast pod is caught by the readiness gate and the rollout stops; a lazily-failing pod goes
`Ready`, takes traffic and 500s.

**Code:**

```java
@ConfigurationProperties(prefix = "streampulse.ingest")
@Validated
public record IngestProperties(
        @NotBlank String bootstrapServers,
        @NotBlank String rawTopic,
        @Min(1) @Max(1000) int maxBatchSize,           // REQ-INGEST: ≤500 events per request
        @NotNull @DurationMin(seconds = 1) Duration idempotencyTtl,
        @Min(1) int maxInFlightBatches                  // the BS-02 backpressure semaphore size
) {}
```

```java
@SpringBootApplication
@ConfigurationPropertiesScan          // finds @ConfigurationProperties records without @EnableConfigurationProperties
public class IngestGatewayApplication { }
```

Records are supported for constructor binding in Boot 3, which also gets you immutability for free.
Relaxed binding means `STREAMPULSE_INGEST_MAX_BATCH_SIZE=500` as an env var maps to `maxBatchSize` —
important in Kubernetes.

The test is one line and belongs in CI:

```java
@Test
void failsFastWhenTopicMissing() {
    new ApplicationContextRunner()
        .withUserConfiguration(IngestConfig.class)
        .withPropertyValues("streampulse.ingest.bootstrap-servers=localhost:9092")  // rawTopic absent
        .run(ctx -> assertThat(ctx).hasFailed()
                                   .getFailure()
                                   .hasMessageContaining("rawTopic"));
}
```

**Follow-ups they'll ask:**
- *"`@Value` vs `@ConfigurationProperties`?"* — `@Value` for a single one-off value; properties classes
  for anything grouped, validated, or documented. Properties classes also get metadata for IDE
  autocomplete via the annotation processor.
- *"How do you handle config that changes at runtime?"* — `@RefreshScope` with Spring Cloud Config, or
  — the StreamPulse choice — push changes as events on `config.changes.v1` via CDC (BS-08 / Phase 6),
  so the change is auditable and replayable rather than invisible.
- *"`@DurationMin` — where's that from?"* Spring Boot's validation constraints for `Duration`.

**Pitfalls / red flags:** `@Value("${x}")` with no default, spread across ten classes, discovered in
production.

**Where in StreamPulse:** BS-01 / REQ-LC-03; `ConfigValidationTest`.

**Verify:** `@DurationMin`/`@DurationMax` and the exact `@ConfigurationPropertiesScan` behaviour —
confirm in the Boot version you pin. Record-based constructor binding is Boot 3.x.

---

## Q7. What is a `BeanPostProcessor`, and when have you needed one?

**30-second answer:** It's a container-level hook that sees **every** bean instance around its
initialization, so it can inspect or replace it — it's how Spring itself implements `@Autowired`
processing, `@Async`, and AOP proxy creation. You reach for one when you need cross-cutting behaviour
applied by *type or annotation* rather than by pointcut expression, and you want it in a shared
library. I used one in the StreamPulse platform starter to wrap every `@TimedPort` bean in a Micrometer
timing proxy, so every outbound adapter is instrumented without each service remembering to do it.

**Deeper:**

```java
/**
 * REQ-LC-08. Any bean annotated @TimedPort gets a timing proxy, so every hexagonal
 * adapter (Kafka, S3, DynamoDB, partner HTTP) reports latency with no per-service code.
 * This also demonstrates the mechanism behind @Transactional: a proxy created in
 * postProcessAfterInitialization — which is why self-invocation bypasses it (Q10).
 */
@Component
class TimedPortBeanPostProcessor implements BeanPostProcessor {

    private final MeterRegistry registry;
    TimedPortBeanPostProcessor(MeterRegistry registry) { this.registry = registry; }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        Class<?> target = AopUtils.getTargetClass(bean);
        if (!target.isAnnotationPresent(TimedPort.class)) {
            return bean;                     // leave everything else untouched
        }
        // JDK dynamic proxy: requires the bean to implement an interface — true for our ports.
        return Proxy.newProxyInstance(
                target.getClassLoader(),
                target.getInterfaces(),
                (proxy, method, args) -> {
                    Timer.Sample sample = Timer.start(registry);
                    try {
                        return method.invoke(bean, args);
                    } finally {
                        sample.stop(registry.timer("port.call",
                                "port", target.getSimpleName(), "method", method.getName()));
                    }
                });
    }
}
```

Two cautions I'd state unprompted:
1. A `BeanPostProcessor` is instantiated **very early**, so it must not depend on beans that aren't
   ready yet — injecting a `MeterRegistry` directly can force premature initialization of other beans.
   The safe form is `ObjectProvider<MeterRegistry>` resolved lazily inside the method.
2. `InvocationTargetException` from `method.invoke` should be unwrapped, otherwise you corrupt the
   caller's exception handling. (The snippet above is simplified for readability — flag that.)

**Follow-ups they'll ask:**
- *"Why not just use AOP / `@Around`?"* — for most cases you should; AOP is the higher-level API. A
  `BeanPostProcessor` is for when you need the bean instance itself (replacing it, registering it
  somewhere, reading annotations across arbitrary types).
- *"JDK proxy vs CGLIB?"* — JDK proxies need an interface and proxy only interface methods; CGLIB
  subclasses the class, so the class and methods cannot be `final`, and it needs a usable constructor.
  Spring Boot defaults to `proxyTargetClass=true` (CGLIB) for `@Transactional`.
- *"Does the proxy break `instanceof` or annotations?"* — it can; use `AopUtils.getTargetClass()` and
  `AnnotationUtils`/`AnnotatedElementUtils` rather than reading annotations off the proxy class.

**Pitfalls / red flags:** proposing a `BeanPostProcessor` for something `@Around` advice solves.

**Where in StreamPulse:** BS-01 / REQ-LC-08, in `libs/platform`.

**Verify:** the default `proxyTargetClass` value for your Boot version, and unwrap
`InvocationTargetException` in real code.

---

## Q8. How does Spring AOP work, and what are its limits?

**30-second answer:** Spring AOP is **proxy-based**, not bytecode weaving: the container wraps your
bean in a JDK dynamic proxy or a CGLIB subclass, and the advice runs in the proxy before delegating to
the target. That's powerful and cheap, but it has three hard limits: it only intercepts calls that go
*through the proxy* (so no self-invocation), it only applies to Spring-managed beans, and with CGLIB it
can't advise `final` classes/methods or `private` methods. AspectJ with load-time or compile-time
weaving removes those limits at the cost of build/agent complexity — I'd only reach for it when the
proxy limitation genuinely blocks the design.

**Deeper:**

```
caller ──▶ [ Proxy ] ──▶ advice chain (@Around, @Before …) ──▶ target bean
              ▲
              └── everything outside the proxy is invisible to AOP
```

Advice types: `@Before`, `@AfterReturning`, `@AfterThrowing`, `@After`, `@Around` (the only one that
can prevent or replace the call). Multiple aspects on one join point are ordered by `@Order` /
`Ordered` — which is exactly the mechanism behind the Resilience4j decorator ordering question in
**BS-04 / REQ-RS-07**.

**Code** — the tenant-tagging aspect used across StreamPulse:

```java
@Aspect
@Component
@Order(10)                                   // runs outside the Resilience4j aspects
class TenantLoggingAspect {

    @Around("@annotation(com.streampulse.platform.TenantScoped)")
    public Object tagWithTenant(ProceedingJoinPoint pjp) throws Throwable {
        String tenant = TenantContextHolder.current();
        try (var ignored = MDC.putCloseable("tenantId", tenant)) {   // shows up in every log line
            return pjp.proceed();
        }
    }
}
```

**Follow-ups they'll ask:**
- *"Give me three things Spring implements with AOP."* — `@Transactional`, `@Async`, `@Cacheable`;
  also `@Retryable`, and the Resilience4j annotations.
- *"Performance cost?"* — a proxy hop and, for `@Around`, a reflective invoke. Negligible against I/O;
  measurable in a tight loop, which is one reason not to annotate hot in-memory methods.
- *"What's a pointcut expression you've actually written?"* — annotation-based pointcuts
  (`@annotation(...)`) are the maintainable kind; broad `execution(* com.acme..*(..))` expressions are
  how you accidentally advise everything.

**Pitfalls / red flags:** describing Spring AOP as bytecode weaving, or not knowing it's proxy-based
— because that's the setup for the next question.

**Where in StreamPulse:** the MDC tenant tag makes every log line in BS-04's multi-tenant outage story
filterable by tenant.

---

## Q9. `@Transactional` self-invocation — what happens and how do you fix it?

**30-second answer:** If a method inside a bean calls another method of the *same* bean, the call goes
directly on `this` — it never passes through the proxy — so `@Transactional`, `@Async`, `@Cacheable`
and `@Retryable` on the inner method are silently ignored. No error, no warning; it just doesn't
happen, which is the dangerous part. The fixes, in order of preference: move the inner method to a
separate bean (usually the right answer, because the need for a different transaction is telling you
it's a different responsibility); inject a self-reference; or use `TransactionTemplate` programmatically.

**Deeper:** this is REQ-DL-04 in **BS-05**, and it has a real consequence there: the audit record for a
rule change must survive the rollback of the rule change itself. That requires `REQUIRES_NEW` — a
genuinely *separate* transaction — so if the audit call is a self-invocation, the audit silently joins
the outer transaction and gets rolled back with it. You lose audit records and never find out.

**Code — the bug and the fix:**

```java
// ❌ BROKEN: audit() is called on `this`, so REQUIRES_NEW never happens.
@Service
class RuleService {
    @Transactional
    public void update(RuleUpdate cmd) {
        audit(cmd);            // direct call — no proxy, no new transaction
        validate(cmd);         // throws → rolls back the rule AND the audit row
        repo.save(cmd.toEntity());
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    void audit(RuleUpdate cmd) { auditRepo.save(AuditRecord.of(cmd)); }
}

// ✅ FIXED: separate bean, so the call crosses a proxy boundary.
@Service
class RuleService {
    private final AuditService audit;          // different bean → proxied call
    private final RuleRepository repo;

    @Transactional
    public void update(RuleUpdate cmd) {
        audit.record(cmd);     // commits in its OWN transaction, survives our rollback
        validate(cmd);
        repo.save(cmd.toEntity());
    }
}

@Service
class AuditService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void record(RuleUpdate cmd) { auditRepo.save(AuditRecord.of(cmd)); }
}
```

And the test that proves it (this is the kind of test that makes an interviewer nod):

```java
@Test
void auditSurvivesRuleRollback() {
    assertThatThrownBy(() -> ruleService.update(invalidRule()))
            .isInstanceOf(ValidationException.class);

    assertThat(ruleRepo.findAll()).isEmpty();          // rule rolled back
    assertThat(auditRepo.findAll()).hasSize(1);        // audit survived — REQUIRES_NEW worked
}
```

**Follow-ups they'll ask:**
- *"Why is the self-injection fix ugly?"* — it works (`@Lazy` self-reference, or
  `AopContext.currentProxy()` with `exposeProxy=true`), but it advertises that the class is doing two
  jobs. Extracting the bean fixes the design, not just the symptom.
- *"Does it apply to private methods?"* — yes, worse: a `@Transactional` private method can never be
  proxied at all with CGLIB.
- *"What else is silently skipped?"* — `@Async`, `@Cacheable`, `@Retryable`, the Resilience4j
  annotations. Same mechanism, same silence.
- *"Does `REQUIRES_NEW` use a second connection?"* — yes: the outer transaction is suspended and a new
  connection is taken from the pool. Two nested `REQUIRES_NEW` chains can exhaust a small pool and
  **self-deadlock** — a genuinely good thing to mention, and it ties to HikariCP sizing (file 06).

**Pitfalls / red flags:** knowing the rule ("self-invocation doesn't work") but not the *mechanism*
(proxy), and not being able to name a case where it caused real data loss.

**Where in StreamPulse:** BS-05 / REQ-DL-04; `SelfInvocationGotchaTest`, `AuditRequiresNewTest`.

---

## Q10. How would you structure a Spring Boot service so the business logic isn't coupled to Spring?

**30-second answer:** Hexagonal architecture — ports and adapters. The domain package has no Spring,
no Kafka, no JPA imports: just entities, value objects and use-case interfaces. Adapters on the
outside implement the ports: a REST controller and a gRPC service on the inbound side, Kafka/Redis/JPA
repositories on the outbound side. The payoff is concrete: the domain is unit-testable with `new` in
milliseconds, and when StreamPulse added a gRPC ingest path (BS-09) it reused the exact same
`IngestUseCase` — the second protocol was an adapter, not a rewrite. I enforce it with ArchUnit so it
doesn't erode.

**Deeper:**

```
inbound adapters          application core                 outbound adapters
┌──────────────────┐     ┌─────────────────────┐          ┌────────────────────┐
│ RestController   │────▶│ IngestUseCase       │─────────▶│ KafkaEventPublisher│
│ GrpcService      │────▶│  (interface)        │          │ KinesisPublisher   │
└──────────────────┘     │ IngestService (impl)│─────────▶│ RedisIdempotency   │
                         │ domain model        │          └────────────────────┘
                         └─────────────────────┘
                          no Spring/Kafka imports
```

Dependencies point **inwards**. The domain declares `EventPublisher` as an interface it owns; the
Kafka adapter depends on the domain, never the reverse. That inversion is what lets BS-09's gRPC
adapter and Phase 10's Kinesis adapter drop in without touching business code.

**Code** — the rule, as an executable test:

```java
@AnalyzeClasses(packages = "com.streampulse", importOptions = ImportOption.DoNotIncludeTests.class)
class ArchitectureTest {

    @ArchTest
    static final ArchRule domain_is_framework_free =
        noClasses().that().resideInAPackage("..domain..")
            .should().dependOnClassesThat()
            .resideInAnyPackage("org.springframework..", "org.apache.kafka..",
                                "jakarta.persistence..", "software.amazon.awssdk..")
            .because("the domain must be testable with plain `new` and survive a framework change");

    @ArchTest
    static final ArchRule no_field_injection =
        noFields().should().beAnnotatedWith(Autowired.class)
            .because("constructor injection only — see Q1");

    @ArchTest
    static final ArchRule adapters_do_not_call_each_other =
        noClasses().that().resideInAPackage("..adapter.in..")
            .should().dependOnClassesThat().resideInAPackage("..adapter.out..")
            .because("inbound adapters go through the use case, not around it");
}
```

**Follow-ups they'll ask:**
- *"Isn't that over-engineering for a CRUD service?"* — yes, often. I'd apply it where there is real
  domain logic or more than one adapter per port, and stay with a simple layered structure otherwise.
  Being able to say *when not to* is the senior part.
- *"Where do DTOs live?"* — in the adapters, with explicit mapping. Leaking a JPA entity to the REST
  layer couples your API contract to your schema.
- *"What breaks it in practice?"* — Jakarta validation annotations and Jackson annotations creeping
  into domain objects. My ArchUnit rule catches it in CI on the first commit.

**Pitfalls / red flags:** naming the pattern but having no enforcement, so the codebase drifts back to
a Spring-flavoured big ball of mud within a quarter.

**Where in StreamPulse:** every service; BS-09 / REQ-GR-02 (`ProtocolParityTest` — the same input via
REST and gRPC produces identical Kafka records); BS-08 / REQ-SF-01 (module boundaries enforced before
the service is extracted).

---

## Q11. `@PostConstruct` runs, but you need something to happen after the *whole* app is ready. What do you use?

**30-second answer:** `ApplicationReadyEvent` — it fires after the context is refreshed, all beans are
initialized and the web server is accepting connections. `@PostConstruct` is too early for anything
that touches other beans' readiness or the network. The related trap is the opposite direction: don't
start consuming Kafka in `@PostConstruct`, because you can begin processing before dependencies you
need are up.

**Deeper:** the useful events, in firing order:
`ApplicationStartingEvent` → `ApplicationEnvironmentPreparedEvent` → `ApplicationContextInitializedEvent`
→ `ApplicationPreparedEvent` → `ContextRefreshedEvent` → `ApplicationStartedEvent` →
`AvailabilityChangeEvent(LivenessState.CORRECT)` → `ApplicationReadyEvent` →
`AvailabilityChangeEvent(ReadinessState.ACCEPTING_TRAFFIC)`.

The last two are what Kubernetes probes read via `/actuator/health/liveness` and `/readiness` — the
BS-01 / REQ-LC-01 drain works by flipping readiness to `REFUSING_TRAFFIC` *before* the listeners stop.

**Code:**

```java
@Component
class TopologyWarmup {

    /** Warm the JIT and the DynamoDB connection pool before the LB sends real traffic. */
    @EventListener(ApplicationReadyEvent.class)
    void warmUp() { /* issue a few no-op queries */ }
}

// Signal REFUSING_TRAFFIC at the very start of shutdown (highest SmartLifecycle phase).
AvailabilityChangeEvent.publish(context, ReadinessState.REFUSING_TRAFFIC);
```

**Follow-ups they'll ask:**
- *"`CommandLineRunner`/`ApplicationRunner` vs `ApplicationReadyEvent`?"* — runners execute *before*
  `ApplicationReadyEvent` and, if they throw, they fail the application start. Good for one-off
  bootstrap work you want to be fatal; events are better for "app is live" side effects.
- *"How do you make an event listener asynchronous?"* — `@Async` on the listener plus `@EnableAsync`,
  with a named executor. Remember it then loses the caller's transaction and request scope.
- *"How does this interact with readiness probes?"* — directly; see above.

**Pitfalls / red flags:** doing network calls in `@PostConstruct` and then being surprised by startup
deadlocks or probe flapping.

**Where in StreamPulse:** BS-01 / REQ-LC-01; Phase 11 (K8s probes), Phase 9 (JIT warmup before load
tests — otherwise your first benchmark measures the interpreter).

---

## Q12. In StreamPulse, how do Spring core features actually prevent event loss on deploy?

*(This is the "tell me how you used it" question — the one that converts knowledge into evidence.)*

**30-second answer:** Four mechanisms, in sequence. First, validated `@ConfigurationProperties` mean a
misconfigured pod fails at startup and never passes readiness, so the rollout halts instead of serving
errors. Second, `SmartLifecycle` phases give a deterministic drain order: readiness goes to
`REFUSING_TRAFFIC`, then the HTTP and gRPC connectors stop, then Kafka listeners stop and commit, then
the S3 buffer flushes, then the producer closes. Third — and this is the important design decision —
if the flush can't finish inside the timeout, we **deliberately do not commit those Kafka offsets**, so
the events are re-delivered rather than lost; correctness comes from at-least-once plus idempotent
writes, not from hoping the flush succeeds. Fourth, profiles select adapters per environment, so the
same artifact runs locally and in AWS with no conditional code. I proved it with a chaos test: 20
rolling restarts under load, counting events in versus events stored.

**The failure it prevents, concretely:**

| Without | With |
|---|---|
| SIGTERM → JVM exits → ~2k in-memory events gone, S3 part-file truncated | Ordered drain → buffer flushed → offsets committed only after a durable write |
| Missing `bootstrap-servers` → pod Ready → 500s on first request | Startup fails → probe never passes → rollout stops |
| `if (env.equals("prod"))` in a service class | `@Profile`-selected adapter; prod and test run the same code path |

**Follow-ups they'll ask:**
- *"How do you verify 'zero loss'?"* — `RollingRestartChaosIT` compares the count of accepted events
  (ingest returned 2xx) against distinct `eventId`s landed in S3 plus DynamoDB. The dedup key makes
  re-delivery invisible, so re-delivered ≠ duplicated.
- *"What if the flush hangs forever?"* — `timeout-per-shutdown-phase` bounds it, then Kubernetes
  SIGKILLs at the grace period. The offsets weren't committed, so the next pod reprocesses.
- *"Does this cost you anything?"* — yes: slower deploys, and duplicate *processing* (not duplicate
  results) on every ungraceful shutdown. That's the trade I'd make every time.

> **Your story:** if you did graceful shutdown or drain work at Couchbase, JustAnswer or NetApp, put
> it here in the same shape — what was lost before, what mechanism fixed it, how you verified it. Keep
> "I built" for what you personally wrote; use "the team built" otherwise. StreamPulse is a portfolio
> project, so present it as one: *"I built this to prove the pattern end to end."*

**Where in StreamPulse:** BS-01 end to end; REQ-LC-01…08; `ShutdownOrderIT`, `RollingRestartChaosIT`.

---

## Key terms

| Term | One-line meaning |
|---|---|
| **IoC container** | The thing that constructs, wires, and manages the lifecycle of your beans. |
| **`ApplicationContext`** | The Spring container plus events, i18n, resource loading, environment. |
| **`BeanFactoryPostProcessor`** | Modifies bean *definitions* before any bean is created. |
| **`BeanPostProcessor`** | Wraps/inspects every bean *instance* around initialization; where proxies are made. |
| **`SmartLifecycle`** | Ordered start/stop with phases and an async stop callback — the graceful-shutdown tool. |
| **Scoped proxy** | A proxy injected into a longer-lived bean that resolves the shorter-lived instance per call. |
| **`@ConditionalOnMissingBean`** | "Only if the app didn't define its own" — why your bean always wins over auto-config. |
| **`AutoConfiguration.imports`** | The `META-INF/spring/...` file listing auto-configuration candidates (replaced `spring.factories`). |
| **`ApplicationContextRunner`** | Test utility to assert auto-configuration behaviour without booting an app. |
| **Self-invocation** | An internal `this.method()` call that bypasses the proxy, silently disabling `@Transactional`/`@Async`. |
| **JDK proxy vs CGLIB** | Interface-based proxy vs subclass-based proxy; CGLIB can't proxy `final`/`private`. |
| **Hexagonal / ports & adapters** | Domain owns interfaces; infrastructure implements them; dependencies point inward. |

---

## Self-check (answer out loud, no notes)

1. Where in the lifecycle is the AOP proxy created, and what does that explain?
2. Why can't `@PreDestroy` give you an ordered drain across beans?
3. A request-scoped bean injected into a singleton — what is actually injected, and what breaks if you
   pass it to another thread?
4. How does user configuration beat auto-configuration when auto-config is on the classpath?
5. Name the four mechanisms that make BS-01's rolling deploy lossless, in order.
