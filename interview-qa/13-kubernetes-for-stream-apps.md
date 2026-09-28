# 13 — Kubernetes for Stream Applications: Probes, Drain, Scaling, and Stateful Streams

**Why this matters for an Event Hub role:** running a stateless REST service on Kubernetes is
commodity knowledge. Running a **stateful stream processor** on it is not, and that's what this team
does. The questions that separate levels here are: *"what happens between SIGTERM and the process
exiting, and where exactly does an in-flight request get dropped?"*, *"why is your JVM getting
OOMKilled when the heap is fine?"*, and *"you scale your Kafka Streams app from 3 pods to 6 — walk me
through what actually happens."* Those have precise answers, and most candidates don't have them.

You already have some Kubernetes exposure, so Q1–Q2 are a fast refresher and everything from Q3
onward is the senior material.

**Scenario anchor — BS-01** supplies the drain requirement: a pod killed mid-batch loses ~2k buffered
events and leaves a half-written S3 file. REQ-LC-01 specifies the exact stop order, and the NFR is
**0 events lost across 20 consecutive rolling restarts under load, with shutdown completing in < 30s
inside a `terminationGracePeriodSeconds: 45`**. **BS-10 REQ-SL-05** supplies the availability
requirement: `maxUnavailable: 0`, PodDisruptionBudget `minAvailable: 2`, readiness gates, topology
spread, HPA/KEDA headroom — each mapped to the failure it absorbs. Phase 11.

---

## Q1. Pods, Deployments, Services, Ingress — the fast version.

**30-second answer:** A **Pod** is the scheduling unit: one or more containers sharing a network
namespace and IP, plus volumes. A **Deployment** manages a **ReplicaSet** which manages Pods, and gives
you declarative rollouts and rollbacks. A **Service** is a stable virtual IP and DNS name in front of a
set of Pods selected by label, with kube-proxy (or eBPF) load balancing at **L4**. An **Ingress** (or
the newer Gateway API) is L7 HTTP routing into the cluster — host/path rules, TLS termination —
implemented by a controller like NGINX or an ALB controller. The piece that actually matters
operationally is the one nobody names: the **EndpointSlice**, which is the live list of ready Pod IPs
behind a Service. Every "traffic went to a dying pod" story is a story about EndpointSlice propagation
lag.

**Deeper:**

```
        Ingress / Gateway  (L7: host, path, TLS)
                │
        Service (ClusterIP)  ──▶ EndpointSlice: [10.1.2.3, 10.1.2.4, 10.1.2.5]
                │                       ▲
         kube-proxy / CNI (L4)          │ maintained by the endpoint controller,
                │                       │ driven by READINESS
          ┌─────┴─────┬──────────┐      │
        Pod A       Pod B      Pod C ───┘
          │
      ReplicaSet ◀── Deployment (rollout strategy, revision history)
```

| Service type | What it does | Use |
|---|---|---|
| `ClusterIP` | virtual IP, in-cluster only | default, service-to-service |
| `NodePort` | opens a port on every node | rarely directly; the building block for LoadBalancer |
| `LoadBalancer` | provisions a cloud LB (NLB/ALB) | cluster ingress |
| `ExternalName` | a CNAME | pointing at an out-of-cluster dependency |
| **Headless** (`clusterIP: None`) | no VIP; DNS returns **all** pod IPs | **StatefulSets**, and any client that wants to do its own load balancing — including gRPC (Q2) and Kafka clients |

`ConcurrentKafkaListenerContainerFactory` consumers don't need a Service at all — they dial out to the
brokers. That's worth noticing: in StreamPulse only `ingest-gateway` and `query-api` are
Service-fronted; `stream-processor` and `sink-writer` have **no inbound traffic**, which changes their
probe and drain design completely (Q3, Q5).

**Follow-ups they'll ask:**
- *"What's the difference between Endpoints and EndpointSlice?"* — EndpointSlice is the scalable
  replacement: Endpoints was a single object listing every address, so a 5,000-pod Service produced
  one enormous object that every node watched. Slices shard it.
- *"Deployment vs StatefulSet vs DaemonSet vs Job?"* — Deployment for interchangeable stateless pods;
  StatefulSet for stable identity, ordered rollout and per-pod persistent volumes (Q11); DaemonSet for
  one-per-node agents (the OTel collector, a log shipper); Job/CronJob for run-to-completion work.
- *"Ingress or Gateway API?"* — Gateway API is the successor with a proper role split (infrastructure
  vs route owner) and first-class support for non-HTTP protocols. **Verify its current maturity and
  controller support** before choosing it for a new cluster.

**Where in StreamPulse:** Phase 11 — `kind` cluster + Helm charts per service.

---

## Q2. Why does gRPC load balance badly in Kubernetes, and how do you fix it?

**30-second answer:** Because a Kubernetes `ClusterIP` Service balances at **L4** — it picks a backend
pod when the **TCP connection** is established, and every subsequent packet on that connection goes to
the same pod. HTTP/1.1 clients open and close connections constantly, so L4 balancing works by
accident. gRPC runs over **HTTP/2 with long-lived connections and request multiplexing**: one
connection carries thousands of RPCs, so all of them land on one pod. The result is that a client that
connected before you scaled up never sees the new pods, and load is wildly uneven. The fixes are
**client-side load balancing** against a headless Service (the client resolves all pod IPs and
round-robins RPCs across its own connections), an **L7 proxy** that understands HTTP/2 streams
(Envoy/service mesh, or an ingress in gRPC mode), or — the cheap mitigation — **max connection age** on
the server so connections are periodically recycled and rebalanced.

**Deeper:**

```
❌ L4 (ClusterIP)                         ✅ Client-side LB (headless Service)
 client ══════ one HTTP/2 conn ═══▶ Pod A   client ─ conn ─▶ Pod A
        (all 10,000 RPCs)            Pod B          ─ conn ─▶ Pod B   RPCs round-robin
                                     Pod C          ─ conn ─▶ Pod C   across all conns
```

| Approach | How | Cost |
|---|---|---|
| Client-side LB | headless Service + gRPC's `dns:///` resolver with `round_robin` | client config; clients must re-resolve on scale changes |
| Service mesh / Envoy sidecar | L7 proxy balances per-**request** | mesh operational overhead |
| Ingress in gRPC mode | terminate HTTP/2 at an L7 ingress | another hop; the ingress becomes the bottleneck |
| `MAX_CONNECTION_AGE` on the server | force periodic reconnect so clients redistribute | still uneven between recycles; simplest to ship |

The `MAX_CONNECTION_AGE` option is worth naming because it's the pragmatic one: the gRPC server closes
connections after N minutes (with a grace period so in-flight RPCs finish), the client reconnects, and
DNS gives it a fresh pod. It doesn't give you per-request balancing, but it bounds how stale a
client's view can be — and it costs one config line. **Verify the exact server builder option names**
for the Java gRPC version you use.

This also interacts with **drain** (Q5): a long-lived HTTP/2 connection doesn't notice that your pod
was removed from the EndpointSlice, because it's already connected. Graceful shutdown for a gRPC
server means sending **GOAWAY** so the client migrates to another pod, then waiting for in-flight RPCs
— `Server.shutdown()` followed by `awaitTermination(timeout)`, wired into the `SmartLifecycle` phase
that stops connectors.

**Code:**

```java
/**
 * BS-09 REQ-GR-07 + BS-01 REQ-LC-01: the gRPC connector is the HIGHEST SmartLifecycle
 * phase, so it stops FIRST — refuse new streams before anything downstream stops.
 */
@Component
class GrpcConnectorLifecycle implements SmartLifecycle {

    private final Server server;

    @Override public int getPhase() { return 300; }   // same phase as the HTTP connector

    @Override
    public void stop(Runnable callback) {
        // shutdown() sends GOAWAY: clients stop opening new streams on this connection
        // and reconnect elsewhere. In-flight RPCs are allowed to finish.
        server.shutdown();
        try {
            if (!server.awaitTermination(20, TimeUnit.SECONDS)) {
                server.shutdownNow();     // hard cancel anything still running
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            server.shutdownNow();
        } finally {
            callback.run();
        }
    }
}
```

```yaml
# Headless Service so gRPC clients can do their own load balancing.
apiVersion: v1
kind: Service
metadata: { name: ingest-grpc-headless }
spec:
  clusterIP: None            # ← DNS returns ALL pod IPs, not one VIP
  selector: { app: ingest-gateway }
  ports: [{ name: grpc, port: 9090 }]
```

**Follow-ups they'll ask:**
- *"Does this affect anything other than gRPC?"* — yes: any long-lived connection. Database connection
  pools, Kafka client connections, HTTP/2 or keep-alive-heavy REST clients, and WebSocket/SSE. The SSE
  path in StreamPulse (Phase 5) has exactly the same property — an SSE client holds one connection for
  hours, so a scale-up doesn't rebalance existing subscribers.
- *"How do clients notice a scale-up with a headless Service?"* — they re-resolve DNS periodically. The
  DNS TTL and the client's re-resolution policy put a floor on how fast new pods get traffic.
- *"Is a service mesh worth it just for this?"* — usually no. A mesh is worth it for mTLS, uniform
  observability and traffic policy across many services; solving gRPC balancing alone doesn't justify
  the operational cost.

**Where in StreamPulse:** BS-09 / REQ-GR-07 ("K8s note: gRPC needs L7 load balancing... documented,
with a client-side LB / service mesh option"); Phase 2 and Phase 11.

**Verify:** gRPC Java server `maxConnectionAge`/`maxConnectionAgeGrace` builder names; Gateway API gRPC
route support.

---

## Q3. Liveness, readiness, startup probes — what's the difference and what's the classic mistake?

**30-second answer:** **Liveness** answers "should this container be **restarted**?" — failure kills
and restarts it. **Readiness** answers "should this pod receive **traffic**?" — failure removes it from
the Service's EndpointSlice but leaves it running. **Startup** answers "has it finished booting yet?" —
it suspends the other two until it passes, which is how you give a slow-starting JVM a long boot window
without also giving it a 90-second liveness timeout forever. The classic, career-defining mistake is
pointing **liveness at a health endpoint that checks dependencies**. When Redis blips, every pod's
liveness fails simultaneously, Kubernetes restarts the entire fleet at once, the restarts hammer the
dependency that was already struggling, and a 30-second blip becomes a 20-minute outage. **Liveness
must never check anything outside the process.**

**Deeper:**

| | Liveness | Readiness | Startup |
|---|---|---|---|
| Failure action | **restart container** | remove from EndpointSlice | keep waiting; after `failureThreshold`, restart |
| Runs | continuously, after startup probe passes | continuously | only until it first succeeds |
| Should check | "is this JVM wedged?" — nothing external | dependencies I cannot serve without | boot progress |
| Should never check | **any network dependency** | — | — |
| Typical config | long `periodSeconds`, high `failureThreshold` | short period, low threshold | high `failureThreshold` × short `periodSeconds` |

The asymmetry to internalize: **a readiness failure is cheap and reversible; a liveness failure is
destructive.** So readiness can be twitchy and liveness must be extremely conservative. A reasonable
default is that liveness only fails if the process is genuinely unrecoverable — deadlocked, or the
event loop is not turning.

```
❌ CASCADING RESTART                          ✅ CORRECT
 Redis blips                                   Redis blips
   → /actuator/health returns DOWN               → readiness group DOWN
   → liveness fails on ALL pods                  → all pods removed from endpoints
   → K8s restarts ALL pods simultaneously        → pods keep running, keep their
   → cold JVMs + reconnect storm                   in-memory state and connections
   → Redis worse → liveness fails again          → Redis recovers
   → crash loop                                  → readiness passes → traffic resumes
```

**The "what about a real deadlock?" question.** If liveness checks nothing, how do you catch a wedged
JVM? Good liveness checks are *internal* liveness signals: a lightweight endpoint that the web layer
must be able to serve (which proves the thread pool isn't exhausted), or — for a consumer with no
inbound traffic — a **heartbeat freshness check**: the listener updates a timestamp on every poll, and
liveness fails only if that timestamp is older than several poll intervals. That's genuinely internal
and genuinely catches a stuck consumer.

**Probe timing arithmetic** is the other thing to get right. Time to detect a failure is roughly
`initialDelaySeconds + periodSeconds × failureThreshold`, and each probe can take up to
`timeoutSeconds`. For liveness, make that number comfortably longer than your worst legitimate GC pause
or your slowest normal operation — otherwise a 12-second ZGC pause under load (Phase 9) restarts a
perfectly healthy pod.

**Code:**

```yaml
# Phase 11 — sink-writer Deployment (BS-01, BS-10 REQ-SL-05)
containers:
  - name: sink-writer
    ports:
      - { name: http,       containerPort: 8080 }
      - { name: management, containerPort: 8081 }   # actuator on a separate port, not routed by Ingress

    # STARTUP: give the JVM + Spring context up to 5 minutes (30 × 10s) to boot,
    # without making liveness that lenient forever.
    startupProbe:
      httpGet: { path: /actuator/health/readiness, port: management }
      periodSeconds: 10
      failureThreshold: 30

    # LIVENESS: liveness group only — no Kafka, no Redis, no DynamoDB (see Q4).
    # Conservative: 3 × 20s = 60s of consecutive failure before a destructive restart.
    livenessProbe:
      httpGet: { path: /actuator/health/liveness, port: management }
      periodSeconds: 20
      timeoutSeconds: 3
      failureThreshold: 3

    # READINESS: twitchy on purpose. Cheap to fail, instantly reversible.
    readinessProbe:
      httpGet: { path: /actuator/health/readiness, port: management }
      periodSeconds: 5
      timeoutSeconds: 2
      failureThreshold: 2
      successThreshold: 1
```

**Follow-ups they'll ask:**
- *"Do you always need a liveness probe?"* — no, and this is a genuinely good senior take: if you have
  no specific unrecoverable-wedge failure mode you can detect, **omitting liveness is safer than a bad
  one**, because a container that crashes on a fatal error is restarted by the kubelet anyway. A
  liveness probe you haven't thought carefully about only adds a way to restart healthy pods.
- *"What about `exec` probes?"* — they fork a process on every probe on every pod; at scale that's real
  CPU. Prefer `httpGet` or `tcpSocket`.
- *"How do probes interact with a pod that has no inbound traffic?"* — `stream-processor` and
  `sink-writer` have no Service, so readiness doesn't gate traffic — but it still matters, because
  **readiness gates the rolling update**: the Deployment won't proceed to the next pod until the new
  one is Ready. That's what makes a bad config fail the rollout instead of silently breaking the
  pipeline (BS-01 REQ-LC-03).
- *"What's `terminationGracePeriodSeconds` got to do with probes?"* — see Q5; readiness is what starts
  the drain.

**Pitfalls / red flags:** liveness on `/actuator/health`; identical config for liveness and readiness;
a liveness timeout shorter than a legitimate GC pause.

**Where in StreamPulse:** BS-01 / REQ-LC-01; BS-10 / REQ-SL-05; Phase 11.

---

## Q4. How do you wire Spring Boot Actuator health groups to those probes?

**30-second answer:** Boot exposes `/actuator/health/liveness` and `/actuator/health/readiness` when
`management.endpoint.health.probes.enabled=true`, backed by two `ApplicationAvailability` states —
`LivenessState` (CORRECT/BROKEN) and `ReadinessState` (ACCEPTING_TRAFFIC/REFUSING_TRAFFIC). The
`liveness` group deliberately includes **only** `livenessState`, and the `readiness` group includes
`readinessState` plus the specific dependency indicators I genuinely cannot serve without. The part
that ties it to BS-01: Boot publishes `AvailabilityChangeEvent`, so my shutdown sequence can flip
readiness to `REFUSING_TRAFFIC` as the **first** thing it does — which is what removes the pod from the
EndpointSlice before anything else stops.

**Deeper:**

The decision of what goes in the readiness group is a design decision with consequences:

| Indicator | In readiness? | Why |
|---|---|---|
| `readinessState` | ✅ always | the state Boot manages for you |
| Kafka Streams state (`KStreams` running?) | ✅ for `stream-processor` | if the topology isn't running we produce nothing |
| Redis | ⚠️ depends | query-api: yes, it's the hot tier. ingest-gateway: **no** — BS-04 says degrade, not go unready |
| DynamoDB / S3 | ❌ no | BS-04 REQ-RS-02 spools to disk on failure; going unready would make a degradable failure total |
| Partner webhook | ❌ never | a third party's outage must never take us out of service |
| MongoDB (`mongo` optional profile) | ❌ no | optional feature, must not gate the whole service |

The pattern to state out loud: **include a dependency in readiness only if the correct response to its
failure is "stop sending me traffic".** If the correct response is "degrade" or "fall back", it belongs
in a circuit breaker (BS-04), not a probe. Most people over-include, and the result is a service that
takes itself out of rotation for something it could have survived.

**Code:**

```yaml
# application-k8s.yml — BS-01 REQ-LC-06 (profile-selected), BS-10 REQ-SL-05
management:
  server:
    port: 8081                       # separate port; Ingress doesn't route it
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,loggers   # `loggers` = runtime log level changes on-call
  endpoint:
    health:
      probes:
        enabled: true
      show-details: never            # leaks internal hostnames otherwise
      group:
        liveness:
          include: livenessState     # DELIBERATELY only this
        readiness:
          include: readinessState,kafkaStreams
```

```java
/**
 * BS-01 REQ-LC-01 step (1): the FIRST thing that stops is readiness.
 * Highest SmartLifecycle phase == stops first (see file 01, Q2).
 */
@Component
class ReadinessGate implements SmartLifecycle {

    private final ApplicationEventPublisher publisher;
    private final ApplicationContext context;
    private volatile boolean running;

    @Override public int getPhase() { return Integer.MAX_VALUE; }   // stops before everything
    @Override public boolean isRunning() { return running; }
    @Override public void start() { running = true; }

    @Override
    public void stop() {
        // /actuator/health/readiness now returns OUT_OF_SERVICE.
        // Kubernetes' next readiness probe removes us from the EndpointSlice.
        // NOTE: this is ASYNCHRONOUS — the endpoint isn't gone the instant we publish.
        // That race is exactly what the preStop hook covers (Q5).
        AvailabilityChangeEvent.publish(publisher, context, ReadinessState.REFUSING_TRAFFIC);
        running = false;
    }
}
```

```java
/**
 * A readiness indicator with a real opinion: the Kafka Streams topology must be RUNNING.
 * REBALANCING is deliberately treated as not-ready — during a rebalance we produce nothing,
 * so we shouldn't advertise readiness (Q11).
 */
@Component("kafkaStreams")
class KafkaStreamsHealthIndicator implements HealthIndicator {

    private final KafkaStreams streams;

    @Override
    public Health health() {
        KafkaStreams.State state = streams.state();
        return switch (state) {
            case RUNNING            -> Health.up().withDetail("state", state).build();
            case REBALANCING        -> Health.outOfService().withDetail("state", state).build();
            case ERROR, NOT_RUNNING -> Health.down().withDetail("state", state).build();
            default                 -> Health.unknown().withDetail("state", state).build();
        };
    }
}
```

**Follow-ups they'll ask:**
- *"Does Boot detect Kubernetes automatically?"* — Boot enables the probe endpoints when it detects a
  Kubernetes environment, and you can force it with the property. **Verify the exact detection
  condition** for your Boot version rather than relying on it — setting the property explicitly is the
  safer habit.
- *"How do you add your own liveness signal?"* — publish
  `AvailabilityChangeEvent.publish(ctx, LivenessState.BROKEN)` from a component that detects an
  unrecoverable internal state. Use it sparingly; it triggers a restart.
- *"Why put actuator on a separate port?"* — so the Ingress can't route `/actuator/**` from the public
  internet, and so probe traffic doesn't compete with request traffic on the same connector/thread pool.

**Pitfalls / red flags:** putting every `HealthIndicator` Boot auto-configures into readiness "for
completeness". That's how a Mongo outage in an optional profile takes down ingest.

**Where in StreamPulse:** BS-01 / REQ-LC-01, REQ-LC-06; Phase 11.

**Verify:** `management.endpoint.health.probes.enabled` and group property paths; Boot's Kubernetes
auto-detection condition.

---

## Q5. Walk me through graceful shutdown, end to end, and tell me where you can still drop a request.

**30-second answer:** The kubelet does two things when a pod is deleted, and they are
**asynchronous and concurrent**: it sends **SIGTERM** to the container, and it tells the endpoint
controller to remove the pod from the EndpointSlice. That propagation — endpoint controller → kube-proxy
on every node → iptables/IPVS rules, plus any cloud load balancer target group — takes seconds, and
during those seconds the pod is **already shutting down but still receiving traffic**. That is the race,
and it is where requests get dropped. The fix is a **preStop hook that sleeps** — the container keeps
serving normally during the sleep while the endpoint removal propagates, and only then does SIGTERM
arrive. After that it's the app's job: `server.shutdown=graceful` finishes in-flight requests, then
the `SmartLifecycle` phases drain in order, and `terminationGracePeriodSeconds` must be larger than
`preStop sleep + the app's full drain budget`, or the kubelet SIGKILLs you mid-flush.

**Deeper:**

```
kubectl delete pod / rolling update
        │
        ├──────────────────────────────┐  (CONCURRENT — no ordering guarantee)
        ▼                              ▼
  preStop hook runs            endpoint controller removes pod from EndpointSlice
  (e.g. sleep 10)                      │
        │                              ├─▶ kube-proxy on node 1 updates iptables
        │  ← still serving! ─────────  ├─▶ kube-proxy on node 2 ...
        │                              └─▶ cloud LB deregisters target (SLOWEST — can be 10s+)
        ▼
  preStop finishes → SIGTERM sent to PID 1
        │
        ▼
  JVM shutdown hook → ApplicationContext.close()
        │
        ├─ SmartLifecycle.stop() in DESCENDING phase order  (BS-01 REQ-LC-01)
        │    MAX  readiness → REFUSING_TRAFFIC
        │    300  HTTP + gRPC connectors stop accepting  (graceful: finish in-flight)
        │    200  Kafka listeners stop, commit offsets
        │    100  S3 buffer flush (≤ 20s, REQ-LC-02)
        │      0  Kafka producer close(Duration)
        ├─ @PreDestroy on remaining singletons
        ▼
  process exits
        │      ...or, if terminationGracePeriodSeconds elapses first:
        ▼
  SIGKILL — nothing runs, buffers lost
```

### The budget arithmetic (BS-01's numbers)

```
preStop sleep                                   10s   ← endpoint propagation
+ graceful HTTP shutdown (spring.lifecycle...)  ≤ 25s  (in-flight requests + phases)
    of which S3 flush timeout                    20s   (REQ-LC-02)
──────────────────────────────────────────────────────
worst-case drain                                ~35s
terminationGracePeriodSeconds                    45s   ← MUST exceed the above
```

The plan targets "< 30s of drain inside a 45s grace period", which leaves headroom. The rule to state:
**`terminationGracePeriodSeconds` > preStop + app drain, with margin.** Getting this backwards means
SIGKILL lands in the middle of your S3 flush — the exact failure BS-01 exists to prevent.

### Why the preStop sleep is not a hack

It feels like one, and interviewers sometimes push on it. The honest framing: Kubernetes has no
mechanism to say "remove me from service, wait for confirmation, *then* signal me" — endpoint removal
is eventually consistent and the pod gets no completion callback. So a fixed delay is the only tool
available, and it's the documented, universal pattern. Flipping readiness to `REFUSING_TRAFFIC` in the
app (Q4) helps, but it's subject to the **same propagation delay** plus the readiness probe period —
so it's complementary, not a replacement.

### Where you can still drop a request

Being able to name the remaining holes is the senior part:

| Hole | Mitigation |
|---|---|
| Endpoint propagation slower than the preStop sleep (cloud LB deregistration is the worst offender) | lengthen the sleep; use a `readinessGate`/pod-readiness-gate integration with the LB controller where available |
| A long-lived connection (gRPC, SSE, WebSocket) that was already established | GOAWAY / SSE close event (Q2); the client must reconnect |
| SIGKILL because the grace period was too short | the arithmetic above |
| **Node failure** — no SIGTERM at all | you can't drain; this is why correctness rests on at-least-once + idempotency, not on a clean shutdown |
| A `kubectl delete pod --force --grace-period=0` | operator discipline |

That fourth row is the one to say out loud: **graceful shutdown is an optimization, not a correctness
mechanism.** The correctness mechanism is REQ-LC-02 — if the flush can't finish, *don't commit the
offsets*, so the events are redelivered. BS-01's design is deliberately belt-and-braces.

**Code:**

```yaml
# Phase 11 — sink-writer Deployment (BS-01 REQ-LC-01/02)
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 45          # > preStop(10) + drain(~25) + margin
      containers:
        - name: sink-writer
          lifecycle:
            preStop:
              exec:
                # Keep serving normally while the EndpointSlice change propagates.
                # `sleep` must exist in the image — one reason NOT to use a fully
                # distroless base for this container, or use the `:debug` variant (Q13).
                command: ["/bin/sh", "-c", "sleep 10"]
```

```yaml
# The matching application config — the two MUST be designed together.
server:
  shutdown: graceful                 # finish in-flight HTTP requests
spring:
  lifecycle:
    timeout-per-shutdown-phase: 25s  # per SmartLifecycle PHASE, not total
streampulse:
  shutdown:
    flush-timeout: 20s               # REQ-LC-02
```

**Follow-ups they'll ask:**
- *"Is `timeout-per-shutdown-phase` the total?"* — no, it's **per phase**, so with four phases the
  theoretical worst case is 4 × 25s. In practice phases finish fast, but size the grace period against
  the realistic worst case and know the distinction. **Verify the exact semantics for your Boot
  version.**
- *"What if `sleep` isn't in the image?"* — distroless images have no shell. Options: use the `:debug`
  variant, use an `httpGet` preStop against an endpoint that blocks, or (in newer Kubernetes) a
  `sleep` action in the lifecycle handler. **Verify the `sleep` lifecycle action's availability** in
  your cluster version.
- *"How do you test this?"* — BS-01's acceptance tests are exactly this: `ShutdownOrderIT` records the
  order of stop callbacks, and `RollingRestartChaosIT` does 20 rolling restarts under load and asserts
  0 lost events by comparing accepted-event count to distinct event IDs landed in S3 + DynamoDB.
- *"What about the Kafka consumer specifically?"* — stopping the listener container triggers the
  consumer to leave the group, which starts a **rebalance** on the remaining members. With
  `CooperativeStickyAssignor` (or Streams' cooperative protocol) that's incremental rather than a
  stop-the-world reassignment — which is why cooperative rebalancing matters for rolling restarts
  (Q11).

**Pitfalls / red flags:** knowing `terminationGracePeriodSeconds` exists but not knowing about the
endpoint-removal race; no preStop hook; a grace period shorter than the app's own shutdown timeout.

**Where in StreamPulse:** BS-01 end to end / REQ-LC-01, REQ-LC-02; `ShutdownOrderIT`,
`RollingRestartChaosIT`; Phase 11.

---

## Q6. How do you configure a rolling update so a deploy doesn't dent availability?

**30-second answer:** The two dials are `maxSurge` (how many pods above the desired count you may
create) and `maxUnavailable` (how many below the desired count you may drop). For BS-10's 99.98% target
I set **`maxUnavailable: 0`** with `maxSurge: 1` — so the new pod is created and becomes **Ready**
before an old one is terminated, and capacity never dips below the desired replica count. The cost is
that the rollout is slower and needs headroom for one extra pod. The mechanism that makes this
meaningful is the readiness probe: "Ready" must genuinely mean "can serve", which is why a validated
fail-fast config (BS-01 REQ-LC-03) matters — a pod that boots successfully but is misconfigured would
otherwise be counted as available and the rollout would happily continue destroying good pods.

**Deeper:**

| Setting | `maxSurge: 25%, maxUnavailable: 25%` (default) | `maxSurge: 1, maxUnavailable: 0` |
|---|---|---|
| Capacity during rollout | dips below desired | never dips |
| Extra resources needed | some | one pod's worth |
| Speed | faster | slower (strictly one at a time, unless surge > 1) |
| Fit | most services | **SLA-bearing services** |

Related settings that matter:

| Field | What it does | Why |
|---|---|---|
| `minReadySeconds` | a pod must stay Ready for N seconds before counting as available | stops a pod that passes readiness then immediately crashes from being counted; forces the rollout to notice |
| `progressDeadlineSeconds` | fail the rollout if it doesn't progress in N seconds | so a broken deploy *stops* instead of hanging forever |
| `revisionHistoryLimit` | how many old ReplicaSets to keep | `kubectl rollout undo` needs at least one |

**The rollout is not a canary.** `maxUnavailable: 0` protects capacity; it does not protect you from a
bad *version*, because a bad version that passes readiness will be rolled out to 100% of pods. For that
you need `minReadySeconds` plus monitoring, or a real progressive delivery tool (Argo Rollouts, Flagger)
that watches your SLI between steps. That's the honest limit of the built-in Deployment strategy and
worth stating, especially since BS-10 has an error-budget gate in CI (REQ-SL-04) which is the
lighter-weight version of the same idea.

For **`stream-processor`**, the calculus is completely different and that's the interesting part: each
pod restart triggers a **rebalance** and potentially a **state store restore** (Q11). So
`maxUnavailable: 0, maxSurge: 1` is *especially* right there — but the real lever is minimizing
rebalances, which is a Kafka config problem (static membership, cooperative assignors), not a
Kubernetes one.

**Code:**

```yaml
# Phase 11 — ingest-gateway (BS-10 REQ-SL-05: "maxUnavailable: 0 rolling updates")
apiVersion: apps/v1
kind: Deployment
metadata: { name: ingest-gateway }
spec:
  replicas: 3
  minReadySeconds: 15              # Ready AND stable for 15s before counting as available
  progressDeadlineSeconds: 600     # fail the rollout rather than hanging
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1                  # create one extra pod first
      maxUnavailable: 0            # never drop below 3 available
```

**Follow-ups they'll ask:**
- *"Blue/green or canary instead?"* — blue/green (two full environments, switch traffic) gives instant
  rollback at double the cost; canary shifts a percentage and watches metrics, which is strictly better
  but needs a controller and a reliable SLI. Given BS-10 already defines per-tenant SLIs (file 11 Q12),
  a canary that gates on burn rate is the natural next step and a good thing to propose.
- *"How do you roll back?"* — `kubectl rollout undo`, or in GitOps terms revert the commit. The
  important part is that rollback must be *rehearsed*: an untested rollback path is discovered during
  the incident.
- *"Does a ConfigMap change trigger a rollout?"* — **no**, not by itself, and this catches people. A
  ConfigMap mounted as a volume updates in place (eventually) but the app won't reread it; a ConfigMap
  consumed as env vars doesn't change at all until the pod restarts. The standard fix is a checksum
  annotation on the pod template (Q12) so changing the ConfigMap changes the pod spec and triggers a
  rollout.
- *"What happens to in-flight work on the pods being replaced?"* — Q5's drain. The two designs are
  inseparable: `maxUnavailable: 0` guarantees capacity, the drain guarantees no loss.

**Pitfalls / red flags:** default rolling-update settings on a service with an SLA; no
`progressDeadlineSeconds`, so a broken rollout hangs; confusing a rolling update with a canary.

**Where in StreamPulse:** BS-10 / REQ-SL-05; BS-01 (the drain that makes it lossless); Phase 11.

---

## Q7. PodDisruptionBudgets, voluntary vs involuntary disruption, and topology spread.

**30-second answer:** A **PodDisruptionBudget** tells the eviction API the minimum number of pods that
must stay available, so a **voluntary** disruption — a node drain for a cluster upgrade, a cluster
autoscaler scale-down, a `kubectl drain` — is blocked or slowed rather than taking out your whole
service. The crucial limitation: a PDB **only constrains voluntary disruptions**. It does nothing about
**involuntary** ones — a node hardware failure, a kernel panic, an OOMKill, a spot instance
reclamation. For those you need **replicas across failure domains**, which is what
`topologySpreadConstraints` gives you: "spread my pods evenly across zones and nodes, and refuse to
schedule if the skew would exceed N". PDB and spread constraints solve different halves of the same
problem, and BS-10 REQ-SL-05 lists both for exactly that reason.

**Deeper:**

| | Voluntary disruption | Involuntary disruption |
|---|---|---|
| Examples | node drain, cluster upgrade, autoscaler scale-down, `kubectl delete pod`, deployment rollout | node crash, kernel panic, OOMKill, spot reclaim, network partition, AZ failure |
| Goes through eviction API? | ✅ yes | ❌ no |
| PDB helps? | ✅ yes | ❌ **no** |
| What helps | PDB | replica count + topology spread + anti-affinity |

**PDB sizing is a trap.** `minAvailable: 2` with `replicas: 2` means **no pod can ever be evicted**,
which blocks cluster upgrades indefinitely and makes you the team the platform group complains about.
BS-10's `minAvailable: 2` is correct only because the deployment runs 3+ replicas. State the rule:
**`minAvailable` must be strictly less than `replicas`**, or express it as `maxUnavailable: 1`, which
is self-adjusting as you scale.

`topologySpreadConstraints` fields:

| Field | Meaning |
|---|---|
| `maxSkew` | max difference in pod count between the most and least populated domain |
| `topologyKey` | the node label defining a domain — `topology.kubernetes.io/zone`, `kubernetes.io/hostname` |
| `whenUnsatisfiable` | `DoNotSchedule` (hard) or `ScheduleAnyway` (soft preference) |
| `labelSelector` | which pods to count |
| `minDomains` | require at least N domains to exist (**verify availability in your cluster version**) |

The practical pattern is **hard across zones, soft across nodes**: you genuinely want AZ spread
(`DoNotSchedule`), but insisting on one-pod-per-node can leave pods Pending when the cluster is tight —
and a Pending pod is worse for availability than two pods on one node.

**Code:**

```yaml
# Phase 11 — BS-10 REQ-SL-05 ("PodDisruptionBudget minAvailable: 2", "topology spread")
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: ingest-gateway }
spec:
  # With replicas: 3, this allows exactly ONE voluntary disruption at a time.
  # maxUnavailable: 1 would be equivalent here and self-adjusts if we scale to 10.
  minAvailable: 2
  selector:
    matchLabels: { app: ingest-gateway }
---
# In the Deployment pod template:
spec:
  topologySpreadConstraints:
    # HARD: never put more than a 1-pod imbalance across AZs. Survives an AZ loss.
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels: { app: ingest-gateway }
    # SOFT: prefer different nodes, but a Pending pod is worse than a co-located one.
    - maxSkew: 1
      topologyKey: kubernetes.io/hostname
      whenUnsatisfiable: ScheduleAnyway
      labelSelector:
        matchLabels: { app: ingest-gateway }
```

**Follow-ups they'll ask:**
- *"Anti-affinity vs topology spread?"* — `podAntiAffinity` is the older mechanism and is essentially
  binary ("don't co-locate"), while spread constraints express *degree* of imbalance and handle
  scale-out gracefully. Spread constraints are the modern choice; anti-affinity is still fine for a
  hard "never two of these on one node".
- *"What happens if a PDB blocks a drain?"* — the eviction API returns 429 and the drain retries. If it
  can never be satisfied (the `minAvailable == replicas` bug), the drain hangs forever and someone
  eventually force-deletes the pods, which defeats the point entirely.
- *"Does a PDB protect against the rolling update?"* — a Deployment rollout doesn't use the eviction
  API, so `maxUnavailable` on the Deployment is what governs there. PDB and rollout strategy are
  separate controls that must agree — this is a genuinely confusing point and worth being precise about.
- *"How does this interact with the cluster autoscaler?"* — the autoscaler respects PDBs when draining
  a node it wants to remove. A too-strict PDB means nodes never scale down and you pay for idle
  capacity.

**Pitfalls / red flags:** believing a PDB protects against node failure; `minAvailable` equal to
`replicas`; no zone spread on an SLA-bearing service.

**Where in StreamPulse:** BS-10 / REQ-SL-05; Phase 11 Helm charts.

**Verify:** `minDomains` availability; whether your cluster version's default
`matchLabelKeys`/spread behaviour differs.

---

## Q8. Resource requests vs limits — and why can CPU limits make things worse?

**30-second answer:** **Requests** are what the scheduler uses to place the pod and what the node
reserves — they're a *guarantee*. **Limits** are the hard ceiling enforced at runtime, and the two
resources behave completely differently at the ceiling. Exceeding a **memory** limit gets the container
**OOMKilled** — a hard stop. Exceeding a **CPU** limit gets you **throttled** by the kernel's CFS
scheduler — your threads are simply not scheduled for the rest of the 100ms period, which shows up as
mysterious latency spikes with normal-looking average CPU. That's why CPU limits are genuinely
controversial: a pod sitting at 40% average CPU can be throttled hard during short bursts, and for a
latency-sensitive service the throttling costs you more than the isolation buys. My position: **always
set memory requests == limits** (predictable, gets you the Guaranteed QoS class), set CPU requests
carefully, and set CPU limits only where you specifically need to bound a noisy neighbour — with
throttling monitored either way.

**Deeper:**

| | CPU | Memory |
|---|---|---|
| Compressible? | ✅ yes — throttled | ❌ no — killed |
| Over the limit | CFS throttling (threads descheduled) | **OOMKill**, exit code 137 |
| Over the request, under the limit | allowed to burst if the node has capacity | allowed |
| Metric to watch | `container_cpu_cfs_throttled_seconds_total` | `container_memory_working_set_bytes`, restart count + reason |

**QoS classes**, which decide who gets evicted when a node runs out of memory:

| Class | Condition | Eviction order |
|---|---|---|
| `Guaranteed` | requests == limits for **both** CPU and memory, on every container | evicted **last** |
| `Burstable` | requests set, limits higher or absent | middle |
| `BestEffort` | nothing set | evicted **first** |

For an SLA-bearing service you want `Guaranteed`. That means setting memory requests equal to limits —
which is fine, because a JVM's memory footprint is bounded anyway (Q9) and you should know it. For CPU
it means setting a limit, which conflicts with the throttling argument — so this is a real trade-off,
and the answer I'd give is: set CPU request = limit for `Guaranteed` **only if** you've sized the limit
from measured peak (Phase 9 data, which is exactly why the plan says "resource requests tuned from
Phase 9 data"), and otherwise prefer a generous request with no limit and rely on the scheduler.

**The CFS throttling mechanism**, because the detail is what makes the answer credible: the kernel
gives the cgroup a quota per 100ms period (`cpu.cfs_quota_us` / `cpu.cfs_period_us`). A limit of
`500m` means 50ms of CPU time per 100ms period. If your app has **4 runnable threads**, they can burn
that 50ms in 12.5ms of wall-clock time — and then the container is **frozen for 87.5ms**. Average
utilization reads as 50%; p99 latency is destroyed. This hits multi-threaded JVMs disproportionately
because GC, JIT compiler threads and your request threads all compete for the same quota.

```
CPU limit 500m, 4 busy threads:
 |████████|                                              |████████|
 0      12.5ms          ← THROTTLED, nothing runs →     100ms    112.5ms
 avg CPU looks like 50%.  p99 latency is +87ms.
```

**Code:**

```yaml
# Phase 11 — sized from Phase 9 Gatling data, not guessed (BS-10 REQ-SL-05 "HPA headroom")
resources:
  requests:
    cpu:    "1000m"      # scheduler reserves a full core
    memory: "2Gi"
  limits:
    # memory limit == request → no surprise OOMKill from a noisy neighbour,
    # and this is half of what Guaranteed QoS requires.
    memory: "2Gi"
    # CPU limit DELIBERATELY OMITTED for the latency-sensitive ingest path.
    # We accept Burstable QoS here and rely on requests + node headroom.
    # For sink-writer (batch, latency-tolerant) we DO set a CPU limit,
    # because bounding a noisy neighbour matters more there than p99.
```

```promql
# The alert that catches the problem CPU limits cause (Phase 8)
rate(container_cpu_cfs_throttled_seconds_total{pod=~"ingest-gateway.*"}[5m]) > 0.1
# "more than 10% of wall-clock time spent throttled" — if this fires,
# the limit is wrong, not the app.
```

**Follow-ups they'll ask:**
- *"How do you size the request?"* — from measurement: run the Phase 9 Gatling load, look at p95/p99
  container CPU and working-set memory, set the request near p95 usage and leave headroom. Vertical Pod
  Autoscaler in recommendation-only mode is a good source of data even if you never let it act.
- *"What's the risk of no CPU limit?"* — a runaway pod can starve co-located pods on the node. The
  mitigations are correct requests everywhere (so the scheduler doesn't overcommit), ResourceQuotas and
  LimitRanges per namespace, and monitoring. It's a real risk; I'd just rather take it than eat
  throttling on a latency-SLO service.
- *"How do requests relate to the HPA?"* — CPU-based HPA targets a **percentage of the request**, so
  an inflated request makes the HPA scale late and a tiny request makes it flap. Wrong requests break
  autoscaling silently.
- *"Ephemeral storage?"* — it has requests and limits too, and exceeding the limit evicts the pod.
  Relevant for StreamPulse: BS-04's local S3 spool (1 GB bounded) and Kafka Streams' RocksDB state both
  write to disk. If that's on `emptyDir` with no accounting, you can get evicted for it.

**Pitfalls / red flags:** copy-pasted resource blocks; CPU limit equal to request on a latency-sensitive
service with no throttling monitoring; not knowing the difference between compressible and
incompressible resources.

**Where in StreamPulse:** Phase 11 ("resource requests tuned from Phase 9 data"); BS-10 REQ-SL-05.

---

## Q9. Your Java pod keeps getting OOMKilled but the heap looks fine. What's happening?

**30-second answer:** **Heap is not the JVM's total memory.** The container's memory limit has to cover
heap *plus* metaspace, the code cache, thread stacks, direct/mapped byte buffers, GC data structures,
the JVM's own native allocations, and anything a native library allocates — for a Kafka/Netty/RocksDB
app the off-heap portion is substantial. So `-Xmx` set to the container limit is a guaranteed OOMKill.
The fix is to size the heap as a *percentage* — `-XX:MaxRAMPercentage`, which works because modern JVMs
are container-aware and read the cgroup limit rather than the host's total memory — and leave real
headroom. And the diagnostic that tells them apart: **OOMKilled is the kernel killing the process
(exit code 137, visible in `kubectl describe pod` as `Reason: OOMKilled`), while `OutOfMemoryError` is
the JVM failing to allocate within its own heap and throwing a Java exception you'll see in the logs.**
Different cause, different fix, and confusing them wastes hours.

**Deeper:**

### What's in a JVM's memory footprint

```
┌─ container memory limit (2 GiB) ──────────────────────────────┐
│ ┌─ JVM heap (-Xmx / MaxRAMPercentage) ──────────┐             │
│ │  young gen + old gen                           │  ~75%       │
│ └────────────────────────────────────────────────┘             │
│  Metaspace (class metadata — grows with framework size)        │
│  Code cache (JIT-compiled methods)                             │
│  Thread stacks (~1 MB × thread count — watch platform threads) │
│  Direct ByteBuffers (Netty, NIO, Kafka client) ← often LARGE   │
│  GC structures (card tables, remembered sets)                  │
│  RocksDB off-heap (Kafka Streams state stores) ← often HUGE    │
│  JVM native / JNI / malloc arenas                              │
└────────────────────────────────────────────────────────────────┘
```

For `stream-processor` the RocksDB row is the dominant one and it is **entirely off-heap**. RocksDB
block cache, write buffers and index/filter blocks are native memory, so a Kafka Streams pod can be
OOMKilled with a heap that's 30% used. Kafka Streams lets you cap it with a custom
`RocksDBConfigSetter` and a shared `WriteBufferManager`/block cache — **verify the current API names
and the recommended bounding approach**, because this is a place where the defaults are not safe in a
container.

### Container awareness

Modern JVMs (JDK 10+, with `-XX:+UseContainerSupport` on by default) read cgroup limits instead of host
memory. Two consequences:

- The **default** max heap is a fraction of the *container* limit (historically 25%), which is
  conservative and wastes memory — hence setting `MaxRAMPercentage` explicitly.
- `Runtime.availableProcessors()` reflects the **CPU limit**, not the node's cores. Which is good —
  thread pools and GC thread counts size correctly — but it also means that a CPU limit of `500m` may
  make the JVM see **1** processor, and things like `ForkJoinPool.commonPool()` and GC parallelism size
  down accordingly. You can override with `-XX:ActiveProcessorCount`. **Verify the rounding behaviour
  for fractional CPU limits in your JDK version** — it's a known source of surprise.

Also relevant: **cgroup v1 vs v2**. Older JDKs only understood v1; modern clusters use v2. Running an
old JVM on a v2 host means container awareness silently doesn't work and the JVM sizes against the
whole node. **Verify your JDK's cgroup v2 support** if you're on anything older than current LTS.

### Telling the two failures apart

| | OOMKilled | `OutOfMemoryError` |
|---|---|---|
| Who kills it | the **kernel** (cgroup OOM killer) | nobody — the **JVM throws** |
| Exit code | **137** (128 + SIGKILL) | whatever your handler/JVM does (often 1) |
| Where you see it | `kubectl describe pod` → `Last State: Terminated, Reason: OOMKilled` | application logs, stack trace |
| Heap dump produced? | ❌ **no** — SIGKILL runs nothing | ✅ yes, with `-XX:+HeapDumpOnOutOfMemoryError` |
| Usual cause | total footprint > limit; often **off-heap** | a genuine heap leak or undersized heap |
| Fix | raise the limit, or lower `MaxRAMPercentage`, or bound off-heap | fix the leak (heap dump), or raise `-Xmx` |

The "no heap dump on OOMKill" row is the operationally painful one: the most common production memory
failure produces the least diagnostic evidence. That's an argument for `NativeMemoryTracking` and for
monitoring `container_memory_working_set_bytes` against the limit *before* it kills you.

**Code:**

```yaml
# Phase 11 — JVM flags for a containerised Spring Boot service
env:
  - name: JAVA_TOOL_OPTIONS
    value: >-
      -XX:MaxRAMPercentage=70.0
      -XX:InitialRAMPercentage=70.0
      -XX:+HeapDumpOnOutOfMemoryError
      -XX:HeapDumpPath=/dumps
      -XX:+ExitOnOutOfMemoryError
      -XX:NativeMemoryTracking=summary
      -XX:+UseZGC
      -Xlog:gc*:file=/dev/stdout:time,uptime,level,tags
resources:
  requests: { memory: "2Gi", cpu: "1000m" }
  limits:   { memory: "2Gi" }
volumeMounts:
  # A dump volume, because a 1.4 GiB heap dump won't fit in ephemeral storage
  # and losing the dump means repeating the incident to get one.
  - { name: dumps, mountPath: /dumps }
```

Why each flag:

| Flag | Reason |
|---|---|
| `MaxRAMPercentage=70` | heap ≈ 1.4 GiB of the 2 GiB limit, leaving ~600 MiB for metaspace, stacks, direct buffers, GC and native. **Tune from measurement, not from 70 being a nice number.** |
| `InitialRAMPercentage` = same | pre-commits the heap so the JVM doesn't grow into a cgroup that's already near its limit; also avoids resize pauses |
| `ExitOnOutOfMemoryError` | a JVM limping along after an OOME is worse than a restart — fail fast, let Kubernetes restart it |
| `HeapDumpOnOutOfMemoryError` + path | the dump is the only way to find the leak (Phase 9's induced-leak lab) |
| `NativeMemoryTracking=summary` | `jcmd <pid> VM.native_memory summary` shows the off-heap breakdown — the tool for the OOMKill case |
| GC logging to stdout | so GC pauses land in Loki alongside everything else |

**Follow-ups they'll ask:**
- *"How do you find what's using off-heap memory?"* — `jcmd VM.native_memory summary` with NMT enabled,
  and for direct buffers the `BufferPool` MBean (Micrometer exposes `jvm.buffer.memory.used`). For a
  native leak outside the JVM's accounting (a JNI library), you're down to `pmap`/`jemalloc` profiling.
- *"Why `MaxRAMPercentage` rather than `-Xmx`?"* — one image runs at different memory limits across
  environments; a percentage adapts, an absolute value has to be templated per environment and will
  eventually be wrong.
- *"G1 or ZGC?"* — Phase 9 benchmarks both under the spike. Broadly: G1 is the balanced default with
  good throughput and pauses in the tens of milliseconds; ZGC targets sub-millisecond pauses at some
  throughput and footprint cost, which matters for a p99 latency SLO. The right answer is "I measured
  it on this workload" — which is exactly why the plan has that lab.
- *"Does the JVM release memory back to the OS?"* — with `ZGC`/`G1` and `-XX:+ShenandoahUncommit`-style
  uncommit behaviour, partially and eventually. Don't count on the working set shrinking quickly, which
  is another reason requests == limits for memory.

**Pitfalls / red flags:** `-Xmx` equal to the container limit; not knowing that OOMKilled produces no
heap dump; treating RocksDB memory as if it were heap.

**Where in StreamPulse:** Phase 9 (JVM lab: JFR, async-profiler, G1 vs ZGC, induced leak found via
heap dump), Phase 11 (the manifests).

**Verify:** default `MaxRAMPercentage`; `ActiveProcessorCount` rounding for fractional CPU limits;
cgroup v2 support in your JDK; the RocksDB memory-bounding API in your Kafka Streams version.

---

## Q10. HPA on CPU vs custom metrics, and how does KEDA scale on Kafka lag?

**30-second answer:** CPU-based HPA is the default and it is the **wrong signal for a consumer**. A
Kafka consumer that's falling behind may be sitting at 20% CPU because it's blocked on I/O — so CPU
says "scale down" exactly when you need to scale up. The correct signal is the **work queue**, which for
Kafka means **consumer group lag**. Kubernetes' HPA can use custom/external metrics via the metrics
API, and **KEDA** is the thing that makes that practical: it ships a Kafka scaler that reads consumer
group lag and drives a standard HPA, including scale-to-zero. The constraint nobody should forget is
that **you cannot usefully scale a consumer group beyond the partition count** — the extra pods get no
partitions and sit idle, so partition count is the hard ceiling on consumer parallelism, and KEDA needs
to know that.

**Deeper:**

| Signal | Good for | Bad for |
|---|---|---|
| CPU | CPU-bound work (the Kafka Streams aggregation, maybe) | I/O-bound consumers, bursty request services |
| Memory | almost nothing — JVMs don't release memory promptly, so it's a ratchet | anything |
| RPS / in-flight requests | HTTP services | consumers |
| **Consumer lag** | **Kafka consumers** — it's literally the backlog | — |
| Queue depth (SQS/Rabbit) | queue workers | — |

Why lag is the right metric, stated precisely: lag is the *integral* of (arrival rate − processing
rate). It goes up exactly when you're under-provisioned and down when you're over-provisioned, with no
dependence on what the bottleneck resource is. That's the definition of a good autoscaling signal.

**The partition ceiling.** In a Kafka consumer group, each partition is assigned to at most one
consumer. With 12 partitions on `telemetry.raw.v1` (the plan's number) and listener `concurrency: 3`
per pod, 4 pods saturate the topic. A 5th pod gets nothing. So:

```
max useful consumer threads = partition count
  12 partitions ÷ 3 listener threads per pod = 4 pods maximum
```

This must be `maxReplicaCount` in the scaler, or KEDA happily scales you to 10 pods, 6 of which do
nothing while still consuming cluster resources — and each scale event triggers a **rebalance**, which
*reduces* throughput. Autoscaling a consumer group past its partition count is actively harmful.

**Rebalance cost is the reason to damp the scaler.** Every scale up or down causes a group rebalance,
during which processing stops (or, with cooperative rebalancing, partially stops). A scaler that reacts
to every lag wiggle produces a rebalance storm. Mitigations: `stabilizationWindowSeconds` on
scale-down, a meaningful `lagThreshold` rather than a tiny one, and cooperative assignors.

**Code:**

```yaml
# Phase 11 — KEDA ScaledObject for sink-writer (BS-10 REQ-SL-05 "HPA/KEDA headroom")
# ⚠️ VERIFY every field name and the apiVersion against current KEDA docs —
#    the Kafka scaler's metadata keys have changed across KEDA versions.
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata: { name: sink-writer }
spec:
  scaleTargetRef:
    name: sink-writer
  minReplicaCount: 2          # BS-10 REQ-SL-05: PDB minAvailable 2 needs at least this
  maxReplicaCount: 4          # ← THE PARTITION CEILING: 12 partitions / 3 threads per pod
  pollingInterval: 15
  cooldownPeriod: 300         # don't scale down for 5 min — rebalances are expensive
  advanced:
    horizontalPodAutoscalerConfig:
      behavior:
        scaleDown:
          stabilizationWindowSeconds: 300    # damp the flapping that causes rebalance storms
          policies: [{ type: Pods, value: 1, periodSeconds: 120 }]
        scaleUp:
          stabilizationWindowSeconds: 0      # scale UP fast — the live-match spike (BS-03)
          policies: [{ type: Pods, value: 2, periodSeconds: 30 }]
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka:9092
        consumerGroup: sink-writer
        topic: telemetry.agg.v1
        lagThreshold: "10000"         # target lag PER REPLICA; KEDA divides total lag by this
        offsetResetPolicy: latest
```

```yaml
# The plain-HPA alternative, if you don't want KEDA: an external metric from
# Prometheus via prometheus-adapter. More moving parts, same idea.
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  metrics:
    - type: External
      external:
        metric:
          name: kafka_consumergroup_lag
          selector: { matchLabels: { consumergroup: sink-writer } }
        target:
          type: AverageValue
          averageValue: "10000"
```

**Follow-ups they'll ask:**
- *"How does KEDA actually work?"* — it's an operator that runs scalers and exposes them as an
  **external metrics server** to the standard HPA, plus a controller that handles the 0↔1 transition
  (which the HPA can't do alone). So it isn't a replacement for the HPA; it's a metrics source plus a
  scale-to-zero controller. **Verify the current architecture description.**
- *"Would you scale `stream-processor` this way?"* — much more carefully. Kafka Streams scaling moves
  **state stores** (Q11), so a scale event can mean gigabytes of restore from the changelog topic
  before the new pod does useful work. Scaling *up* under lag pressure can therefore make lag worse
  before it makes it better. I'd scale it on a long window, or not automatically at all.
- *"What about scale-to-zero?"* — great for genuinely idle workloads. For a Kafka consumer it means
  leaving the group entirely, so the first message after idle pays a rebalance plus a cold JVM start.
  Fine for a nightly batch consumer; not for the telemetry hot path.
- *"How do you set `lagThreshold`?"* — from your drain rate and your freshness SLO. If a pod processes
  5,000 records/s and the freshness SLO is 10s, a lag of 50,000 per pod is already at the limit — so
  the threshold should be comfortably below that. Deriving it from the SLO rather than picking a round
  number is the senior move, and it links straight back to BS-10.

**Pitfalls / red flags:** CPU-based autoscaling for a consumer; no `maxReplicaCount` tied to partition
count; no scale-down stabilization, producing rebalance storms.

**Where in StreamPulse:** Phase 11 ("KEDA Kafka-lag autoscaling for `sink-writer`"); BS-10 REQ-SL-05;
BS-03 (the 10× live-match spike is the load this must absorb).

**Verify:** the KEDA Kafka scaler's trigger metadata field names, `apiVersion`, and whether
`lagThreshold` is per-replica or total. The plan itself flags this ("verify KEDA Kafka scaler config").

---

## Q11. What actually happens when you scale a Kafka Streams app from 3 pods to 6?

**30-second answer:** A **rebalance**, and then a **state restore** — and the second one is what people
forget. Kafka Streams tasks are tied to partitions, and each stateful task owns a local RocksDB store
backed by a compacted **changelog topic** in Kafka. When you add three pods, the group rebalances and
tasks are reassigned; every task that moved to a new pod must **rebuild its state store by replaying
the changelog from the beginning** before it can process anything. For a large windowed aggregation
that can be gigabytes and minutes, during which throughput is *worse* than before you scaled. The
mitigations are: **standby replicas**, so a warm copy of the state already exists elsewhere;
**persistent volumes with a StatefulSet plus static group membership**, so a restarting pod reuses its
own local state instead of restoring; and **cooperative rebalancing**, so unaffected tasks keep running
instead of everything stopping. And the ceiling above all of it: you cannot exceed the partition count.

**Deeper:**

### What a stateful task actually holds

```
 telemetry.raw.v1 (12 partitions)
        │
   ┌────┴────┬─────────┬─────────┐
 task 0-0  task 0-1  ...       task 0-11
   │          │
 RocksDB    RocksDB      ← local, on disk, in the pod
   │          │
 changelog topic (compacted, in Kafka) ← the durable source of truth
```

The changelog is what makes the local store recoverable — and what makes recovery *slow*, because
recovery means consuming the whole compacted changelog partition.

### The rebalance protocol matters

| Protocol | Behaviour |
|---|---|
| Eager (old default) | **stop-the-world**: every member revokes all partitions, then a new assignment is computed. Total processing stop. |
| **Cooperative / incremental** | only the partitions that actually move are revoked; everyone else keeps processing. |

Kafka Streams uses a cooperative protocol and a **high-availability task assignor** that tries to
assign a task to an instance that already has (most of) its state, moving tasks only gradually and
letting a "warming up" instance catch up before the task is handed over. **Verify the exact config names
and default assignor for your Kafka Streams version** — this area has evolved significantly
(`acceptable.recovery.lag`, `max.warmup.replicas`, `probing.rebalance.interval.ms` are the ones I
recall, but check them).

### The four levers

| Lever | Config | What it buys | Cost |
|---|---|---|---|
| **Standby replicas** | `num.standby.replicas=1` | a warm copy of every store on another instance → failover restores in seconds, not minutes | 2× state storage and 2× changelog consumption |
| **Static group membership** | `group.instance.id` (stable per pod) + a longer `session.timeout.ms` | a pod restart within the session timeout does **not** trigger a rebalance at all — perfect for rolling restarts | a genuinely dead pod is detected more slowly |
| **Persistent volume + StatefulSet** | `state.dir` on a PVC | the restarted pod finds its own RocksDB files and skips the restore entirely | StatefulSets are more operationally involved; PVs are zone-bound |
| **Cooperative rebalancing** | assignor config | unaffected tasks keep running | more rebalance rounds |

**Static membership is the highest-value one for rolling restarts** and is worth calling out
specifically: without it, a rolling restart of 3 pods triggers 6 rebalances (one per leave, one per
join). With it and a session timeout longer than your restart time, it triggers **zero**. That single
setting transforms deployment cost for a Streams app — and it pairs directly with BS-01's drain budget,
because the drain must finish inside the session timeout.

### Why StatefulSet changes the recovery story

| | Deployment | StatefulSet |
|---|---|---|
| Pod identity | random name, random ordinal | stable: `stream-processor-0`, `-1`, `-2` |
| Volume | shared or ephemeral | **one PVC per pod**, reattached to the same ordinal |
| DNS | via Service VIP | per-pod DNS via a headless Service |
| Rollout | parallel-ish | ordered by default (`podManagementPolicy: OrderedReady`) |
| Restart recovery | full changelog restore | **local RocksDB reused** → near-instant |

The catch: an EBS volume is **zone-bound**, so `stream-processor-0`'s PVC pins that pod to one AZ. If
that AZ fails, the pod can't be rescheduled elsewhere until the volume is. So the PV approach trades
*restart* speed for *relocation* flexibility — which is precisely why standby replicas are the
complementary mechanism rather than an alternative. Say both.

**Code:**

```yaml
# Phase 11 — stream-processor as a StatefulSet (BS-01 drain + Phase 3 state stores)
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: stream-processor }
spec:
  serviceName: stream-processor-headless
  replicas: 3
  podManagementPolicy: Parallel      # we don't need ordered startup; faster rollouts
  template:
    spec:
      terminationGracePeriodSeconds: 60   # Streams close() can take a while; > drain budget
      containers:
        - name: app
          env:
            # STATIC MEMBERSHIP: stable identity survives a restart, so a rolling
            # restart inside session.timeout.ms causes NO rebalance at all.
            - name: KAFKA_GROUP_INSTANCE_ID
              valueFrom: { fieldRef: { fieldPath: metadata.name } }   # stream-processor-0/1/2
          volumeMounts:
            - { name: state, mountPath: /var/lib/streampulse/state }
  volumeClaimTemplates:
    - metadata: { name: state }
      spec:
        accessModes: [ReadWriteOnce]
        resources: { requests: { storage: 50Gi } }
```

```yaml
# application.yml — the Streams side
spring:
  kafka:
    streams:
      properties:
        # Where RocksDB lives — the PVC mount above, so it survives a restart.
        state.dir: /var/lib/streampulse/state
        # A warm standby copy of every store on another instance.
        num.standby.replicas: 1
        # Must EXCEED the pod's full restart time (drain + start) for static
        # membership to avoid a rebalance. Tune from measured restart duration.
        session.timeout.ms: 60000
        processing.guarantee: exactly_once_v2
```

**Follow-ups they'll ask:**
- *"What's the real ceiling on parallelism?"* — the **partition count** of the source topic (and of any
  repartition topic in the topology). 12 partitions means at most 12 stream threads doing useful work,
  no matter how many pods. Increasing partitions later is possible but changes the key→partition
  mapping, which breaks per-key ordering across the change and invalidates existing state — so
  partition count is a decision you want to get roughly right up front, with headroom.
- *"How do you make a rolling restart cheap for a Streams app?"* — static membership + a session
  timeout longer than the restart, persistent state on a PVC, `maxUnavailable: 0`, and standby replicas
  as the backstop. That's the complete answer and it ties BS-01, Q6 and this question together.
- *"What does `exactly_once_v2` cost during a rebalance?"* — EOS uses transactions, so a rebalance
  involves aborting and fencing in-flight transactions. It also means consumers downstream reading with
  `read_committed` see records only after commit, which adds latency bounded by `commit.interval.ms`.
  Worth knowing that EOS is not free, and that it covers the Kafka-to-Kafka path only — a write to S3 or
  DynamoDB is outside the transaction, which is exactly why sink-writer needs idempotent writes anyway.
- *"How do you monitor restore progress?"* — Kafka Streams exposes restore metrics and a
  `StateRestoreListener` callback. The pod's readiness indicator (Q4) returning `REBALANCING` is the
  operational surface of the same fact.

**Pitfalls / red flags:** describing Kafka Streams scaling as if it were stateless; not knowing about
the changelog restore; scaling past the partition count.

**Where in StreamPulse:** Phase 3 (topology, state stores, standby replicas documented), Phase 11
("K8s ops for stateful stream apps (Kafka Streams state + rebalances on scale)" — the phase's stated
interview proof), BS-01 (drain must fit inside the session timeout).

**Verify:** Kafka Streams assignor config names (`acceptable.recovery.lag`, `max.warmup.replicas`,
`probing.rebalance.interval.ms`), the default assignor in your version, and `group.instance.id`
semantics.

---

## Q12. ConfigMaps and Secrets — and why did my baked-in property stop working?

**30-second answer:** Because **environment variables beat everything** in Spring Boot's property
resolution order. Spring Boot's precedence puts OS environment variables above profile-specific
`application-{profile}.yml` and above plain `application.yml` — and relaxed binding maps
`STREAMPULSE_INGEST_MAX_BATCH_SIZE` to `streampulse.ingest.max-batch-size`. So a ConfigMap consumed
with `envFrom` silently overrides the value in your jar. That's usually exactly what you want, and
occasionally it's a two-hour debugging session, which is why the first thing I check is
`/actuator/env` — it shows every property source in order and which one won.

**Deeper:**

| Mount style | Updates without restart? | Spring picks it up? | Use |
|---|---|---|---|
| `envFrom` / `env.valueFrom.configMapKeyRef` | ❌ never (env is fixed at process start) | n/a | simple scalar config |
| Volume mount | ✅ eventually (kubelet syncs, with a delay) | ❌ not unless the app rereads the file | config files, certs |
| Volume + Spring Cloud Kubernetes reload | ✅ | ✅ with a refresh mechanism | dynamic config |

StreamPulse deliberately does **none** of the dynamic options for business config: BS-01 REQ-LC-03 wants
config **validated at startup and fail-fast**, and the plan's answer to runtime config change is
`config.changes.v1` via Debezium CDC (Phase 6) — an auditable, replayable event, not an invisible file
mutation. That's a good design argument to make: "config that changes silently under a running process
is config you can't debug or audit."

**Secrets.** Kubernetes Secrets are **base64-encoded, not encrypted**, in etcd by default. Making them
real requires: encryption at rest for etcd, RBAC so only the right service accounts can read them, and
ideally not storing the secret in Kubernetes at all — External Secrets Operator or the Secrets Store CSI
driver pull from AWS Secrets Manager / Vault at mount time, and **IRSA** (IAM Roles for Service
Accounts) removes the need for static AWS credentials entirely. That last one is the answer to "how do
your pods authenticate to AWS": a service account annotated with an IAM role, and the AWS SDK's default
credential chain picks up the projected web identity token. No secrets at all.

**The ConfigMap-doesn't-trigger-a-rollout problem** (from Q6): changing a ConfigMap does not change the
Deployment's pod template, so nothing restarts. The idiomatic fix is a checksum annotation.

**Code:**

```yaml
# Helm — changing the ConfigMap changes this annotation, which changes the pod
# template, which triggers a normal rolling update. (Phase 11)
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    spec:
      serviceAccountName: sink-writer        # annotated with the IRSA role — no AWS keys
      containers:
        - name: sink-writer
          envFrom:
            - configMapRef: { name: sink-writer-config }
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "k8s,aws"               # BS-01 REQ-LC-06
            # A secret that genuinely must be an env var.
            - name: STREAMPULSE_KAFKA_SASL_PASSWORD
              valueFrom:
                secretKeyRef: { name: kafka-credentials, key: password }
```

```yaml
# The ConfigMap — note the SCREAMING_SNAKE keys: relaxed binding maps
# STREAMPULSE_INGEST_MAX_BATCH_SIZE → streampulse.ingest.max-batch-size
apiVersion: v1
kind: ConfigMap
metadata: { name: sink-writer-config }
data:
  STREAMPULSE_KAFKA_BOOTSTRAP: "kafka-0.kafka:9092,kafka-1.kafka:9092"
  STREAMPULSE_S3_BUCKET: "streampulse-raw-prod"
  STREAMPULSE_SHUTDOWN_FLUSH_TIMEOUT: "20s"     # BS-01 REQ-LC-02
  MANAGEMENT_SERVER_PORT: "8081"
```

**Follow-ups they'll ask:**
- *"How do you debug 'which value won'?"* — `/actuator/env` lists every `PropertySource` in precedence
  order with the resolved value, and `/actuator/configprops` shows the bound `@ConfigurationProperties`
  objects. Restrict both in production.
- *"IRSA vs node IAM role?"* — a node role grants permissions to **every pod on the node**, which is a
  blast-radius disaster. IRSA scopes an IAM role to a Kubernetes service account, so each workload gets
  exactly its own permissions. Always IRSA (or EKS Pod Identity — **verify which your cluster supports**).
- *"What about secret rotation?"* — the Secrets Store CSI driver can rotate a mounted secret, but your
  app must reread it; for a DB password that means recreating the connection pool. Many teams just
  restart the pods on rotation, which is simpler and honest.
- *"How does this interact with GitOps?"* — you can't put secrets in Git in plaintext. Sealed Secrets
  (encrypted, decryptable only in-cluster) or External Secrets (a reference in Git, the value in a
  vault) are the two patterns. External Secrets is the better fit when you already have Secrets Manager.

**Pitfalls / red flags:** thinking Kubernetes Secrets are encrypted; storing AWS access keys in a
Secret when IRSA exists; no checksum annotation, so config changes never take effect and someone
"fixes" it by manually deleting pods.

**Where in StreamPulse:** BS-01 / REQ-LC-03, REQ-LC-06; Phase 11.

**Verify:** EKS Pod Identity vs IRSA availability; Spring Boot's exact property precedence list for
your version.

---

## Q13. How do you build the container image?

**30-second answer:** **Jib**, for a Spring Boot service. It builds an optimally layered OCI image
straight from Gradle with **no Dockerfile and no Docker daemon**, which means CI doesn't need a
privileged builder, builds are reproducible, and — the practical win — it splits dependencies, resources
and application classes into separate layers, so a code change only pushes the small top layer instead
of a 300 MB fat jar. Base image is a **JRE, not a JDK** (you don't need javac in production), the
container runs as a **non-root** user, and for maximum reduction **distroless** removes the shell and
package manager entirely, which cuts both image size and CVE surface. The one gotcha with distroless is
that it has no `/bin/sh`, so the `preStop` sleep hook from Q5 doesn't work — you either use the `:debug`
variant or a non-exec preStop.

**Deeper:**

| Approach | Image size (order of magnitude) | Layer caching | Needs Docker daemon | Notes |
|---|---|---|---|---|
| Naive `FROM openjdk` + fat jar | large | ❌ one giant layer | ✅ | every code change repushes everything |
| Multi-stage Dockerfile + Boot layered jar | small-ish | ✅ | ✅ | explicit, portable, more to maintain |
| **Jib** | small | ✅ automatic | ❌ **no** | best default for a Gradle/Maven JVM project |
| Buildpacks (`bootBuildImage`) | small | ✅ | usually ✅ | zero config, opinionated, good memory tuning built in |
| Jib + **distroless** base | smallest | ✅ | ❌ | no shell — mind the preStop hook |

**Why layering matters**, concretely: a Spring Boot fat jar is mostly third-party dependencies that
change rarely, and a small amount of your code that changes every commit. If they're in one layer, every
build pushes and pulls the whole thing. Split into `dependencies / spring-boot-loader / snapshot-deps /
application`, a typical code change pushes a few hundred KB. On a pipeline that builds six services on
every commit, that's the difference between a 2-minute and a 12-minute deploy.

**Code:**

```kotlin
// build.gradle.kts — Phase 11 ("Small images with Jib (or Buildpacks), JRE-only, non-root")
jib {
    from {
        // JRE, not JDK. Distroless: no shell, no package manager, tiny CVE surface.
        // NOTE: `:debug` variant includes busybox — needed if you use an exec preStop (Q5).
        image = "gcr.io/distroless/java21-debian12:nonroot"
        platforms {
            platform { architecture = "amd64"; os = "linux" }
            platform { architecture = "arm64"; os = "linux" }   // your Mac builds natively
        }
    }
    to {
        image = "ghcr.io/streampulse/${project.name}"
        tags = setOf(version.toString(), "latest")
    }
    container {
        user = "nonroot:nonroot"          // never root; also satisfies a restricted PodSecurity policy
        ports = listOf("8080", "8081")    // app + management (Q4)
        // Container-aware heap sizing (Q9) baked in, overridable by JAVA_TOOL_OPTIONS.
        jvmFlags = listOf(
            "-XX:MaxRAMPercentage=70.0",
            "-XX:+ExitOnOutOfMemoryError",
            "-XX:+HeapDumpOnOutOfMemoryError"
        )
        // Reproducible builds: Jib uses a fixed epoch timestamp by default so the
        // same source produces the same digest. Set explicitly if you need real times.
        creationTime = "USE_CURRENT_TIMESTAMP"
    }
}
```

```dockerfile
# The multi-stage equivalent, for when you can't use Jib (e.g. the web/ React build).
# Shown because interviewers often want to see you can write one.
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /build
COPY gradlew settings.gradle.kts build.gradle.kts ./
COPY gradle ./gradle
RUN ./gradlew dependencies --no-daemon        # cache the dependency layer separately
COPY src ./src
RUN ./gradlew bootJar --no-daemon
# Explode the Boot layered jar so each layer becomes a Docker layer.
# VERIFY the layertools invocation for your Boot version — the jarmode
# arguments changed in recent Boot releases.
RUN java -Djarmode=layertools -jar build/libs/*.jar extract --destination /extracted

FROM eclipse-temurin:21-jre-alpine
RUN addgroup -S app && adduser -S app -G app   # non-root
USER app
WORKDIR /app
# Ordered least-changing → most-changing, so the cache does its job.
COPY --from=builder /extracted/dependencies/ ./
COPY --from=builder /extracted/spring-boot-loader/ ./
COPY --from=builder /extracted/snapshot-dependencies/ ./
COPY --from=builder /extracted/application/ ./
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

**Follow-ups they'll ask:**
- *"Why non-root?"* — defence in depth: a container escape or an RCE starts with fewer privileges, and
  restricted PodSecurity Standards will refuse to schedule a root container. It also forces you to
  notice that you can't bind port 80 — which is fine, you bind 8080 and let the Service map it.
- *"Alpine or Debian-slim base?"* — Alpine uses musl libc rather than glibc, and while modern JDK
  builds for Alpine exist, some native libraries misbehave. For a JVM app the size saving is modest
  relative to the risk; I'd default to a slim Debian-based JRE or distroless and only use Alpine after
  testing.
- *"How do you scan images?"* — Trivy or Grype in CI, failing on high/critical CVEs with a documented
  allowlist. Distroless reduces the findings dramatically because there's no OS package surface.
- *"Do you tag with `latest`?"* — never for deployment. Deploy by **immutable digest** or a
  commit-SHA tag, so what's running is unambiguous and a rollback is exact. `latest` exists for
  convenience and causes incidents.

**Pitfalls / red flags:** `FROM openjdk` with a fat jar COPY and no layering; running as root; deploying
`:latest`.

**Where in StreamPulse:** Phase 11 ("Small images with Jib (or Buildpacks), JRE-only, non-root").

**Verify:** the `-Djarmode=layertools` invocation and `JarLauncher` class name for your Boot version —
both changed in recent releases. Also verify the distroless Java 21 image tag.

---

## Q14. Tie it together: how does your Kubernetes setup deliver zero-loss deploys and keep 99.98% achievable?

*(The applied question.)*

**30-second answer:** Two independent chains. The **zero-loss chain (BS-01)** is: preStop sleep covers
the endpoint-removal race; readiness flips to `REFUSING_TRAFFIC` first; graceful HTTP shutdown finishes
in-flight requests; `SmartLifecycle` phases drain in a fixed order — connectors, then Kafka listeners
committing offsets, then the S3 buffer flush, then the producer close; and
`terminationGracePeriodSeconds: 45` exceeds the whole budget so SIGKILL never lands mid-flush. And
underneath it, the correctness backstop: if the flush doesn't finish, we **don't commit the offsets**,
so the events are redelivered — at-least-once plus idempotent writes, not "hope the flush worked". The
**availability chain (BS-10 REQ-SL-05)** is: `maxUnavailable: 0` so capacity never dips during a
deploy; PDB `minAvailable: 2` so a node drain can't take the service out; topology spread across zones
so an AZ loss costs a third of capacity, not all of it; KEDA headroom on consumer lag for the 10×
spike; and fail-fast config validation so a bad pod never goes Ready and the rollout halts instead of
serving errors. Then I prove it: `RollingRestartChaosIT` does 20 rolling restarts under load and
asserts zero lost events, and `PodKillChaosTest` kills 1 of 3 pods every 2 minutes and measures what
fraction of the 8.64-minute monthly budget that consumes.

**Deeper:**

### The zero-loss sequence, with the failure each step prevents

| Step | Mechanism | Failure prevented |
|---|---|---|
| 1 | `preStop: sleep 10` | requests routed to a pod that already got SIGTERM (endpoint propagation race) |
| 2 | readiness → `REFUSING_TRAFFIC` (phase MAX) | new traffic after drain starts |
| 3 | `server.shutdown: graceful`, connectors stop (phase 300) | in-flight HTTP/gRPC requests cut mid-response |
| 4 | Kafka listeners stop and commit (phase 200) | offsets committed for work not yet durable |
| 5 | S3 buffer flush ≤ 20s (phase 100) | ~2k buffered events lost, half-written S3 part-file |
| 6 | producer `close(Duration)` (phase 0) | unsent records dropped |
| 7 | `terminationGracePeriodSeconds: 45` > total | SIGKILL mid-flush |
| **8** | **don't commit offsets if the flush failed** | **silent loss — this is the only step that is a correctness guarantee rather than an optimization** |

Step 8 is the one to emphasize, because steps 1–7 all fail simultaneously on a **node crash**, where
there is no SIGTERM at all. The design is deliberately layered: graceful shutdown makes the common case
clean and fast; at-least-once plus idempotency makes the uncommon case *correct*.

### The availability mechanisms, each mapped to what it absorbs (REQ-SL-05)

| Mechanism | Absorbs | File/Q |
|---|---|---|
| `maxUnavailable: 0`, `maxSurge: 1`, `minReadySeconds` | capacity dip during a deploy | Q6 |
| PodDisruptionBudget `minAvailable: 2` | node drain / cluster upgrade / autoscaler scale-down | Q7 |
| Topology spread across zones (hard) | AZ or node failure | Q7 |
| Readiness gate + ordered drain | in-flight loss on every pod replacement | Q3, Q5 |
| Validated fail-fast config | a misconfigured pod going Ready and serving 500s | BS-01 REQ-LC-03 |
| Liveness that checks nothing external | cascading fleet-wide restart on a dependency blip | Q3 |
| KEDA on consumer lag, `maxReplicaCount` = partition ceiling | the BS-03 live-match 10× spike | Q10 |
| Static group membership + standby replicas + PVC state | rebalance and restore cost on every restart | Q11 |
| Memory requests == limits, `MaxRAMPercentage`, no CPU limit on the latency path | OOMKills and CFS throttling | Q8, Q9 |
| Resilience4j breaker + spool fallback (BS-04) | a 3-minute S3 outage without going unready | file 05 |

### The proof, not the claim

The two acceptance tests are what convert this from a design doc into evidence:

- **`RollingRestartChaosIT`** — Testcontainers plus kill signals, 20 consecutive rolling restarts under
  steady load, comparing accepted-event count (ingest returned 2xx) against **distinct `eventId`s**
  landed in S3 + DynamoDB. Distinct is the key word: redelivery is expected and the dedup key makes it
  invisible, so the test measures *loss*, not *duplication*.
- **`PodKillChaosTest`** — `kind` cluster under Gatling load, kill 1 of 3 pods every 2 minutes, measure
  the actual SLI. If each kill costs ~3 seconds of errors, 15 kills ≈ 45 seconds ≈ 8.7% of the
  8.64-minute budget. The NFR in BS-10 is "< 10% of the monthly budget", and now that's a measured
  number rather than an assertion.

### The one honest caveat

`kind` is a local single-node-or-few-node cluster. It genuinely tests the drain sequence, probe
behaviour, rollout strategy and the app's shutdown ordering. It does **not** test real AZ failure, real
cloud load-balancer deregistration timing (which is usually the slowest part of the endpoint race), or
real node preemption. So the preStop sleep value I'd ship is derived from the cloud LB's documented
deregistration behaviour, not from what worked on `kind` — and I'd say so rather than implying the
number was validated end to end.

> **Your story:** if you have run services on Kubernetes at Couchbase or elsewhere — particularly
> anything involving a bad deploy, a probe misconfiguration, an OOMKill hunt, or a drain that didn't
> drain — that goes here, in the same shape: what broke, which mechanism fixed it, how you verified.
> Your Datadog/Grafana on-call experience is the natural pairing: the metrics in file 11 are what told
> you these mechanisms were working. Keep "I built" for what you personally wrote; StreamPulse is a
> portfolio project and should be presented as *"I built this to prove the pattern end to end."*

**Follow-ups they'll ask:**
- *"What's the weakest link?"* — the endpoint-removal race, because the delay is outside my control and
  varies by load balancer. It's mitigated by a fixed sleep, which is a guess dressed up as a
  configuration value. The honest mitigation is that step 8 makes even a failure there non-fatal.
- *"What would you add next?"* — progressive delivery (Argo Rollouts/Flagger) gating each canary step on
  the burn-rate SLI from BS-10, so a bad version is caught at 10% of traffic instead of 100%. That's the
  natural extension of the error-budget gate already in REQ-SL-04.
- *"How long does a deploy take with `maxUnavailable: 0` and `minReadySeconds: 15`?"* — roughly
  `replicas × (pod startup + 15s + drain)`. For 3 replicas with a 40-second JVM start that's a few
  minutes, and that's the price of the SLA. If it became a problem I'd raise `maxSurge`, not
  `maxUnavailable`.
- *"What breaks first at 10× scale?"* — partition count. Every Kubernetes lever above scales
  horizontally; the consumer group's parallelism ceiling does not. That's the constraint I'd plan
  around first.

**Pitfalls / red flags:** describing the mechanisms without the ordering; claiming zero loss without
naming the offset-commit rule that actually guarantees it; having no test behind the claim.

**Where in StreamPulse:** BS-01 end to end (REQ-LC-01, REQ-LC-02, REQ-LC-03, REQ-LC-06); BS-10 /
REQ-SL-05; Phase 11; acceptance tests `ShutdownOrderIT`, `RollingRestartChaosIT`, `PodKillChaosTest`.

---

## Key terms

| Term | One-line meaning |
|---|---|
| **EndpointSlice** | The live list of Ready pod IPs behind a Service; its propagation delay is the drain race. |
| **Liveness / readiness / startup probe** | Restart me / stop sending traffic / I'm still booting. |
| **Cascading restart** | Liveness pointed at a dependency check → one blip restarts the whole fleet. |
| **preStop hook** | Runs before SIGTERM; a sleep here covers endpoint-removal propagation. |
| **`terminationGracePeriodSeconds`** | Time from SIGTERM to SIGKILL; must exceed preStop + full app drain. |
| **`maxSurge` / `maxUnavailable`** | Pods above / below desired count allowed during a rollout. |
| **PodDisruptionBudget** | Minimum availability the **eviction API** must respect; voluntary disruptions only. |
| **Voluntary vs involuntary disruption** | Drain/upgrade/autoscaler vs node crash/OOMKill/spot reclaim. |
| **`topologySpreadConstraints`** | Even pod distribution across zones/nodes with a `maxSkew` ceiling. |
| **QoS class** | Guaranteed / Burstable / BestEffort — decides eviction order under node memory pressure. |
| **CFS throttling** | CPU limit enforcement: threads descheduled for the rest of the 100ms period. |
| **OOMKilled vs `OutOfMemoryError`** | Kernel SIGKILLs the container (exit 137, no heap dump) vs the JVM throws inside its own heap. |
| **`MaxRAMPercentage`** | Heap as a share of the **container** limit; the container-aware replacement for `-Xmx`. |
| **KEDA** | Event-driven autoscaling: exposes external metrics (e.g. Kafka lag) to the HPA, plus scale-to-zero. |
| **Partition ceiling** | Max useful consumers in a group == partition count; scaling past it is harmful. |
| **Changelog restore** | Rebuilding a Kafka Streams state store from its compacted changelog after task reassignment. |
| **Standby replica** | A warm copy of a Streams state store on another instance; turns minutes of restore into seconds. |
| **Static group membership** | `group.instance.id` + long session timeout → a pod restart causes **no** rebalance. |
| **IRSA** | IAM Roles for Service Accounts — per-pod AWS credentials, no static keys. |
| **Jib / distroless** | Daemon-less layered image builds / a base image with no shell or package manager. |

---

## Self-check (answer out loud, no notes)

1. Between `kubectl delete pod` and the process exiting, name every step in order — and say where a
   request can still be dropped.
2. Why must liveness never check Redis, and what *should* a liveness probe check for a Kafka consumer
   with no inbound traffic?
3. Your pod is OOMKilled with 40% heap usage. What are the three things you check?
4. You scale `stream-processor` from 3 to 6 pods. Describe what happens, and why throughput might get
   worse before it gets better.
5. List the mechanisms that keep BS-10's 99.98% achievable, and say which one is a correctness
   guarantee rather than an optimization.
