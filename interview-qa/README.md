# StreamPulse Interview Q&A

Study material for senior backend (Java/Spring, distributed systems) interviews, built around the
**StreamPulse** portfolio project ([../StreamPulse_Project_Plan.md](../StreamPulse_Project_Plan.md)).

Every answer is tied back to a **business scenario (BS-01…BS-10)** and a **requirement ID** from the
plan, so when you later write the code you are reinforcing something you already studied — and in the
interview you can say *"here's how I actually built it"* rather than *"here's what the docs say."*

---

## How to use this

1. **One file per sitting.** Read it, then close it and answer the questions out loud from memory.
2. **The 30-second answer is the real deliverable.** That is what you say in the room. Everything
   below it is ammunition for the follow-ups.
3. **Fill in the `> Your story:` placeholders.** They mark spots where your real Couchbase /
   JustAnswer / NetApp experience goes. I cannot invent those — and an invented one collapses on the
   first follow-up.
4. **Check every `Verify:` line before you rely on it.** Those are version-sensitive or things I am
   not fully certain of.

---

## Reading order

### Tier 1 — asked in almost every round
| # | File | Topic |
|---|---|---|
| 01 | [01-spring-core.md](01-spring-core.md) | IoC/DI, bean lifecycle, scopes, profiles, auto-configuration, AOP proxies |
| 02 | [02-spring-mvc-vs-webflux.md](02-spring-mvc-vs-webflux.md) | MVC request flow, WebFlux/Reactor, backpressure, SSE, virtual threads vs reactive |
| 03 | [03-concurrency.md](03-concurrency.md) | JMM, locks, atomics, thread pools, `CompletableFuture`, virtual threads, deadlock |
| 04 | [04-kafka-and-spring-kafka.md](04-kafka-and-spring-kafka.md) | Partitions, consumer groups, delivery semantics, DLT, tuning, Kafka Streams |
| 05 | [05-resilience4j.md](05-resilience4j.md) | Circuit breaker states, retry+jitter, rate limiter, bulkhead, decorator order |
| 06 | [06-data-layer.md](06-data-layer.md) | JPA vs JDBC, N+1, transactions, isolation, HikariCP, SQL vs NoSQL |

### Tier 2 — the distributed-systems core of this role
| # | File | Topic |
|---|---|---|
| 07 | [07-cdc-outbox-sse.md](07-cdc-outbox-sse.md) | Dual-write problem, outbox, Debezium CDC, SSE vs WebSocket |
| 08 | [08-messaging-rabbitmq-kafka-sqs.md](08-messaging-rabbitmq-kafka-sqs.md) | RabbitMQ internals, and the broker decision table |
| 09 | [09-microservices-migration.md](09-microservices-migration.md) | Boundaries, strangler fig, shadow traffic, when *not* to split |
| 10 | [10-grpc.md](10-grpc.md) | Protobuf, streaming, deadlines, status codes, the K8s L7 LB problem |
| 11 | [11-observability-and-slos.md](11-observability-and-slos.md) | Micrometer, tracing, SLI/SLO/SLA, error budgets, burn-rate alerts |

### Tier 3 — platform & production
| # | File | Topic |
|---|---|---|
| 12 | [12-aws-and-terraform.md](12-aws-and-terraform.md) | Lambda, VPC, Kinesis vs MSK, DynamoDB modeling, cross-region, Terraform |
| 13 | [13-kubernetes-for-stream-apps.md](13-kubernetes-for-stream-apps.md) | Probes, graceful shutdown, PDB, KEDA lag autoscaling, stateful stream apps |
| 14 | [14-testing.md](14-testing.md) | JUnit 5, Mockito, test slices, Testcontainers, `TopologyTestDriver`, contract tests |
| 15 | [15-jvm-performance.md](15-jvm-performance.md) | Memory model, G1 vs ZGC, JIT, leaks, JFR/async-profiler, diagnosing prod |

### Tier 4 — the interview rounds themselves
| # | File | Topic |
|---|---|---|
| 16 | [16-system-design-telemetry-and-feed.md](16-system-design-telemetry-and-feed.md) | Two full worked designs: telemetry pipeline, Twitter-like feed |
| 17 | [17-coding-lru-and-graphs.md](17-coding-lru-and-graphs.md) | LRU/LFU from scratch, BFS/DFS set, narration script |
| 18 | [18-ai-tooling.md](18-ai-tooling.md) | MCP, agentic workflows, spec-driven development story |

---

## Answer template

Every question follows the same shape:

- **30-second answer** — what you say out loud.
- **Deeper** — mechanism, flow of control, data flow.
- **Code** — small Java 21 snippet.
- **Follow-ups they'll ask** — the second and third questions.
- **Pitfalls / red flags** — what a weak answer sounds like.
- **Where in StreamPulse** — BS-xx / REQ-xx-nn / Phase n.
- **Verify** — anything to confirm in current docs before you say it.

---

## Business scenario cheat sheet

| ID | One-line scenario | Skills it proves |
|---|---|---|
| BS-01 | Zero-loss rolling deploys & per-env config | Spring lifecycle, scopes, profiles, auto-config |
| BS-02 | Title Command Center page + S3 offload | `CompletableFuture`, thread pools, locks, atomics, deadlock |
| BS-03 | Championship-final cost & freshness tuning | Kafka producer/consumer tuning with measured numbers |
| BS-04 | Peak-night degraded dependencies | All Resilience4j patterns |
| BS-05 | Tenant usage billing & ops console | JPA vs JDBC, transactions, N+1, pooling, NoSQL |
| BS-06 | Archive compaction & regional DR | Lambda, VPC, cross-region, RPO/RTO |
| BS-07 | Partner alert routing | RabbitMQ, broker trade-offs |
| BS-08 | Extracting the alert engine | Strangler fig, shadow run, cutover |
| BS-09 | Smart-TV gRPC ingest | gRPC streaming, deadlines, flow control |
| BS-10 | 99.98% enterprise SLA | SLI/SLO, error budgets, burn-rate alerts |

---

## Honesty rules baked into this material

- Scenarios BS-01…BS-10 are **fictional design exercises** for a portfolio project. Present them that
  way. Do not describe them as production incidents you lived through.
- Numbers in the plan (p95 targets, 30% byte reduction, RPO/RTO) are **targets chosen for the
  project**, to be replaced with measured values in Phase 9.
- Where an answer says `> Your story:`, that is where your real experience goes — and only what you
  can defend under three follow-ups.
