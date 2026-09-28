# 09 — Monolith to Microservices: Boundaries, Strangler Fig, Sagas, and Knowing When Not To

**Why this matters for an Event Hub role:** an event hub is almost always an *extraction* — something
that used to live inside a bigger application, pulled out because it scales differently. So the
interview question is rarely "what is a microservice"; it is *"we have this monolith, it's slow during
peaks, what would you do?"* The senior answer starts by asking whether to split at all, then makes the
**data** cut before the **process** cut, then moves traffic incrementally with a way back at every
step. The junior answer starts drawing boxes.

**Scenario anchor — BS-08 (REQ-SF-01…06):** StreamPulse's alert engine starts as a module *inside*
control-plane. During peaks, CPU-heavy rule evaluation starves the admin API — rule-CRUD p99 rises from
80 ms to 900 ms in load tests — and the two scale on different axes: alerts with Kafka partitions,
admin with users. It must come out with **no missed and no duplicate alerts**, and the move must be
reversible with a flag flip at any moment.

> The `> **Your story:**` blocks in this file are where your real monolith-to-microservices work goes.
> This material gives you the vocabulary and the failure modes; the specifics — which service, why,
> what broke — are yours and are worth more than any diagram here.

---

## Q1. How do you find service boundaries?

**30-second answer:** Four lenses, applied in this order. First, **language**: where does the same word
mean two different things? In StreamPulse a "rule" to the admin API is an editable record with
validation and audit; to the alert engine it is a compiled predicate evaluated 50,000 times a second.
Same word, two models — that is a bounded-context seam. Second, **data ownership**: draw the tables and
ask who writes each one. A candidate service that must write another's tables is not a service. Third,
**change coupling**: run `git log` and find which files change together. Things that always change
together belong together, whatever the diagram says. Fourth, **team topology** — Conway's law says the
architecture will end up mirroring the org chart anyway, so either design for the teams you have or be
prepared to change the teams.

**Deeper:**

| Lens | The question | The signal | Tool |
|---|---|---|---|
| **Bounded context (DDD)** | does this word mean the same thing on both sides? | one word, two models → seam | event storming, a glossary per context |
| **Data ownership** | who is the sole writer of this table? | two candidate services writing one table → not a seam | a write-matrix of services × tables |
| **Change coupling** | which files change in the same commit? | high co-change across the proposed boundary → wrong boundary | `git log --name-only` aggregated into a co-change matrix |
| **Team topology / Conway** | who is on call for this? | one service, three teams → constant coordination | the org chart, honestly read |
| **Scaling axis** | what makes this need more capacity? | different axes → a genuine reason to split | your own load-test data |

**Conway's law, said precisely:** organisations design systems that mirror their own communication
structures. The practical use is the *inverse* manoeuvre — deliberately shape teams around the
architecture you want. The failure mode it predicts is concrete: if one team owns a service and
another team owns a feature that lives in it, every change needs a cross-team ticket, and within two
quarters someone will have added a back door to avoid the ticket.

**The write-matrix, which is the exercise I'd actually do:**

```
                    tenant  app  title  alert_rule  service_node  alert_history  outbox
control-plane (admin)  W     W     W        W            W              -          W
alert-engine           -     -     -        R            R              W          -
                                          ↑ read-only ⇒ a clean seam
```

That matrix is the whole boundary analysis in one picture. If `alert-engine` had a `W` under
`alert_rule`, the seam would not exist — you would have two services that must agree on writes to one
table, which is a distributed monolith with extra latency. In BS-08 it reads rules and writes only its
own history, which is why the extraction is even a candidate.

**The scaling-axis test.** "It's slow" is not a reason to split; it is a reason to profile. The reason
to split is that two things inside one deployable need capacity for **different reasons** and you
cannot give it to one without paying for the other. In BS-08, alert evaluation scales with the number
of Kafka partitions and the rule count; admin CRUD scales with the number of logged-in operators.
Bundled together, a live sports event forces you to scale the whole control plane — database
connections, JVM heap, Kafka consumer group membership — to satisfy the CPU-bound half.

**Code — the boundary, enforced before the split (REQ-SF-01):**

```java
/**
 * The boundary exists as a compile-and-test-time rule BEFORE it exists as a network hop.
 * If these rules pass for a full release cycle, the extraction is mechanical.
 * If they fail, you have just learned the extraction would have failed too — cheaply.
 */
@AnalyzeClasses(packages = "com.streampulse.controlplane")
class AlertEngineBoundaryArchTest {

    @ArchTest
    static final ArchRule engine_uses_only_the_published_read_api =
        noClasses().that().resideInAPackage("..alerts.engine..")
            .should().dependOnClassesThat().resideInAnyPackage(
                    "..admin.repository..",      // no reaching into the admin's JPA repos
                    "..admin.entity..")          // and no sharing its entities
            .because("REQ-SF-01: the engine consumes the rules read API / CDC topic, never the tables");

    @ArchTest
    static final ArchRule admin_does_not_call_the_engine_directly =
        noClasses().that().resideInAPackage("..admin..")
            .should().dependOnClassesThat().resideInAPackage("..alerts.engine..")
            .because("the seam must be one-directional, or it is not a seam");
}
```

**Follow-ups they'll ask:**
- *"How big should a service be?"* — there is no size rule worth quoting. The tests I use: can one team
  own it end to end? Can it be deployed without coordinating with another team? Does it own its data?
  Can a new engineer understand it in a day? "Small" is a consequence of those, not a target.
- *"What if two services genuinely need the same data?"* — one owns it and the other gets a **copy**,
  kept fresh by events, shaped for its own reads. That is Q8. Shared writes are the thing to refuse.
- *"Where do you start in a big legacy codebase?"* — at the edges: something with few inbound
  dependencies, a clear owner, and a measurable pain. Not the core domain. Your first extraction is
  about building the *muscle* — the pipeline, the observability, the rollback — not about value.

**Pitfalls / red flags:** boundaries drawn by technical layer ("user-service, order-service,
notification-service" derived from the table list), or by team convenience with no data-ownership
analysis. Splitting by layer gives you a distributed monolith on the first try.

**Where in StreamPulse:** BS-08 / REQ-SF-01; `AlertEngineBoundaryArchTest`.

---

## Q2. What is a distributed monolith, and how would you recognise one?

**30-second answer:** A distributed monolith is a system that paid every cost of microservices and
collected none of the benefits: you have network calls, serialization, partial failure and N
deployment pipelines, but you still cannot deploy any one service independently. The diagnostic
question is simply *"can you release service A without releasing service B?"* If the honest answer is
"well, they have to go out together", you have one application in several processes. The usual causes
are a shared database, synchronous call chains three or four deep, and a shared library that carries
domain types between services.

**Deeper — the symptom checklist:**

| Symptom | Why it means "not actually separate" |
|---|---|
| Services deployed together, in a fixed order | the API contract is not stable, so they are one unit |
| A shared database schema written by several services | the *real* coupling is the schema; the code split is cosmetic |
| A shared "common"/"domain" jar containing entities and DTOs | a field change forces every service to rebuild and redeploy |
| A request fans out through 4+ synchronous hops | availability multiplies down, latency adds up (Q13) |
| One team must change three services to ship one feature | the boundary cuts across the change, i.e. it is in the wrong place |
| Integration tests require the whole stack up | no service can be reasoned about alone |
| A schema migration requires a coordinated release | data ownership was never split (Q7) |

**The availability arithmetic**, which I'd show rather than assert. If four services are each up
99.9% of the time and a request needs all four **synchronously**:

```
0.999^4 = 0.996    →  ~0.4% failure  ≈  ~2.9 hours/month of failed requests
```

Compared with 99.9% on a monolith (~43 min/month), the split made availability **worse** — unless each
hop has a timeout, a fallback and a circuit breaker (BS-04), or the call is made asynchronous. That
number is the honest cost of a synchronous chain, and being able to produce it on the spot is worth
more than any amount of enthusiasm about microservices.

**The shared-library trap** deserves its own note, because it is the one teams walk into while feeling
virtuous about DRY. A shared jar containing *technical* things — a tracing starter, a ProblemDetail
error model, JSON config, test fixtures — is good; that is exactly StreamPulse's `libs/platform`. A
shared jar containing *domain* types — the `AlertRule` entity, shared DTOs — recreates compile-time
coupling across the network boundary, and now a field addition is a coordinated release of everything.
The rule I use: **share technical infrastructure, never the domain model.** Each service maps the wire
contract (Avro/OpenAPI/proto, which *is* versioned and compatibility-checked) into its own internal
types.

**Follow-ups they'll ask:**
- *"Isn't duplicating DTOs a DRY violation?"* — DRY is about duplicating *knowledge*, not about
  duplicating *shapes*. Two services having a field called `tenantId` is not knowledge duplication; it
  is two independent models that happen to agree today and must be free to diverge tomorrow. Enforcing
  their sameness with a shared jar is the actual coupling.
- *"How do you get out of one?"* — fix the data ownership first (Q7), then stabilise the contracts with
  versioning and consumer-driven contract tests, then break the synchronous chains with events. The
  code layout is the last thing you change, not the first.
- *"What's the fastest way to check?"* — look at the deploy history. If the services' release timestamps
  are always within minutes of each other, they are one application.

**Pitfalls / red flags:** treating "we have microservices" as the goal. The goal is independent
deployability; services are a means.

**Where in StreamPulse:** BS-08 / REQ-SF-06 — the ADR explicitly documents *when not to extract*, and
`libs/platform` is deliberately technical-only.

---

## Q3. When would you *not* split — and what do you do instead?

**30-second answer:** Most of the time. A split buys independent deployability, independent scaling and
fault isolation; it costs you network latency on every former method call, partial failure as a
permanent condition, distributed tracing to debug anything, eventual consistency where you used to
have a transaction, and N times the operational surface. If the pain is "the code is messy" or "the
build is slow", a **modular monolith** with enforced boundaries fixes that at a fraction of the cost —
you get the architecture without the distribution. I'd only split when there is a *specific,
measurable* driver: different scaling axes, different availability requirements, a team-ownership
boundary, a technology constraint, or a compliance/data-residency boundary. StreamPulse's alert engine
qualifies on the first of those, with load-test numbers to back it.

**Deeper:**

| Driver | Split? | Why |
|---|---|---|
| Different scaling axes, measured | **yes** | you cannot give capacity to one half without paying for both |
| Different availability tiers | **yes** | a best-effort feature shouldn't share a JVM with the critical path |
| Separate team ownership + deploy cadence | **yes** | Conway; the coordination cost is real and recurring |
| Technology constraint (native library, different runtime) | **yes** | genuinely can't coexist |
| Compliance / data residency | **yes** | the boundary is imposed from outside |
| "The codebase is messy" | **no** | modules and enforced boundaries; distribution doesn't clean code, it hides mess behind network calls |
| "The build is slow" | **no** | fix the build (incremental compilation, test parallelism, build cache) |
| "Microservices are best practice" | **no** | — |
| "We might need to scale later" | **no** | build the *module boundary* now; the split is then a day's work, not a quarter's |

**The modular monolith, concretely.** One deployable, one database, one transaction — but internal
modules with enforced boundaries:

```
control-plane (one Spring Boot app, one Postgres)
├── admin/            owns tenant, app, title, alert_rule tables
│   ├── api/          ← the only package other modules may import
│   └── internal/     ← repositories, entities: package-private or ArchUnit-fenced
├── alerts/
│   ├── api/
│   └── engine/       ← may import admin.api, NEVER admin.internal
└── platform/         shared technical concerns only
```

Enforced by ArchUnit (Q1), by Java modules, or by Spring Modulith if you want the tooling. The payoff
is precise: **you get the boundary discipline now, and the option to distribute later is nearly free**,
because the seam already exists and has been tested. The reverse — splitting first and discovering the
boundary was wrong — costs a quarter and a lot of credibility.

**The costs a split imposes, which I'd list before drawing any box:**

| Cost | Was | Becomes |
|---|---|---|
| Latency | a method call, ~ns | a network round trip, ~ms — 10⁶× |
| Failure | an exception | a timeout, a partial success, or an *unknown* (Q13) |
| Consistency | one ACID transaction | a saga with compensations (Q9) |
| Debugging | a stack trace | a distributed trace you must have built beforehand |
| Refactoring | an IDE rename | a versioned API change across two release cycles (Q12) |
| Operations | one deploy, one dashboard, one log stream | N of each, plus the glue |
| Testing | one process | contract tests, and an integration environment |

**Follow-ups they'll ask:**
- *"How do you keep a modular monolith from rotting?"* — the boundary must be **executable**. An
  ArchUnit test that fails the build is a boundary; a wiki page is a suggestion. Add the rule on day
  one, because retrofitting it means fixing 400 violations.
- *"Doesn't a monolith mean you can't scale?"* — no: you scale it horizontally like anything else. What
  you can't do is scale *one part* independently, which is precisely the BS-08 driver and precisely why
  it is the one thing StreamPulse extracts.
- *"Have you ever recommended not splitting?"* — this is where a real story lands hardest. Interviewers
  are tired of enthusiasm; restraint with a reason is memorable.

> **Your story:** if you have been in a migration where part of the system should have stayed put — or
> where a split was reversed — that is one of the most credible things you can say in this round. What
> was the driver people claimed, what was the real one, and what happened?

**Pitfalls / red flags:** no cost list. If a candidate describes splitting as pure upside, they have
not operated one.

**Where in StreamPulse:** BS-08 / REQ-SF-06 (the ADR covers "when not to extract"), REQ-SF-01 (the
module boundary exists and is enforced for a full phase before the process split).

---

## Q4. Walk me through the strangler fig pattern, step by step.

**30-second answer:** You put a **facade** in front of the old system, then move functionality behind
it one piece at a time, routing a growing share of traffic to the new implementation, until the old one
is dead and you delete it. The name comes from the fig that grows around a host tree and eventually
replaces it. The essential properties are that every step is small, every step ships to production, and
every step is reversible — which is what makes it different from a rewrite. In BS-08 the facade is a
configuration flag plus the Kafka topic: the new alert-engine consumes the same topics under a
different consumer group, publishes to a shadow topic while we compare, and then a flag promotes it.

**Deeper — the six steps, with the exit criterion for each:**

```
0. PREPARE   module boundary enforced in the monolith (REQ-SF-01)
             ├ exit: ArchUnit rules green for a full release cycle
             └ observability in place: you cannot compare what you cannot measure

1. SEAM      the module talks to the rest only through an interface + events (REQ-SF-02)
             ├ exit: no direct repository/table access across the boundary
             └ THIS IS THE DATA CUT — see Q7. It is the hard part and it comes FIRST.

2. BUILD     new service implements the same interface; deployed, running, receiving NO traffic
             └ exit: it starts, passes health checks, has dashboards

3. SHADOW    new service consumes the same input; output goes to a shadow sink; comparator diffs
             ├ exit: REQ-SF-03 — 0 mismatches over a 1-hour run under load
             └ this is where you find the bugs, at zero user risk

4. CUTOVER   flag flips reads/writes to the new service, incrementally if possible (REQ-SF-04)
             ├ exit: error rate and latency within budget at 100%
             └ flag flips BACK in seconds — the rollback is not a deploy

5. REMOVE    delete the old code path and the flag
             └ exit: nobody has flipped back for N weeks; the old path has zero traffic in metrics
```

Step 5 is the one teams skip, and skipping it is how you end up maintaining both forever. Put a
deletion date in the ADR and hold yourself to it; a feature flag with no removal date is permanent
complexity.

**Where the facade lives** depends on what kind of thing you are extracting:

| Extraction type | Facade | Rollback mechanism |
|---|---|---|
| HTTP endpoint | reverse proxy / API gateway route, or a controller-level branch | route weight, instantly |
| Message consumer | consumer group + topic routing | flag: which consumer group is "live" |
| Scheduled job | flag in the scheduler | flag |
| Library call | an interface with two implementations + `@ConditionalOnProperty` | flag |

BS-08 is the second row, and it is the *easiest* kind of extraction — which is worth saying, because it
shows you know the difference. A Kafka consumer can be shadow-run trivially: add a second consumer
group and it reads the same data with **zero** impact on the existing one. An HTTP-request extraction
needs actual traffic mirroring, which is harder to do without side effects (Q5).

**Code — the cutover seam (REQ-SF-04):**

```java
/**
 * The in-monolith engine and the extracted service both consume telemetry.agg.v1.
 * Which one is LIVE is a runtime property, not a deployment:
 *   alerts.engine=module   → the monolith listener runs, publishes to alerts.v1
 *   alerts.engine=shadow   → both run; the service publishes to alerts.shadow.v1
 *   alerts.engine=service  → the monolith listener STOPS; the service publishes to alerts.v1
 *
 * It is a PROPERTY, not a Spring profile, precisely so it can be flipped within one
 * environment without a redeploy (file 01, Q5).
 */
@Component
@ConditionalOnProperty(name = "alerts.engine", havingValue = "module", matchIfMissing = true)
class InMonolithAlertEngine implements SmartLifecycle {

    /**
     * Stopping is SmartLifecycle, not just "don't process": we must stop the listener
     * cleanly and commit offsets, so the cutover has a well-defined last-processed window
     * and the new engine can start exactly there. This reuses BS-01's ordered shutdown.
     */
    @Override public void stop() {
        listenerRegistry.getListenerContainer("alert-engine").stop();
    }
    @Override public int getPhase() { return 200; }
}
```

**Follow-ups they'll ask:**
- *"How long does a strangler migration take?"* — longer than a rewrite would *appear* to, and it
  finishes, which the rewrite frequently does not. The honest framing: each step ships, so value and
  risk are both incremental, and you can stop at any point with a working system.
- *"What if the old system has no seams at all?"* — then step 0 is the whole project: add tests
  (characterisation tests that pin current behaviour, even the wrong bits), then introduce the
  interface, then move the implementation. You cannot strangle what you cannot isolate.
- *"How do you handle a piece that needs both old and new data?"* — during the overlap, one side owns
  the write and the other reads a copy kept fresh by events (Q8). If both must write, you have not
  finished step 1.

**Pitfalls / red flags:** "we'd do a big-bang cutover over a weekend." Also: describing the pattern but
having no rollback mechanism, which turns every step into a one-way door.

**Where in StreamPulse:** BS-08 / REQ-SF-01…05; `CutoverFlagIT` flips the flag mid-stream and asserts
no gaps and no duplicates.

---

## Q5. What is shadow traffic / dual-run, and how do you compare the outputs?

**30-second answer:** You send the same input to both the old and the new implementation, but only the
old one's output is real — the new one writes to a shadow sink. A comparator then diffs the two
streams and exports a mismatch metric. This is how you get production-fidelity confidence before
anyone is exposed: production data, production volume, production weirdness, zero user risk. The
subtlety is **side effects**: the shadow path must not send emails, charge cards or page anyone, so
either its outbound adapters are no-ops or its output goes somewhere inert. And the comparison must be
tolerant of *legitimate* differences — timestamps, generated ids, map ordering — or you drown in false
mismatches on day one and stop looking at the metric.

**Deeper:**

```
telemetry.agg.v1 ──┬──▶ [monolith engine]  group=alerts-v0 ──▶ alerts.v1        ← REAL, users see this
                   │
                   └──▶ [alert-engine svc] group=alerts-v1 ──▶ alerts.shadow.v1 ← inert

                          ┌──────────────┐
    alerts.v1        ────▶│  comparator  │──▶ alertengine.shadow.mismatch{kind}  (metric)
    alerts.shadow.v1 ────▶│ join by key, │──▶ a sample of mismatches to a topic for inspection
                          │ window, diff │
                          └──────────────┘
```

**The comparator is the real engineering**, and it is where candidates thin out. Four decisions:

| Decision | Options | StreamPulse choice |
|---|---|---|
| Join key | natural id, or a synthetic correlation id | `alertId = hash(ruleId, windowStart)` — deterministic on both sides (REQ-SF-05), so the join is exact |
| Time window | how long to wait for the other side before declaring a mismatch | 2× the window length + grace; a late arrival is not a mismatch |
| Normalisation | which fields to ignore | ignore `emittedAt`, `engineVersion`, `traceId`; compare severity, rule, window, value |
| Mismatch taxonomy | one counter, or several | several: `MISSING_IN_SHADOW`, `EXTRA_IN_SHADOW`, `VALUE_DIFFERS` — they have completely different causes |

That last row matters. `MISSING_IN_SHADOW` usually means the new engine didn't get the rule (a data
problem — step 1 isn't finished). `EXTRA_IN_SHADOW` usually means a suppression or dedup rule wasn't
ported. `VALUE_DIFFERS` usually means a rounding, timezone or window-boundary difference. One
undifferentiated "mismatch" counter tells you that something is wrong; three tell you what to fix.

**Code:**

```java
/**
 * REQ-SF-03 — shadow comparator. Runs as a Kafka Streams job joining the two alert
 * streams on the deterministic alertId within a window.
 *
 * Exit criterion for the cutover: 0 mismatches over a 1-hour run under load.
 * "Zero" is achievable ONLY because alertId is deterministic; with random ids you
 * would be comparing sets fuzzily and would never get a clean signal.
 */
KStream<String, Alert> live   = builder.stream("alerts.v1");
KStream<String, Alert> shadow = builder.stream("alerts.shadow.v1");

live.outerJoin(shadow,
        (l, s) -> compare(l, s),
        JoinWindows.ofTimeDifferenceAndGrace(Duration.ofMinutes(2), Duration.ofSeconds(30)))
    .foreach((key, result) -> meter.counter("alertengine.shadow.mismatch",
                                            "kind", result.kind().name()).increment());

static Comparison compare(Alert live, Alert shadow) {
    if (shadow == null) return Comparison.missingInShadow(live);   // new engine didn't fire
    if (live  == null)  return Comparison.extraInShadow(shadow);   // new engine over-fired
    // Normalise before comparing: these fields are EXPECTED to differ.
    return live.withoutVolatileFields().equals(shadow.withoutVolatileFields())
            ? Comparison.match() : Comparison.valueDiffers(live, shadow);
}
```

**For an HTTP extraction** (not BS-08, but they will ask): you mirror requests — at the proxy (nginx
`mirror`, Envoy shadow/request-mirror), or in the application by forking the request to the new
service on a separate executor with a short timeout and discarding the response. Rules: never let the
shadow call's latency or failure affect the real response; make the shadow's writes go to a separate
schema or be suppressed; and be careful with non-idempotent operations — mirroring a `POST /payments`
to a system with real side effects is how this pattern gets a bad name.

**Follow-ups they'll ask:**
- *"What does shadow running cost?"* — double the compute for that path, double the read load on shared
  dependencies, and the comparator itself. For a Kafka consumer it is cheap (an extra consumer group is
  pure read). For an HTTP path with database writes it can be significant, and you run it for a bounded
  window rather than indefinitely.
- *"What if the outputs legitimately differ?"* — then say so explicitly and encode it: a documented
  allow-list of differences with an owner and an expiry. What you must not do is widen the tolerance
  until the metric goes green.
- *"Can you shadow with a percentage?"* — yes, and for expensive paths you should: mirror 5% and scale
  up as confidence grows. Watch for sampling bias — 5% of *random* requests, not 5% of one tenant.

**Pitfalls / red flags:** shadow-running with no automated comparison ("we eyeballed the logs"), or a
shadow path that shares a database with the live path and quietly writes to it.

**Where in StreamPulse:** BS-08 / REQ-SF-03; `ShadowComparatorIT`.

---

## Q6. How do you actually cut over, and how do you roll back?

**30-second answer:** With a runtime flag, not a deploy. The new path ships to production **disabled**;
the cutover is a configuration change that takes seconds and reverses in seconds. That decoupling —
deploy is not release — is the single most valuable property, because it means the rollback path is not
"revert the commit, rebuild, redeploy, wait eight minutes" while the incident runs. Where the traffic
shape allows it I ramp: 1% → 10% → 50% → 100%, watching the SLIs at each step, with an automatic
rollback trigger rather than a human judgement call at 3 a.m. And before the flip I write down what
would make me flip back — specific metrics, specific thresholds — because deciding that *during* the
incident is how people talk themselves into staying.

**Deeper:**

| Cutover style | Mechanism | Granularity | Good for |
|---|---|---|---|
| Big-bang flag | one boolean | all-or-nothing | idempotent consumers where a shadow run already proved parity (BS-08) |
| Percentage ramp | hash(requestId) % 100 < n | request-level | HTTP paths; stable per-user if you hash the user id |
| Per-tenant | flag keyed by tenant | tenant-level | multi-tenant systems — start with the smallest tenant (`kidsplus`), end with the biggest |
| Blue/green | two full environments, switch the LB | environment | infrastructure changes |
| Canary | a subset of pods run the new version | pod-level | code changes, not behaviour changes |

For BS-08 the choice is a **big-bang flag** rather than a ramp, and the reason is instructive: with a
Kafka consumer you cannot meaningfully run "10% of the alerts" — the partition assignment decides who
sees what, and splitting evaluation across two engines would break windowed state. So the risk is
managed by the *shadow run* instead: an hour of zero mismatches at full production volume, then a flip.
Different traffic shape, different risk-control mechanism. Being able to say *why* the ramp doesn't
apply is a better answer than reciting the ramp.

**Rollback criteria, written before the flip:**

| Signal | Threshold | Action |
|---|---|---|
| alert delivery latency p99 | > 2× baseline for 5 min | flip back |
| missed alerts (comparator, still running) | any | flip back immediately |
| duplicate alerts reaching PagerDuty | > 0 after dedup | flip back, investigate dedup |
| alert-engine error rate | > 1% for 3 min | flip back |
| admin API p99 (the *reason* for the split) | not improved after 30 min | keep the flag on; investigate — this is a "the split didn't help" signal, not an incident |

The last row is the one people forget: you also need a criterion for **"it worked but didn't achieve
the goal"**, which is a different decision from "it broke" (Q14).

**The overlap window is the dangerous moment.** At the instant of the flip, the old engine has consumed
up to offset X and the new one is at offset Y. If Y < X, the new engine reprocesses and *duplicates*.
If Y > X, there is a **gap** and alerts are missed. Two mechanisms make this safe:

1. **Stop the old listener cleanly first** — `SmartLifecycle.stop()` commits offsets, so X is known and
   final (this is why BS-01's ordered shutdown is a prerequisite, not a nicety).
2. **Deterministic ids** — `alertId = hash(ruleId, windowStart)` (REQ-SF-05) means that if the windows
   *do* overlap, both engines produce the *same* id and the downstream consumer dedupes it away. Gaps
   are prevented by ordering; duplicates are made harmless by identity. Two independent mechanisms,
   because one of them will have a bug.

**Code:**

```java
@Test
@DisplayName("REQ-SF-04/05: flipping the engine flag mid-stream loses no alerts and duplicates none")
void cutoverIsSeamless() {
    producer.sendWindows(1..100);                 // continuous input across the flip

    awaitWindow(40);
    config.set("alerts.engine", "service");       // ← the flip, mid-stream
    awaitWindow(100);

    List<Alert> delivered = sqsSink.all();
    assertThat(delivered).extracting(Alert::alertId).doesNotHaveDuplicates();   // no dups
    assertThat(delivered).hasSize(100);                                         // no gaps
}
```

**Follow-ups they'll ask:**
- *"Where does the flag live?"* — a config property refreshed from `config.changes.v1` (the same CDC
  path as everything else, so the change is auditable and replayable), or a feature-flag service. Not
  an environment variable that needs a pod restart — that is a deploy wearing a flag's clothes.
- *"How do you stop flags accumulating?"* — every flag gets an owner and a removal date in the same PR
  that adds it, and a periodic report of flags older than 90 days. Flag debt is real debt: each one
  doubles the code paths you must reason about, and they multiply.
- *"What if you can't roll back — the data has moved on?"* — then design so that you can: dual-write
  during the overlap, keep the old store readable, and don't delete anything until the flag is removed.
  If a cutover is genuinely irreversible, it needs a *much* longer shadow run and an explicit sign-off,
  and you say that out loud rather than pretending the flag protects you.

**Pitfalls / red flags:** "rollback = redeploy the previous version." That is minutes of outage and
assumes the previous version still works against the current data.

**Where in StreamPulse:** BS-08 / REQ-SF-04, REQ-SF-05; `CutoverFlagIT`.

---

## Q7. Everyone says "split the data first". Why is that the hard part?

**30-second answer:** Because code is easy to move and data is not. You can extract a class in an
afternoon; you cannot extract a table that four features join against without deciding what happens to
those joins — and the answer is always "they stop being joins", which means either a network call or a
replicated copy, and either way the transaction that used to wrap them is gone. That is the real
migration: giving up a foreign key and an ACID guarantee you have been relying on without noticing. So
I do the data cut **while still in the monolith**, where it is cheap to get wrong: make the module talk
to the other module's data through an API or an event stream, run it that way for a release, and only
then move the process. If the data cut works in-process, the extraction is mechanical.

**Deeper:**

```
BEFORE                                   AFTER (data cut, still one process)
┌──────────────────────────┐             ┌──────────────────────────┐
│  control-plane           │             │  control-plane           │
│  ┌────────┐  ┌────────┐  │             │  ┌────────┐              │
│  │ admin  │  │ engine │  │             │  │ admin  │──owns──▶ [alert_rule]
│  └───┬────┘  └───┬────┘  │             │  └───┬────┘              │
│      │  JOIN     │       │             │      │ publishes RuleChanged
│      ▼           ▼       │             │      ▼                   │
│   [alert_rule] (shared)  │             │  config.changes.v1       │
└──────────────────────────┘             │      │                   │
   both modules query it,                │      ▼                   │
   both rely on one tx                   │  ┌────────┐              │
                                         │  │ engine │──owns──▶ [alert_history]
                                         │  └────────┘  holds its OWN rule cache
                                         └──────────────────────────┘
                                          REQ-SF-02: rules arrive as EVENTS, never a join
```

**Why the shared database is the anti-pattern to refuse.** It looks like the cheap option — two
services, one Postgres, done in a day — and it makes every subsequent step impossible:

| Problem | Consequence |
|---|---|
| The schema is the contract | any migration is a coordinated, cross-team release (file 06, Q12) |
| No encapsulation | service B depends on column semantics nobody documented; you can't refactor |
| Locking and connection pools are shared | service B's slow report blocks service A's writes; one pool exhaustion takes both down |
| Can't scale independently | the database is the bottleneck for both, which was often the original problem |
| Ownership is unclear | at 3 a.m., who owns the corrupt row? |

The one *defensible* transitional form is **read-only access during the migration**, with an explicit
expiry date. A service that reads another's tables and never writes is a temporary crutch you can
remove. Shared writes are not recoverable without redoing the work.

**The four techniques for breaking a join across the new boundary:**

| Technique | Shape | Cost | Use when |
|---|---|---|---|
| **API call** | B asks A at request time | latency on every call; A's availability becomes B's | low volume, freshness matters, A is fast |
| **Replicated read model** | A publishes events; B keeps its own copy, shaped for its reads | eventual consistency; storage; invalidation | high volume, staleness of seconds is fine — **BS-08's choice** |
| **Data denormalised into the event** | the event carries what B needs (the rule's threshold, not just its id) | bigger events; a versioned contract | B only needs a snapshot at event time |
| **Move the operation** | give the work to whoever owns the data | an API change | the join was really A's job all along |

For BS-08 the replicated read model is right and the reasoning is a calculation: the engine evaluates
rules at Kafka-partition rate — thousands of times a second — so an API call per evaluation is
impossible. Rules change a handful of times a day. Enormous read rate, tiny write rate, seconds of
staleness acceptable: that is exactly the profile a replicated cache fed by CDC serves best. The rule
cache is a `ConcurrentHashMap` refreshed by `config.changes.v1`, and the topic is **compacted** and
keyed by `ruleId`, so a cold start replays the topic and rebuilds the full current state with no API
call and no database access at all.

**Follow-ups they'll ask:**
- *"What about referential integrity?"* — you lose the foreign key. It moves into the application as a
  validation at the boundary, plus a reconciliation job that detects orphans and an operational
  decision about what to do with them. This is a genuine loss, and the honest answer names it rather
  than pretending events solve it.
- *"How do you migrate the physical data?"* — dual-write to both stores, backfill the new one, verify
  with a row-count and checksum comparison, switch reads, stop writing the old, then drop. It is
  expand/contract (file 06, Q12) at the database-instance level, and each step is reversible.
- *"How do you handle a query that genuinely needs both sides?"* — a read model built by joining the two
  event streams (CQRS), or an API composition at the edge, or — the most honest answer — admit the
  boundary was wrong and move it. Needing a cross-service join for a core query is evidence, not a
  detail.

**Pitfalls / red flags:** "both services will just use the same database for now." That sentence is
where most microservice migrations die, and it dies quietly, a year later.

**Where in StreamPulse:** BS-08 / REQ-SF-02 — *"Rules reach the engine via `config.changes.v1` (CDC)
and never by a DB join. This makes the data-ownership cut before the process cut."*

---

## Q8. Data is now duplicated across services. How do you keep it sane?

**30-second answer:** By being explicit about which copy is the **source of truth** and which are
**derived projections**. There is exactly one writer per piece of data; every other copy is built by
consuming that writer's event stream and can be **thrown away and rebuilt** by replaying it. That
rebuildability is the property that makes duplication safe rather than terrifying — a corrupt
projection is a replay, not an incident. Then you make the staleness visible: measure the lag from
source commit to projection update, put an SLO on it, and show it in the UI when it matters. Users
tolerate "updated 3 seconds ago" far better than silently wrong data.

**Deeper:**

| Property | Source of truth | Derived projection |
|---|---|---|
| Writers | exactly one | none (only the event consumer) |
| Correctness | authoritative | eventually consistent with the source |
| On corruption | restore from backup — a real incident | replay the topic — a routine operation |
| Shape | normalised for writing | denormalised for **its own** reads |
| Example | `alert_rule` in control-plane Postgres | the engine's in-memory rule cache; query-api's title LRU |

**The compacted topic is the mechanism that makes this work.** `config.changes.v1` is keyed by
aggregate id and compacted, so Kafka retains at least the latest value per key forever. A new service,
or a service that lost its state, subscribes from the beginning and receives the current state of every
rule. No API call, no bulk export endpoint, no coordination with the owning team. That is a genuinely
elegant property and it is worth naming explicitly, because it is the difference between "we copy data
around" and "we have a rebuildable materialised view".

**Consistency you can actually promise:**

```
operator saves rule  ──┬─── control-plane Postgres         [strongly consistent, immediately]
                       │
                       ├─── config.changes.v1              [+ ~ms, commit-ordered]
                       │
                       ├─── engine rule cache              [+ ~10-100ms  → alerts use the new rule]
                       └─── query-api title LRU            [+ ~10-100ms  → dashboard label updates]

              read-your-own-writes: NOT automatic. After the save, the admin UI may re-read
              from a projection that hasn't caught up and see the OLD value. Fixes:
                (a) read back from the source of truth on the write path (simplest);
                (b) return the new state in the write response, and render that;
                (c) version token in the response; the read waits for the projection to reach it.
              StreamPulse uses (b): the PUT returns the updated rule, the UI renders it.
```

Read-your-own-writes is the anomaly that generates bug reports, and naming it unprompted signals you
have actually shipped an eventually consistent system rather than read about one.

**Bounding the staleness, operationally:**

| Control | What it does |
|---|---|
| Projection lag metric | `now() − sourceCommitTimestamp` on the consumed event; alert if > SLO |
| Periodic reconciliation | compare counts/checksums between source and projection nightly; alert on drift |
| Full rebuild runbook | reset the consumer group to earliest and replay; practise it, time it, write the number down |
| Version/`updatedAt` on every projected record | lets a consumer detect and discard out-of-order updates |

That last one matters more than it looks: if the projection can be updated by more than one path
(replay plus live stream, say), an old event can arrive after a new one. Carrying the source version
and ignoring anything not newer makes the projection **idempotent and order-insensitive** — which is
what lets you replay safely at any time.

**Follow-ups they'll ask:**
- *"What if a projection is wrong and you don't notice?"* — that is what reconciliation is for, and it
  is the part everyone skips. A nightly job comparing counts and checksums per tenant, with a metric,
  costs a day to build and finds the class of bug that otherwise surfaces as a customer complaint.
- *"Isn't this a lot of machinery for a rule cache?"* — for one cache, yes. The machinery is the CDC
  pipeline, which exists anyway for other reasons. That is the real argument: build the event backbone
  once and every subsequent projection is nearly free.
- *"How much staleness is acceptable?"* — it is a product question, and I'd get it answered explicitly.
  For an alert rule, seconds. For a billing rate, zero — which is why billing reads the source of truth
  directly, and I'd rather have that inconsistency in the *architecture* than pretend one policy fits
  everything.

**Pitfalls / red flags:** "data duplication is bad, we normalise" — that is a single-database instinct
applied where it does not hold. Or duplication with no source of truth, which is two writers and
guaranteed divergence.

**Where in StreamPulse:** BS-08 / REQ-SF-02; Phase 5 (title LRU invalidated by `config.changes.v1`);
Phase 1 (compaction and key choice documented in the AsyncAPI contract).

---

## Q9. You lost your transaction. How do you keep a multi-service operation consistent?

**30-second answer:** With a **saga** — a sequence of local transactions where each step publishes an
event that triggers the next, and each step has a **compensating action** that semantically undoes it if
a later step fails. "Semantically" is the important word: you do not roll back, you *compensate* — you
don't un-charge a card, you issue a refund; you don't un-send an email, you send a correction. The
system is never in a globally consistent state mid-saga, only eventually. Two styles: **choreography**,
where each service reacts to events with no central coordinator (simple, decoupled, and hard to follow
once you have more than three or four steps), and **orchestration**, where one component owns the state
machine and tells each service what to do (visible, testable, debuggable, at the cost of a component
that knows the whole flow).

**Deeper:**

```
CHOREOGRAPHY                                 ORCHESTRATION
  A ──event──▶ B ──event──▶ C                     ┌──────────────┐
  │            │            │                     │ Orchestrator │  owns the state machine,
  └─ compensate on failure ─┘                     │  (persisted) │  retries, timeouts
                                                  └──┬───┬───┬───┘
  + no central component                             ▼   ▼   ▼
  + services fully decoupled                         A   B   C
  − the flow exists in NO single place
  − cyclic dependencies creep in                  + the flow is one readable object
  − "why didn't step C run?" is an                + timeouts and retries in one place
    archaeology exercise across 3 log streams     + testable as a unit
                                                  − a component that knows everyone
                                                  − can drift into a god service
```

My rule: **choreography up to about three steps, orchestration beyond that** — and always orchestration
when there is a timeout, a human approval, or a compensation sequence, because those need persisted
state that no single event can carry.

**Compensations are not rollbacks**, and the differences are the part interviewers probe:

| | Rollback (ACID) | Compensation (saga) |
|---|---|---|
| Visibility of intermediate state | never visible | **visible to everyone**, including users |
| Mechanism | the database undoes it | you write code that does the inverse |
| Always possible? | yes | **no** — an email is sent; a webhook fired |
| Ordering | automatic | reverse order of completion, and you must implement it |
| Failure of the undo itself | impossible | very possible — needs retries and a dead-letter path for human intervention |

The consequences you must design for: **isolation is gone**. Another transaction can read the
intermediate state (a "dirty read" at the business level) — mitigated with semantic locks (a
`PENDING` status that other operations refuse to act on), commutative updates, or simply re-ordering
steps so the externally visible one happens last. And **compensations must be idempotent**, because the
compensator will be retried.

**Code — the alert-notification saga (BS-07/BS-08 territory), orchestrated:**

```java
/**
 * Orchestration chosen over choreography because there are 4 steps, two timeouts,
 * and a compensation that must run in reverse order. The state machine is persisted,
 * so an orchestrator restart resumes rather than restarting the saga.
 */
@Service
class AlertDeliverySaga {

    void handle(AlertRaised e) {
        SagaState s = store.start(e.alertId());       // persisted; the whole point
        try {
            s.step("PERSIST",  () -> history.record(e));             // compensate: mark VOID
            s.step("SUPPRESS", () -> suppression.check(e));          // compensate: none (read-only)
            s.step("ROUTE",    () -> rabbit.publish(e));             // compensate: publish a retraction
            s.step("PAGE",     () -> pager.notify(e));               // compensate: resolve the page
        } catch (StepFailed f) {
            // Compensate in REVERSE order of completion. Each compensation is idempotent
            // and retried independently; a compensation that keeps failing goes to a DLQ
            // for a human, because a stuck saga must never be silently abandoned.
            s.compensateCompleted();
        }
    }
}
```

**Follow-ups they'll ask:**
- *"What if a compensation fails?"* — retry with backoff, then dead-letter it to a queue a human owns,
  and alert. There is no automatic answer; the system's job is to make the stuck saga *visible* rather
  than silently inconsistent. Any design that assumes compensations always succeed is incomplete.
- *"How do you test a saga?"* — each step and each compensation as a unit; the state machine with a
  fake for every service; then failure-injection integration tests that fail at step 3 and assert steps
  2 and 1 were compensated. The failure paths need *more* tests than the happy path, because they run
  rarely and are therefore where the bugs live.
- *"Does StreamPulse actually need a saga?"* — mostly no, and I'd say so. Most of its flows are a single
  local transaction plus an event (the outbox pattern, file 07), which is much simpler. Sagas are for
  multi-service *write* operations, and the design deliberately has few of them. Recognising that the
  outbox covers most cases is a stronger answer than reaching for a saga reflexively.

**Pitfalls / red flags:** describing a saga as "distributed transactions" or believing compensations
restore the prior state exactly.

**Where in StreamPulse:** BS-07 (partner delivery with retry/DLQ), BS-08; the concept appears in
`docs/design-doc.md`. The Go `ticket-booking` project in the same portfolio has a fuller saga —
worth referencing as prior art.

**Verify:** the saga concept comes from a 1987 paper by Garcia-Molina and Salem; check the exact title
and venue before citing it in an interview.

---

## Q10. Why not just use two-phase commit across services?

**30-second answer:** Because it trades availability for a consistency guarantee you can usually get
more cheaply another way. In 2PC a coordinator asks every participant to prepare, and once a
participant votes yes it must hold its locks until the coordinator says commit or abort. If the
coordinator dies in that window, the participants are **blocked** — locks held, rows unavailable, and
no participant may unilaterally decide. In practice you also need every participant to be an XA
resource, which Kafka, most HTTP APIs and most NoSQL stores are not. So the whole system's availability
becomes the product of all participants', with an added single point of failure. The industry answer is
sagas plus idempotency plus reconciliation — weaker guarantees, vastly better availability, and failures
that are visible and fixable rather than a wedged cluster.

**Deeper:**

```
  coordinator            participant A            participant B
       │ ── PREPARE ────────▶ │                         │
       │ ── PREPARE ──────────┼────────────────────────▶│
       │ ◀── yes ─────────────│                         │
       │ ◀── yes ─────────────┼─────────────────────────│
       │                      │  locks HELD             │  locks HELD
       ✖ coordinator dies     │  …                      │  …
                              │  BLOCKED indefinitely   │  BLOCKED indefinitely
                              │  cannot commit (maybe   │
                              │  the other voted no)    │
                              │  cannot abort (maybe    │
                              │  the other committed)   │
```

That blocking window is not a theoretical concern; it is the defining property. And it interacts badly
with everything else: held locks mean queued writers, which means connection-pool exhaustion (file 06,
Q11), which means the service is down for operations that had nothing to do with the transaction.

| | 2PC | Saga |
|---|---|---|
| Consistency | atomic across services | eventual |
| Isolation | yes | **no** — intermediate state is visible |
| Availability | product of all participants, minus the coordinator | each step independent |
| Blocking | yes, if the coordinator fails mid-protocol | no |
| Participant requirements | XA-capable resource manager | any service with an API and a compensator |
| Latency | two round trips per participant, locks held throughout | one local transaction per step |
| Works with Kafka / HTTP / DynamoDB | no | yes |
| Operational familiarity | low; few teams have run it | high |

**Where 2PC genuinely is used:** within a single database (Postgres `PREPARE TRANSACTION` for
distributed setups), between a message broker and a database in some classic Java EE stacks, and inside
distributed databases that implement it internally with a consensus-backed coordinator (which fixes the
blocking problem by making the coordinator itself fault-tolerant). That last point is the nuance worth
having: **2PC is not "wrong", it is expensive and needs a highly available coordinator** — which is what
Raft/Paxos-based systems provide internally and what an application-level coordinator does not.

**What you do instead, in order of preference:**

1. **Don't distribute the transaction.** Redesign so the atomic operation lives in one service. This is
   the best answer and the one most often skipped: needing a cross-service transaction is usually
   evidence the boundary is wrong (Q1).
2. **Outbox + idempotent consumer** (file 07). Covers "change my data and tell the world", which is the
   overwhelming majority of real cases.
3. **Saga with compensations** (Q9) for genuine multi-service writes.
4. **Reconciliation** as the safety net underneath all of it: a job that detects divergence and either
   fixes it or raises it. This is what real financial systems do — they do not prevent every
   inconsistency, they *detect and settle* them.

**Follow-ups they'll ask:**
- *"What about Kafka transactions / `exactly_once_v2`?"* — that is atomicity between consumed offsets
  and produced records **within Kafka**, not across Kafka and your database. Useful and real, but it
  does not make your DynamoDB write part of the transaction. (File 07, Q6 has the hop-by-hop version.)
- *"Is there a middle ground?"* — try-confirm-cancel (TCC): reserve, then confirm or cancel. It is a
  saga with an explicit reservation step that gives you a form of isolation — good for inventory and
  seat booking. Same failure handling, better business semantics.
- *"How do you explain eventual consistency to a product owner?"* — in their units: "the change is
  guaranteed to happen, and it will be visible everywhere within N seconds; here is the number, here is
  the alert if it is exceeded, and here is what the UI shows in the meantime." A bounded, measured delay
  is a specification. "Eventually" on its own is not.

**Pitfalls / red flags:** proposing XA for a Kafka + Postgres flow, or dismissing 2PC without being
able to explain the blocking problem.

**Where in StreamPulse:** the reason the outbox pattern exists (file 07, REQ-DL-06) rather than any
distributed transaction.

---

## Q11. How do idempotency and deterministic IDs help during a cutover?

**30-second answer:** They convert the two scary cutover failure modes into non-events. During the
overlap window you cannot guarantee the old and new implementations processed exactly disjoint input —
so you must assume some input is processed **twice**. If the output id is derived deterministically from
the input (`alertId = hash(ruleId, windowStart)`), both implementations produce the *same* id for the
same logical event, and the downstream consumer's idempotent write collapses them. That is REQ-SF-05.
The alternative — random UUIDs — means two engines produce two different ids for the same alert and
someone gets paged twice, which is exactly the outcome a migration is supposed to avoid.

**Deeper:**

```
❌ random id                                  ✅ deterministic id
old engine: alertId = uuid4() = "a1b2…"       old engine: hash(rule=r42, window=14:30) = "7f3c…"
new engine: alertId = uuid4() = "c3d4…"       new engine: hash(rule=r42, window=14:30) = "7f3c…"
consumer sees 2 alerts → 2 pages              consumer sees the SAME id twice → 1 page
                                                   INSERT … ON CONFLICT DO NOTHING
                                                   / DynamoDB attribute_not_exists(PK)
                                                   / Redis SET NX
```

**Choosing what goes into the hash** is the actual design decision, and it is where this gets
interesting. It must contain everything that makes two outputs *logically different*, and nothing that
varies between implementations:

| Include | Exclude |
|---|---|
| the rule id | the engine's hostname or version |
| the window start (the logical time) | the wall-clock time of evaluation |
| the tenant / entity id | the Kafka partition or offset |
| the rule *version*, if a re-evaluation with a changed threshold should be a new alert | a random nonce, obviously |

That fourth row is a genuine judgement call worth raising unprompted: if an operator edits the rule
mid-window, should the alert be "the same alert, updated" or "a new alert"? Include the rule version in
the hash and it is new; exclude it and it is the same. There is no universally right answer — but
picking one deliberately, and writing down why, is the senior move.

**Idempotency has to be implemented on the *write*, not just asserted.** The three shapes:

| Store | Mechanism |
|---|---|
| Postgres | `INSERT … ON CONFLICT (alert_id) DO NOTHING`, or a unique index and catching the violation |
| DynamoDB | `ConditionExpression: attribute_not_exists(PK)` — succeeds exactly once (file 06, Q13) |
| Redis | `SET key value NX EX ttl` |
| Kafka consumer | a dedup table/cache keyed by event id, with a TTL longer than your worst replay window |

**Code:**

```java
/**
 * REQ-SF-05. The id is a pure function of the alert's identity, so the in-monolith engine
 * and the extracted service independently compute the SAME value. This is what makes the
 * cutover overlap window harmless without any coordination between the two engines.
 *
 * SHA-256 truncated to 128 bits and rendered as a UUID: stable across JVMs and versions
 * (String.hashCode() would NOT be — never use it for anything persisted or cross-process).
 */
static UUID alertId(String ruleId, Instant windowStart) {
    byte[] digest = sha256((ruleId + '|' + windowStart.toEpochMilli()).getBytes(UTF_8));
    return UUID.nameUUIDFromBytes(digest);        // deterministic, collision-resistant enough here
}
```

```java
@Test
@DisplayName("REQ-SF-05: both engines produce the same id for the same logical alert")
void deterministicAcrossImplementations() {
    var window = Instant.parse("2026-09-28T14:30:00Z");
    assertThat(monolithEngine.evaluate("r-42", window).alertId())
        .isEqualTo(extractedEngine.evaluate("r-42", window).alertId());
}
```

**Follow-ups they'll ask:**
- *"What if the two implementations disagree on the window boundary?"* — then they produce different ids
  and you get duplicates, which is precisely the kind of bug the shadow comparator (Q5) exists to catch
  *before* cutover. The determinism and the comparator are complementary: one makes duplicates harmless,
  the other proves there aren't any.
- *"How long do you keep the dedup keys?"* — longer than your worst-case replay. If a consumer group can
  be reset to six hours ago, a one-hour TTL means duplicates. I'd size it to the topic retention for
  critical paths, and monitor the dedup store's size.
- *"Does this apply outside cutovers?"* — constantly. It is the same mechanism that makes at-least-once
  delivery safe everywhere (file 07, Q6), and the same reason the ingest gateway dedups on `eventId`.
  A migration just makes the need obvious.

**Pitfalls / red flags:** using `String.hashCode()` or `Objects.hash()` for a persisted, cross-process
identifier. `hashCode` is not specified to be stable across JVM versions or implementations, and for
`String` it is trivially collidable.

**Where in StreamPulse:** BS-08 / REQ-SF-05; Phase 2 (`eventId` dedup in Redis), Phase 4 (DynamoDB
conditional writes).

---

## Q12. How do you version APIs during a migration?

**30-second answer:** The rule is that **during a migration both versions run at once**, so every change
must be backward-compatible for at least one release cycle — exactly like a database migration
(expand/contract). Additive changes need no version bump: new optional fields, new endpoints, new enum
values that clients are required to tolerate. Breaking changes — removing a field, renaming one,
tightening validation, changing semantics — need a new version that runs in parallel while consumers
migrate, with usage metrics telling you when the old one is genuinely dead. The mistake I watch for is
treating "v2" as a licence to redesign everything at once, which guarantees nobody migrates.

**Deeper:**

| Change | Breaking? | Handling |
|---|---|---|
| Add an optional response field | no | ship it; clients must ignore unknown fields (say so in the contract) |
| Add a required request field | **yes** | make it optional with a default first, then tighten later |
| Remove a response field | **yes** | deprecate, measure usage, remove after the window |
| Rename a field | **yes** | add the new one, populate both, switch clients, remove the old — expand/contract |
| Change a field's type | **yes** | new field, new name; never reuse the name with a new type |
| Add an enum value | **maybe** | breaking if clients switch exhaustively; the contract must say "tolerate unknown values" |
| Tighten validation | **yes** — and easy to miss | someone is sending the now-invalid value; measure first |
| Change semantics with the same shape | **yes, and the worst kind** | no type system catches it; needs a new field or version |

That last row is the dangerous one: changing `status: "ACTIVE"` from "the rule is enabled" to "the rule
is enabled and currently matching" breaks every consumer without changing a single type. Semantic
changes need a new name.

**Versioning styles:**

| Style | Example | Notes |
|---|---|---|
| URI path | `/v1/rules` | most visible, easiest to route at a proxy — **StreamPulse's choice**, and the same `v1` suffix appears on Kafka topics for consistency |
| Header | `Accept: application/vnd.streampulse.v2+json` | purer REST, harder to route and to curl |
| Query param | `?version=2` | easy to forget, easy to cache wrong |
| No versioning, additive only | — | ideal, and achievable for a long time if you are disciplined |

For **events** the equivalent is the schema registry with `BACKWARD` compatibility (file 07, Q7), and
topic-level versioning (`config.changes.v1` → `.v2`) for genuinely breaking changes, with dual
publishing during the overlap.

**Consumer-driven contract tests** are the mechanism that makes this safe rather than hopeful: each
consumer publishes what it actually needs, and the provider's CI fails if a change would break any
registered consumer. That converts "we think nobody uses that field" into a build result. StreamPulse
uses this between `query-api` and `web/` (Phase 5). The honest caveat: contract tests only cover
consumers you *know about*; a public API needs usage telemetry per field as well.

**Code:**

```java
/**
 * Both versions live, mapped from one domain model. v2 renamed `expr` → `expression`
 * and made `severity` an object rather than a string.
 *
 * Deprecation is machine-readable AND measured: without the counter you will never
 * know when it is safe to delete, and "v1" will still be here in three years.
 */
@GetMapping(value = "/v1/rules/{id}")
@Deprecated(since = "2026-09", forRemoval = true)
RuleV1 getV1(@PathVariable UUID id) {
    meter.counter("api.deprecated.calls", "endpoint", "GET /v1/rules/{id}",
                  "client", clientId()).increment();     // ← who still calls this, and how often
    return RuleV1.from(service.find(id));
}

@GetMapping(value = "/v2/rules/{id}")
RuleV2 getV2(@PathVariable UUID id) { return RuleV2.from(service.find(id)); }
```

**Follow-ups they'll ask:**
- *"How long do you keep the old version?"* — until the usage metric is zero for a full cycle of your
  slowest client, plus a documented sunset date communicated in advance (a `Deprecation`/`Sunset` header
  is a reasonable mechanism — verify the exact header semantics before citing them). For an internal API
  with known consumers, weeks. For a partner-facing one like the BS-09 OEM SDK, quarters, because you do
  not control when a TV gets a firmware update.
- *"What about the database, does it version too?"* — no: the database has one schema, and
  *both API versions map onto it*. That is why the expand/contract discipline (file 06, Q12) and the API
  versioning discipline have to be in step.
- *"How do you stop v2 becoming a redesign?"* — scope the version bump to the specific breaking change.
  If v2 is a rewrite, nobody migrates, and you maintain two full APIs indefinitely.

**Pitfalls / red flags:** removing a field because "nobody uses it" with no telemetry, or versioning
every change and ending up at v7 after a year.

**Where in StreamPulse:** Phase 1 (OpenAPI is the source of truth, TS client generated from it),
Phase 5 (contract tests between query-api and `web/`), BS-08 during the extraction.

---

## Q13. What new failure modes does the split introduce?

**30-second answer:** Every method call you turned into a network call gained three new outcomes:
slow, failed, and — the dangerous one — **unknown**. A timeout does not tell you whether the other side
did the work. That single fact drives most of the design: every remote call needs a timeout, a retry
policy, and an idempotency key, because retrying an unknown is the only safe thing you can do and it is
only safe if the operation is idempotent. Then the aggregate effects: latency adds up along the chain,
availability multiplies down, and a slow dependency propagates backwards into *your* thread or
connection pool until you fall over too — cascading failure. Retries make all of it worse unless they
are bounded, jittered and circuit-broken.

**Deeper — the fallacies of distributed computing** (commonly attributed to L. Peter Deutsch and
colleagues at Sun, with a later addition by James Gosling; the attribution is widely repeated but
verify before quoting it as fact):

| Fallacy | What it costs you when assumed |
|---|---|
| The network is reliable | no retries, no idempotency → lost or duplicated work |
| Latency is zero | chatty APIs; an N+1 across the network, which is 1000× worse than an N+1 in SQL |
| Bandwidth is infinite | shipping whole entities where an id would do |
| The network is secure | no mTLS/authz between services; a flat trust zone |
| Topology doesn't change | hard-coded addresses; no service discovery; broken during a rolling deploy |
| There is one administrator | nobody knows who owns the failing hop at 3 a.m. |
| Transport cost is zero | serialization CPU and egress charges that nobody budgeted for |
| The network is homogeneous | cross-AZ/cross-region assumptions that hold only in one environment |

**Latency arithmetic**, to have ready:

```
in-process method call          ~ 10 ns
same-AZ network round trip      ~ 0.5 ms         →  ~50,000× slower
cross-AZ round trip             ~ 1-2 ms         (approximate; measure your own)
cross-region round trip         ~ 50-150 ms      (speed of light + routing; measure)

A request that fans out to 4 services sequentially:  4 × 0.5 ms = 2 ms of pure network,
plus serialization on both sides, plus TLS, plus connection acquisition. In parallel it is
~0.5 ms — which is why CompletableFuture composition (BS-02) is not a micro-optimisation,
it is the difference between a viable and an unviable decomposition.
```

Those numbers are order-of-magnitude, not measurements — quote them as such and measure your own.

**Cascading failure, mechanically:**

```
1. dependency D slows from 20 ms to 2 s
2. service C's callers hold threads/connections 100× longer
3. C's pool saturates → C's own latency rises → C starts timing out
4. C's callers (B) now see C as slow → B saturates
5. retries from B multiply load on C by the retry factor → C gets slower still
6. the whole chain is down because ONE leaf got slow
```

The defences are exactly BS-04's toolkit, and it is worth naming which one stops which step:

| Defence | Stops |
|---|---|
| **Timeout** (short, explicit, everywhere) | step 2 — bounds how long a thread is held |
| **Bulkhead** (per-dependency, per-tenant) | step 3 — saturation is contained to one pool |
| **Circuit breaker** | step 4/5 — stops calling a known-bad dependency and sheds load from it |
| **Retry with exponential backoff + jitter, capped** | step 5 — no synchronised retry storm |
| **Retry budget** (cap retries as a % of total requests) | step 5 — the one people forget; a per-call retry limit still permits a 3× load multiplier system-wide |
| **Load shedding / 429 + `Retry-After`** | step 6 — fail fast and visibly rather than queueing |
| **Fallback / graceful degradation** | user impact — a stale panel with a badge beats a blank page (BS-02 / REQ-CC-02) |

**Partial failure is the one with no clean answer.** A request updates service A, then service B fails.
A is already committed. Your options are: compensate (Q9), retry until B succeeds (needs idempotency and
durable intent — the outbox), or accept the inconsistency and reconcile later. What you cannot do is
pretend it did not happen. Designing the *specific* answer per operation is the work.

**Follow-ups they'll ask:**
- *"How do you debug across services?"* — distributed tracing with propagated context, and it must be in
  place **before** the split, not after. Trace ids in every log line, and — the bit people miss — the
  trace has to survive asynchronous hops: Kafka headers, and the `trace_id` column in the outbox
  (file 07). Otherwise the trace stops at the message broker, which is where the interesting problems are.
- *"What does a good timeout look like?"* — derived from the downstream's measured p99 plus headroom, and
  **shorter than the caller's own budget**. The classic bug is a chain where every service times out at
  30 s, so the user waits 30 s to learn the 4th hop failed at 2 s. Deadlines propagated down the chain
  (gRPC does this natively — file 10) fix it properly.
- *"Have you seen a retry storm?"* — a great place for a real story if you have one.

> **Your story:** a cascading failure, a retry storm, or a partial failure you had to reconcile — what
> the trigger was, how you found it, and what you changed. Concrete incident stories outperform any
> amount of pattern vocabulary in this round.

**Pitfalls / red flags:** not distinguishing "failed" from "unknown". That distinction is the whole
reason idempotency keys exist.

**Where in StreamPulse:** BS-04 (all of Resilience4j, file 05), Phase 8 (tracing through Kafka
headers), BS-02 / REQ-CC-02 (per-panel budget and DEGRADED status).

---

## Q14. How do you know the split actually worked?

**30-second answer:** You wrote down the number before you started. The BS-08 driver is measurable:
admin API p99 degrades from 80 ms to 900 ms under peak alert load. So the success criterion is "admin
p99 stays under 100 ms during a peak-load run" — the same Gatling scenario, before and after. If you
cannot state a metric that should move, you do not have a reason to split, you have a preference. I'd
also track the costs honestly: deploy frequency, change failure rate, mean time to restore, and the new
line items — one more pipeline, one more dashboard, one more on-call surface, plus end-to-end latency
which will have gone **up**.

**Deeper:**

| Category | Metric | BS-08 expectation |
|---|---|---|
| **The driver** | admin API p99 during peak alert load | 900 ms → < 100 ms (REQ-SF-04 NFR) |
| | alert evaluation throughput at fixed CPU | should scale with partitions independently |
| **Correctness** | shadow comparator mismatches | 0 over 1 h under load (REQ-SF-03) |
| | duplicate alerts after cutover | 0 (REQ-SF-05 dedup) |
| **Delivery (DORA-style)** | deploy frequency per service | should rise — independent deployability was the point |
| | change failure rate | should not rise; if it does, the boundary or the tests are wrong |
| | mean time to restore | should not rise; if it does, observability lagged the split |
| **The bill** | end-to-end alert latency | **will rise** by a network hop — budget it and verify it stays inside the SLO |
| | infrastructure cost | 2 deployables, 2 pipelines, more pods |
| | operational surface | one more thing on call, one more runbook |

Presenting the costs alongside the wins is what makes this credible. A candidate who reports only
improvements has either not measured or is not telling you everything.

**The "when to reverse it" question.** Sometimes the honest outcome is that the split did not pay for
itself, and a mature team merges it back. The signals: the two services always deploy together anyway
(Q2), the network hop dominates the latency budget, the on-call load doubled with no reliability gain,
or the boundary turns out to be crossed by every new feature. That last one is the strongest signal —
if every feature touches both sides, the boundary is in the wrong place, and either move it or remove
it.

**Code — the measurement, as an executable gate:**

```java
/**
 * The BS-08 acceptance criterion, as a test rather than a claim. Runs the same
 * Gatling profile against control-plane with the alert engine in-process and extracted,
 * and asserts the admin API's p99 under peak alert load.
 */
@Test
@DisplayName("REQ-SF-04: admin API p99 < 100ms during peak alert evaluation, engine extracted")
void adminApiIsNotStarvedByAlertEvaluation() {
    loadGen.runPeakAlertLoad(Duration.ofMinutes(5));
    Percentiles p = metrics.forEndpoint("PUT /v1/rules/{id}");
    assertThat(p.p99()).isLessThan(Duration.ofMillis(100));
}
```

**Follow-ups they'll ask:**
- *"What if the numbers don't improve?"* — then the bottleneck was not what you thought, and the right
  move is to profile rather than to keep splitting. This is the answer that shows judgement: the flag is
  still there, flipping back costs seconds, and you have learned something concrete for the cost of one
  extraction.
- *"How do you attribute a regression to the split?"* — the flag. Flip it and re-measure. That is the
  under-appreciated benefit of the strangler approach: for as long as the flag exists you have a
  controlled A/B of your own architecture.
- *"What about the team's velocity?"* — worth tracking, hard to measure honestly. Deploy frequency and
  lead time per service are the least-gameable proxies.

**Pitfalls / red flags:** no before-measurement, so success is a feeling. Or declaring victory on
"cleaner architecture" with no operational metric attached.

**Where in StreamPulse:** BS-08 NFRs (0 shadow mismatches; admin p99 < 100 ms); Phase 9 provides the
before-numbers, which is why the extraction is deliberately scheduled *after* the perf phase.

---

## Q15. Walk me through extracting StreamPulse's alert engine, end to end.

*(The applied question. Tell it as a sequence with a decision and an exit criterion at each step.)*

**30-second answer:** The alert engine started as a module inside control-plane with an ArchUnit-enforced
boundary — it could use the rules read API and nothing else, no repositories, no entities. That boundary
ran green for a full phase, which is what made the rest mechanical. Then the **data cut**: rules reach
the engine only via `config.changes.v1`, the compacted CDC topic, never by a join — so the engine already
owned its own state before it owned its own process. Then I stood the service up consuming the same
topics under a different consumer group, publishing to `alerts.shadow.v1`, and ran a comparator joining
live and shadow on a deterministic `alertId = hash(ruleId, windowStart)`; the exit criterion was zero
mismatches over an hour under load. Cutover is a property, `alerts.engine=service`, which stops the
in-monolith listener via `SmartLifecycle` — committing its offsets so the boundary is exact — and
promotes the service's output. Because the alert id is deterministic, an overlap produces duplicates
that the consumer dedupes rather than pages. And the flag flips back in seconds.

**The sequence, with the decision at each step:**

| Step | What | Decision & exit criterion |
|---|---|---|
| 0 | module inside control-plane, ArchUnit-fenced | REQ-SF-01. Exit: rules green for a release; engine touches no admin repository |
| 1 | **data cut**: rules via `config.changes.v1`, not a join | REQ-SF-02. Decision: replicated read model, not an API call — evaluation runs at partition rate, rule changes are rare (Q7). Exit: the engine's rule cache is rebuilt from the compacted topic on cold start, with no DB access |
| 2 | new service deployed, zero traffic | Exit: healthy, dashboards live, consumer group registered |
| 3 | **shadow**: same input, `alerts.shadow.v1`, comparator | REQ-SF-03. Exit: 0 mismatches / 1 h under load, with the mismatch taxonomy split three ways |
| 4 | **cutover**: `alerts.engine=service` | REQ-SF-04. Decision: big-bang flag, not a ramp — you cannot split a windowed Kafka consumer by percentage (Q6). Safety: ordered stop + deterministic ids |
| 5 | remove the in-monolith engine and the flag | Exit: no flip-backs for N weeks, zero traffic on the old path. Deletion date in the ADR |

**What made it safe, in one list:**
- The **boundary existed before the network did** — the expensive discovery (a hidden dependency) happens
  at compile time, not in production.
- The **data cut came first**. If step 1 had failed, steps 2–5 would have failed more expensively.
- **Shadow running at production volume** found the differences with zero user exposure.
- **Deterministic ids** made the one thing you cannot fully control — the overlap window — harmless.
- **`SmartLifecycle`** gave the old listener a clean stop with committed offsets, so there was no gap
  (this is BS-01's ordered shutdown paying a second dividend — file 01, Q2).
- The **rollback is a property flip**, not a deploy.

**What it cost, stated honestly:**
- One more deployable, pipeline, dashboard, runbook and on-call surface.
- End-to-end alert latency gained a Kafka hop — measured and inside the freshness SLO, but not free.
- The engine's rule view is now **eventually consistent**: a rule edited at T is live in the engine at
  T + tens of ms. For alerting that is fine, and it is written down as a number rather than assumed.
- Two code paths and a flag for the duration of the migration, with a deletion date.

**Follow-ups they'll ask:**
- *"Why extract this and nothing else?"* — it is the only component in StreamPulse with a genuinely
  different scaling axis and a measured interference problem. Everything else stays in its module.
  Knowing where to stop is the point of REQ-SF-06.
- *"What would you have done if the shadow run showed mismatches?"* — not cut over. Classify them
  (missing / extra / value-differs — each has a different root cause), fix, re-run the hour. The
  comparator staying on *after* cutover is also worth it for a while, because it becomes a regression
  detector for the new engine.
- *"What if the monolith had no module boundary to start with?"* — then step 0 is the whole project, and
  I'd say so up front: characterisation tests first to pin current behaviour, then the interface, then
  the module, then everything above. Attempting step 2 without step 0 is how migrations become
  eighteen-month projects that get cancelled.

> **Your story:** this is where your monolith-to-microservices work goes, and it is likely the highest-
> value story in this file. Frame it in the same shape: what was the *measured* driver (not "it was a
> monolith"), where did you cut the data, how did you de-risk the cutover, what was the rollback, and
> what did the number do afterwards. Be precise about scope — "I led", "I built", "I was on the team
> that" — because everything here invites three follow-ups. If part of it went badly, that is usually
> the better story: what you would do differently is the most senior thing you can say.

> **Your story:** if you have ever argued *against* a split, or had to merge services back together,
> lead with that. Restraint backed by a reason is rarer and more memorable than another extraction.

**Where in StreamPulse:** BS-08 end to end; REQ-SF-01…06; Phase 6 (v0 module) → after Phase 9
(extraction, using measured load data); `AlertEngineBoundaryArchTest`, `ShadowComparatorIT`,
`CutoverFlagIT`.

---

## Key terms

| Term | One-line meaning |
|---|---|
| **Bounded context** | A boundary within which one model and one ubiquitous language hold; the primary seam-finding tool. |
| **Conway's law** | Systems mirror the communication structures of the organisations that build them. |
| **Change coupling** | Files/modules that change in the same commits; high co-change across a boundary means the boundary is wrong. |
| **Distributed monolith** | Services that cannot be deployed independently — all the costs of distribution, none of the benefits. |
| **Modular monolith** | One deployable with enforced internal boundaries; the split becomes cheap if you ever need it. |
| **Strangler fig** | Incrementally routing functionality from an old system to a new one behind a facade, reversibly. |
| **Shadow traffic / dual-run** | Sending real input to the new implementation with its output discarded, and diffing against the old. |
| **Comparator** | The job that joins live and shadow outputs and exports a typed mismatch metric. |
| **Cutover flag** | A runtime property that switches which implementation is live — deploy ≠ release. |
| **Data ownership** | Exactly one service writes a given piece of data; everyone else reads a copy. |
| **Shared-database anti-pattern** | Several services writing one schema; the schema becomes an undocumented contract. |
| **Derived projection** | A read-optimised copy built by consuming events; throwable-away and rebuildable by replay. |
| **Read-your-own-writes** | The anomaly where a user reads a projection that hasn't caught up with their own write. |
| **Saga** | A sequence of local transactions with compensating actions; orchestrated or choreographed. |
| **Compensation** | A semantic inverse (a refund, a retraction) — not a rollback; intermediate state was visible. |
| **2PC** | Prepare/commit across participants; blocking if the coordinator dies, and rarely usable across modern infrastructure. |
| **Deterministic id** | An output id derived from the input, so two implementations agree and duplicates collapse. |
| **Retry budget** | A system-wide cap on retries as a fraction of traffic — stops per-call retry limits from multiplying load. |
| **Fallacies of distributed computing** | The eight assumptions (reliable network, zero latency, …) that break distributed designs. |

---

## Self-check (answer out loud, no notes)

1. Four lenses for finding a service boundary — name them, and say which one you'd apply first and why.
2. Four services at 99.9% each, one synchronous chain. What is the availability, and what do you do
   about it?
3. Why must the data cut happen before the process cut? Give the concrete BS-08 example.
4. During a cutover, what makes duplicates harmless and what makes gaps impossible? Two different
   mechanisms — name both.
5. Your extraction is live. Name three metrics that should improve and three costs that will have gone
   up.
