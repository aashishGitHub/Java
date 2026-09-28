# Verification checklist

Claims across this material that are **version-sensitive, or that I was not fully certain of**.
Confirm each against current documentation before stating it in an interview.

My knowledge has a cutoff, and library defaults, annotation names, JVM flags and cloud
quotas all change. A confidently-wrong technical claim costs more in an interview than
saying "I'd check the current docs on the exact value." — which is itself a good answer.

Auto-generated from every `**Verify:**` line in the Q&A files.

## 01-spring-core.md

- [ ] the `allow-circular-references` default changed in Spring Boot 2.6 — confirm the behaviour in the Boot version you pin.
- [ ] the exact `SmartLifecycle` phase semantics and `timeout-per-shutdown-phase` key against your Spring version's docs before quoting them.
- [ ] the `AutoConfiguration.imports` path replaced `spring.factories` in Boot 2.7/3.x — confirm for your pinned version. `@AutoConfiguration` is Boot 2.7+; earlier code uses `@Configuration` + `@AutoConfigureAfter`.
- [ ] `@DurationMin`/`@DurationMax` and the exact `@ConfigurationPropertiesScan` behaviour — confirm in the Boot version you pin. Record-based constructor binding is Boot 3.x.
- [ ] the default `proxyTargetClass` value for your Boot version, and unwrap `InvocationTargetException` in real code.

## 02-spring-mvc-vs-webflux.md

- [ ] default content-negotiation behaviour (path extension / `favorParameter`) changed across Spring versions — confirm for the Boot version you pin.
- [ ] the `createProblemDetail(...)` signature and the `problemdetails.enabled` property keys differ across Spring 6.x / Boot 3.x minor versions — check the version you pin.
- [ ] Spring 6.1 changed method-parameter validation to throw `HandlerMethodValidationException` rather than `ConstraintViolationException` — confirm the behaviour in your version before claiming it.
- [ ] Reactor Netty's default worker count, the `boundedElastic` cap formula, and the virtual-thread scheduler property — all version-sensitive.
- [ ] `Sinks.EmitFailureHandler.busyLooping(Duration)` availability, and the exact `autoCancel` default for `multicast().onBackpressureBuffer(...)` in your Reactor version.
- [ ] `ServerSentEvent.Builder#comment` and `#retry` exist in Spring's reactive codec package — confirm the exact builder methods for your Spring version.
- [ ] `spring.threads.virtual.enabled` (Boot 3.2+), `-Djdk.tracePinnedThreads` status, JEP 491's delivery version, and `StructuredTaskScope`'s current API — all changed recently.
- [ ] `RestClient` requires Spring Framework 6.1 / Boot 3.2 or later; HTTP-interface client class names changed across 6.x minors.
- [ ] `@MockitoBean` replaced `@MockBean` in recent Boot/Spring Framework versions — check which one your version wants.

## 03-concurrency.md

- [ ] the `synchronized`-pins-carrier behaviour is JDK-version-dependent (JEP 491); biased locking removal is also JDK-version-dependent.
- [ ] `ForkJoinPool.commonPool()` parallelism is `availableProcessors() - 1` and is influenced by `-Djava.util.concurrent.ForkJoinPool.common.parallelism` — confirm the behaviour on your JDK, especially inside a CPU-limited container.
- [ ] JEP 491's delivery version, `-Djdk.tracePinnedThreads` vs the JFR pinning event, `ScopedValue`'s preview status, and `spring.threads.virtual.enabled` — all moved recently.

## 04-kafka-and-spring-kafka.md

- [ ] your broker version, your client version, and whether your cluster runs KRaft or (on an > old install) ZooKeeper. Defaults below are "as documented for recent 3.x/4.x" — confirm against the > version you pin.
- [ ] whether `min.insync.replicas` defaults to 1 on your broker, and the exact segment-rolling config names (`segment.ms`, `segment.bytes`) for your version.
- [ ] which Kafka and client versions in your stack have the new protocol enabled, and the config to opt in (`group.protocol`) — this changed across the 3.7→4.x line.
- [ ] the default `partition.assignment.strategy` for your client version — it changed in the 3.x line to a list that includes the cooperative assignor for upgrade purposes. - *"How do you migrate from eager to cooperative safely?"* — it's a documented two-rolling-restart upgrade (first deploy adds the cooperative assignor to the list while keeping the old one, second removes the old one). You cannot flip it in one deploy. **Verify** the exact procedure in the docs for your version before doing it. - *"More consumers than partitions?"* — the extras sit idle. Not an error, just wasted pods; a legitimate pattern for fast failover. - *"Why does `enable.auto.commit=true` lose data?"* — it commits on a timer based on what `poll()` *returned*, not what you *processed*. A crash between the commit and the processing loses those records. It can also, less famously, duplicate. Manual ack after the durable write is the only honest option.
- [ ] the `max.in.flight.requests.per.connection` limit under idempotence (I believe 5), the Kafka 3.0 default changes, and producer-state expiry config names.
- [ ] accepted `processing.guarantee` values and `transaction.timeout.ms` defaults in your version.
- [ ] `DefaultErrorHandler`'s default `BackOff`, and `@RetryableTopic` attribute names/defaults in your Spring Kafka version (this annotation has evolved).
- [ ] default values for `linger.ms`, `batch.size`, `fetch.min.bytes`, `fetch.max.wait.ms`, `max.poll.records`, and which compression codecs your broker version supports.
- [ ] default `delete.retention.ms`, `min.cleanable.dirty.ratio`, `segment.ms` for your broker.
- [ ] the default compatibility mode of your registry deployment (it is configurable globally and per subject), and the exact Gradle plugin/task names.
- [ ] `TimeWindows.ofSizeAndGrace` / `ofSizeWithNoGrace` replaced the older `TimeWindows.of(...)` — confirm the API for your Kafka Streams version, and the default `num.standby.replicas` (I believe 0).
- [ ] `SessionWindows.ofInactivityGapAndGrace` and `TimeWindows.ofSizeAndGrace` API names for your version (the older `.of(...)`/`.grace(...)` forms are deprecated), and the default `max.task.idle.ms`.

## 05-resilience4j.md

- [ ] the `CircuitBreaker.State` enum's full member list (I believe it also includes `METRICS_ONLY`), and every default value quoted anywhere in this file.
- [ ] defaults for `slidingWindowSize`, `minimumNumberOfCalls`, `failureRateThreshold`, `slowCallRateThreshold`, `slowCallDurationThreshold`, `waitDurationInOpenState`, `permittedNumberOfCallsInHalfOpenState` and `automaticTransitionFromOpenToHalfOpenEnabled` in the Resilience4j version you pin.
- [ ] `IntervalFunction` factory method signatures, the `resilience4j.retry.calls` tag values, and whether your R4j version has any retry-budget feature (I don't believe it does).
- [ ] R4j RateLimiter defaults (`limitForPeriod`, `limitRefreshPeriod` — I recall the refresh default being an unusually small value like 500ns, `timeoutDuration`), and Bucket4j's current Redis/`ProxyManager` API names.
- [ ] `ThreadPoolBulkhead` defaults (`coreThreadPoolSize`, `maxThreadPoolSize`, `queueCapacity`, `keepAliveDuration`) and `SemaphoreBulkhead` defaults (`maxConcurrentCalls`, `maxWaitDuration`), plus current reactive-support status for `ThreadPoolBulkhead`.
- [ ] `cancelRunningFuture` default, and the AWS SDK v2 override-configuration method names (`apiCallTimeout` / `apiCallAttemptTimeout` / `retryPolicy` vs `retryStrategy` — this API changed).
- [ ] the exact Micrometer metric names and tag values for your R4j + Spring Boot version (the Prometheus-rendered names differ from the Micrometer names by the usual `.`→`_` mangling).
- [ ] the default numeric aspect orders (`retryAspectOrder`, `circuitBreakerAspectOrder`, `rateLimiterAspectOrder`, `timeLimiterAspectOrder`, `bulkheadAspectOrder`) and which direction the number means "outermost" — I am confident about the documented *nesting*, not the numbers.
- [ ] `ThreadPoolBulkhead` reactive support status, annotation support for reactive return types, and current virtual-thread pinning behaviour for your JDK.

## 06-data-layer.md

- [ ] the Hibernate 6 parameter-logging category (`org.hibernate.orm.jdbc.bind`) differs from Hibernate 5's (`org.hibernate.type.descriptor.sql`). Confirm against the version you pin.
- [ ] the exact `Statistics` accessor names (`getPrepareStatementCount()`, `getQueryExecutionCount()`, `getEntityLoadCount()`) against your Hibernate version before quoting them in code.
- [ ] the Hibernate 6 default-distinct behaviour and the exact `MultipleBagFetchException` conditions against your pinned version.
- [ ] `JdbcClient` exists from Spring Framework 6.1 (Boot 3.2). On an older version, use `NamedParameterJdbcTemplate`. Also confirm which window functions your Hibernate version supports in HQL before claiming JPQL cannot do it.
- [ ] `NESTED` support for `JpaTransactionManager` in your Spring version, and savepoint support in your JDBC driver.
- [ ] all HikariCP defaults above, and the `spring.threads.virtual.enabled` property name, for the exact versions you pin. Also verify the current state of `synchronized`-related pinning in your HikariCP and JDBC driver versions — this has been changing.
- [ ] the PG-11 fast-default behaviour, `cleanDisabled` defaults, and how your Flyway version marks a migration as non-transactional for `CREATE INDEX CONCURRENTLY`.
- [ ] DynamoDB limits (400 KB item, 1 MB page, 25-item batch) and the current adaptive-capacity behaviour against the AWS docs — these have changed over time.

## 07-cdc-outbox-sse.md

- [ ] the default `plugin.name` for your Debezium Postgres connector version (it has moved between `decoderbufs` and `pgoutput`), PG 16's failover-slot behaviour, and `max_slot_wal_keep_size` semantics, all in current docs.
- [ ] every connector option name above, the snapshot-mode names, the Outbox Event Router config keys, and Connect's exactly-once source support — all have changed across Debezium/Connect versions. Read the docs for the exact versions you pin before quoting them.

## 08-messaging-rabbitmq-kafka-sqs.md

- [ ] `publisher-confirm-type` values, `setReturnsCallback` vs the deprecated `setReturnCallback`, and fsync semantics per queue type in your RabbitMQ version.
- [ ] the `default-requeue-rejected` default, and `ackMode` attribute support on `@RabbitListener` in your Spring AMQP version.
- [ ] Spring AMQP's default prefetch value for your version (it has been a fairly large number — in the hundreds — for some time; do not assume 1).
- [ ] `x-death` header structure and field names, and the current status/limits of the delayed message exchange plugin.
- [ ] the quorum-queue feature matrix (per-message TTL, priorities), `x-delivery-limit` behaviour, and classic-queue lazy defaults in your RabbitMQ version.
- [ ] `DefaultJackson2JavaTypeMapper` API names and `setTrustedPackages` behaviour in your Spring AMQP version.
- [ ] the `spring.rabbitmq.listener.simple.retry.*` property names and `RepublishMessageRecoverer` method names for your Spring Boot version.
- [ ] the default memory/disk watermark values and the exact `x-overflow` policy names for your version.
- [ ] every AWS number in this section — SQS visibility timeout defaults and maxima, message size limit, retention limits, delay maximum, FIFO throughput limits (standard and high-throughput mode), Kinesis per-shard quotas, enhanced fan-out limits, and Kinesis retention maxima. These are documented service quotas and they change; I have quoted the *mechanisms* confidently and the *numbers* only as "documented limits — check them".

## 09-microservices-migration.md

- [ ] the saga concept comes from a 1987 paper by Garcia-Molina and Salem; check the exact title and venue before citing it in an interview.

## 10-grpc.md

- [ ] the exact unknown-field-retention behaviour by proto3 version, and the one-byte tag range, against current Protobuf docs before you quote them.
- [ ] the `buf` CLI's current command names and whether it's still the standard tool.
- [ ] the Spring gRPC integration's annotation and starter names — there have been both a community `grpc-spring-boot-starter` and newer official Spring support; confirm which you're using.
- [ ] the exact error-details types (`google.rpc.RetryInfo`, `ErrorInfo`) and the canonical gRPC↔HTTP code mapping table.
- [ ] `ServerCallStreamObserver.disableAutoRequest()`, `isReady()`, `setOnReadyHandler()` method names against your grpc-java version — this API has changed over time (an older `disableAutoInboundFlowControl()` existed).
- [ ] the gRPC service-config JSON schema (`retryPolicy`, `hedgingPolicy` field names, `retryableStatusCodes`) and whether retries are enabled by default in your grpc-java version.
- [ ] grpc-java's test helpers — `GrpcCleanupRule` is a JUnit 4 `TestRule`; confirm the current recommended JUnit 5 approach (typically managing `InProcessServerBuilder`/`InProcessChannelBuilder` lifecycles yourself) before relying on a specific class name.

## 11-observability-and-slos.md

- [ ] Micrometer builder method names (`publishPercentileHistogram`, `serviceLevelObjectives`) against the Micrometer version you pin — the API is stable but check before quoting it.
- [ ] the `management.metrics.distribution.*` property paths and whether your Boot version has moved any of them under `management.observations.*`. Also verify native-histogram support status.
- [ ] the `management.endpoint.health.group.*` and `probes.enabled` property paths for your Boot version, and whether Boot auto-enables probes when it detects Kubernetes (it does detect the environment, but confirm the exact condition).
- [ ] built-in Boot structured-logging property names and version; `logstash-logback-encoder` coordinates.
- [ ] OTel semantic convention attribute names (several were renamed at stabilization), the propagator config key, and `Context.taskWrapping`.
- [ ] `setObservationEnabled` availability in your Spring Kafka version; OTel agent coverage for Kafka Streams; `messaging.*` semantic convention attribute names.
- [ ] Collector `tail_sampling` policy type names and the `loadbalancing` exporter config — Collector config schema changes between releases. Verify `OTEL_TRACES_SAMPLER` values too.
- [ ] the exact burn-rate factors and window pairings in the Google SRE Workbook. Also verify `promtool test rules` syntax for your Prometheus version.

## 12-aws-and-terraform.md

- [ ] all limits above; init billing; `/tmp` sizing options.
- [ ] SnapStart runtime/architecture support and compatibility constraints; CRaC API coordinates; Spring Boot's CRaC support version; `-XX:TieredStopAtLevel` effect on your workload.
- [ ] ESM parameter names, SQS scaling behaviour, retry counts for async push sources.
- [ ] SQS FIFO dedup window; Powertools idempotency module name/API.
- [ ] NAT gateway pricing components; EKS VPC CNI IP allocation behaviour and prefix delegation.
- [ ] current Lambda-in-VPC cold-start behaviour; interface endpoint pricing; the exact service names for endpoints in your region.
- [ ] every quota in this answer. Kinesis per-shard read/write limits, `GetRecords` call rate, SQS FIFO throughput, retention maximums, MSK Serverless limits.
- [ ] per-partition throughput limits, item size limit, adaptive capacity behaviour.
- [ ] transaction item limit, capacity-mode spike behaviour, GSI throttling semantics, Streams retention.
- [ ] per-prefix request rates, storage class minimum sizes/durations, Express One Zone / S3 Tables availability.
- [ ] MM2 checkpoint/offset-translation behaviour for your Kafka version; MSK Replicator as the managed alternative; DynamoDB global table replication lag SLAs (if any).
- [ ] S3-native state locking availability and version; `import`/`moved`/`removed` block support; current AWS provider major version.
- [ ] `s3_use_path_style` vs the older `s3_force_path_style` argument name for your AWS provider version; moto's current service coverage; whether moto Lambda works on Apple Silicon.

## 13-kubernetes-for-stream-apps.md

- [ ] gRPC Java server `maxConnectionAge`/`maxConnectionAgeGrace` builder names; Gateway API gRPC route support.
- [ ] `management.endpoint.health.probes.enabled` and group property paths; Boot's Kubernetes auto-detection condition.
- [ ] `minDomains` availability; whether your cluster version's default `matchLabelKeys`/spread behaviour differs.
- [ ] default `MaxRAMPercentage`; `ActiveProcessorCount` rounding for fractional CPU limits; cgroup v2 support in your JDK; the RocksDB memory-bounding API in your Kafka Streams version.
- [ ] the KEDA Kafka scaler's trigger metadata field names, `apiVersion`, and whether `lagThreshold` is per-replica or total. The plan itself flags this ("verify KEDA Kafka scaler config").
- [ ] Kafka Streams assignor config names (`acceptable.recovery.lag`, `max.warmup.replicas`, `probing.rebalance.interval.ms`), the default assignor in your version, and `group.instance.id` semantics.
- [ ] EKS Pod Identity vs IRSA availability; Spring Boot's exact property precedence list for your version.
- [ ] the `-Djarmode=layertools` invocation and `JarLauncher` class name for your Boot version — both changed in recent releases. Also verify the distroless Java 21 image tag.

## 14-testing.md

- [ ] JUnit 5's parallel-execution property names (`junit.jupiter.execution.parallel.*`) against current docs before quoting them.
- [ ] `@MockitoBean` replaced `@MockBean` in recent Spring Framework/Boot versions — confirm which annotation your pinned version wants. Also confirm the context-cache logging category name.
- [ ] `@ServiceConnection` (Boot 3.1+) support for each container type, the `KafkaContainer` class name and image coordinates for your Testcontainers version, and the reuse flag (`testcontainers.reuse.enable`).
- [ ] `TopologyTestDriver`'s current API (`createInputTopic`/`createOutputTopic` replaced older `pipeInput`/`readOutput` forms) for your Kafka Streams version.
- [ ] jcstress usage and current coordinates before recommending it as a build dependency.
- [ ] Gatling's current Java DSL injection-profile API names (`constantUsersPerSec`, `rampUsersPerSec` etc.) before quoting them.
- [ ] PIT (pitest) as the standard JVM mutation-testing tool and its current Gradle plugin coordinates.
- [ ] JUnit 5's test-order configuration property names.

## 15-jvm-performance.md

- [ ] default `-Xss` and the compressed-oops threshold vary by platform and JDK build — confirm with `java -XX:+PrintFlagsFinal -version | grep -i -E 'ThreadStackSize|UseCompressedOops'` on your actual image.
- [ ] biased-locking status, and whether `-XX:+EliminateLocks` is still a separate flag, differ across JDK versions. Check `-XX:+PrintFlagsFinal` on your JDK.
- [ ] generational ZGC's flag name and default status changed across JDK 21 → 24. Check `java -XX:+PrintFlagsFinal -version | grep -i ZGenerational` and the release notes for your exact JDK before you name a flag in an interview. Shenandoah is not present in every vendor's build.
- [ ] flag availability and defaults are JDK- and vendor-specific. Check every flag above with `java -XX:+PrintFlagsFinal -version` on your actual base image before putting it in a Dockerfile.
- [ ] unified-logging tag names and decorator syntax (`-Xlog:gc*:file=...:time,uptime:filecount=...`) are stable since JDK 9 but worth confirming with `java -Xlog:help` on your JDK.
- [ ] compile-threshold flag names and defaults differ by JDK. Do not quote a specific threshold number; say "thresholds are tunable and JDK-specific" and move on.
- [ ] JMH annotation names above are stable, but confirm the current Gradle/Maven plugin coordinates in the JMH docs for the version you pin.
- [ ] `ScopedValue` preview/final status and its exact API shape (`where(...).run(...)` vs `callWhere`) changed across JDK releases — confirm against your JDK's javadoc before using it.
- [ ] async-profiler's CLI changed between the 2.x `profiler.sh` script and the 3.x `asprof` binary, and event names/options differ. Run `asprof --help` (or `profiler.sh --help`) on the version you actually ship and confirm `-e wall`, `-e lock`, `-e alloc` and `-t` before quoting them.
- [ ] `jmap` is deprecated in favour of `jcmd` on modern JDKs, and a JRE-only container image (Jib/Buildpacks default) may not include either. Confirm the tooling is present in your Phase 11 image *before* you need it at 2am.
- [ ] cgroup v1 vs v2 file paths, `UseContainerSupport` default, and `MinRAMPercentage` thresholds — confirm on your actual base image and JDK.

## 18-ai-tooling.md

- [ ] line. Check it against >   current documentation before you say it out loud. > - If an interviewer asks something concrete about the current state of the ecosystem and you don't >   know, say *"I'd check the current spec — what I know is the shape of it, which is ___."* That is a >   strong answer in this topic specifically, because everyone knows it moves.
- [ ] the protocol's current specification version, its primitive names, the transport mechanisms, and any SDK or Spring integration's artifact coordinates and annotation names. All of these have changed since the protocol's introduction. Check the official specification and the SDK's own docs before writing code or quoting details.
- [ ] guidance on injection defences is evolving, and client applications increasingly implement their own mitigations and approval flows. Check current guidance from the protocol's specification and your client's documentation rather than relying on this summary.
- [ ] do not state specific latency figures, prices, context limits or rate limits — all are vendor- and time-specific, and all change. Say "order of seconds" and "cost scales with tokens", and check current documentation for anything precise.
- [ ] the Spring AI MCP server starter's artifact coordinates, annotation names and version compatibility; the client-side registration file format and its schema; and whether the transport you choose is still the recommended one. All of these are version-specific and all have changed. Check the current documentation before writing code, and before describing implementation details in an interview.

---

**Total: 118 items.**
