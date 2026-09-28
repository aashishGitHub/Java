# Java — Senior Engineer Action Items (Distributed-Systems SaaS)

**Target role:** Senior Software Engineer, distributed-systems SaaS company
**Your leverage:** 12y UI + 6y fullstack .NET + some k8s/Docker + AWS & system design in progress
**Tools available:** Claude AI, DesignGuru (Grokking), YouTube, Udemy

> Principle: The language/framework layer is your *easy 20%* (you already know OOP, DI, REST,
> ORMs, async). Spend real effort on JVM internals, concurrency, and distributed failure modes —
> that is where "wrote Java" becomes "senior Java."

> Java version note: Target **Java 21 LTS** (virtual threads GA here). If a later LTS (25) is what
> the target company runs, adjust — *verify the current LTS and the company's actual version.*

---

## How to read this list
Each item has:
- **[ ]** a checkable action
- **Why (senior):** why it earns senior signal
- **Proof:** the artifact or explanation that proves you actually own it (not just watched a video)

Do the workstreams roughly in order, but B/C/F are the differentiators — protect that time.

---

## Workstream 0 — Setup & baseline (Days 1–3)

- [ ] Install JDK 21 (Temurin/Adoptium), set up **IntelliJ IDEA** (Community is fine), Maven **and** Gradle hello-worlds.
  - **Why (senior):** you must be fluent in the build tools; seniors debug dependency/classpath hell.
  - **Proof:** build + run a multi-module Maven project and the same in Gradle.
- [ ] Set up a GitHub repo for a single **capstone service** you grow all the way through (see bottom).
- [ ] Configure Claude AI as your porting/review partner (prompts in the last section).

---

## Workstream A — Java language & idioms (Week 1, fast)

Leverage .NET; only drill the *differences*.

- [ ] Collections deeply: `ArrayList/LinkedList`, `HashMap` internals (buckets, resize, treeify), `TreeMap`, `LinkedHashMap`, `EnumMap`.
- [ ] Streams API vs LINQ: `map/filter/reduce/collect`, `groupingBy`, `flatMap`, when **not** to use streams (hot loops).
- [ ] `Optional` — correct use (return type, not fields/params).
- [ ] Generics differences from C#: **type erasure**, wildcards `? extends` / `? super` (PECS), bounded types.
- [ ] `equals()`/`hashCode()` contract; `record` types (Java's value objects); `sealed` classes; pattern matching for `switch`.
- [ ] Exceptions: checked vs unchecked, try-with-resources, why checked exceptions are controversial.
- [ ] Read **Effective Java (Joshua Bloch)** — items on generics, equals/hashCode, immutability, builders. *(Real, confidently recommended.)*
  - **Why (senior):** interviewers quote this book; it is the shared vocabulary of senior Java.
  - **Proof:** port a non-trivial C# class to idiomatic Java 21 and have Claude review it against "Effective Java" idioms.

---

## Workstream B — JVM internals & performance ⭐ (Week 2) — DIFFERENTIATOR

This is the #1 thing that separates senior from mid. .NET has GC too, but the questions differ.

- [ ] Memory model: heap vs stack, object layout, references, escape analysis (concept).
- [ ] Garbage collectors: generational model, **G1** (default), **ZGC**/Shenandoah (low-pause), when each is chosen.
- [ ] GC tuning flags: heap sizing (`-Xms/-Xmx`), what causes long pauses, allocation rate.
- [ ] Class loading basics; JIT compilation & warmup (why the first requests are slow).
- [ ] Memory leaks in a GC'd language: static collections, unclosed resources, `ThreadLocal` leaks, classloader leaks.
- [ ] Tooling: read a heap dump / flamegraph — **VisualVM**, **JFR (Java Flight Recorder)**, **async-profiler** (names to know).
  - **Why (senior):** you'll be asked "service latency spiked / OOM in prod — how do you diagnose?"
  - **Proof:** deliberately introduce a leak in your capstone, capture it with JFR/VisualVM, and explain the fix.
- [ ] YouTube deep-dives here (search current talks): "JVM garbage collection G1 vs ZGC", "Java Flight Recorder profiling". *(Pick recent, high-view talks; verify quality yourself.)*

---

## Workstream C — Concurrency & virtual threads ⭐ (Week 2–3) — DIFFERENTIATOR

Java's model differs from .NET `async/await`. Expect hard questions.

- [ ] Threads, `Runnable`/`Callable`, `ExecutorService`, `ThreadPoolExecutor` (core/max/queue/rejection).
- [ ] `synchronized`, `volatile`, the **Java Memory Model** (happens-before), `ReentrantLock`, `ReadWriteLock`.
- [ ] `java.util.concurrent`: `ConcurrentHashMap`, `BlockingQueue`, `CountDownLatch`, `Semaphore`, atomics.
- [ ] `CompletableFuture` — composition (`thenCompose`, `thenCombine`, exception handling). This is your `Task` analog.
- [ ] **Virtual threads (Project Loom, Java 21)** — what problem they solve, structured concurrency, pinning pitfalls, why they change the "thread-per-request" cost model.
- [ ] Deadlock / livelock / race conditions — cause, detect, prevent.
- [ ] Read **Java Concurrency in Practice (Brian Goetz)** — the canonical reference (pre-Loom but foundational). *(Real, confidently recommended.)*
  - **Why (senior):** concurrency bugs are the expensive ones in SaaS; seniors are trusted to get them right.
  - **Proof:** implement a bounded producer/consumer two ways (blocking queue vs virtual threads) and explain the trade-offs.

---

## Workstream D — Spring Boot in depth (Week 3–4)

You know ASP.NET Core — map, then go deeper than tutorials.

- [ ] Spring Core: IoC container, bean scopes, lifecycle (`@PostConstruct`), constructor injection (why over field).
- [ ] **Auto-configuration** — how it actually works (`@Conditional`, starters, `spring.factories`/`AutoConfiguration.imports`). Seniors get asked this.
- [ ] Spring MVC: `@RestController`, validation (`@Valid`), `@ControllerAdvice` global error handling, content negotiation.
- [ ] Configuration & profiles: `@ConfigurationProperties`, externalized config, secrets handling.
- [ ] Spring Security: filter chain (be able to draw it), JWT, OAuth2/OIDC, method security.
- [ ] AOP: proxies, `@Transactional` is AOP — understand the self-invocation gotcha.
- [ ] **Spring WebFlux / reactive** — know *when* it's worth it vs virtual threads (increasingly "don't, use Loom"); have an opinion.
  - **Why (senior):** juniors use Spring; seniors explain the machinery and choose when *not* to use a feature.
  - **Proof:** one well-reviewed Udemy Spring Boot course finished end-to-end (*verify ratings/recency before buying*), plus your capstone.
- [ ] Reference site: **Baeldung** (baeldung.com) and official **spring.io/guides** for anything specific. *(Real, confidently recommended.)*

---

## Workstream E — Data & persistence (Week 4)

- [ ] JPA/Hibernate: entity mapping, `@OneToMany`/`@ManyToOne`, fetch types.
- [ ] **N+1 problem** — reproduce it, fix with fetch joins / `@EntityGraph`. Classic senior screen.
- [ ] Transactions: `@Transactional` propagation & isolation levels; where boundaries belong.
- [ ] Connection pooling: **HikariCP** (defaults, sizing, why pool size ≠ "bigger is better").
- [ ] When to drop the ORM: raw SQL / jOOQ for hot paths; read vs write models.
- [ ] Flyway/Liquibase migrations (SaaS teams gate schema changes on these).
  - **Why (senior):** most SaaS latency/incidents trace to the data layer.
  - **Proof:** capture a slow query, run `EXPLAIN`, add an index, show before/after.

---

## Workstream F — Distributed systems & messaging ⭐ (Week 5–6) — CORE OF THE ROLE

This is *why it's a distributed-systems SaaS role*. Tie to your AWS + system-design study.

- [ ] Communication: REST vs **gRPC**, idempotency keys, API versioning, pagination at scale.
- [ ] **Kafka** (or the company's broker): topics/partitions/consumer groups, ordering guarantees, exactly-once vs at-least-once, consumer lag.
- [ ] Caching: **Redis**, cache-aside vs write-through, invalidation, `@Cacheable`, stampede protection.
- [ ] Consistency: eventual consistency, the **saga pattern** & outbox pattern for distributed transactions (there is no cross-service 2PC in practice).
- [ ] Resilience: **Resilience4j** — circuit breaker, retry (with backoff+jitter), bulkhead, timeout, rate limiter.
- [ ] Coordination concepts: leader election, distributed locks (and why they're dangerous), consistent hashing, quorum.
- [ ] Delivery/failure semantics: retries → duplicates → need for idempotency; poison messages; DLQs.
  - **Why (senior):** the interview loop *is* distributed failure modes; the job is keeping a multi-tenant system up.
  - **Proof (DesignGuru):** work **Grokking the System Design Interview** end-to-end; be able to design URL shortener, rate limiter, notification system, a multi-tenant SaaS metering service.

---

## Workstream G — Observability, resilience & production ops (Week 6, ongoing)

- [ ] **Spring Boot Actuator** + **Micrometer** → Prometheus metrics; the RED/USE method.
- [ ] Structured logging (SLF4J/Logback), correlation/trace IDs.
- [ ] **Distributed tracing**: OpenTelemetry / Zipkin — follow a request across services.
- [ ] Health checks, readiness vs liveness (ties to your k8s knowledge), graceful shutdown.
- [ ] Containerize: multi-stage Dockerfile for a small JRE image; deploy to k8s (you have a head start here).
- [ ] Feature flags, blue/green & canary, rollback thinking.
  - **Why (senior):** seniors own services in prod, not just merge PRs.
  - **Proof:** capstone exposes `/actuator`, ships metrics to Prometheus, and you can show a Grafana panel + a trace.

---

## Workstream H — Testing at the senior bar (throughout)

- [ ] JUnit 5 + **Mockito** (mock/spy, argument captors).
- [ ] Spring test slices: `@WebMvcTest`, `@DataJpaTest`, `@SpringBootTest`, `MockMvc`.
- [ ] **Testcontainers** — real Postgres/Kafka/Redis in integration tests. Strong senior signal; .NET folks rarely bring this.
- [ ] Contract tests (Spring Cloud Contract / Pact) for service boundaries.
- [ ] Some load testing (k6/Gatling) tied back to your JVM/perf work.
  - **Why (senior):** seniors are trusted because their code is *provably* correct under real dependencies.
  - **Proof:** capstone has unit + Testcontainers integration tests green in CI.

---

## Workstream I — Coding & system-design interview (Weeks 7–8, overlap earlier)

- [ ] LeetCode in **Java** — ~50–75 problems, patterns not grind (two pointers, sliding window, heaps, graphs, DP basics). Get fluent typing Java under time.
- [ ] DesignGuru **Grokking the Coding Interview** (patterns) to cut the volume.
- [ ] System design: 8–10 full designs out loud, whiteboard/diagram, always cover: requirements → estimates → API → data model → scaling → failure modes → trade-offs.
- [ ] Mock interviews with Claude (prompt it to be a skeptical senior interviewer and push back).
  - **Proof:** you can talk through a design for 40 min unprompted and defend trade-offs.

---

## Workstream J — Senior signal, behavioral & leadership (ongoing)

- [ ] Prep **STAR stories** from your 18y: an architecture decision, a production incident, a hard trade-off, mentoring, a disagreement resolved.
- [ ] Frame the transition as strength: "I've built scalable systems in .NET; I'm applying the same distributed principles in Java, and here's a service I built to prove it."
- [ ] Be able to explain **trade-offs**, not just facts — senior = judgment. For every tech in your capstone, know why you chose it and what you'd choose instead at 10x scale.
- [ ] Avoid the red flags (don't hand-wave concurrency, prod experience, or distributed consistency).

---

## The one capstone that ties it all together

Build **one** service and grow it across every workstream — it's your portfolio and your interview story:

> **Multi-tenant "usage metering / billing events" service** (very SaaS-relevant):
> - Spring Boot REST + gRPC ingest of usage events
> - Kafka pipeline, consumer with idempotency + DLQ
> - Postgres (Flyway migrations) + Redis cache
> - Resilience4j on all external calls
> - JWT auth, per-tenant isolation
> - Actuator + Micrometer + OpenTelemetry tracing
> - Testcontainers integration tests, CI pipeline
> - Dockerized, deployed to k8s
> - A written **design doc** (requirements → architecture → trade-offs → failure modes)

That design doc + running service answers 80% of a senior loop before they even ask.

---

## Using Claude AI as your accelerator (highest-leverage prompts)
- "Port this C# class to idiomatic Java 21; flag anything a senior reviewer would reject."
- "Act as a skeptical senior interviewer. Ask me about the Java memory model and push back on weak answers."
- "Review this Spring service for concurrency and transaction-boundary bugs."
- "Design-review my metering service: where does it break at 100x load? What are the failure modes?"
- "Quiz me on Kafka delivery semantics until I get exactly-once vs at-least-once right."

---

## Suggested pace (adjust to your hours/week)
| Weeks | Focus |
|------|-------|
| 1 | A (language) + start B |
| 2–3 | **B (JVM) + C (concurrency)** ← protect this |
| 3–4 | D (Spring) + E (data) |
| 5–6 | **F (distributed) + G (observability)** ← core of role |
| 7–8 | I (interview) + J (behavioral); H testing throughout |

> Honest note: for a senior distributed-systems role, ~8 weeks part-time makes you *interview-ready
> to tell a credible story*; true depth in JVM/concurrency/distributed is a longer game. Bias the
> capstone toward B/C/F and you'll punch above the timeline.
