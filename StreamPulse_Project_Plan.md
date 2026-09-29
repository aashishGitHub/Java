
# StreamPulse — Real-time Telemetry Event Hub in Java (Interview Portfolio Project)

> ## ▶ Picking this back up? Read `~/Documents/GitHub/streampulse/RESUME-HERE.md` first.
>
> **Last worked: 2026-09-29. Phases 0, 1 and 2 are complete and merged.**
>
> `main` is clean — one branch, no worktrees, `./gradlew clean build` green with
> **149 tests, 0 failures**, including Testcontainers integration tests against moto
> (Kinesis), Redis and Keycloak.
>
> The ingest gateway accepts batched telemetry over **REST and gRPC** through one use case
> (proven by a parity test that decodes the Avro off Kinesis and compares every field), behind
> Keycloak OIDC with real JWKS/issuer/audience validation, per-tenant Redis quotas, semaphore
> backpressure, atomic idempotency, `Retry( CircuitBreaker( TimeLimiter( publish ) ) )`, and an
> ordered graceful drain.
>
> **The next session opens with a decision, not code.** Phase 3 below specifies Kafka Streams,
> but the pivot to Kinesis (ADR-0008) leaves no direct equivalent. `RESUME-HERE.md` §3 lays out
> the four options against the one constraint that matters here — it has to run free and
> locally, with no AWS account.

## Context

**Why:** Aashish is preparing for backend-focused Senior Engineer rounds on a real-time telemetry "Event Hub" team (prep doc: "Disney Event Hub — Interview Prep"). The JD asks for Java 17+/Spring Boot, message-based distributed systems (Kafka/Kinesis), SQL + NoSQL (Postgres, DynamoDB), a React/TypeScript frontend, Terraform/Docker/K8s, Datadog/Grafana observability, and AI tooling (Claude Code, MCP, agentic workflows). Known interview content: BFS (Rotting Oranges), LRU Cache, system design of a real-time telemetry system and a Twitter-like feed, and Spring MVC questions.

**What exists already (so we don't repeat it):** Two Go projects: `ticket-booking` (Postgres CAS, outbox, saga, waiting room, WebSocket, cognito-local, EventBridge/SQS on moto, Terraform) and `offline-sync-app` (offline-first PWA, idempotent sync, LWW, outbox, WebSocket, MinIO, Terraform). The Java project should **reuse their proven local-AWS approach** and **add what they don't show**:
- high-throughput streaming: Kafka partitions, Kafka Streams windowing, exactly-once
- SSE with backpressure
- virtual threads vs WebFlux
- JVM profiling
- Spring internals
- Debezium CDC, which mirrors the Couchbase CDC→SSE story

**Outcome:** One runnable, multi-module Java 21 repo you can demo locally on a Mac (Docker). Every JD item and every likely interview question maps to a real file, test, dashboard or ADR. It is built phase by phase with **spec-driven development**.

**Decisions (confirmed):** full scope, phased · multi-module, ~5 services + UI · Gradle Kotlin DSL + Java 21 · new repo · free local stack (no paid LocalStack).

> Naming: generic "StreamPulse" with fictional tenants (`streamflix`, `sportsnow`, `kidsplus`). No real company branding in the repo.

---

## 1. Product scenario (the story you tell)

Video players on many apps/devices emit telemetry: `session_start`, `heartbeat`, `buffering_start/end`, `bitrate_change`, `error`, `session_end`. StreamPulse:
1. **Ingests** batched events at high throughput. The ingest is idempotent, rate-limited per tenant and applies backpressure.
2. **Processes** them in near-real time: concurrent viewers per title/region, rebuffer ratio, startup-time buckets, error rate, in 10s tumbling / 1m hopping windows.
3. **Serves** live dashboards over **SSE**, plus historical queries (hot: Redis/DynamoDB; cold: S3 NDJSON).
4. **Alerts** on rules, with **root-cause suppression** via a service-dependency graph (BFS).
5. **Propagates config changes** (tenants, rules) via **Debezium CDC** from Postgres.
6. Exposes an **MCP server** so Claude Code can ask "what's wrong with sportsnow right now?"
7. Includes an **activity feed** (the Twitter-like design, implemented as "friends are watching").

---

## 2. Architecture

### Services (Gradle multi-module)

| Module | Style | Owns | Key interview topics |
|---|---|---|---|
| `services/ingest-gateway` | Spring MVC + **virtual threads** | `POST /v1/events:batch` | Validation, idempotency keys, Kafka idempotent producer, partition key choice, 429 + Retry-After backpressure, Bucket4j/Redis rate limit, Resilience4j, OAuth2 resource server (JWT) |
| `services/stream-processor` | **Kafka Streams** | windowed aggregates | Topology, state stores, `exactly_once_v2`, grace/late events, suppression, repartitioning, `TopologyTestDriver` |
| `services/sink-writer` | **Spring Kafka** consumers | Redis, DynamoDB, S3, Postgres writes | Batch listeners, manual acks, `@RetryableTopic` + DLT, idempotent consumer, consumer lag, rebalancing |
| `services/query-api` | **Spring WebFlux** | REST + **SSE** + feed | Reactive vs blocking, backpressure (`onBackpressureLatest`/sample), heartbeats, `Last-Event-ID` resume, Caffeine caching, reactive Redis |
| `services/control-plane` | Spring MVC + **JPA/Postgres** | tenants, titles, alert rules, alert engine | Flyway, N+1 demo/fix, `@Transactional` propagation/self-invocation gotcha, HikariCP sizing, outbox → **Debezium CDC**, BFS suppression, topo-sort cycle detection |
| `services/mcp-server` (small) | Spring AI MCP server | read-only tools over query-api | AI tooling/agentic story (verify Spring AI MCP starter name/version in current docs) |
| `web/` | React 19 + TS + Vite | live dashboard + admin | SSE hook with reconnect/backoff, generated TS client from OpenAPI, Vitest + Playwright |
| `libs/contracts` | Avro + OpenAPI codegen | shared schemas | Schema evolution, compatibility modes |
| `libs/platform` | shared starter (your own **auto-configuration**) | tracing, error model (RFC 7807 ProblemDetail), idempotency, JSON, test fixtures | Shows *how* Spring auto-config works (`@AutoConfiguration`, `@Conditional*`, `AutoConfiguration.imports`) |
| `tools/load-gen` | Gatling (Java DSL) | steady + "live match spike" | Perf/JVM phase |

### Data flow (happy path)
```
Player SDK/load-gen ──HTTP batch──▶ ingest-gateway ──(key=sessionId)──▶ Kafka: telemetry.raw.v1 (12 partitions, Avro)
                                                                     │
                     ┌───────────────────────────────────────────────┼──────────────────────────┐
                     ▼                                               ▼                          ▼
           stream-processor (Kafka Streams, EOS v2)        sink-writer (raw → S3 NDJSON,   control-plane alert engine
           ─▶ telemetry.agg.v1 (per tenant/title/window)    dt=/hour= partitions)           (consumes agg + rules)
                     │                                                                          │
                     ▼                                                                          ▼
           sink-writer ─▶ Redis (hot, TTL) + DynamoDB (single-table history)          alerts.v1 ─▶ SQS (moto) + SSE
                     │
                     ▼
           query-api (WebFlux) ──SSE──▶ web dashboard / MCP server ──▶ Claude Code

control-plane Postgres ── outbox table ── Debezium (Kafka Connect) ──▶ config.changes.v1 ──▶ all services refresh caches
```

### Patterns (to name and justify in the interview)
- **Hexagonal (ports & adapters)** in every service: domain has no Spring/Kafka imports, so it's unit-testable.
- **Strategy:** `EventPublisher` (Kafka vs Kinesis adapter), `RuleEvaluator` per rule type.
- **Decorator:** caching around repositories.
- **CQRS:** control-plane (write model, Postgres) vs query-api (read models in Redis/DynamoDB).
- **Transactional outbox + CDC**, idempotent consumer, DLQ/retry topics, circuit breaker/bulkhead/rate limiter.
- **SOLID/DRY/KISS** enforced via the shared `platform` starter and ArchUnit tests (e.g. "domain must not depend on adapters").

### Local stack (Docker Compose, all free, Mac-friendly)
Compose **profiles**, so you only run what the current phase needs (Docker/OrbStack/Colima with ~10–12 GB RAM recommended for `full`; this is an estimate to verify on your machine):

| Profile | Containers |
|---|---|
| `core` | Kafka (Apache Kafka, KRaft, single broker), Schema Registry, Postgres 16, Redis 7, Keycloak (OIDC/JWT: the Java-ecosystem standard, and a new skill vs cognito-local) |
| `aws` | **moto server** (S3, SQS, Kinesis), **DynamoDB Local** (official AWS image) |
| `cdc` | Kafka Connect + Debezium Postgres connector |
| `obs` | OpenTelemetry Collector, Prometheus, Grafana, Tempo, Loki (or the all-in-one `grafana/otel-lgtm` image; verify it's still maintained) |
| `ui` | Kafka UI (e.g. provectus/kafbat kafka-ui; verify current image), web dev server |
| `chaos` | Toxiproxy (in front of Kafka/Redis/Postgres) |

Things to verify at build time (I am not certain of their current status):
- MinIO community image/licensing changed in 2025. **Use moto S3 instead** to avoid that dependency.
- Reactor Kafka's maintenance status is unclear. **Plan: Spring Kafka listener → Reactor `Sinks.Many` for SSE fan-out**, no reactor-kafka.
- KCL (Kinesis Client Library) against moto may be flaky. The Kinesis path will be producer + simple SDK v2 `GetRecords` consumer, documented as a "comparison adapter", not the main path.
- Confluent Schema Registry image license (Community License, fine locally). Apicurio is the Apache-2 fallback.

---

## 3. Spec-driven development workflow (applies to every phase)

Repo layout for specs:
```
specs/
  00-foundation/        requirements.md · design.md · tasks.md
  01-contracts/ ...     (one folder per phase)
api/openapi/*.yaml      # REST contracts (source of truth → codegen)
api/asyncapi/*.yaml     # Kafka topic contracts (channels, keys, schemas, retention)
schemas/avro/*.avsc     # event schemas, BACKWARD compatibility enforced
docs/adr/NNN-*.md       # architecture decision records
docs/design-doc.md      # the living system design doc (interview centerpiece)
```
Per-phase loop (the Definition of Done is identical each time):
1. **requirements.md**: user stories plus acceptance criteria in EARS/Given-When-Then form, plus NFRs (throughput, p99 latency, durability).
2. **design.md**: flow of control, data flow, sequence diagram (Mermaid), failure modes, test plan, links to ADRs.
3. **Contracts first**: OpenAPI/AsyncAPI/Avro updated. Codegen produces interfaces/DTOs (openapi-generator `interfaceOnly`, Avro Gradle plugin); the build fails if the implementation drifts.
4. **tasks.md**: small checklist, each task = test first, then code.
5. **Tests, in this order:** acceptance tests from criteria (fail) → unit → slice (`@WebMvcTest`/`@DataJpaTest`/`@WebFluxTest`) → Testcontainers integration → contract tests (Spring Cloud Contract or Pact between query-api ↔ web).
6. **Demo script + ADR**: record verified command output in `MILESTONES.md` (same honesty convention as ticket-booking).
7. Claude Code support: repo `CLAUDE.md` and custom slash commands (`/spec-new`, `/spec-tasks`, `/spec-verify`) that drive this loop. This doubles as the "agentic workflow" story.

---

## 4. Phases

Each phase is independently demoable. The **Interview proof** line is what you'll point to.

### Phase 0 — Foundation
- New repo `~/Documents/GitHub/streampulse` (confirm path at start). Gradle Kotlin DSL, version catalog (`libs.versions.toml`), convention plugins in `build-logic/` (java-conventions, spring-service, test-conventions with a separate `integrationTest` source set).
- Java 21 toolchain, Spotless, ErrorProne (optional), JaCoCo, ArchUnit.
- `docker-compose.yml` with profiles, `Makefile` (`up-core`, `up-full`, `test`, `it`, `demo-*`).
- GitHub Actions: build, unit, integration tests (Testcontainers on the GH runner).
- Spec templates, ADR-001 (multi-module monolith-repo), ADR-002 (Gradle), ADR-003 (local AWS strategy).
- **Domain primitives:** `TenantId` and `EventId` in `libs/platform` — validated value objects used by every later phase. Teaches records, compact constructors, boundary validation and `equals`/`hashCode` correctness on material the project actually uses.

> **Scope note (decided 2026-09-28):** algorithm/DSA practice is **out of scope** for StreamPulse. This project is for learning the Java ecosystem in a cloud environment. Coding-round preparation lives in `interview-qa/17-coding-lru-and-graphs.md`.
- **Interview proof:** clean build, CI green, "why multi-module" ADR.

### Phase 1 — Contracts & schemas
- Avro: `TelemetryEvent` (envelope: eventId, tenantId, sessionId, deviceType, titleId, region, ts, type, payload union), `MetricAggregate`, `AlertEvent`, `ConfigChange`.
- AsyncAPI for topics `telemetry.raw.v1`, `telemetry.agg.v1`, `alerts.v1`, `config.changes.v1`, `*.dlt`. Document partitions, keys, retention and compaction.
- OpenAPI for ingest, query, control-plane. Generate the TS client for `web/`.
- Schema Registry compatibility = BACKWARD. A test proves an incompatible change fails the build.
- **Interview proof:** schema evolution, why sessionId as key (ordering per session vs hot partitions), topic naming/versioning.

### Phase 2 — Ingest gateway
- Spring MVC with `spring.threads.virtual.enabled=true` (verify property in the Spring Boot version you pin). `POST /v1/events:batch` (≤500 events), Bean Validation, ProblemDetail errors.
- Idempotency: `Idempotency-Key` header and per-event `eventId` dedup in Redis (`SET NX EX`).
- Kafka producer: `enable.idempotence=true`, `acks=all`, linger/batch tuning, compression. Async send with `CompletableFuture`, and a bounded in-flight `Semaphore` that returns **429 + Retry-After** when saturated (backpressure).
- Per-tenant rate limiting (Bucket4j + Redis). Resilience4j circuit breaker + timeout around the producer. OAuth2 resource server with Keycloak JWT, tenant claim → authorization.
- `EventPublisher` port with Kafka adapter (Kinesis adapter arrives in Phase 10).
- **Tests:** `@WebMvcTest`, Testcontainers Kafka+Redis, a concurrency test that fires duplicate batches in parallel and asserts exactly one publish.
- **Interview proof:** Spring MVC deep questions, virtual threads (and pinning pitfalls with `synchronized`), producer delivery semantics.

### Phase 3 — Stream processing (hot path)
- Kafka Streams topology: filter/branch invalid events → DLT. Concurrent viewers per (tenant, title, region) via session state. Rebuffer ratio, startup-time histogram buckets, error rate. 10s tumbling + 1m hopping windows, grace period, `suppress(untilWindowCloses)`.
- `processing.guarantee=exactly_once_v2`, RocksDB state stores, standby replicas (documented), interactive query endpoint (optional).
- **Tests:** `TopologyTestDriver` (out-of-order events, late events, duplicates), plus a Testcontainers end-to-end run.
- **Interview proof:** at-least-once vs exactly-once (what EOS covers and what it does not), windowing, repartition cost, scaling via partitions.

### Phase 4 — Sinks & storage (hot + cold)
- `sink-writer` with Spring Kafka batch listeners and manual ack.
  - Aggregates → Redis (latest window, TTL) and **DynamoDB** single-table (`PK=TENANT#t#METRIC#m`, `SK=WINDOW#ts`, GSI for title). Conditional writes for idempotency.
  - Raw → S3 (moto) gzip NDJSON batched by size/time, `tenant=/dt=/hour=` layout (Parquet as a stretch goal).
- `@RetryableTopic` (exponential backoff) → DLT. Poison-message test. Consumer lag metric. Cooperative-sticky rebalancing.
- ADR: Postgres vs DynamoDB vs Mongo for time-series aggregates (access patterns first).
- **Interview proof:** DLQ design, idempotent consumers, NoSQL modeling from access patterns, hot vs cold path.

### Phase 5 — Query API + real-time SSE
- WebFlux: `GET /v1/metrics/{tenant}/{metric}?from&to` (Redis first, then DynamoDB) and `GET /v1/stream/metrics` as **SSE**. Kafka listener → `Sinks.Many` multicast per tenant, `onBackpressureLatest`, 15s heartbeat, `Last-Event-ID` resume from a Redis ring buffer.
- A thread-safe title-metadata cache invalidated by `config.changes.v1`. ADR compares a hand-rolled LRU against **Caffeine** — the interesting content is the eviction-policy and sizing trade-off, not the data structure.
- **Benchmark ADR:** the same endpoint on MVC + virtual threads vs WebFlux, under Gatling, with the numbers recorded.
- **Tests:** `@WebFluxTest`, `StepVerifier`, SSE integration test, contract test with `web/`.
- **Interview proof:** "WebFlux vs MVC, and when each fits" answered with your own data, plus cache eviction policy and sizing as a design decision.

### Phase 6 — Control plane, CDC & alert engine
- JPA entities: Tenant, App, Title, AlertRule, ServiceNode/Dependency. Flyway migrations. HikariCP tuning notes.
- **Deliberate N+1 lab:** reproduce it (Hibernate statistics/SQL log), fix it with fetch join/`@EntityGraph`, and add a test asserting the query count.
- `@Transactional` propagation and self-invocation demo test.
- Outbox table written in the same transaction → **Debezium** connector (outbox event router) → `config.changes.v1`. Services consume it to refresh caches. This is the Couchbase CDC→SSE story, rebuilt in Java.
- Alert engine: consumes `telemetry.agg.v1`, `RuleEvaluator` strategies (threshold, rate-of-change, absence). Dependency graph (CDN → packager → player) where **BFS suppresses downstream alerts** when an upstream root cause fires. **Topo sort** rejects cyclic rule dependencies at save time. Output goes to `alerts.v1` → SQS (moto) + SSE.
- **Interview proof:** JPA depth, CDC vs dual-write, BFS/topo-sort used for real.

### Phase 7 — Web dashboard
- React 19 + TS + Vite: `useEventSource` hook (reconnect with jittered backoff, Last-Event-ID), live charts (throttled rendering), alert feed, tenant switcher (Keycloak login via OIDC PKCE), admin CRUD for rules via the generated client.
- Vitest + Testing Library. Playwright e2e against the compose stack.
- **Interview proof:** your frontend-architect strength connected to the backend real-time path.

### Phase 8 — Observability ("system health" as the JD asks)
- Micrometer → Prometheus. OTel Java agent → Collector → Tempo (trace from HTTP ingest through Kafka headers to the consumer). JSON logs with traceId → Loki.
- Grafana dashboards-as-code: RED per service, Kafka consumer lag, end-to-end freshness (event ts → SSE delivery), JVM (GC pauses, heap, virtual threads), DLT rate.
- `docs/system-health.md`: SLIs/SLOs (e.g. p99 ingest latency, freshness < N s, DLT rate). Prometheus alert rules. Optional `datadog` Spring profile using the Micrometer Datadog registry (requires your own Datadog trial; verify).
- **Interview proof:** "what metrics define health for this system?" answered with a real dashboard.

### Phase 9 — Resilience, performance & JVM
- Gatling scenarios: steady state, plus a "live match kickoff" 10× spike. Record throughput/latency in `docs/perf-report.md`.
- Toxiproxy chaos: Kafka latency, Redis down, Postgres slow. Verify circuit breaker/bulkhead/fallback behavior and graceful degradation.
- JVM lab: JFR recordings + async-profiler flame graph. G1 vs ZGC under the spike. Induced memory leak (static map / ThreadLocal) found via heap dump, then fixed. Virtual-thread pinning detection (`-Djdk.tracePinnedThreads`; verify flag for your JDK).
- **Interview proof:** "latency spiked / OOM in prod, what do you do?" answered with artifacts.

### Phase 10 — AWS simulation & Terraform
- Terraform modules: `kinesis`, `dynamodb`, `s3`, `sqs`, `iam`. `apply` runs against moto/DynamoDB Local via provider endpoint overrides (same technique as your Go repos). `msk`, `vpc`, `eks` are `validate`/`plan`-only, with notes.
- **Kinesis adapter** for `EventPublisher` (Strategy, selected by Spring profile) + a simple consumer. ADR: Kafka vs Kinesis (shards vs partitions, retention, ordering, cost model).
- S3 compaction function with Spring Cloud Function (Lambda-shaped handler, run locally).
- Doc: multi-region / cross-region replication design (MSK replication / MirrorMaker 2, DynamoDB global tables); design only, not simulated.
- **Interview proof:** AWS + IaC fluency, and trade-offs between managed streaming options.

### Phase 11 — Containers & Kubernetes
- Small images with Jib (or Buildpacks), JRE-only, non-root. `kind` cluster + Helm charts per service.
- Liveness/readiness via Actuator health groups, graceful shutdown, PodDisruptionBudget, resource requests tuned from Phase 9 data.
- **KEDA** Kafka-lag autoscaling for `sink-writer` (verify KEDA Kafka scaler config).
- **Interview proof:** K8s ops for stateful stream apps (Kafka Streams state + rebalances on scale).

### Phase 12 — Activity feed (the Twitter-like design, implemented)
- "Friends are watching" feed: follow graph in Postgres, watch events from Kafka. **Fan-out-on-write** to Redis sorted sets for normal users; **fan-out-on-read** for high-follower accounts (the hybrid approach). Simple ranking (recency × affinity). Cursor pagination.
- `docs/feed-design.md` with the full system-design write-up and scale numbers.
- **Interview proof:** the Twitter-feed design question, with code backing it.

### Phase 13 — AI tooling
- `mcp-server` (Spring AI MCP server) with read-only tools: `get_live_metrics`, `list_active_alerts`, `explain_alert` (pulls the rule, recent aggregates and the dependency-graph root cause). Registered in Claude Code via `.mcp.json`.
- Repo-level Claude Code setup: `CLAUDE.md`, spec slash commands, a hook that runs `./gradlew spotlessCheck test` before commit (optional).
- Optional: an LLM incident summarizer using the Claude API (model id from the current docs).
- **Interview proof:** the concrete "what I built with AI tooling" story the prep doc asks for.

### Phase 14 — Interview packaging
- `docs/design-doc.md` finalized: requirements → estimates → API → data model → architecture → failure modes → trade-offs → "at 10×/100×".
- ADR index, runbook (on-call scenarios: lag spike, DLT growth, Redis outage), 5-min and 20-min demo scripts.
- `docs/interview-map.md`: each likely question → file/test/dashboard. STAR stories cross-linked (ownership = CDC pipeline, optimization = perf report, etc.).

---

## 5. JD → phase coverage matrix

| JD / prep-doc item | Phase(s) |
|---|---|
| Java 17+/Spring Boot core, DI, auto-config | 0, 2, `libs/platform` |
| Spring MVC (asked directly) | 2, 6 |
| WebFlux vs MVC, backpressure | 5 (benchmark ADR) |
| Concurrency (`CompletableFuture`, executors, virtual threads) | 2, 5, 9 |
| Kafka delivery semantics, idempotent consumers, DLQ | 2, 3, 4 |
| Kinesis | 10 |
| Resilience4j | 2, 9 |
| Postgres/JPA/transactions/pooling | 6 |
| DynamoDB/NoSQL trade-offs | 4 |
| React/TypeScript frontend | 7 |
| Terraform/Docker/K8s | 0, 10, 11 |
| Datadog/Grafana, tracing, "system health" metrics | 8 |
| AI tooling (Claude Code, MCP, agentic) | §3 spec workflow, 13 |
| RabbitMQ, gRPC, strangler-fig, SLA error budgets | see §5a |
| System design: telemetry pipeline | whole project + 14 |
| System design: Twitter-like feed | 12 |
| JUnit/Mockito/Testcontainers | every phase |

---

## 5a. Gap closures (added after audit against the prep doc's "Java/Spring deep dive" + resume bullets)

| Gap | Where it lands | What gets built |
|---|---|---|
| Spring core: bean lifecycle, scopes, profiles | Phase 0 + `libs/platform` | `spring-internals-lab` test module: `@PostConstruct`/`@PreDestroy`, `SmartLifecycle` (used for real to start/stop Kafka listeners in order on graceful shutdown), a custom `BeanPostProcessor`, request vs prototype vs singleton scope tests, `@Profile` + `application-{local,aws,k8s,datadog}.yml`, `@ConfigurationProperties` with validation. Includes a doc explaining constructor vs field injection. |
| Concurrency primitives | Phase 2, 5, 9 | Explicit `ThreadPoolExecutor` (core/max/queue/rejection policy) for S3 upload offload, compared with virtual-thread executor. `ReentrantReadWriteLock` in the title-metadata cache. `AtomicLong`/`LongAdder` counters. `CountDownLatch`/`CyclicBarrier` in concurrency tests. `CompletableFuture` fan-out (`allOf`, `thenCompose`, `exceptionally`, `orTimeout`) in query-api for aggregating Redis+DynamoDB. Plus a bounded producer/consumer implemented two ways, and a deliberate deadlock detected via thread dump. |
| Kafka tuning with numbers | Phase 9 | `docs/kafka-tuning.md` experiment matrix: producer `linger.ms`/`batch.size`/`compression.type` and consumer `fetch.min.bytes`/`max.poll.records`/listener concurrency, measured as throughput, p99 and bytes on disk/network. Gives a real "direction of the number" story in Java. |
| Resilience4j: all four, states and configs | Phase 2 + 9 | Replace Bucket4j with **Resilience4j RateLimiter** (keep Bucket4j+Redis only for distributed per-tenant quotas, and explain why: R4j is per-instance). Add **Retry** with exponential backoff + jitter on DynamoDB/S3 calls. Add **Bulkhead** (semaphore vs thread-pool). Tests drive CB through CLOSED→OPEN→HALF_OPEN→CLOSED and assert events/metrics. `docs/resilience.md` maps each pattern to the failure mode it guards against. |
| JPA vs raw JDBC; Mongo/MySQL | Phase 6 | Same query implemented with JPA and with `JdbcClient` (verify API exists in your Spring version; it's in Spring 6.1+ as far as I know), benchmarked. Hot read path uses JDBC. ADR covers Postgres vs MySQL (MVCC, JSONB) and Mongo vs DynamoDB. Optional `mongo` profile storing alert history in MongoDB to show Spring Data portability. |
| AWS: Lambda, VPC, cross-region | Phase 10 | Lambda: Spring Cloud Function handler deployed to **moto's Lambda** (needs Docker-in-Docker; not certain it works on Mac, fallback is the AWS SAM CLI `sam local invoke`). VPC: Terraform module with subnets/SGs/VPC endpoints, applied against moto (API-level only; real network isolation can't be simulated locally). Cross-region: **two local Kafka clusters + MirrorMaker 2** (region-a → region-b), with a failover drill documented. DynamoDB global tables stay design-only. |
| RabbitMQ (JustAnswer story) | Phase 6 | Alert notifications published to **RabbitMQ** via Spring AMQP (topic exchange, DLX, publisher confirms) as a second transport behind a `NotificationPublisher` port. ADR: Kafka (log, replay, partitions) vs RabbitMQ (routing, per-message acks) vs SQS. Adds `rabbitmq` to compose profile `core`. |
| Monolith → microservices | Phase 6 | Alert engine starts **inside** control-plane as a module, then is extracted into its own service with the **strangler-fig** pattern (route via config flag, dual-run, compare outputs, cut over). ADR + before/after diagram. |
| gRPC (NetApp story) | Phase 2 | Second ingest endpoint over **gRPC** (protobuf, client streaming, deadlines, status codes), sharing the same domain service (hexagonal payoff). Benchmark HTTP batch vs gRPC stream. |
| 99.98% SLA / error budgets | Phase 8 + 11 | `docs/system-health.md` includes error-budget math (99.98% ≈ 8.6 min/month of downtime; verify the arithmetic), burn-rate alerts, and what keeps it there: readiness gates, PDBs, rolling updates, retries+idempotency. |
| Rehearsal items (authorship wording, cert date, questions to ask) | Phase 14 | Not code. `docs/interview-map.md` links each resume bullet to the matching project artifact, as a *fresh Java re-implementation, not a claim of production use*. The "questions to ask" pair with your own perf-report numbers ("my local baseline was X ev/s; what's yours at peak?"). |

---

## 5b. Business scenarios & detailed requirements for the gap skills

**How to read this section**
- Every skill gets a **real business reason** to exist in StreamPulse, so in the interview you explain *why* it was needed, not just that you used it.
- Requirement IDs (`REQ-<area>-NN`) are traced through tests (`@DisplayName("REQ-LC-03 ...")`), specs and `docs/interview-map.md`.
- Acceptance criteria use **EARS** style: *WHEN <trigger> THE SYSTEM SHALL <response>*.
- All numbers are **design targets we choose**, not industry facts. They get validated in Phase 9 and adjusted with measured data.
- Each scenario lists: context → actors → user stories → functional reqs → NFRs → flow of control / data flow → skills & interview questions answered → acceptance tests → phase.

**Tenants used in scenarios** (fictional): `streamflix` (general VOD, 40% of traffic), `sportsnow` (live sports, spiky, enterprise SLA), `kidsplus` (small, strict data rules).

---

### BS-01 · Zero-loss rolling deploys & per-environment config
**Skills:** Spring bean lifecycle, `SmartLifecycle` phases, scopes, profiles, `@ConfigurationProperties`, DI style.

**Context:** SRE ships sink-writer and ingest-gateway several times a day. In the last (fictional) incident, a pod was killed mid-batch: ~2k events were buffered in memory and lost, and the S3 part-file was left half-written. Also, a missing Kafka bootstrap property in staging was only found at first request, not at startup.

**Actors:** SRE, release pipeline, Kubernetes (SIGTERM), on-call engineer.

**User stories**
- As an SRE, I want a pod shutdown to drain in-flight work before exiting, so deploys never lose telemetry.
- As an on-call engineer, I want misconfiguration to fail the pod at startup, so bad config never receives traffic.
- As a developer, I want the same artifact to run in `local`, `aws`, `k8s` and `datadog` profiles with no code changes.

**Functional requirements**
- **REQ-LC-01** WHEN the app receives SIGTERM THE SYSTEM SHALL stop components in this order: (1) readiness=OUT_OF_SERVICE, (2) stop accepting HTTP/gRPC, (3) stop Kafka listeners (commit offsets), (4) flush S3 buffers, (5) close producers. The order is implemented as `SmartLifecycle` beans with explicit `getPhase()` values.
- **REQ-LC-02** WHEN the S3 buffer holds un-flushed events at shutdown THE SYSTEM SHALL flush them within `streampulse.shutdown.flush-timeout` (default 20s). IF the timeout is exceeded, it SHALL NOT commit those Kafka offsets, so the events are re-delivered (at-least-once) rather than lost.
- **REQ-LC-03** WHEN a required property (`streampulse.kafka.bootstrap`, `streampulse.s3.bucket`, …) is missing or invalid THE SYSTEM SHALL fail startup with a message naming the property (`@ConfigurationProperties` + `@Validated`).
- **REQ-LC-04** THE SYSTEM SHALL resolve a **request-scoped** `TenantContext` from the JWT, injected into singleton services via a scoped proxy. A test proves two concurrent requests never see each other's tenant.
- **REQ-LC-05** THE SYSTEM SHALL use **prototype-scoped** `BatchAccumulator` beans (one per listener container), and a test proves they are distinct instances.
- **REQ-LC-06** THE SYSTEM SHALL select adapters by profile: `local` → moto/DynamoDB Local endpoints, `aws` → real endpoints, `datadog` → Micrometer Datadog registry, `k8s` → Actuator probes enabled. No `if (env == ...)` in code; it is done with `@Profile`/`@ConditionalOnProperty`.
- **REQ-LC-07** The `libs/platform` starter SHALL auto-configure the tracing, error model and idempotency beans with `@ConditionalOnMissingBean`, so a service can override any of them. A test with `ApplicationContextRunner` proves both the default and the override paths.
- **REQ-LC-08** A custom `BeanPostProcessor` SHALL wrap every bean annotated `@TimedPort` with a Micrometer timer proxy. This demonstrates how Spring proxies (and so `@Transactional`) work.

**NFRs:** 0 events lost across 20 consecutive rolling restarts under steady load. Shutdown completes in < 30s (fits K8s `terminationGracePeriodSeconds: 45`).

**Flow of control (shutdown):** K8s SIGTERM → JVM shutdown hook → `ApplicationContext.close()` → `SmartLifecycle.stop()` called in **descending phase order** → readiness probe fails (K8s removes the endpoint) → Kafka containers stop and commit → `S3BufferFlusher.stop(callback)` flushes → producer `close(Duration)` → `@PreDestroy` on remaining singletons.

**Interview questions answered:** bean lifecycle order; `@PostConstruct` vs `InitializingBean` vs `SmartLifecycle`; why constructor injection; how auto-config decides; how scoped proxies work; what `@Transactional` actually is.

**Acceptance tests:** `ShutdownOrderIT` (records the order of stop callbacks); `RollingRestartChaosIT` (Testcontainers + kill signals, counts events end to end); `ConfigValidationTest`; `TenantContextConcurrencyTest`; `PlatformAutoConfigurationTest`.
**Phase:** 0 (platform starter) + 4 (sink-writer) + 11 (K8s drill).

---

### BS-02 · "Title Command Center" page & archive offload
**Skills:** `CompletableFuture` composition, `ThreadPoolExecutor` tuning, virtual threads vs pools, locks, atomics, latches, deadlock diagnosis.

**Context:** The operations team wants one page per title during a live event showing live viewers (Redis), the 24h trend (DynamoDB), title metadata (control-plane, via Caffeine), active alerts, and top error codes. Calling these one after another takes ~600ms (target breach). Separately, S3 uploads in sink-writer block Kafka listener threads, which causes consumer lag and rebalances.

**Actors:** live-ops operator, query-api, sink-writer.

**User stories**
- As a live-ops operator, I want the Command Center to load in < 250ms p95, even if one panel is slow, showing partial data with a "stale" badge.
- As a platform engineer, I want slow S3 uploads never to stall Kafka consumption.
- As a finance analyst, I want an accurate per-tenant ingest counter with no lock contention at 50k ev/s.

**Functional requirements**
- **REQ-CC-01** WHEN `GET /v1/titles/{id}/command-center` is called THE SYSTEM SHALL fetch the 5 panels **in parallel** (`CompletableFuture.supplyAsync` on a named executor) and combine them with `allOf` / `thenCombine`.
- **REQ-CC-02** WHEN any panel exceeds its budget (`orTimeout`, per-panel config, default 150ms) THE SYSTEM SHALL return the page with that panel marked `status: "DEGRADED"` (`exceptionally`/`completeOnTimeout`) and increment `commandcenter.panel.degraded{panel}`.
- **REQ-CC-03** Metadata SHALL come from a **Caffeine** cache (size- and time-bounded, configurable), invalidated by `config.changes.v1`. The ADR covers why a library beats a hand-rolled cache here — Caffeine's W-TinyLFU admission policy, its non-blocking reads, and the fact that a naive LRU mutates access order on a *read*, so a `ReadWriteLock` around one would need the **write** lock on every get. That last point is the discussion worth having out loud.
- **REQ-CC-04** sink-writer SHALL hand S3 uploads to a **bounded `ThreadPoolExecutor`** (core=4, max=16, `ArrayBlockingQueue(200)`, named threads, `CallerRunsPolicy`). WHEN the queue is full, the caller (listener) thread runs the upload itself, which naturally slows consumption (backpressure) instead of dropping or OOMing.
- **REQ-CC-05** An alternate implementation using `Executors.newVirtualThreadPerTaskExecutor()` + `Semaphore(16)` SHALL exist behind a property. Phase 9 compares the two, and the ADR explains why an unbounded virtual-thread executor still needs a semaphore (downstream protection).
- **REQ-CC-06** Per-tenant ingest counters SHALL use `LongAdder` in a `ConcurrentHashMap<TenantId, LongAdder>` (`computeIfAbsent`) and be exported as Micrometer gauges. A JMH micro-benchmark compares `AtomicLong`, `LongAdder` and `synchronized` under contention.
- **REQ-CC-07** **Quota rebalancing (deadlock lab):** `POST /admin/quota/transfer {from, to, amount}` moves ingest quota between tenant buckets. v1 locks `from` then `to`, which can deadlock under crossed transfers (reproduced by a test with `CyclicBarrier`). v2 fixes it with **lock ordering by tenantId** plus `tryLock(timeout)`. The thread dump that shows the deadlock is committed to `docs/labs/deadlock.md`.
- **REQ-CC-08** Concurrency tests SHALL use `CountDownLatch` to start N threads simultaneously, and assert invariants (total quota conserved; no lost counter increments).

**NFRs:** Command Center p95 < 250ms with one panel artificially slowed to 2s. Kafka consumer lag stays < 10k with S3 latency injected at 1s (Toxiproxy).

**Data flow:** browser → query-api → {Redis, DynamoDB, cache→control-plane, alerts (Redis), top-errors (Redis ZSET)} in parallel → merged DTO → JSON.

**Interview questions answered:** `thenApply` vs `thenCompose`; which pool `supplyAsync` uses by default and why that's dangerous; `ThreadPoolExecutor` sizing and rejection policies; `LongAdder` vs `AtomicLong`; deadlock conditions and prevention; virtual threads vs pools.

**Acceptance tests:** `CommandCenterTimeoutTest` (stubbed slow panel); `S3OffloadBackpressureIT`; `QuotaTransferDeadlockTest` (v1 times out, v2 passes 10k crossed transfers); `LruConcurrencyStressTest`; JMH report.
**Phase:** 4, 5, 9.

---

### BS-03 · Championship-final cost & freshness tuning
**Skills:** Spring Kafka producer/consumer config, delivery semantics, measured tuning (the "cut infra cost" story in Java).

**Context:** Finance flags the Kafka bill (broker storage + network). Product needs freshness (event → dashboard) < 5s p99 during the final. The team must cut **bytes on the wire/disk by ≥ 30%** without breaking freshness or delivery guarantees.

**Actors:** platform engineer, finance, product owner.

**User stories**
- As finance, I want a documented before/after cost proxy (bytes in/out, broker disk), so I can approve the change.
- As product, I want freshness to stay < 5s p99 at 10× baseline load.

**Functional requirements**
- **REQ-KT-01** Tuning parameters SHALL be externalized as Spring Kafka properties under profiles `kafka-baseline` and `kafka-tuned`: producer `linger.ms`, `batch.size`, `compression.type` (none/lz4/zstd), `acks`, `max.in.flight.requests.per.connection`; consumer `fetch.min.bytes`, `fetch.max.wait.ms`, `max.poll.records`, listener `concurrency`.
- **REQ-KT-02** A Gatling scenario SHALL run each configuration for a fixed duration and record: producer throughput, `record-size-avg`, `compression-rate-avg`, bytes-out, broker log size, p99 freshness, and consumer lag.
- **REQ-KT-03** The chosen config SHALL keep `enable.idempotence=true` and `acks=all`. The report SHALL explain why raising `max.in.flight` above 5 is disallowed with idempotence (verify the exact limit in current Kafka docs).
- **REQ-KT-04** Consumer config SHALL guarantee `max.poll.records × worst-case processing time < max.poll.interval.ms`. A test with an artificially slow handler proves the rebalance storm happens when the rule is violated, and does not happen when it is followed.
- **REQ-KT-05** Batch listeners SHALL use manual ack (`AckMode.MANUAL_IMMEDIATE` or `BATCH`, chosen and justified in the ADR). Idempotent downstream writes make redelivery safe.
- **REQ-KT-06** `docs/kafka-tuning.md` SHALL contain the matrix, charts, the chosen config and the reasoning.

**NFRs:** ≥ 30% bytes reduction (target). p99 freshness < 5s. No duplicate aggregates (verified by the idempotency table).

**Interview questions answered:** how batching/compression trade latency for throughput; idempotent producer internals (PID + sequence numbers); consumer rebalances and poll-interval pitfalls; at-least-once + idempotent consumer = effectively-once.

**Acceptance tests:** `KafkaTuningBenchmark` (Gatling + metrics scrape), `SlowConsumerRebalanceIT`, and a report committed with the real numbers.
**Phase:** 9.

---

### BS-04 · Peak-night degraded dependencies
**Skills:** Resilience4j CircuitBreaker, Retry (backoff + jitter), RateLimiter, Bulkhead, TimeLimiter: states, configs, metrics.

**Context:** During a big match (fictional scenario):
- (a) DynamoDB throttles writes.
- (b) S3 returns 503s for 3 minutes.
- (c) A partner webhook (a broadcaster's ops pager) allows only 50 req/s and gets slow.
- (d) Slow partner calls exhaust threads and take down alert evaluation for *all* tenants.

**Actors:** sink-writer, alert engine, partner webhook, on-call.

**User stories**
- As on-call, I want transient throttling to self-heal without paging me.
- As a partner, I want StreamPulse never to exceed my contracted rate.
- As the `kidsplus` tenant, I want `sportsnow`'s partner outage not to delay my alerts.

**Functional requirements**
- **REQ-RS-01 Retry:** WHEN DynamoDB returns a throttling error THE SYSTEM SHALL retry with **exponential backoff + jitter** (`IntervalFunction.ofExponentialRandomBackoff`, e.g. 100ms base, ×2, max 5 attempts). It SHALL NOT retry validation errors (`retryExceptions` / `ignoreExceptions` configured). Metrics: `resilience4j.retry.calls{kind}`.
- **REQ-RS-02 CircuitBreaker:** WHEN the S3 failure rate ≥ 50% over a count-based window of 20 calls (min 10) THE SYSTEM SHALL open the breaker for 30s, then allow 5 trial calls in HALF_OPEN. While OPEN, the fallback SHALL **spool batches to local disk** (bounded, 1 GB), and a background job SHALL replay the spool when the breaker closes. Slow-call threshold: > 2s counts as failure.
- **REQ-RS-03 RateLimiter (outbound, per instance):** Partner webhook calls SHALL go through a R4j `RateLimiter` (`limitForPeriod = 50 / replicas`, `limitRefreshPeriod=1s`, `timeoutDuration=0`). WHEN no permit is available THE SYSTEM SHALL enqueue the notification to RabbitMQ for delayed delivery (links BS-07). The ADR explains why a per-instance limiter must be divided by replica count, or replaced by a distributed limiter.
- **REQ-RS-04 Distributed quota (inbound):** Per-tenant ingest quotas SHALL stay on **Bucket4j + Redis** (cluster-wide). The doc contrasts R4j RateLimiter (local, protects callee) with Bucket4j (distributed, enforces a business quota).
- **REQ-RS-05 Bulkhead:** Partner calls SHALL be isolated **per tenant** with `ThreadPoolBulkhead` (core 2, max 4, queue 20). WHEN a tenant's bulkhead is full THE SYSTEM SHALL reject fast (`BulkheadFullException` → queue to RabbitMQ). The semaphore bulkhead is used on the reactive query-api path instead, and the doc explains why.
- **REQ-RS-06 TimeLimiter:** Every outbound call SHALL have an explicit timeout (default 2s). There are no infinite waits anywhere (ArchUnit rule: no `HttpClient` without a timeout).
- **REQ-RS-07 Ordering:** The decorator order SHALL be documented and tested: `Retry( CircuitBreaker( RateLimiter( TimeLimiter( Bulkhead( call )))))`. This matches, as I understand it, the default aspect order of the Resilience4j Spring Boot starter; verify in current docs. The doc explains why retry-outside-breaker matters.
- **REQ-RS-08 Observability:** Breaker state transitions SHALL emit events to logs + Micrometer. A Grafana panel shows state over time, and an alert fires if a breaker stays OPEN > 5 min.
- **REQ-RS-09** `docs/resilience.md` SHALL contain a table: pattern → failure mode it guards → config → what the user sees.

**NFRs:** Under a 3-minute S3 outage, zero raw events are lost (spool replay verified). `kidsplus` alert latency is unaffected (< 2s p99) while `sportsnow`'s partner is down.

**Flow (S3 outage):** batch ready → CB(S3) CLOSED → PUT fails ×10 → OPEN → fallback writes spool file → Kafka offsets committed only after the spool write is durable (fsync) → 30s → HALF_OPEN → 5 trial PUTs OK → CLOSED → `SpoolReplayer` uploads and deletes spool files.

**Interview questions answered:** the CB state machine and count- vs time-based windows; why jitter; retry storms; bulkhead semaphore vs thread-pool; local vs distributed rate limiting; decorator ordering.

**Acceptance tests:** `CircuitBreakerStateTransitionTest` (drives CLOSED→OPEN→HALF_OPEN→CLOSED with a stub); `S3OutageSpoolReplayIT` (Toxiproxy/moto fault); `PartnerRateLimitIT` (asserts ≤ 50 rps observed at a WireMock partner); `TenantBulkheadIsolationIT`; `RetryJitterTest` (asserts the intervals fall within the randomized bounds).
**Phase:** 2, 4, 6, 9.

---

### BS-05 · Tenant usage billing & ops console data layer
**Skills:** JPA vs raw JDBC (`JdbcClient`), transactions (propagation, isolation), N+1, HikariCP, SQL vs NoSQL trade-offs.

**Context:** Each tenant is billed monthly on ingested events + retained GB. Finance needs a usage report (heavy aggregate SQL over millions of rows). The ops console lists tenants → apps → titles (and is slow: N+1). Every rule change must be audited *even if the change itself fails validation later*.

**Actors:** finance analyst, tenant admin, ops console user, auditor.

**User stories**
- As finance, I want `GET /v1/billing/usage?month=2026-10` in < 2s for all tenants.
- As an ops user, I want the tenant tree to load with a bounded number of SQL queries.
- As an auditor, I want every attempted rule change recorded, including rejected ones.

**Functional requirements**
- **REQ-DL-01** The Admin CRUD (tenants, apps, titles, rules) SHALL use **Spring Data JPA** with Flyway-managed schema, optimistic locking (`@Version`), and ProblemDetail on conflict (409).
- **REQ-DL-02** The billing report SHALL use **`JdbcClient` + hand-written SQL** (window functions, `GROUP BY tenant, date_trunc`) over a `usage_hourly` table rolled up from Kafka. The same query is also written in JPQL; the benchmark and the reasoning for choosing JDBC go in the ADR.
- **REQ-DL-03 N+1:** `GET /v1/admin/tenants/tree` SHALL execute **≤ 3 SQL statements** regardless of tenant count. A test uses Hibernate `Statistics` to assert the query count. The broken v1 (lazy loops) is kept in a lab branch/test to demonstrate the problem.
- **REQ-DL-04 Propagation:** `RuleService.update()` (`REQUIRED`) SHALL call `AuditService.record()` (`REQUIRES_NEW`). WHEN rule validation throws after auditing THE SYSTEM SHALL roll back the rule but keep the audit row. A test proves it, plus a test showing the **self-invocation gotcha** (calling a `@Transactional` method from inside the same bean skips the proxy).
- **REQ-DL-05 Isolation:** Monthly quota allocation SHALL prevent two admins over-allocating the same pool: use `SELECT … FOR UPDATE` (or `SERIALIZABLE` + retry; choose one and justify it). A concurrency test proves no over-allocation.
- **REQ-DL-06 Outbox:** Rule/tenant changes SHALL write an `outbox` row in the **same transaction** (for Debezium, Phase 6). A test proves that on rollback neither the entity nor the outbox row persists.
- **REQ-DL-07 HikariCP:** The pool size SHALL be set from measurement (starting from the commonly cited formula `connections ≈ cores × 2 + effective spindles`, which should be treated as a heuristic only). `leakDetectionThreshold` is enabled in `local`. Pool metrics go on Grafana. A lab shows pool exhaustion with virtual threads and a `Semaphore` guarding the pool.
- **REQ-DL-08 NoSQL choice:** Alert history (heterogeneous per rule type, append-heavy, queried by tenant+time) SHALL be stored in **MongoDB** under the optional `mongo` profile, alongside the DynamoDB aggregate store. The ADR compares Postgres JSONB vs Mongo vs DynamoDB for this access pattern, and Postgres vs MySQL (MVCC, JSONB, `FOR UPDATE SKIP LOCKED`).

**NFRs:** billing report < 2s over 10M `usage_hourly` rows (seeded). Tenant tree < 100ms for 200 tenants × 20 apps × 50 titles.

**Interview questions answered:** when to drop the ORM; the N+1 fix options (fetch join, `@EntityGraph`, batch size); propagation types; isolation levels and anomalies; why a bigger pool isn't better; SQL vs NoSQL chosen by access pattern.

**Acceptance tests:** `TenantTreeQueryCountTest`, `BillingReportIT` (Testcontainers Postgres + seeded data + `EXPLAIN ANALYZE` captured), `AuditRequiresNewTest`, `SelfInvocationGotchaTest`, `QuotaAllocationConcurrencyIT`, `OutboxAtomicityIT`, `MongoAlertHistoryIT`.
**Phase:** 6.

---

### BS-06 · Serverless archive compaction & regional resilience
**Skills:** AWS Lambda (Spring Cloud Function), VPC design in Terraform, cross-region replication, RPO/RTO.

**Context:** sink-writer writes many small S3 objects (one per flush), which makes Athena-style queries slow and expensive. Security requires that services reach S3/DynamoDB only through private networking. The business requires surviving the loss of a region for `sportsnow` (enterprise contract).

**Actors:** data engineer, security reviewer, SRE, `sportsnow` account manager.

**User stories**
- As a data engineer, I want small objects compacted into hourly files automatically.
- As security, I want no service subnet to have a route to the internet for data-plane traffic.
- As the `sportsnow` account manager, I want a documented and drilled failover with RPO ≤ 1 min and RTO ≤ 15 min (targets).

**Functional requirements**
- **REQ-AW-01 Lambda:** WHEN an object lands in `raw/…/hour=H/` THE SYSTEM SHALL trigger (S3 event → SQS → Lambda) a Spring Cloud Function `compactHour` that merges part-files once the hour is closed (plus a grace period) into `compacted/…/hour=H.ndjson.gz` and writes a manifest. The function is idempotent: re-running produces the same output (deterministic ordering, overwrite by key).
- **REQ-AW-02** The function SHALL be deployable to moto Lambda (`local`), with a documented fallback of `sam local invoke` if moto-Lambda doesn't work on Mac. Cold-start is measured, and SnapStart is discussed (verify current Java runtime support).
- **REQ-AW-03 VPC (Terraform):**
  - VPC with 3 AZ private subnets for services and public subnets only for load balancers.
  - **Gateway endpoints** for S3/DynamoDB and interface endpoints for SQS/Kinesis.
  - Security groups least-privilege (ingest SG → MSK SG on 9092/9094 only).
  - Applied against moto at the API level; `terraform plan` output committed.
  - `docs/network.md` states plainly that real isolation cannot be verified locally.
- **REQ-AW-04 Cross-region (simulated):** Compose profile `multiregion` runs Kafka `region-a` and `region-b` with **MirrorMaker 2** replicating `telemetry.agg.v1` and `config.changes.v1`. A consumer offset-sync/translation drill is documented (verify MM2 checkpoint behavior in current docs).
- **REQ-AW-05 Failover drill:** WHEN `region-a` is stopped THE SYSTEM SHALL let query-api (pointed at `region-b` via profile/flag) serve dashboards. The drill measures data lag (RPO proxy) and time-to-serve (RTO proxy). Results go into `docs/dr-drill.md`.
- **REQ-AW-06** Terraform SHALL use modules + remote-state pattern (local backend in dev), `tflint`, `terraform validate` in CI, and per-env `tfvars`.
- **REQ-AW-07** The design doc SHALL cover DynamoDB global tables, S3 CRR and MSK replication as the production approach, with the trade-offs (conflict resolution, cost, replication lag).

**Interview questions answered:** Lambda triggers and idempotency; the small-files problem; VPC endpoints (gateway vs interface); active-passive vs active-active; RPO vs RTO.

**Acceptance tests:** `CompactHourFunctionTest` (pure unit), `CompactionLambdaIT` (moto), `terraform validate/plan` in CI, `MirrorMakerFailoverIT` (scripted drill, manual/nightly).
**Phase:** 10.

---

### BS-07 · Partner alert routing over RabbitMQ
**Skills:** RabbitMQ / Spring AMQP (exchanges, routing keys, DLX, publisher confirms, manual acks), Kafka vs RabbitMQ vs SQS trade-off.

**Context:** Broadcast partners (fictional) want to subscribe to alert subsets: "all CRITICAL for sportsnow", "any CDN alert for any tenant". Each partner has its own delivery endpoint and reliability. Kafka topics-per-partner would explode in number, and per-message routing + per-message retry is RabbitMQ's strength.

**Actors:** partner integration manager, alert engine, partner delivery workers.

**User stories**
- As a partner manager, I want to register a subscription with a routing pattern, with no code deploy.
- As a partner, I want failed deliveries retried and eventually parked for inspection, not lost.

**Functional requirements**
- **REQ-MQ-01** The alert engine SHALL publish every alert to topic exchange `alerts.x` with routing key `alert.<tenant>.<severity>.<category>` via a `NotificationPublisher` port (Rabbit adapter). SQS stays as the second adapter, for AWS parity.
- **REQ-MQ-02** `POST /v1/partners/{id}/subscriptions {pattern: "alert.sportsnow.CRITICAL.*"}` SHALL declare a durable queue `partner.<id>` bound with that pattern (`RabbitAdmin`/`Declarables`).
- **REQ-MQ-03** Publishing SHALL use **publisher confirms + mandatory flag**. WHEN a publish is nacked or unroutable THE SYSTEM SHALL log it, count it and keep the alert in the outbox for retry.
- **REQ-MQ-04** Delivery workers SHALL use **manual ack**. On failure: retry via a TTL-based delay queue ×3 (exponential), then route to DLX `partner.<id>.dlq`. An admin endpoint lists and replays DLQ messages.
- **REQ-MQ-05** Prefetch (`basic.qos`) SHALL be configured per partner (default 10). The ADR explains how prefetch affects fairness and memory.
- **REQ-MQ-06** Partner calls go through BS-04's RateLimiter + Bulkhead.
- **REQ-MQ-07** The ADR SHALL compare Kafka (log/replay/ordering per partition), RabbitMQ (routing/per-message ack/priority) and SQS/SNS (managed, fan-out via SNS filter policies).

**Interview questions answered:** exchange types; ack/nack/requeue semantics; DLX; confirms vs transactions; when RabbitMQ beats Kafka and vice versa (maps to the JustAnswer backbone story).

**Acceptance tests:** `PartnerRoutingIT` (Testcontainers RabbitMQ; asserts only matching queues receive), `PublisherConfirmIT`, `DlqRetryIT` (failing WireMock partner → 3 retries → DLQ → replay).
**Phase:** 6.

---

### BS-08 · Extracting the alert engine (monolith → microservice)
**Skills:** strangler-fig migration, shadow traffic/dual-run, feature flags, service boundaries, data ownership.

**Context:** The alert engine starts as a module inside control-plane. During peaks its CPU-heavy evaluation slows the admin API (p99 of rule CRUD rises from 80ms to 900ms in load tests), and the two need different scaling (alerts scale with Kafka partitions, admin with users). It must be extracted with **no missed or duplicate alerts**.

**Actors:** platform team, on-call, tenant admins.

**User stories**
- As a tenant admin, I want the rule UI to stay fast during live events.
- As on-call, I want the extraction reversible at any moment with a flag flip.

**Functional requirements**
- **REQ-SF-01 v0 (monolith module):** The alert engine is a separate Gradle module inside control-plane, with **package-level boundaries enforced by ArchUnit** (the engine may use only the `rules` read API, never the JPA repositories directly).
- **REQ-SF-02 Seam:** Rules reach the engine via `config.changes.v1` (CDC) and never by a DB join. This makes the data-ownership cut before the process cut.
- **REQ-SF-03 Shadow mode:** The new `alert-engine` service SHALL consume the same topics under a different consumer group and publish to `alerts.shadow.v1`. A comparator job diffs `alerts.v1` vs `alerts.shadow.v1` per window and exports a mismatch metric.
- **REQ-SF-04 Cutover:** WHEN the flag `alerts.engine=service` is set THE SYSTEM SHALL stop the in-monolith engine's listener (`SmartLifecycle` stop, from BS-01) and promote the service's output to `alerts.v1`. The flip is atomic per deploy, and flipping it back restores v0.
- **REQ-SF-05 Dedup across cutover:** Alerts carry a deterministic `alertId = hash(ruleId, windowStart)`, and consumers dedup on it, so an overlap window produces no duplicate pages.
- **REQ-SF-06** The ADR + before/after diagrams SHALL cover when *not* to extract (the cost of a network hop, operational overhead).

**NFRs:** 0 mismatches over a 1-hour shadow run under load before cutover. Admin API p99 back to < 100ms after extraction.

**Interview questions answered:** how to migrate a monolith safely; how to find service boundaries; data ownership; the dual-write danger; rollback strategy.

**Acceptance tests:** `AlertEngineBoundaryArchTest`, `ShadowComparatorIT`, `CutoverFlagIT` (flip mid-stream, assert no gaps or dups).
**Phase:** 6 (v0) → after Phase 9 (extraction, using measured load data).

---

### BS-09 · Smart-TV gRPC streaming ingest
**Skills:** gRPC (protobuf, client streaming, deadlines, status codes, flow control), REST vs gRPC trade-offs, hexagonal reuse.

**Context:** A (fictional) smart-TV OEM SDK keeps a long-lived connection and emits heartbeats every 10s per device. HTTP batch overhead (TLS handshakes, JSON) dominates device CPU and battery. The OEM asks for a binary streaming protocol.

**Actors:** TV SDK, ingest-gateway, OEM partner engineer.

**User stories**
- As the OEM, I want to stream events over one connection and receive periodic acks with the last durable sequence number.
- As platform, I want both protocols to share the same validation, idempotency and publishing logic.

**Functional requirements**
- **REQ-GR-01** `telemetry.proto` SHALL define `rpc StreamEvents(stream TelemetryEvent) returns (stream Ack)` (bidi) and `rpc SendBatch(Batch) returns (BatchAck)` (unary). The proto lives in `libs/contracts`, and `buf lint` + breaking-change check run in CI (verify buf tooling).
- **REQ-GR-02** Both gRPC and REST adapters SHALL call the same `IngestUseCase` port. An ArchUnit test ensures no business logic lives in the adapters.
- **REQ-GR-03** WHEN the client sets a deadline THE SYSTEM SHALL honor it (cancel downstream work on `Context` cancellation). Errors SHALL map to status codes: validation → `INVALID_ARGUMENT`, quota → `RESOURCE_EXHAUSTED` (with retry info in trailers), overload → `UNAVAILABLE`, auth → `UNAUTHENTICATED`.
- **REQ-GR-04** Server-side flow control SHALL use manual inbound flow control (`request(n)`) tied to the same in-flight semaphore as REST (backpressure parity).
- **REQ-GR-05** Auth: JWT in metadata, validated by a `ServerInterceptor`. The tracing interceptor propagates W3C trace context.
- **REQ-GR-06** A benchmark (ghz or a Gatling gRPC plugin; verify tool availability) compares REST batch vs gRPC stream on CPU, bytes and p99. Results go in the ADR.
- **REQ-GR-07** K8s note: gRPC needs L7 load balancing (long-lived HTTP/2 connections defeat L4 balancing). This is documented, with a client-side LB / service mesh option.

**Interview questions answered:** gRPC streaming types; deadlines vs timeouts; status codes; HTTP/2 multiplexing; the gRPC load-balancing problem in K8s (maps to the NetApp story).

**Acceptance tests:** `GrpcIngestIT` (in-process server + real Kafka via Testcontainers), `GrpcDeadlineCancellationTest`, `GrpcStatusMappingTest`, `ProtocolParityTest` (same input via REST and gRPC gives identical Kafka records).
**Phase:** 2 (after REST ingest is green).

---

### BS-10 · Enterprise SLA 99.98% for `sportsnow`
**Skills:** SLI/SLO/SLA, error budgets, burn-rate alerting, the K8s availability mechanics that make the number achievable.

**Context:** `sportsnow` signs a (fictional) contract: **99.98% monthly availability** of the ingest API and **dashboard freshness < 10s**, with service credits on breach. Engineering must prove it can measure and defend this.

**Actors:** account manager, SRE, engineering manager, release pipeline.

**User stories**
- As the account manager, I want a monthly SLA report per tenant.
- As an SRE, I want to be paged when we're burning budget fast, not on every blip.
- As an engineering manager, I want a release-freeze policy tied to the remaining budget.

**Functional requirements**
- **REQ-SL-01 SLIs:**
  - Ingest availability = good requests (2xx + 429s within quota policy; the definition is justified in the doc) / valid requests.
  - Freshness = share of windows where (SSE delivery time − event time) < 10s.
  - Both are recorded as Prometheus recording rules per tenant.
- **REQ-SL-02 Budget math:** 99.98% over 30 days = 0.02% × 43,200 min = **8.64 min** of allowed unavailability per month. The doc shows the calculation and the equivalent request-based budget.
- **REQ-SL-03 Burn-rate alerts:** multi-window, multi-burn-rate alerts (e.g. 1h/5m windows at 14.4× and 6h/30m at 6×). The pattern is as described in the Google SRE Workbook, *from memory: verify the exact recommended factors there*.
- **REQ-SL-04 Freeze policy:** WHEN the remaining monthly budget is < 25% THE SYSTEM SHALL fail the CD pipeline's deploy gate for non-fix releases (a GitHub Actions job queries Prometheus). Override requires a labelled PR.
- **REQ-SL-05 What keeps it there:** Helm values and tests for:
  - `maxUnavailable: 0` rolling updates
  - PodDisruptionBudget `minAvailable: 2`
  - readiness gates (BS-01 drain)
  - topology spread across nodes/zones
  - HPA/KEDA headroom
  - retries + idempotency end to end (BS-04)
  - Each item is listed in `docs/system-health.md` with the failure it absorbs.
- **REQ-SL-06 Report:** `GET /v1/sla/{tenant}?month=` returns the SLI values, budget consumed and incidents (from alert history). Grafana SLO dashboard per tenant.

**NFRs:** A chaos run (killing 1 of 3 pods every 2 min during load) consumes < 10% of the monthly budget.

**Interview questions answered:** SLA vs SLO vs SLI; error budgets; burn-rate alerting; what "99.98%" means operationally (maps to the NetApp 99.98% story, rebuilt with your own numbers).

**Acceptance tests:** `promtool test rules` for recording/alert rules, `PodKillChaosTest` (kind + load), `SlaReportIT`, deploy-gate workflow dry run.
**Phase:** 8 + 11.

---

### Traceability summary

| Scenario | Gap skill | Requirement IDs | Phase |
|---|---|---|---|
| BS-01 | Spring core: lifecycle, scopes, profiles, auto-config | REQ-LC-01..08 | 0, 4, 11 |
| BS-02 | Concurrency primitives | REQ-CC-01..08 | 4, 5, 9 |
| BS-03 | Kafka tuning with numbers | REQ-KT-01..06 | 9 |
| BS-04 | Resilience4j (all patterns) | REQ-RS-01..09 | 2, 4, 6, 9 |
| BS-05 | JPA vs JDBC, transactions, pooling, NoSQL | REQ-DL-01..08 | 6 |
| BS-06 | AWS Lambda, VPC, cross-region | REQ-AW-01..07 | 10 |
| BS-07 | RabbitMQ | REQ-MQ-01..07 | 6 |
| BS-08 | Monolith → microservices | REQ-SF-01..06 | 6 → post-9 |
| BS-09 | gRPC | REQ-GR-01..07 | 2 |
| BS-10 | 99.98% SLA / error budgets | REQ-SL-01..06 | 8, 11 |

When implementation starts, each BS block is copied into the matching `specs/NN-*/requirements.md`. Then `design.md` (sequence diagrams per flow above) and `tasks.md` are written, and the tests named under "Acceptance tests" are created **first**, failing, before any production code.

---

## 6. Verification (per phase and end-to-end)

- **Per phase:** `make up-<profile>` → `./gradlew test` (unit/slice, no Docker) → `./gradlew integrationTest` (Testcontainers) → `make demo-<phase>`. The verified output is pasted into `MILESTONES.md`, and every acceptance criterion in `specs/NN/requirements.md` is checked off.
- **End-to-end (after Phase 7):**
  1. `make up-full`.
  2. `make seed` (tenants, titles, rules via the control-plane API).
  3. `make load-steady` (Gatling).
  4. Open the dashboard: live viewers/rebuffer charts update within ~N seconds (target to be set in the Phase 5 spec).
  5. Trigger a rule via `make load-spike-errors`: the alert appears in the UI and in SQS (moto) with downstream alerts suppressed.
  6. Change a rule in the UI and watch the Debezium `config.changes.v1` event invalidate caches (visible in logs/trace).
  7. In Grafana: RED, lag and freshness panels, plus one trace from ingest to SSE.
  8. From Claude Code: ask the MCP server for active alerts.
- **CI:** GitHub Actions runs unit + integration tests on every push. Playwright e2e runs on a manual/nightly workflow (compose in CI is heavier).

## 7. First step after approval
Create the repo and write `specs/00-foundation/{requirements,design,tasks}.md` plus the Gradle skeleton, then stop for your review before Phase 1 (spec-first, one phase at a time).
