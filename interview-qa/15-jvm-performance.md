# 15 — JVM Performance: Memory, GC, JIT, Leaks and Production Diagnosis

**Why this matters for an Event Hub role:** a telemetry hub is an *allocation machine*. Every event
that arrives is a byte buffer, a deserialized record, a domain object, a serialized output record —
four allocations per event, millions of events per second. At that rate the JVM's memory behaviour
*is* your latency profile: a 200 ms GC pause on the ingest path is 200 ms of consumer lag, and on the
SSE path it is a visible stall on someone's live dashboard. Juniors say "the GC handles it." Seniors
can say which collector is running, what the allocation rate is, where the pause time went, and how
they found out — with an artifact, not an anecdote.

This file is the "something is wrong in production" round. Interviewers use it to separate people who
have *read* about G1 from people who have *sat in front of a flame graph at 2am*.

**Scenario anchor — BS-03 + Phase 9 (the JVM lab):** the championship-final spike drives 10× baseline
load through `ingest-gateway` and `stream-processor`. Three things are deliberately broken and then
found with tools:
1. an induced memory leak (a static map keyed by `sessionId`, never evicted) found via heap dump and
   the Eclipse MAT dominator tree;
2. G1 vs ZGC compared under the same spike, with the pause distribution recorded in
   `docs/perf-report.md`;
3. an async-profiler flame graph showing where wall-clock time actually goes on the request path.

> **Important about numbers in this file:** every GC log excerpt, flame-graph shape and pause figure
> below is **constructed for illustration**, not a capture from a real system. The *shapes* are real
> — the digits are made up so you can practise reading them. Replace them with your own Phase 9
> output before you quote any number in an interview.

---

## Q1. Walk me through the JVM's runtime memory areas. Why can a container OOMKill a JVM whose heap looks fine?

**30-second answer:** Heap is only one of several regions, and it is the only one `-Xmx` bounds. The
JVM process also holds Metaspace (class metadata, native memory, only loosely bounded by default), one
stack per platform thread (`-Xss`, default around a megabyte, so a thousand threads is a gigabyte of
native memory), the JIT code cache, GC bookkeeping structures that scale with heap size, direct
`ByteBuffer`s used by NIO and by Kafka/Netty, and the native allocator's own arenas. A container limit
applies to **all** of that. So the classic production surprise is: heap graph is flat at 60%, no
`OutOfMemoryError` in the logs, and Kubernetes still kills the pod with exit code 137 — because the
native side grew. The right mental model is "`-Xmx` is a fraction of the container limit, not the
limit."

**Deeper:**

```
                     ┌──────────────────────────── container memory limit ────────────────────────────┐
                     │                                                                                │
  JVM process        │  ┌───────── Java heap (-Xmx) ─────────┐  ┌── Metaspace ──┐ ┌── code cache ──┐   │
  address space      │  │ young (eden + 2 survivor) │ old    │  │ class meta,   │ │ JIT-compiled   │   │
                     │  │  ← TLABs carved from eden          │  │ native mem    │ │ native code    │   │
                     │  └────────────────────────────────────┘  └───────────────┘ └────────────────┘   │
                     │  ┌── thread stacks ──┐ ┌── GC structures ──┐ ┌── direct/mapped ByteBuffers ──┐  │
                     │  │ N × -Xss          │ │ card tables,      │ │ Kafka/Netty network buffers,  │  │
                     │  │ (platform threads)│ │ remembered sets,  │ │ memory-mapped files           │  │
                     │  └───────────────────┘ │ mark bitmaps      │ └───────────────────────────────┘  │
                     │  ┌── malloc arenas, JIT compiler scratch, JNI, symbol/string tables ─────────┐  │
                     │  └──────────────────────────────────────────────────────────────────────────┘  │
                     └────────────────────────────────────────────────────────────────────────────────┘
```

| Region | Bounded by | Grows with | Typical failure |
|---|---|---|---|
| Heap | `-Xmx` / `-XX:MaxRAMPercentage` | live object graph, allocation rate | `OutOfMemoryError: Java heap space` |
| Metaspace | `-XX:MaxMetaspaceSize` (unbounded-ish by default) | number of loaded classes | `OutOfMemoryError: Metaspace` — classloader leak, or code-generation gone wild |
| Thread stacks | `-Xss` × thread count | **platform** threads | `OutOfMemoryError: unable to create native thread`, or silent native growth |
| Code cache | `-XX:ReservedCodeCacheSize` | compiled methods, inlining | "CodeCache is full. Compiler has been disabled" → everything reverts to interpreted, throughput collapses |
| Direct / mapped buffers | `-XX:MaxDirectMemorySize` | NIO, Kafka client, Netty, gRPC | `OutOfMemoryError: Direct buffer memory`, or native growth if buffers are pooled and never released |
| GC structures | proportional to heap, collector-specific | heap size, region count | quiet native overhead — ZGC and Shenandoah cost more here than Parallel |

**The container arithmetic that actually matters.** Illustrative, with the assumptions stated:

```
Container limit                                 2048 MB   (assumption: what you set in the Pod spec)
  - thread stacks: 200 platform threads × 1 MB  ~200 MB   (assumption: -Xss default ≈ 1 MB)
  - Metaspace (a mid-size Spring Boot app)      ~150 MB   (assumption — MEASURE yours)
  - code cache                                  ~ 80 MB
  - direct buffers (Kafka producer + consumer)  ~128 MB   (assumption: bounded by config)
  - GC + JVM native overhead                    ~150 MB
  ─────────────────────────────────────────────────────
  headroom needed before heap                   ~708 MB
  → -Xmx of about 1200-1300 MB, i.e. ~60-65% of the limit
```

That is why `-XX:MaxRAMPercentage=65` is a more honest container setting than `-Xmx1900m`. Do not
treat 65 as a law — it is a starting point you then correct with Native Memory Tracking.

**Code:**

```bash
# Enable Native Memory Tracking (small, permanent overhead — acceptable in a perf environment,
# usually left OFF in prod unless you are hunting native growth).
java -XX:NativeMemoryTracking=summary -jar app.jar

# Then, against the running PID:
jcmd <pid> VM.native_memory summary          # where the native bytes actually are
jcmd <pid> VM.native_memory baseline         # snapshot...
# ...wait for the growth to happen...
jcmd <pid> VM.native_memory summary.diff     # ...and diff: this is the leak finder for native memory

# Fast orientation without NMT:
jcmd <pid> GC.heap_info                      # heap used/committed/max
jcmd <pid> VM.flags -all | grep -i -E 'MaxHeapSize|MaxMetaspace|ReservedCodeCache|MaxDirectMemory'
```

**Follow-ups they'll ask:**
- *"Where do interned strings and class statics live?"* — since JDK 7/8 the string table and static
  fields live in the heap (not PermGen/Metaspace). Metaspace holds class metadata only. This matters
  because a static `Map` leak shows up as a **heap** leak, not a Metaspace one.
- *"What replaced PermGen and why?"* — Metaspace, in JDK 8. It is native memory that grows on demand,
  which removed the classic "PermGen full" but replaced it with "the container killed us" if you never
  set `MaxMetaspaceSize`.
- *"How do you bound direct memory?"* — `-XX:MaxDirectMemorySize`, and more importantly configure the
  clients: Kafka's `buffer.memory` on the producer, Netty's pooled allocator settings. Leaving it
  implicit means it defaults to roughly `-Xmx`, which doubles your worst case.
- *"Compressed oops?"* — below a heap of roughly 32 GB the JVM stores 32-bit object references,
  saving significant memory. Crossing that boundary can make a 33 GB heap hold *less* live data than a
  31 GB one. Worth knowing as a sizing cliff.

**Pitfalls / red flags:** "the JVM uses `-Xmx` memory" — full stop. Or treating OOMKilled and
`OutOfMemoryError` as the same event (Q13).

**Where in StreamPulse:** Phase 9 (JVM lab), Phase 11 (K8s resource requests/limits tuned from Phase 9
data), BS-01 (a pod that dies badly is exactly what the graceful-shutdown work is defending against).

**Verify:** default `-Xss` and the compressed-oops threshold vary by platform and JDK build —
confirm with `java -XX:+PrintFlagsFinal -version | grep -i -E 'ThreadStackSize|UseCompressedOops'` on
your actual image.

---

## Q2. Explain the generational hypothesis, TLABs, and why allocation rate is the metric you watch.

**30-second answer:** The generational hypothesis is the empirical observation that most objects die
very young — a request-scoped DTO lives microseconds, a cache entry lives hours, and there is very
little in between. Generational collectors exploit that by collecting the young generation frequently
and cheaply: the cost of a young collection is proportional to the *surviving* objects, not to the
garbage, so if 98% of eden is dead the collection is nearly free. Allocation itself is a pointer bump
inside a thread-local allocation buffer, so it costs a few nanoseconds with no locking. The metric I
actually watch is **allocation rate in MB/s**, because it directly sets young-GC *frequency*, and
frequency times pause is your GC overhead. Tuning a collector without first asking "why are we
allocating 2 GB/s?" is treating the symptom.

**Deeper:**

Flow of an allocation:

```
new TelemetryEvent(...)
   │
   ├─ fits in the thread's TLAB?  ──yes──▶ bump a pointer. No lock, no CAS. ~ a few ns.
   │                                       (TLABs are private slices of eden; this is why
   │                                        allocation scales linearly with cores)
   ├─ TLAB exhausted? ──▶ retire it, request a new TLAB from eden (a CAS on a shared cursor)
   │
   ├─ object too big for a TLAB? ──▶ allocate directly in eden (slow path, shared)
   │
   └─ eden full? ──▶ YOUNG COLLECTION
                      • trace from GC roots (stacks, statics, JNI) through young objects only
                      • copy survivors to a survivor space, bumping their age
                      • age > MaxTenuringThreshold, or survivor space overflows → promote to old
                      • reclaim all of eden in O(1) by resetting the pointer
```

Two consequences seniors are expected to name:

1. **Young GC cost ∝ live set, not heap size.** Doubling eden does not double the pause; it halves the
   *frequency*. That is the real lever: a bigger young generation trades more infrequent, slightly
   longer pauses for less total overhead — until the extra pause breaks your latency SLO.
2. **Premature promotion is the pathology.** If survivor spaces are too small, or the live set spikes
   (a burst of buffered events during a Kafka rebalance), objects that would have died get copied into
   the old generation. Old-generation growth then forces concurrent marking and mixed/full collections
   — which is how a *throughput* problem turns into a *pause* problem. In StreamPulse the classic
   trigger is BS-03's spike: at 10× load the in-flight batch set is 10× larger and suddenly survives.

**Allocation rate arithmetic** (this is the number to compute out loud):

```
Assumption (illustrative): eden is 1 GB; GC log shows a young collection every 2 s under steady load.
  allocation rate ≈ 1 GB / 2 s = 500 MB/s

At 10× load (BS-03):                    ≈ 5 GB/s  → a young GC every 200 ms
If each young pause is 15 ms:            GC overhead ≈ 15 / 200 = 7.5% of wall time
                                         and a 15 ms stall every 200 ms is VISIBLE at p99.
```

The fix list, in the order I would actually try it:
1. **Allocate less.** Reuse buffers, avoid boxing in hot loops, avoid `String.format` on the hot path,
   deserialize lazily, use primitive collections where it matters. This is almost always the biggest
   win and the one people skip.
2. **Make eden bigger** so the same allocation rate produces fewer collections.
3. **Change collector** only if the pause profile, not the total overhead, is the problem.

**Code:**

```bash
# Get the allocation rate without guessing: JFR has an allocation view, and the GC log gives it
# indirectly (eden size reclaimed ÷ interval between young collections).
jcmd <pid> JFR.start name=alloc settings=profile duration=60s filename=/tmp/alloc.jfr
# Then open /tmp/alloc.jfr in JDK Mission Control → "TLAB Allocations" / "Allocation" pages,
# which attribute allocation BY CALL SITE — that is what tells you which code to fix.
```

```java
// ❌ Allocates a new String, a new StringBuilder, and boxes the long — per event, at 5M events/s.
log.debug("event " + event.id() + " for tenant " + tenantId + " seq=" + seq);

// ✅ Parameterized logging: the array is still allocated, but formatting is skipped entirely
//    when DEBUG is off, and the guard removes even that.
if (log.isDebugEnabled()) {
    log.debug("event {} for tenant {} seq={}", event.id(), tenantId, seq);
}
```

**Follow-ups they'll ask:**
- *"Is allocation expensive in Java?"* — no; it is a pointer bump, often cheaper than `malloc`. The
  cost is not the allocation, it is the *collection* it eventually forces. That reframing is the point.
- *"What is `MaxTenuringThreshold`?"* — the survivor age at which an object is promoted to old. G1
  adapts it dynamically; setting it manually is rarely right and mostly a way to hide a sizing problem.
- *"What is a humongous object in G1?"* — an object larger than half a G1 region. It is allocated
  directly in the old generation across contiguous regions, is not compacted normally, and can cause
  fragmentation. Big `byte[]` batch buffers are the usual source — very relevant to a telemetry
  ingest service that builds large arrays before an S3 flush.
- *"Do escape-analysed objects count toward allocation rate?"* — no, if they are scalar-replaced they
  are never allocated at all (Q3).

**Pitfalls / red flags:** jumping straight to "I'd increase the heap" without knowing the allocation
rate or the live-set size.

**Where in StreamPulse:** BS-03 (10× spike), Phase 9 (`docs/perf-report.md` records allocation rate
before and after the buffer-reuse change), BS-02 / REQ-CC-04 (the S3 offload buffers are the
humongous-object suspect).

---

## Q3. What is escape analysis, and what does the JIT do with a short-lived object?

**30-second answer:** Escape analysis is a JIT optimization that proves an object never becomes
reachable outside the compiling method — it does not "escape" to a field, a static, another thread, or
a return value. If it can prove that, C2 can do **scalar replacement**: delete the allocation entirely
and keep the object's fields in registers or on the stack. It can also eliminate the associated locks
(lock elision) since no other thread can see the object. The practical consequence is that a lot of
"obviously wasteful" small allocations cost nothing in a hot, inlined method — and equally, that a
micro-optimization you make to avoid an allocation may be optimizing something the JIT already
removed. That is a big part of why you cannot reason about performance without measuring (Q8).

**Deeper:**

Object layout, conceptually, on HotSpot with compressed oops:

```
┌──────────────┬──────────────┬──────────────────────┬──────────┐
│ mark word    │ klass ptr    │ instance fields      │ padding  │
│ (8 bytes)    │ (4 compressed│ (aligned, reordered  │ to 8-byte│
│ hash, GC age,│  / 8 bytes)  │  by the JVM, not by  │ boundary │
│ lock state   │              │  source order)       │          │
└──────────────┴──────────────┴──────────────────────┴──────────┘
```

So a "tiny" object with one `long` field is not 8 bytes — it is header plus field plus padding. At
millions of objects per second the header overhead alone is real. This is the argument for value-type
style designs (packing several fields into a `long`, using primitive arrays instead of
`List<Integer>`) *on the hot path only*.

Escape analysis has three outcomes:

| Escape state | Meaning | JIT can |
|---|---|---|
| NoEscape | never visible outside the method | scalar-replace (no allocation at all), elide locks |
| ArgEscape | passed to a callee but does not outlive the call | elide locks, sometimes stack-allocate |
| GlobalEscape | stored in a field/static/returned/thrown | nothing — real heap allocation |

Two things that defeat it, worth naming: **inlining failure** (escape analysis runs after inlining, so
if the method is too big to inline the object escapes into the callee's frame and the analysis gives
up), and **megamorphic call sites** (an interface with many implementations called from one place —
the JIT cannot devirtualize, so it cannot inline, so it cannot analyse).

**Code:**

```java
// The JIT can usually scalar-replace `window` here: it never leaves the method.
// After C2 compiles this, there may be NO allocation at all — just two long registers.
long rebufferRatio(long bufferingMs, long watchedMs) {
    var window = new Window(bufferingMs, watchedMs);   // candidate for scalar replacement
    return window.ratioBasisPoints();
}

// ...but one change makes it GlobalEscape and the allocation is real again:
private Window last;                                   // a field
long rebufferRatio(long bufferingMs, long watchedMs) {
    this.last = new Window(bufferingMs, watchedMs);    // escapes → heap allocation, every call
    return last.ratioBasisPoints();
}
```

```bash
# These are diagnostic flags — useful in a lab, not in production.
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining -XX:+PrintCompilation -jar app.jar
# Escape analysis and scalar replacement are ON by default on C2; you turn them OFF to
# measure their effect, which is the only honest way to prove they mattered:
java -XX:-DoEscapeAnalysis -jar app.jar
java -XX:-EliminateAllocations -jar app.jar
```

**Follow-ups they'll ask:**
- *"Does Java have stack allocation?"* — not as a language feature. Scalar replacement is the closest
  thing and it is a JIT optimization, not a guarantee. Never *depend* on it.
- *"Why is `synchronized` sometimes free?"* — lock elision on non-escaping objects, plus biased
  locking historically (biased locking was deprecated and later removed — verify for your JDK).
- *"How does this interact with records?"* — records are ordinary heap objects; they get the same
  treatment. Their value is API clarity, not a different memory model. Project Valhalla's value
  classes are the thing that would change that, and it is not final — do not claim it as available.

**Pitfalls / red flags:** claiming "the JVM stack-allocates small objects" as a rule. It is an
optimization the compiler *may* apply after inlining succeeds.

**Where in StreamPulse:** Phase 9; the JMH micro-benchmark in BS-02 / REQ-CC-06 (`AtomicLong` vs
`LongAdder` vs `synchronized`) is exactly the kind of measurement where escape analysis silently
changes the answer if you write the benchmark badly.

**Verify:** biased-locking status, and whether `-XX:+EliminateLocks` is still a separate flag, differ
across JDK versions. Check `-XX:+PrintFlagsFinal` on your JDK.

---

## Q4. Compare Serial, Parallel, G1, ZGC and Shenandoah. How do you choose?

**30-second answer:** They sit on one trade-off curve: throughput and footprint on one end, pause time
on the other. Serial and Parallel are stop-the-world collectors that maximise throughput per CPU —
fine for batch jobs and tiny containers. G1 is the general-purpose default: region-based, mostly
concurrent marking, with a *pause-time goal* it tries to meet by choosing how many regions to collect.
ZGC and Shenandoah do the evacuation itself concurrently, so pauses become roughly independent of heap
size — sub-millisecond in ZGC's design — but you pay in CPU and footprint because every load goes
through a barrier. My selection rule: start with G1; move to ZGC or Shenandoah when the measured p99
is dominated by GC pause and I have CPU headroom to spend; move to Parallel when it is a throughput
job that nobody is waiting on. And I would never change collector before I had a GC log proving the
pause was the problem.

**Deeper:**

| | Serial | Parallel | G1 | ZGC | Shenandoah |
|---|---|---|---|---|---|
| Young collection | STW, 1 thread | STW, N threads | STW, N threads (region-based) | concurrent (generational mode) | concurrent |
| Old collection | STW mark-compact | STW mark-compact | concurrent mark + **STW mixed evacuation** | concurrent mark + **concurrent relocate** | concurrent mark + **concurrent evacuate** |
| Pause scales with | heap | heap | *collection set size* (bounded by the goal) | root-set size, ~not heap | root-set size, ~not heap |
| Compaction | yes | yes | yes (evacuating) | yes (concurrent) | yes (concurrent) |
| Key mechanism | — | — | regions, remembered sets, SATB marking | colored pointers + **load barriers** | **load reference barriers**, forwarding |
| Throughput | good (single core) | **best** | good | lower (barrier cost) | lower (barrier cost) |
| Footprint / CPU | lowest | low | moderate | **highest** | high |
| Sweet spot | tiny heaps, CLI tools, single-CPU containers | batch, ETL, anything where only total runtime matters | 2–32 GB, mixed latency/throughput, the default | large heaps + strict tail latency | same niche as ZGC, different vendor lineage |

**What each mechanism actually buys you:**

- **G1: regions + a collection set.** The heap is split into equal regions (typically 1–32 MB). At
  each young collection G1 picks a *collection set* and evacuates it. Because it chooses how many
  regions to include, it can aim at `MaxGCPauseMillis`: fewer regions → shorter pause → but the old
  generation fills faster. Concurrent marking (SATB — snapshot-at-the-beginning) runs alongside the
  application and identifies which old regions are mostly garbage; those get folded into subsequent
  **mixed** collections. G1's failure mode is *evacuation failure / to-space exhausted*: it wanted to
  copy survivors and had nowhere to put them, so it falls back to a full compacting collection — the
  multi-second pause you see in Q6's log.
- **ZGC: colored pointers + load barriers.** Metadata bits are stored in the pointer itself; a load
  barrier on every reference read checks those bits and, if the object has been relocated, fixes the
  reference in place. That is what allows relocation to happen *while the application runs*, which is
  what makes pause time independent of live-set size. The cost is a barrier on every load and extra
  memory for the mapping.
- **Shenandoah: load reference barriers.** Same goal, different implementation lineage (Red Hat);
  concurrent evacuation with barriers that forward references to the moved copy.

**Choosing — what actually changes the answer:**

| If… | then… | because |
|---|---|---|
| heap < ~4 GB, latency matters modestly | G1 | defaults are good; ZGC's overhead is not repaid at small heaps |
| heap is large (tens of GB) **and** p99 is GC-bound | ZGC / Shenandoah | pause stops scaling with heap |
| you are CPU-constrained (tight container CPU limit) | G1 or Parallel | concurrent collectors *spend CPU* to buy pause time you may not have to spend |
| it is a batch/ETL job, throughput only | Parallel | no barrier overhead, best mutator throughput |
| single-CPU container, small heap, short-lived | Serial | lowest overhead; the JVM may select it automatically |
| allocation rate is the problem | **none of the above** | fix the allocation first (Q2) |

**Code:**

```bash
# Selection is one flag each. Always paired with a GC log, or you are guessing.
java -XX:+UseG1GC          -Xlog:gc*:file=/var/log/gc.log:time,uptime,level,tags:filecount=5,filesize=20M -jar app.jar
java -XX:+UseZGC           -Xlog:gc* ... -jar app.jar
java -XX:+UseParallelGC    -Xlog:gc* ... -jar app.jar
java -XX:+UseShenandoahGC  -Xlog:gc* ... -jar app.jar   # availability depends on your JDK build/vendor

# What is actually running right now (never assume — the JVM's ergonomic default depends on
# the container's CPU and memory):
jcmd <pid> VM.flags | tr ' ' '\n' | grep -i -E 'UseG1GC|UseZGC|UseParallelGC|UseSerialGC|UseShenandoah'
```

**Follow-ups they'll ask:**
- *"What is the default collector?"* — G1 on a "server-class" machine, but the JVM's ergonomics can
  select Serial on a small single-CPU container. That surprise is common in Kubernetes with a CPU
  limit of 1. Always check rather than assume.
- *"Is CMS still an option?"* — no, CMS was deprecated in JDK 9 and removed in JDK 14. If a codebase
  still passes `-XX:+UseConcMarkSweepGC` it will refuse to start on a modern JDK. Good thing to
  mention, because it shows you have done an upgrade.
- *"Does ZGC have a young generation?"* — generational ZGC was added later than ZGC itself; in the
  JDK 21 timeframe it was opt-in via a flag, and in a later JDK it became the default with the
  non-generational mode deprecated and then removed. **This is exactly the kind of detail to state
  carefully and then verify** — see the Verify line.
- *"When would you tune region size?"* — `-XX:G1HeapRegionSize` only when you have humongous-object
  churn (objects > half a region) that you cannot eliminate at the source. Evidence first.

**Pitfalls / red flags:** "ZGC is better, so use ZGC." It is not better, it is *different*: you pay
throughput and footprint for pause. A candidate who cannot name the cost has read a blog post.

**Where in StreamPulse:** Phase 9 — the G1 vs ZGC comparison under the BS-03 spike, with the pause
distribution (not the mean — the p99 and max) recorded in `docs/perf-report.md`.

**Verify:** generational ZGC's flag name and default status changed across JDK 21 → 24. Check
`java -XX:+PrintFlagsFinal -version | grep -i ZGenerational` and the release notes for your exact JDK
before you name a flag in an interview. Shenandoah is not present in every vendor's build.

---

## Q5. Which GC tuning is legitimate, and which is usually a mistake?

**30-second answer:** The legitimate list is short: set `-Xms` equal to `-Xmx` (or use
`MaxRAMPercentage` in a container) so the heap never resizes; pick a collector on evidence; set a
pause *goal* if you are on G1 and you know your latency budget; bound the things that are unbounded by
default — Metaspace, direct memory, code cache. Everything past that — survivor ratios, tenuring
thresholds, IHOP, region sizes, GC thread counts — is a mistake *unless a GC log told you to*. The
reason is that modern collectors are adaptive: hand-setting a value freezes an adaptive controller at
whatever was right on the day you measured, and it silently becomes wrong when the workload shifts.
The highest-leverage "GC tuning" I have done is usually a code change that stops allocating.

**Deeper:**

**Legitimate, with the reason:**

| Setting | Why |
|---|---|
| `-Xms` == `-Xmx` (server) | The JVM starts small and grows, and each growth is a GC plus an OS-level commit. Equal values mean the heap is committed once at startup — no resize pauses, no surprise at traffic onset, and you find out at boot if the memory isn't there. |
| `-XX:MaxRAMPercentage=<n>` (container) | The correct container idiom: the JVM computes `-Xmx` from the *cgroup* limit, so one image works across Pod sizes. Pair with `InitialRAMPercentage` for the `-Xms` == `-Xmx` effect. |
| `-XX:MaxMetaspaceSize` | Turns an unbounded native growth into a diagnosable `OutOfMemoryError: Metaspace` with a heap dump, instead of an OOMKill with no evidence. |
| `-XX:MaxDirectMemorySize` | Same argument for NIO/Netty/Kafka buffers. |
| `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=<vol>` | Costs nothing until it matters; without it, your one reproduction is wasted. The path must be a volume that survives the container. |
| `-Xlog:gc*:file=...` with rotation | You cannot diagnose a pause you did not record. |
| `-XX:MaxGCPauseMillis=<n>` (G1 only) | A **goal**, not a guarantee: G1 shrinks the collection set to try to meet it. Setting it too low makes G1 collect too little per cycle, the old generation fills, and you get *more* full GCs. |
| `-XX:+ExitOnOutOfMemoryError` or `+CrashOnOutOfMemoryError` | For a replicated stateless service, dying fast and letting the orchestrator replace you beats limping along in a GC death spiral. A deliberate decision, not a default. |

**Usually a mistake:**

| Anti-tuning | Why it backfires |
|---|---|
| `-XX:MaxGCPauseMillis=10` on a 16 GB heap with G1 | G1 shrinks the collection set to chase an impossible goal, promotion accelerates, and you trade frequent short pauses for occasional multi-second full GCs. If you need 10 ms, you need a different collector, not a smaller number. |
| Hand-setting `SurvivorRatio`, `MaxTenuringThreshold`, `NewRatio` on G1 | G1 sizes the young generation adaptively to meet the pause goal. Pinning young size *disables* that adaptation. |
| `-XX:InitiatingHeapOccupancyPercent` without evidence | IHOP is adaptive in modern G1. Fixing it is only right if you have logs showing marking starts too late (to-space exhaustion) or too early (wasted CPU). |
| Copying flags from a blog | Different heap size, different allocation rate, different JDK. The flags that helped someone else's JVM are noise in yours. |
| Adding heap to fix a leak | Buys hours and makes the eventual heap dump bigger and slower to analyse. |
| `-XX:+AggressiveOpts`, `-XX:+UseFastAccessorMethods` and friends | Long obsolete; many were removed and will fail to start a modern JVM. |
| Tuning GC when the CPU is the bottleneck | Check the flame graph first. |

**Code — a defensible container baseline:**

```bash
JAVA_TOOL_OPTIONS="\
  -XX:MaxRAMPercentage=65 -XX:InitialRAMPercentage=65 \   # heap ≈ 65% of the cgroup limit, committed up front
  -XX:MaxMetaspaceSize=256m \                             # bound it so it fails loudly, not silently
  -XX:MaxDirectMemorySize=256m \                          # Kafka/Netty buffers cannot eat the container
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/dumps \                               # MUST be a mounted volume that outlives the pod
  -XX:+ExitOnOutOfMemoryError \                           # deliberate: die, get replaced, don't limp
  -Xlog:gc*,safepoint:file=/var/log/gc.log:time,uptime,level,tags:filecount=5,filesize=20M \
  -XX:StartFlightRecording=name=cont,settings=profile,maxsize=256m,maxage=12h,dumponexit=true,filename=/dumps/app.jfr \
  -XX:NativeMemoryTracking=summary"                       # keep in perf envs; weigh the overhead for prod
```

Two notes I would state unprompted: `-XX:+ExitOnOutOfMemoryError` and
`-XX:+HeapDumpOnOutOfMemoryError` together means "write the dump, then die" — check that ordering
holds on your JDK before relying on it, because losing the dump defeats the purpose. And a heap dump
of a 4 GB heap is a multi-gigabyte file written during an already-degraded moment; make sure the disk
and the shutdown timeout can take it (this interacts directly with BS-01's grace period).

**Follow-ups they'll ask:**
- *"Why does `-Xms` == `-Xmx` matter more in a container?"* — because the container's memory is
  reserved anyway. A JVM that starts at 256 MB and grows to 2 GB gives you an OOMKill at minute 40 of
  a load test instead of a failure to start at second 0. Fail early.
- *"What about `-XX:+UseStringDeduplication`?"* — G1 can deduplicate equal `String` character arrays
  during collection. It genuinely helps workloads with lots of repeated strings — a telemetry pipeline
  with repeated tenant/title/region labels is a plausible candidate. Measure it; it is not free.
- *"How do you roll out a GC change safely?"* — canary one instance, compare pause distributions
  (p99 and max, not mean) and throughput on the same load, keep the old flags one flip away.

**Pitfalls / red flags:** a candidate who recites ten flags. The senior answer is short and each entry
has a *reason*.

**Where in StreamPulse:** Phase 9 (the flag baseline lives in `docs/perf-report.md` with the
justification per flag), Phase 11 (the same flags in the Helm values, sized from Phase 9 data).

**Verify:** flag availability and defaults are JDK- and vendor-specific. Check every flag above with
`java -XX:+PrintFlagsFinal -version` on your actual base image before putting it in a Dockerfile.

---

## Q6. Show me how you read a GC log. What does a long-pause pattern tell you?

**30-second answer:** I read four things, in this order: the *pause distribution* (max and p99, never
the mean), the *heap occupancy after* each collection, the *interval* between collections, and the
*cause* string. Rising post-collection occupancy across many cycles means the live set is growing —
that is a leak or a genuinely growing cache, and no collector setting will save you. Flat occupancy
with short frequent pauses means a high allocation rate — a code problem. And an isolated
multi-second `Pause Full` preceded by "to-space exhausted" is G1 failing to evacuate: it ran out of
free regions, which means either the heap is too small for the live set, or promotion is happening
faster than concurrent marking can reclaim.

**Deeper — a constructed G1 log, annotated.** (Illustrative; not a real capture.)

```
# Healthy steady state: eden reclaimed, occupancy returns to ~the same level, pauses ~10-20 ms.
[  2.456s][info][gc] GC(12) Pause Young (Normal) (G1 Evacuation Pause)  1024M->118M(4096M)  14.325ms
[  4.512s][info][gc] GC(13) Pause Young (Normal) (G1 Evacuation Pause)  1024M->121M(4096M)  13.902ms
[  6.588s][info][gc] GC(14) Pause Young (Normal) (G1 Evacuation Pause)  1026M->119M(4096M)  15.110ms
#          ▲                ▲                                           ▲      ▲    ▲         ▲
#          │                │                                           │      │    │         └ pause
#          │                │                                           │      │    └ heap capacity
#          │                │                                           │      └ occupancy AFTER  ← watch this
#          │                │                                           └ occupancy BEFORE
#          │                └ cause: normal eden fill
#          └ interval ≈ 2.06 s → allocation rate ≈ (1024-119) MB / 2.06 s ≈ 440 MB/s
```

```
# Load spike (BS-03). Interval collapses; pauses lengthen because more objects SURVIVE.
[ 61.100s][info][gc] GC(88) Pause Young (Normal) (G1 Evacuation Pause)  1024M->402M(4096M)  38.771ms
[ 61.310s][info][gc] GC(89) Pause Young (Concurrent Start) (G1 Evacuation Pause) 1210M->600M(4096M) 41.203ms
[ 61.311s][info][gc] GC(90) Concurrent Mark Cycle
[ 61.702s][info][gc] GC(90) Concurrent Mark Cycle 391.004ms          ← concurrent: NOT a pause
[ 61.930s][info][gc] GC(91) Pause Young (Prepare Mixed) (G1 Evacuation Pause) 1400M->710M(4096M) 33.115ms
[ 62.140s][info][gc] GC(92) Pause Young (Mixed) (G1 Evacuation Pause) 1502M->640M(4096M)  44.980ms
#   interval ≈ 210 ms → allocation rate ≈ 5 GB/s. GC overhead ≈ 40/210 ≈ 19% of wall time.
#   Mixed collections are now reclaiming old regions — this is G1 working as designed.
```

```
# The pathology. Occupancy after each collection keeps climbing, then evacuation fails.
[301.44s][info][gc] GC(410) Pause Young (Normal) 3810M->3402M(4096M)  92.110ms
[302.01s][info][gc] GC(411) Pause Young (Normal) 3901M->3690M(4096M) 121.450ms
[302.60s][info][gc] GC(412) To-space exhausted                        ← no free regions to copy INTO
[302.94s][info][gc] GC(412) Pause Young (Normal) 4010M->3988M(4096M) 341.220ms
[303.02s][info][gc] GC(413) Pause Full (G1 Compaction Pause) 3988M->3901M(4096M) 2104.880ms   ← 2.1 s STW
[305.20s][info][gc] GC(414) Pause Full (G1 Compaction Pause) 3990M->3950M(4096M) 2298.100ms
#   Two full GCs back to back that reclaim almost nothing = the live set really is ~3.9 GB.
#   This is a LEAK (or an undersized heap), not a GC tuning problem. Next step: heap dump (Q12).
```

**The decision table I keep in my head:**

| Pattern in the log | Diagnosis | First action |
|---|---|---|
| Short pauses, short intervals, flat post-GC occupancy | high allocation rate | profile allocation by call site (JFR), reduce garbage; consider bigger eden |
| Post-GC occupancy rising monotonically over minutes/hours | growing live set: leak or unbounded cache | heap dump → MAT dominator tree (Q9, Q12) |
| Occasional long `Pause Full` after "to-space exhausted" | G1 evacuation failure | more heap, or reduce promotion rate; check for humongous allocations |
| Long *concurrent* phases but short pauses | fine — concurrent time is not stop-the-world | nothing; make sure you are not misreading concurrent time as pause time |
| Pause time far above the `MaxGCPauseMillis` goal | goal is unachievable for this live set/heap | raise the goal or change collector; do not lower it further |
| Long pauses with tiny collection sets, and `safepoint` log shows long "time to safepoint" | **not GC at all** — a thread is slow to reach a safepoint (a long counted loop, a slow JNI call) | `-Xlog:safepoint`; this is the trap that makes people blame GC wrongly |

That last row is the one that impresses. Total pause time includes *time to safepoint*, and a
long-running counted loop without a safepoint poll can stall every other thread while GC waits. If
`-Xlog:safepoint` shows most of the pause is "reaching safepoint" rather than the operation itself,
your problem is in application code, not the collector.

**Code:**

```bash
# Enable everything you need to answer the table above, with rotation so it survives a long soak.
-Xlog:gc*,gc+heap=info,safepoint:file=/var/log/gc.log:time,uptime,level,tags:filecount=10,filesize=20M

# Quick triage without a tool: the pauses, sorted.
grep -oE '[0-9]+\.[0-9]+ms' /var/log/gc.log | sort -g | tail -20      # worst 20 pauses
grep -c 'Pause Full' /var/log/gc.log                                   # full GCs — should be ~0
grep 'To-space exhausted' /var/log/gc.log                              # evacuation failures

# For anything beyond triage, use a log analyser (GCeasy, GCViewer or JDK Mission Control)
# to get the pause histogram and the occupancy-after-GC trend line.
```

**Follow-ups they'll ask:**
- *"What is a safepoint?"* — a point where all application threads are stopped and the heap is in a
  consistent state, so the JVM can do GC, deoptimization, biased-lock revocation, thread dumps and so
  on. Threads poll for it; a thread that does not poll (long counted loop, native call) delays
  everyone.
- *"Mean pause vs max pause?"* — the mean is meaningless for latency. A p99 SLO is broken by the max.
  I report the pause *distribution*.
- *"Is concurrent time free?"* — no: it is CPU stolen from the application. On a CPU-limited container
  concurrent marking can raise your latency without ever showing up as a pause.

**Pitfalls / red flags:** reading the "before → after" numbers and ignoring the "after" trend; or
quoting an average pause.

**Where in StreamPulse:** Phase 9; the before/after GC-log excerpts for the induced leak and for the
G1-vs-ZGC comparison are committed to `docs/perf-report.md`.

**Verify:** unified-logging tag names and decorator syntax (`-Xlog:gc*:file=...:time,uptime:filecount=...`)
are stable since JDK 9 but worth confirming with `java -Xlog:help` on your JDK.

---

## Q7. How does the JIT work? Why are the first N requests slow, and what is deoptimization?

**30-second answer:** HotSpot starts by *interpreting* bytecode, counting method invocations and loop
back-edges. When a counter crosses a threshold the method is compiled by C1 — fast to compile,
moderately optimized, and instrumented with profiling counters. If it stays hot, those profiles feed
C2, which produces aggressively optimized code using assumptions taken from the profile: this call
site only ever saw one receiver type, so devirtualize and inline; this branch was never taken, so
don't emit it; this class has no subclasses, so the method is effectively final. Those assumptions can
later be violated — a second implementation gets loaded, the rare branch fires — and then the JVM
**deoptimizes**: it discards the compiled code, rebuilds the interpreter frame mid-execution, and
falls back, recompiling later with the new information. The practical consequence is warmup: your
first few thousand requests run interpreted or C1-compiled code, so any benchmark or load test that
does not warm up is measuring the wrong thing.

**Deeper:**

```
                 invocation & back-edge counters
   ┌────────────┐      ──────────▶      ┌────────────┐     ──────────▶    ┌────────────┐
   │ Interpreter│                       │  C1 (tier 1-3)                  │  C2 (tier 4)│
   │  tier 0    │ ◀── deoptimize ────── │ quick compile,│ ◀── deopt ───── │ aggressive, │
   │            │                       │ + profiling   │                 │ speculative │
   └────────────┘                       └──────────────┘                  └────────────┘
        slow                              ~5-10× faster                    fastest
```

Tiered compilation (on by default) is the whole pipeline: tier 0 interpreter, tiers 1–3 C1 variants
(with/without profiling), tier 4 C2. The point of C1-with-profiling is to get decent speed *while*
collecting the type profile C2 needs.

**What C2 speculates on, and what breaks it:**

| Optimization | Assumption | Invalidated by |
|---|---|---|
| Inlining | callee is small enough and hot | a huge method body; the inlining budget |
| Devirtualization (monomorphic) | this call site has seen exactly one receiver class | loading a second implementation → bimorphic, then **megamorphic** (no inlining) |
| Class hierarchy analysis | this class has no loaded subclass | a subclass being classloaded later |
| Branch pruning / uncommon traps | this branch was never taken during profiling | the branch actually firing |
| Null-check elimination | this reference was never null | a null appearing |
| Loop unrolling, range-check elimination | array bounds provably safe | — |

The **megamorphic call site** point matters for real Spring code: a `Map<String, RuleEvaluator>`
Strategy dispatch (BS-01's `RuleEvaluator`, BS-04's resilience decorators) with many implementations,
called from one hot loop, cannot be devirtualized or inlined. That is usually fine — it is a virtual
call, not a catastrophe — but it is the honest answer to "why is my abstraction slower than the
if-else version?", and it is also why measuring one implementation in isolation misleads you.

**Warmup, concretely.** This is the part that ties to load testing:

```
Phase                    What is running               What you'd measure
─────────────────────────────────────────────────────────────────────────────
request 1..~200          interpreted                   10-50× slower than steady state
request ~200..~10,000    C1-compiled, profiling        2-5× slower
after ~10,000 (varies)   C2-compiled, steady state     the number you actually care about
```

Those counts are **illustrative** — the real thresholds depend on `-XX:CompileThreshold`,
`-XX:Tier3InvocationThreshold` and friends, on the shape of the code, and on the JDK. The *principle*
is what you assert in an interview: a load test with a 10-second ramp and no warmup period is
measuring the interpreter, and a "p99 regression" that disappears after two minutes is warmup, not a
regression.

**Code:**

```bash
# See it happen. In a lab only — this is very verbose.
java -XX:+PrintCompilation -jar app.jar 2>&1 | head -50
#   the "made not entrant" / "made zombie" lines ARE deoptimization events

java -XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining -jar app.jar 2>&1 | grep 'too large\|hot method too big'
#   "@ n  Foo::bar (123 bytes)   too large" tells you an inlining budget was hit

# Prove warmup exists, and quantify it, with JFR rather than a blog post:
jcmd <pid> JFR.start name=warmup settings=profile duration=120s filename=/tmp/warmup.jfr
#   JMC "Compilations" page shows compile activity settling; correlate with your latency series.

# Force interpreted-only, to see the floor (useful for teaching, useless in prod):
java -Xint -jar app.jar
# Force C1-only — a legitimate choice for very short-lived processes (CLI tools, some Lambdas):
java -XX:TieredStopAtLevel=1 -jar app.jar
```

```java
// StreamPulse: warm the hot paths before the load balancer sends real traffic.
// This is the same @EventListener(ApplicationReadyEvent) hook used for connection-pool warmup
// in file 01 Q11 — one mechanism, two reasons.
@Component
class JitWarmup {
    private final IngestUseCase ingest;

    @EventListener(ApplicationReadyEvent.class)
    void warm() {
        var synthetic = TelemetryEvent.synthetic();          // tagged, never published downstream
        for (int i = 0; i < 20_000; i++) {                   // the count comes from MEASUREMENT,
            ingest.validateOnly(synthetic);                  // not from this number being magic
        }
    }
}
```

**Follow-ups they'll ask:**
- *"What is OSR?"* — on-stack replacement: compiling a method whose *loop* is hot while the method
  invocation itself is not, and swapping the running frame over to the compiled version. It is why a
  long `for` loop speeds up mid-execution.
- *"Does the JIT ever make things slower?"* — briefly, yes: compilation consumes CPU, and a
  deoptimization storm (an uncommon trap firing repeatedly) can be pathological. Rare, but the log
  shows it.
- *"How do AOT/CDS/native-image fit?"* — Class Data Sharing reduces class-loading time; GraalVM native
  image removes warmup entirely at the cost of peak throughput and of dynamic features (reflection,
  proxies) needing configuration. For a Spring service that lives for days, JIT peak performance is
  usually worth the warmup; for a Lambda that lives for 200 ms, it is not. That trade-off is the
  answer, not a preference.
- *"How does this interact with blue/green deploys?"* — a freshly started pod is cold. If you shift
  100% of traffic at once you get a latency spike that is not a bug. Ramp, or warm up before readiness.

**Pitfalls / red flags:** "Java is slow because it's interpreted." Or running a Gatling test with no
warmup and presenting the p99.

**Where in StreamPulse:** Phase 9 (Gatling scenarios have an explicit warmup ramp before the measured
window; `docs/perf-report.md` shows the warmup curve), BS-03 (the freshness measurement must exclude
warmup or it is meaningless), Phase 10 (the Spring Cloud Function compaction Lambda is the case where
`TieredStopAtLevel=1` or native image is worth discussing).

**Verify:** compile-threshold flag names and defaults differ by JDK. Do not quote a specific threshold
number; say "thresholds are tunable and JDK-specific" and move on.

---

## Q8. Why does a microbenchmark without JMH lie?

**30-second answer:** Because everything the JVM does to make real code fast makes naive benchmarks
wrong. Dead-code elimination deletes your loop body when the result is unused. Constant folding
evaluates it at compile time when the input is a `static final`. Loop unrolling and OSR change the
shape of what you measured. The JIT profiles the *whole* benchmark, so running two implementations in
the same JVM makes the second one's call site megamorphic and slower purely by ordering. And the
measurement window usually includes warmup and a GC pause. JMH exists to control all of that: forked
JVMs per benchmark, explicit warmup and measurement iterations, `Blackhole` to consume results, and
`@State` objects the compiler cannot fold. My rule is: if a performance claim has a number in it and
no JMH harness or production measurement behind it, I treat it as a hypothesis.

**Deeper — the five ways a hand-rolled benchmark lies:**

| Lie | Mechanism | JMH's answer |
|---|---|---|
| "It runs in 0 ns" | dead-code elimination: result unused → the JIT deletes the computation | `Blackhole.consume(...)` or return the value |
| "It runs in 0 ns" (variant) | constant folding: inputs are `static final` → computed at compile time | `@State` instance fields, not constants |
| Second implementation looks slower | profile pollution: same call site now sees two receiver types → megamorphic | `@Fork` — a fresh JVM per benchmark |
| Wildly variable numbers | warmup, GC, background compilation inside the measurement window | `@Warmup` / `@Measurement` iterations, and reporting the distribution |
| "10% faster" that isn't | run-to-run variance larger than the effect | multiple forks + JMH's error bars; if the intervals overlap, you measured nothing |

**Code:**

```java
// ❌ This "benchmark" measures approximately nothing.
long start = System.nanoTime();
for (int i = 0; i < 1_000_000; i++) {
    counter.increment();          // result never read → may be eliminated entirely
}
System.out.println((System.nanoTime() - start) / 1_000_000 + " ns/op");
//  - no warmup: mostly interpreted
//  - single JVM: whichever runs second is polluted by the first
//  - a GC pause anywhere in the loop silently doubles the answer
```

```java
// ✅ BS-02 / REQ-CC-06: the real question is "AtomicLong vs LongAdder under contention",
//    and contention is exactly what a single-threaded loop cannot show.
@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.SECONDS)
@State(Scope.Benchmark)                 // shared across threads — that IS the contention
@Fork(value = 3)                        // 3 separate JVMs: kills profile pollution, exposes variance
@Warmup(iterations = 5, time = 2)       // let C2 finish compiling before measuring
@Measurement(iterations = 10, time = 2)
@Threads(8)                             // the parameter that actually decides the winner
public class CounterBenchmark {

    private AtomicLong atomic;
    private LongAdder  adder;

    @Setup(Level.Iteration)
    public void setup() { atomic = new AtomicLong(); adder = new LongAdder(); }

    @Benchmark
    public long atomicIncrement() {
        return atomic.incrementAndGet();      // returned → cannot be eliminated
    }

    @Benchmark
    public void adderIncrement(Blackhole bh) {
        adder.increment();
        bh.consume(adder.sum());              // consumed → cannot be eliminated
    }
}
```

The interesting part of that benchmark is the *expected shape* of the result, which you should be able
to predict before running it: `AtomicLong` is a single cache line contended by all threads, so
throughput degrades as thread count rises; `LongAdder` spreads increments across per-thread cells and
pays only on `sum()`. So `LongAdder` should win for write-heavy, read-rare counters and lose where you
read constantly. **Predict, then measure, then explain the gap** — that sequence is what a senior
performance answer looks like.

**Follow-ups they'll ask:**
- *"When is a microbenchmark the wrong tool entirely?"* — almost always for a service. The system-level
  answer is a load test (Gatling) plus a profiler, because the thing that dominates is I/O, locking,
  serialization and GC interaction — none of which a microbenchmark reproduces. I use JMH for a
  *specific, isolated* algorithmic question, like the counter one above.
- *"What is `@CompilerControl`?"* — JMH's escape hatch to prevent or force inlining of a specific
  method, when inlining is the thing you are trying to study.
- *"What does JMH's error interval mean?"* — the confidence interval across forks/iterations. If two
  candidates' intervals overlap, the honest conclusion is "no measurable difference", and saying that
  out loud is worth more than picking a winner.

**Pitfalls / red flags:** quoting a performance number with no harness, no variance, and no thread
count.

**Where in StreamPulse:** BS-02 / REQ-CC-06 (the counter JMH benchmark), Phase 5 (the MVC+virtual
threads vs WebFlux comparison is deliberately a *Gatling* benchmark, not JMH — because the question is
about the whole request path, not a method).

**Verify:** JMH annotation names above are stable, but confirm the current Gradle/Maven plugin
coordinates in the JMH docs for the version you pin.

---

## Q9. How do you leak memory in a garbage-collected language? Name the patterns.

**30-second answer:** A GC only frees what is *unreachable*, so a "leak" in Java is always an
unintended strong reference from a long-lived root. There are five patterns I look for, and they cover
almost everything I have seen: a `static` collection that only ever grows; a cache with no eviction
policy and no size bound; a `ThreadLocal` set on a *pooled* thread and never removed; a registration —
listener, callback, metric, subscription — that has no deregistration; and a classloader leak, where a
single reference from an old classloader pins every class it loaded. The diagnostic is the same for
all of them: heap dump, dominator tree, ask "what is the *shortest path to a GC root* for the thing
that is big?"

**Deeper:**

| Pattern | The root that holds it | StreamPulse-shaped example | Fix |
|---|---|---|---|
| Static collection | the class, which the classloader holds forever | `static Map<String, SessionState> ACTIVE` keyed by `sessionId` — sessions end, entries never do. **This is the Phase 9 induced leak.** | bound it, TTL it, or key the lifecycle to something that ends |
| Unbounded cache | the cache instance, held by a singleton bean | an LRU built without a capacity check; a `ConcurrentHashMap` "cache" with `computeIfAbsent` and no eviction | a real cache with size + TTL bounds (Caffeine, or the bounded hand-written LRU of BS-02 / REQ-CC-03) |
| `ThreadLocal` on a pooled thread | the `Thread` object, which the pool holds forever | tenant context set on a Tomcat worker thread and never cleared — the *next* request on that thread also sees it, which is a correctness bug as well as a leak | `try { ... } finally { threadLocal.remove(); }`, always |
| Listener / callback registration | the publisher's subscriber list | a per-tenant Micrometer meter or a Reactor `Sinks.Many` subscriber registered per SSE connection and never removed on disconnect | explicit deregistration in the disconnect/cancel path; `doFinally` on the reactive stream |
| Classloader leak | a static field, a `ThreadLocal`, or a JDBC driver registration in a *parent* classloader holding a child's object | mostly an app-server redeploy problem; also shows up with dynamic class generation | avoid cross-classloader statics; unregister drivers and shutdown hooks |
| "Not a leak, just growth" | nothing — it's real | 30-day retention of in-memory window state; a Kafka Streams state store that legitimately grows | not a code fix — a capacity and architecture decision |

That last row is worth saying out loud in an interview, because it is the honest failure mode of leak
hunting: sometimes the live set really is that big and the answer is more heap, spill-to-disk, or a
different data structure — not a bug fix.

**The `ThreadLocal` mechanism, precisely** (this is the detail interviewers dig into): `ThreadLocalMap`
holds its **keys weakly** (the `ThreadLocal` object itself) but its **values strongly**. So if the
`ThreadLocal` becomes unreachable, the key is cleared — but the value stays in the map until some
later `get`/`set`/`remove` on that thread happens to run the stale-entry cleanup. On a thread pool,
"some later operation" may be never. Hence: `remove()` in a `finally`, every time.

**Code:**

```java
// ❌ The Phase 9 induced leak, in the shape it usually appears in real code:
//    a "temporary" static map that someone added for a debugging session and never removed.
public class SessionTracker {
    private static final Map<String, SessionState> ACTIVE = new ConcurrentHashMap<>();

    public static void onEvent(TelemetryEvent e) {
        ACTIVE.computeIfAbsent(e.sessionId(), SessionState::new).apply(e);
        // session_end arrives... and nothing removes the entry.
        // At 50k new sessions/minute this is a textbook slow leak: hours to matter,
        // so it passes every short test and dies on a Saturday night.
    }
}

// ✅ Bounded and time-aware. Two changes, both of which you should be able to justify:
//    (1) a size bound, so the worst case is known rather than discovered;
//    (2) expiry, because "session ended" messages can be lost — never rely on a cleanup event.
private final Cache<String, SessionState> active = Caffeine.newBuilder()
        .maximumSize(200_000)                                  // sized from measured concurrent sessions
        .expireAfterAccess(Duration.ofMinutes(30))             // longer than the longest plausible session
        .recordStats()                                         // hit rate and eviction count → Micrometer
        .build();
```

```java
// ❌ ThreadLocal on a pooled (platform) thread.
private static final ThreadLocal<String> TENANT = new ThreadLocal<>();

void handle(Request r) {
    TENANT.set(r.tenantId());
    process();
    // exception? early return? → never cleared. The next request on this pooled thread
    // inherits the previous tenant. A leak AND a cross-tenant data bug.
}

// ✅ Always paired, always in a finally.
void handle(Request r) {
    TENANT.set(r.tenantId());
    try {
        process();
    } finally {
        TENANT.remove();          // non-negotiable on a pooled thread
    }
}
```

**Follow-ups they'll ask:**
- *"What about `WeakHashMap` / `SoftReference`?"* — `WeakHashMap` clears entries whose *key* is
  otherwise unreachable; useful for metadata keyed by an object you don't own, useless if the value
  references the key (a classic self-defeating bug). `SoftReference` is cleared "when memory is tight",
  which in practice means right before an OOM, after you have already had the pauses — a bad basis for
  a cache. Use a real cache with explicit bounds.
- *"Do finalizers / `Cleaner` help?"* — finalization is deprecated for removal and was always a
  latency hazard. `java.lang.ref.Cleaner` is the modern mechanism for native-resource cleanup; it is
  still a safety net, not a design.
- *"How do you prove it's a leak and not growth?"* — two heap dumps an hour apart, compared by object
  count and retained size per class. Growth that tracks traffic is capacity; growth that continues
  when traffic is flat is a leak.
- *"How do you catch it before prod?"* — a soak test: steady load for hours, with the post-GC heap
  occupancy trend as the pass/fail signal. This belongs in Phase 9, not in a unit test.

**Pitfalls / red flags:** "Java can't leak memory, it has a GC."

**Where in StreamPulse:** Phase 9 (the induced leak, found via heap dump, then fixed and re-soaked),
BS-02 / REQ-CC-03 (the bounded hand-written LRU), Phase 5 (SSE subscriber deregistration on
disconnect — a real leak source in a fan-out server).

---

## Q10. `ThreadLocal` leaks in a thread pool — what changes with virtual threads?

**30-second answer:** With platform threads in a pool, a `ThreadLocal` value outlives the task, because
the *thread* outlives the task — that is both a leak and a correctness hazard. Virtual threads change
the lifetime story: a virtual thread is created per task and discarded when the task ends, so its
thread-locals die with it. There is no pool to carry stale state into the next request. But it
introduces the opposite cost: at a million concurrent virtual threads, a per-thread `ThreadLocal`
holding even a few kilobytes is gigabytes of live heap, and inheritable thread-locals get copied into
every child. So the rule flips from "always `remove()`" to "don't put anything large in a
thread-local, and prefer an immutable value scoped to the call". Scoped values are the language-level
answer to exactly this, though you must check their preview/final status for your JDK.

**Deeper:**

| | Pooled platform threads | Virtual threads |
|---|---|---|
| Thread lifetime | long (pool lifetime) | one task |
| Stale `ThreadLocal` after task | **yes** — leak + cross-request bleed | no — the carrier's locals are separate from the virtual thread's |
| Cost of a `ThreadLocal` value | bounded by pool size (hundreds) | scales with *concurrency* (potentially millions) |
| `InheritableThreadLocal` | copied per new thread (rare) | copied per **task** — can be very expensive |
| Correct discipline | `set` / `finally remove` | keep values small and immutable; prefer scoped values |

The framework consequence for StreamPulse is concrete. Spring's request-scoped `TenantContext`
(BS-01 / REQ-LC-04) is backed by thread-bound request attributes. On a Tomcat worker pool, forgetting
to clear them leaks and can bleed across tenants. On `spring.threads.virtual.enabled=true`, the
thread is per-request anyway, so the bleed risk goes away — but every one of those concurrent virtual
threads now carries its own copy of whatever you put there. Neither model removes the need to think;
they move where the cost is.

**Code:**

```java
// The pattern that is correct on BOTH models: pass the value, don't stash it.
// (Same advice as file 01 Q3 — capture before you fork.)
String tenantId = tenantContext.tenantId();                 // read once, on the request thread
CompletableFuture.supplyAsync(() -> loadMetrics(tenantId),  // pass the VALUE across the boundary
                              panelExecutor);
```

```java
// Scoped values: an immutable, structurally scoped binding — no remove() to forget,
// automatically visible to child tasks in a structured-concurrency scope, and cheap to inherit.
// STATUS WARNING: scoped values moved through several preview rounds. VERIFY whether they are
// preview or final in YOUR JDK before using (or mentioning) them without a caveat.
private static final ScopedValue<String> TENANT = ScopedValue.newInstance();

ScopedValue.where(TENANT, request.tenantId())
           .run(() -> handle(request));      // TENANT is bound for exactly this dynamic scope
```

**Follow-ups they'll ask:**
- *"Do virtual threads make `ThreadLocal` free?"* — no. They make the *stale value across tasks*
  problem disappear and make the *memory per value* problem worse.
- *"What about MDC for logging?"* — MDC is a `ThreadLocal` under the hood. On virtual threads it works
  per-task, which is what you want; across an async boundary it still needs explicit propagation
  (Reactor context, Micrometer context propagation, or manual copy). This is a real source of "the
  traceId is missing from half my logs".
- *"Is there a case where you still want a pooled executor?"* — yes: to *bound* concurrency against a
  downstream that cannot take more (BS-02 / REQ-CC-05 — a virtual-thread executor plus a `Semaphore`
  gives you unbounded threads with bounded downstream pressure, which is usually the right shape).

**Pitfalls / red flags:** "virtual threads fixed our thread-local leaks" stated as a general fact,
with no mention of the per-thread memory cost at high concurrency.

**Where in StreamPulse:** BS-01 / REQ-LC-04 (`TenantContextConcurrencyTest`), BS-02 / REQ-CC-05
(virtual-thread executor + semaphore), Phase 2 (`spring.threads.virtual.enabled`), Phase 9 (memory
behaviour of both models under the spike).

**Verify:** `ScopedValue` preview/final status and its exact API shape (`where(...).run(...)` vs
`callWhere`) changed across JDK releases — confirm against your JDK's javadoc before using it.

---

## Q11. Production runbook: "p99 latency just spiked." Walk me through what you do.

**30-second answer:** I work outside-in and I do not guess. First, is it *us*? Check the RED dashboard
— rate, errors, duration — per service, and the end-to-end freshness metric, to find which hop grew.
Second, is it GC, CPU, lock contention, or a downstream? That is four questions with four different
signals: GC pause panel, CPU utilisation and throttling, thread states, and downstream client latency.
Third, prove it with a profile: I take a continuous JFR recording (it should already be running) and a
30-second async-profiler **wall-clock** flame graph, because a latency problem usually lives in
*waiting*, not in burning CPU — and a CPU flame graph is blind to waiting. Fourth, mitigate before I
finish diagnosing: shed load, scale out, or trip a breaker. Then find root cause with the artifacts I
captured.

**Deeper — the decision tree:**

```
p99 spike
 │
 ├─ Is it all endpoints or one?          one → that code path / that downstream
 │                                       all → shared resource: GC, CPU, pool, network
 │
 ├─ GC pause panel: max pause up?
 │     yes → Q6 decision table (allocation rate? live-set growth? evacuation failure?)
 │     no  → keep going
 │
 ├─ CPU: utilisation high, or cgroup THROTTLING?
 │     throttled → the container is being CPU-capped. GC threads and compiler threads suffer
 │                 FIRST, so this often LOOKS like a GC problem. Check throttling before GC. (Q13)
 │     high & not throttled → CPU flame graph: where are the cycles?
 │
 ├─ Thread dump ×3, 10 s apart: are threads BLOCKED on the same monitor?
 │     yes → lock contention. The repeated dump shows the same stack at the top → that's your lock.
 │     WAITING on a pool/connection? → pool exhaustion (Hikari, Kafka producer buffer, semaphore)
 │
 ├─ Downstream client latency (Redis, DynamoDB, S3, partner HTTP) up?
 │     yes → it's not your JVM. Resilience4j breaker/timeout should already be shaping this (BS-04)
 │
 └─ Nothing obvious → WALL-CLOCK flame graph. This is the step people skip and it is the one
                      that finds "we're blocked 80% of the time in a call nobody suspected".
```

**Why wall-clock vs CPU matters so much here** — the single most useful profiling distinction:

| | CPU profiling | Wall-clock profiling |
|---|---|---|
| Samples | only threads **on-CPU** | threads regardless of state, including blocked/waiting |
| Finds | hot loops, inefficient algorithms, serialization cost | lock waits, I/O waits, pool waits, sleeping, "why is p99 400 ms when CPU is 12%?" |
| Misses | anything that waits | genuinely CPU-bound hotspots get diluted among idle threads |
| Use when | throughput/CPU cost is the problem | **latency** is the problem ← usually this one |

A latency incident with low CPU is *by definition* a waiting problem, and only a wall-clock profile
will show you what you are waiting on.

**Code:**

```bash
# 0. What are we even running?
jcmd -l                                        # find the PID
jcmd <pid> VM.uptime VM.flags

# 1. Continuous JFR should already be on (see Q5's flag block). Dump the last N minutes:
jcmd <pid> JFR.dump name=cont filename=/dumps/spike-$(date +%s).jfr
#    In JMC: "Garbage Collections", "Lock Instances" (contention), "Socket I/O", "File I/O",
#    "Method Profiling", "Thread Park" — the JFR pages map 1:1 to the branches of the tree above.

# 1b. Or start a targeted recording right now:
jcmd <pid> JFR.start name=spike settings=profile duration=120s filename=/dumps/spike.jfr

# 2. Thread dumps — three of them, spaced, because ONE dump proves nothing.
for i in 1 2 3; do jcmd <pid> Thread.print > /dumps/threads-$i.txt; sleep 10; done
grep -c 'java.lang.Thread.State: BLOCKED'  /dumps/threads-*.txt
grep -A3 'Found one Java-level deadlock'   /dumps/threads-*.txt     # jcmd reports deadlocks explicitly

# 3. Flame graph — WALL CLOCK for a latency incident.
./asprof -d 30 -e wall -t -f /dumps/wall.html <pid>     # -t splits by thread; -e wall = all states
./asprof -d 30 -e cpu       -f /dumps/cpu.html  <pid>   # for comparison
./asprof -d 30 -e lock      -f /dumps/lock.html <pid>   # lock contention, attributed to stacks
./asprof -d 30 -e alloc     -f /dumps/alloc.html <pid>  # allocation by call site (ties to Q2)

# 4. GC, if the tree pointed there:
tail -200 /var/log/gc.log
jcmd <pid> GC.heap_info
```

**The mitigations I would reach for while still diagnosing** (say this — it shows operational
instinct): scale out if the service is stateless; raise the Resilience4j timeout/bulkhead limits or
trip the breaker manually on a known-bad downstream; shed load with the 429 + `Retry-After` path
(BS-02 backpressure) so the system degrades predictably instead of collapsing; and roll back if the
spike correlates with a deploy. Root cause can wait an hour; the SLO cannot (BS-10).

**Follow-ups they'll ask:**
- *"What if you can't attach a profiler in prod?"* — that is an argument for *continuous* JFR with
  `dumponexit`, running all the time at low overhead, so the evidence exists before the incident. Also
  make sure the container image has `jcmd` (a JRE-only image often does not — a real Phase 11 gotcha).
- *"How do you know a profiler is safe?"* — JFR is designed for continuous production use at low
  overhead with the default settings; `settings=profile` costs more. async-profiler uses OS-level
  sampling and is widely used in production, but it is an agent attaching to your JVM — test it in a
  perf environment first and know your rollback.
- *"What if p99 is up but p50 is flat?"* — that shape points at something episodic: GC pauses, a lock
  held occasionally, a retry path, a cache miss cliff, or one bad shard/partition. A mean-based
  dashboard would have shown nothing.
- *"How do you find WHICH tenant?"* — per-tenant metric tags and the MDC tenant tag from file 01 Q8.
  Noisy-neighbour is a real cause and you want it to be one dashboard filter away (BS-04).

**Pitfalls / red flags:** immediately blaming GC; or running one thread dump and drawing a conclusion;
or taking a CPU flame graph for a latency problem and concluding "nothing is hot, so nothing is wrong."

**Where in StreamPulse:** Phase 8 (the dashboards this runbook reads), Phase 9 (the flame graphs and
JFR recordings committed as artifacts), BS-04 (the resilience behaviour under a degraded downstream),
BS-10 (the burn-rate alert that pages you in the first place).

**Verify:** async-profiler's CLI changed between the 2.x `profiler.sh` script and the 3.x `asprof`
binary, and event names/options differ. Run `asprof --help` (or `profiler.sh --help`) on the version
you actually ship and confirm `-e wall`, `-e lock`, `-e alloc` and `-t` before quoting them.

---

## Q12. Production runbook: "OOM in prod." What do you do?

**30-second answer:** First I establish *which* OOM, because there are two completely different
events. An `OutOfMemoryError` in the application log with a message like "Java heap space" means the
JVM could not satisfy an allocation — and if I set `-XX:+HeapDumpOnOutOfMemoryError` I now have a heap
dump to analyse. An OOMKill — exit code 137, `Reason: OOMKilled` in the Pod status, nothing in the
application log — means the *kernel* killed the process for exceeding the cgroup memory limit, which
is usually native memory, not heap. Those two need different tools: Eclipse MAT for the first, Native
Memory Tracking for the second. Then: get the heap dump off the dead container, open the dominator
tree, find the object with the largest *retained* size, and trace its shortest path to a GC root. That
path is the bug.

**Deeper:**

| | `OutOfMemoryError` | OOMKilled (exit 137) |
|---|---|---|
| Who killed it | the JVM | the Linux kernel (cgroup memory limit) |
| Evidence | stack trace in the app log, heap dump if configured | `kubectl describe pod` → `Last State: Terminated, Reason: OOMKilled`; `dmesg`; **nothing in the app log** |
| Usual cause | heap leak, undersized heap, one huge allocation | native growth: thread stacks, Metaspace, direct buffers, or `-Xmx` set too close to the limit |
| Tool | heap dump → Eclipse MAT | NMT (`VM.native_memory summary.diff`), thread count, Metaspace trend |
| Common trigger | the Q9 patterns | a thread-per-connection explosion; an unbounded Netty/Kafka buffer; `-Xmx` = 95% of the limit |

**The heap-dump workflow, in the order I actually do it:**

```
1. Get the dump.
   - Ideally it already exists: -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps
     on a PersistentVolume or emptyDir that a sidecar copies out. If the path is inside the
     container's ephemeral filesystem, the dump dies with the pod and you learn nothing.
   - On a live, still-sick process:   jcmd <pid> GC.heap_dump /dumps/heap.hprof
     (NOTE: this triggers a full GC and a long STW pause. On a live-traffic pod, take it from a
      drained instance, or accept the outage. Say this out loud — it's the operationally aware bit.)

2. Open it in Eclipse MAT (or JMC / VisualVM / heaphero).

3. Leak Suspects report first — it is often right, and it is 30 seconds.

4. Dominator tree. Sort by RETAINED heap, not shallow heap.
     shallow  = the object's own bytes
     retained = everything that would be freed if this object were collected  ← the number that matters
   The dominator tree answers "which single object is keeping 3 GB alive?"

5. On the suspect: "Path to GC Roots" → "exclude weak/soft references".
   That path is literally the line of code holding the reference. Read it top to bottom:
     static field  →  ConcurrentHashMap  →  Node[]  →  SessionState  →  ... 3.1 GB
   That is the Phase 9 induced leak, found.

6. "Group by class" / histogram, sorted by retained size, to confirm the shape:
   2.4M SessionState instances when the system has ~40k concurrent sessions = the entries never leave.

7. Compare two dumps if it's ambiguous: MAT can diff histograms. Growth in a class between
   two dumps taken an hour apart under flat traffic is the definition of a leak.
```

**Code:**

```bash
# Which OOM was it?
kubectl describe pod <pod> | grep -A5 'Last State'          # Reason: OOMKilled, Exit Code: 137
kubectl logs <pod> --previous | grep -i 'OutOfMemoryError'  # java.lang.OutOfMemoryError: Java heap space

# Live capture (accept the STW pause, or do it on a drained pod):
jcmd <pid> GC.heap_dump /dumps/heap.hprof                   # full GC + long pause — be deliberate
jmap -dump:live,format=b,file=/dumps/heap.hprof <pid>       # older tool, same caveat; :live forces a GC first

# A fast histogram without a full dump — often enough to name the suspect class:
jcmd <pid> GC.class_histogram | head -30
jmap -histo:live <pid> | head -30

# Get it out of the container:
kubectl cp <ns>/<pod>:/dumps/heap.hprof ./heap.hprof

# Native-side OOM instead (no OutOfMemoryError in the log):
jcmd <pid> VM.native_memory summary.diff                    # requires -XX:NativeMemoryTracking
jcmd <pid> Thread.print | grep -c 'java.lang.Thread.State'  # thread explosion?
jcmd <pid> GC.heap_info                                     # heap is fine? then it IS native
```

**Follow-ups they'll ask:**
- *"What is retained size?"* — the memory freed if that object became unreachable, i.e. the object
  plus everything only it keeps alive. Shallow size is just the object's own fields. Sorting a
  histogram by shallow size is how people miss the actual culprit.
- *"`OutOfMemoryError: GC overhead limit exceeded`?"* — the JVM spent an excessive fraction of time in
  GC while reclaiming very little. It is the *polite* version of a heap leak: the JVM noticed the death
  spiral and gave up rather than grinding. Same diagnosis path.
- *"Can you catch an `OutOfMemoryError`?"* — technically yes, practically no: it is an `Error`, the
  JVM state is unreliable, and catching it usually just hides the problem. Prefer
  `-XX:+ExitOnOutOfMemoryError` and let the orchestrator restart you.
- *"How do you make the next OOM cheap to diagnose?"* — that is the real senior answer: heap dump on
  OOM to a persistent path, continuous JFR with `dumponexit`, NMT in perf, a per-service memory
  dashboard with post-GC occupancy (not just "used heap", which is sawtoothed and useless), and a
  soak test in CI that fails on occupancy trend.

**Pitfalls / red flags:** "we increased the heap and it went away." That is a delay, not a fix, and a
senior interviewer will ask what the retained-size winner was.

**Where in StreamPulse:** Phase 9 (the induced leak, the dump, the MAT screenshot in
`docs/labs/leak.md`), Phase 11 (the `/dumps` volume and the heap-dump sidecar), BS-01 (the shutdown
path has to tolerate a dump being written inside the grace period).

**Verify:** `jmap` is deprecated in favour of `jcmd` on modern JDKs, and a JRE-only container image
(Jib/Buildpacks default) may not include either. Confirm the tooling is present in your Phase 11 image
*before* you need it at 2am.

---

## Q13. Container specifics: CPU limits, cgroups, GC threads, OOMKilled. What bites people?

**30-second answer:** The JVM has been container-aware for a long time — it reads the cgroup memory
and CPU limits rather than the host's — but "aware" is not "correct for you". Three things bite. One:
`-Xmx` sized as a percentage of the *limit* must leave room for all the native memory from Q1, or you
get OOMKilled with a healthy-looking heap graph. Two: a CPU **limit** in Kubernetes is a CFS quota,
and when you exceed it the kernel *throttles* you — your threads are descheduled for the rest of the
period. GC threads and JIT compiler threads get throttled too, which makes throttling masquerade as a
GC problem or a warmup problem. Three: the JVM derives `ParallelGCThreads`, the common ForkJoinPool
size, and the default collector choice from the CPU count it sees, so a limit of 1 can silently give
you the Serial collector and a parallelism-1 common pool. I check `availableProcessors()` and the
throttling metric before I touch any GC flag.

**Deeper:**

| Symptom | Real cause | How to confirm |
|---|---|---|
| Pod OOMKilled, heap flat at 60% | native memory (Metaspace, threads, direct buffers) + `-Xmx` too close to the limit | NMT `summary.diff`; thread count; Metaspace trend |
| Latency spikes every ~100 ms, CPU "only 70%" | **CFS throttling** — the quota is consumed early in each period | `container_cpu_cfs_throttled_seconds_total` / throttled periods ratio |
| GC suddenly single-threaded and slow | CPU limit ≤ 1 → JVM picks Serial GC, `ParallelGCThreads=1` | `jcmd <pid> VM.flags \| grep -E 'UseSerialGC\|ParallelGCThreads'` |
| `parallelStream()` does nothing in parallel | common ForkJoinPool sized from `availableProcessors()` = 1 | `Runtime.getRuntime().availableProcessors()` |
| Works on my laptop, dies in the cluster | laptop sees 10 cores and 32 GB; the pod sees 1 core and 1 GB | compare `availableProcessors()` and `maxMemory()` in both |

**The CPU limit argument worth having.** A CPU *request* is a scheduling guarantee; a CPU *limit* is a
hard throttle. For a latency-sensitive JVM service, many teams set requests and deliberately **omit**
the limit (or set it generously), because throttling a GC thread to save CPU you weren't using anyway
is a bad trade. The counter-argument is noisy-neighbour protection and predictable capacity planning.
There is no universal answer — but having the argument, with "throttling hits GC and JIT threads
first" as your evidence, is the senior position. Measure the throttling ratio and decide from data.

**Code:**

```bash
# What does the JVM think it has? Run this INSIDE the container, not on your laptop.
jcmd <pid> VM.flags | tr ' ' '\n' | grep -E 'MaxHeapSize|ParallelGCThreads|ConcGCThreads|ActiveProcessorCount|UseSerialGC|UseG1GC'

jshell -q <<'EOF'
System.out.println("cores  = " + Runtime.getRuntime().availableProcessors());
System.out.println("maxMem = " + Runtime.getRuntime().maxMemory() / (1<<20) + " MB");
EOF

# The cgroup truth (path differs between cgroup v1 and v2 — check which you're on):
cat /sys/fs/cgroup/memory.max          # v2
cat /sys/fs/cgroup/cpu.max             # v2: "<quota> <period>", e.g. "100000 100000" = 1 CPU
cat /sys/fs/cgroup/memory/memory.limit_in_bytes   # v1

# Throttling, from the kubelet's cAdvisor metrics — this is the one to put on a Grafana panel:
#   rate(container_cpu_cfs_throttled_periods_total[5m])
#     / rate(container_cpu_cfs_periods_total[5m])          ← sustained > ~0.05 deserves investigation
```

```yaml
# Phase 11 shape. The numbers are placeholders to be replaced with Phase 9 measurements.
resources:
  requests:
    cpu:    "2"           # scheduling guarantee: enough for app threads + GC threads
    memory: "2Gi"
  limits:
    memory: "2Gi"         # memory limit == request: no overcommit surprises, predictable OOMKill
    # cpu: deliberately omitted — see the argument above. Document the decision in the ADR.
env:
  - name: JAVA_TOOL_OPTIONS
    value: "-XX:MaxRAMPercentage=65 -XX:InitialRAMPercentage=65 -XX:ActiveProcessorCount=2 ..."
    # ActiveProcessorCount: only if you need to OVERRIDE what the JVM detects. Usually you don't —
    # but it is the escape hatch when the detection is wrong for your runtime.
```

**Follow-ups they'll ask:**
- *"Is container support automatic?"* — yes on modern JDKs (`-XX:+UseContainerSupport`, on by
  default), including cgroup v2. Ancient JVMs (pre-JDK 8u191) saw the *host's* memory and are a classic
  legacy landmine.
- *"Why does memory limit == request?"* — memory is incompressible: exceeding it is death, not
  slowdown. Setting them equal gives the Pod the Guaranteed QoS class and removes an eviction risk.
- *"How do you size it in the first place?"* — from a load test at target throughput, with headroom:
  live-set size from post-GC occupancy, plus the native budget from NMT, plus margin. That is exactly
  what Phase 9 produces and Phase 11 consumes.
- *"What about `MaxRAMPercentage` with a very small limit?"* — below a threshold the JVM applies
  `MinRAMPercentage` instead, which surprises people on tiny sidecars. Verify the thresholds for your
  JDK.

**Pitfalls / red flags:** tuning GC flags in a container without ever checking `availableProcessors()`
or the throttling ratio.

**Where in StreamPulse:** Phase 11 (Helm resources, probes, PDB), Phase 9 (the measurements that set
those numbers), BS-10 / REQ-SL-05 (these settings are part of what defends the SLA).

**Verify:** cgroup v1 vs v2 file paths, `UseContainerSupport` default, and `MinRAMPercentage`
thresholds — confirm on your actual base image and JDK.

---

## Q14. What are the performance characteristics of virtual threads, and how do you detect pinning?

**30-second answer:** Virtual threads are cheap to create and cheap to block — the JDK mounts them on
a small pool of carrier (platform) threads, and when a virtual thread blocks on JDK-managed I/O it
*unmounts*, freeing the carrier. So the win is concurrency, not raw speed: you can have a million
blocked-on-I/O tasks for the cost of their stacks instead of a million OS threads. They do not make
CPU-bound work faster — there is no more CPU. The failure mode to know is **pinning**: in some
situations the virtual thread cannot unmount and holds its carrier while blocked. Historically the two
causes were blocking inside a `synchronized` block and blocking inside a native/JNI frame. If enough
carriers are pinned, throughput collapses in a way that looks like a deadlock. The `synchronized` case
was addressed in a later JDK, so the *correct* answer depends on your exact JDK version — and how you
*detect* it depends on the version too.

**Deeper:**

```
   1,000,000 virtual threads
            │  mounted on demand
            ▼
   ┌──────────────────────────────┐
   │ ForkJoinPool carrier threads │   default parallelism ≈ availableProcessors()
   │  [ C0 ][ C1 ][ C2 ][ C3 ]    │
   └──────────────────────────────┘
      blocking on JDK I/O  →  UNMOUNT: the carrier picks up another virtual thread ✅
      blocking while PINNED →  the carrier is stuck ❌  (only 4 carriers → 4 stuck = total stall)
```

| Property | Platform thread | Virtual thread |
|---|---|---|
| Creation cost | expensive (OS thread, ~MB stack) | cheap (heap-allocated stack chunks that grow/shrink) |
| Practical count | thousands | millions |
| Blocking on JDK I/O | blocks an OS thread | unmounts — the carrier is reused |
| CPU-bound work | same | **same** — no speedup, and long CPU-bound tasks starve carriers |
| Pooling | required | **anti-pattern** — create one per task |
| Thread-locals | pooled, must `remove()` | per task, but multiply with concurrency (Q10) |
| Synchronization | `synchronized` fine | prefer `ReentrantLock` on older JDKs (pinning); see Verify |
| Observability | mature (thread dumps, JMX) | thread dumps need the newer format; JFR has virtual-thread events |

**The performance traps that are specific to virtual threads:**

1. **Unbounded concurrency against a bounded downstream.** A virtual-thread-per-task executor will
   happily launch 50,000 concurrent DynamoDB calls. The database, the connection pool or the partner
   API will not. The old thread pool was accidentally also your rate limiter; remove it and you must
   add the limit back explicitly — a `Semaphore`, a Resilience4j bulkhead, or the pool itself. This is
   exactly BS-02 / REQ-CC-05 and the reason that requirement pairs the virtual-thread executor with a
   `Semaphore(16)`.
2. **Connection-pool inversion.** With platform threads, pool size ≥ thread count meant no waiting.
   With a million virtual threads and a Hikari pool of 20, you now have a million tasks queueing on 20
   connections. The queue moved; it did not disappear. (BS-05 / REQ-DL-07 has this as an explicit lab.)
3. **CPU-bound tasks starve carriers.** A virtual thread doing a long computation does not yield. A
   handful of them can occupy every carrier. Keep CPU-bound work on a dedicated platform-thread pool.
4. **Pinning**, below.

**Code:**

```java
// ✅ Virtual threads for I/O-bound fan-out, with an EXPLICIT bound on downstream concurrency.
//    (BS-02 / REQ-CC-05: the semaphore is the point, not an afterthought.)
private final ExecutorService vt   = Executors.newVirtualThreadPerTaskExecutor();
private final Semaphore      limit = new Semaphore(16);      // protects S3, not the JVM

void upload(Batch batch) {
    vt.submit(() -> {
        limit.acquire();                 // bounded pressure downstream, unbounded threads upstream
        try { s3.put(batch); }
        finally { limit.release(); }
    });
}
```

```java
// ❌ On older JDKs this pins the carrier for the whole blocking call.
synchronized (lock) {
    httpClient.send(request, ofString());     // blocking I/O inside a monitor → PINNED
}

// ✅ ReentrantLock does not pin: the virtual thread can unmount while holding it.
lock.lock();
try {
    httpClient.send(request, ofString());
} finally {
    lock.unlock();
}
```

```bash
# DETECTION — and this is the part that is genuinely version-sensitive. Read the Verify line.
#
# (a) The historical system property that printed a stack trace when a virtual thread pinned:
java -Djdk.tracePinnedThreads=full -jar app.jar     # values were 'full' and 'short'
#     This was tied to the era when synchronized caused pinning; it was deprecated/removed
#     once that cause was addressed. CONFIRM whether it exists on your JDK before relying on it.
#
# (b) JFR. There is a virtual-thread pinning event in the JFR event set; the StreamPulse
#     approach is to enable the virtual-thread events in a JFR recording and look for pinned
#     samples in JMC. CONFIRM the exact event names against `jfr summary` on your own recording:
jcmd <pid> JFR.start name=vt settings=profile duration=120s filename=/dumps/vt.jfr
jfr summary /dumps/vt.jfr | grep -i -E 'virtual|pin'      # ← let the tool tell you the real names
jfr print --events <NameYouJustFound> /dumps/vt.jfr | head -40
#
# (c) Carrier-pool sizing (system properties, also version-sensitive — verify):
-Djdk.virtualThreadScheduler.parallelism=N
-Djdk.virtualThreadScheduler.maxPoolSize=N
#
# (d) The blunt but always-available signal: a thread dump. If every carrier thread is inside a
#     blocking call and virtual threads are piling up, you are pinned regardless of what the
#     flags are called this release.
jcmd <pid> Thread.print                      # newer JDKs also offer a JSON dump format for virtual threads
```

**Follow-ups they'll ask:**
- *"When would you NOT use virtual threads?"* — CPU-bound work; anything already fully reactive and
  performing well; code with heavy `ThreadLocal` state (Q10); native-heavy paths that pin.
- *"Virtual threads vs WebFlux?"* — virtual threads give you reactive-like scalability with
  *imperative* code, which is a huge maintainability win — straightforward debugging, real stack
  traces, ordinary try/catch. WebFlux gives you true end-to-end backpressure, which virtual threads do
  not: a blocking call has no way to say "slow down". For StreamPulse's SSE fan-out, backpressure is
  the whole problem, which is why Phase 5 keeps `query-api` on WebFlux while `ingest-gateway` uses
  virtual threads — and Phase 5's benchmark ADR is where that decision gets its evidence.
- *"Do virtual threads help throughput?"* — only if you were thread-limited. If you were CPU- or
  downstream-limited, they change nothing except where the queue lives.

**Pitfalls / red flags:** pooling virtual threads; claiming they make everything faster; or stating a
pinning flag name with confidence when the flag has changed across releases.

**Where in StreamPulse:** Phase 2 (`spring.threads.virtual.enabled=true` on ingest-gateway), BS-02 /
REQ-CC-05 (virtual-thread executor + semaphore, benchmarked against `ThreadPoolExecutor`), BS-05 /
REQ-DL-07 (the pool-exhaustion lab), Phase 9 (pinning detection and the comparison).

**Verify — read this before you say any of it out loud:** (1) whether `-Djdk.tracePinnedThreads`
exists on your JDK, or has been removed; (2) whether `synchronized` still pins on your JDK — the
behaviour changed in a later release than 21; (3) the exact JFR event name for pinning — get it from
`jfr summary` on your own recording rather than from memory; (4) the
`jdk.virtualThreadScheduler.*` property names. These four are the most commonly mis-stated facts in
this whole topic, and an interviewer on a modern JDK will know.

---

## Q15. In StreamPulse, what did the JVM lab actually find?

*(The applied question — the one that converts knowledge into evidence.)*

**30-second answer:** Phase 9 is a deliberate three-part lab, and each part produces an artifact I can
show. First, an **induced leak**: a static `ConcurrentHashMap` keyed by `sessionId` with no eviction.
Under a soak test the post-GC heap occupancy climbs monotonically while traffic is flat — that trend
line *is* the leak signal, and it is why my dashboard plots occupancy-after-collection rather than
"heap used". I took a heap dump, opened the dominator tree, sorted by retained size, and the path to
GC roots landed on the static field in three clicks. The fix was a bounded, expiring cache, and the
proof was a re-run of the same soak with a flat occupancy trend. Second, **G1 vs ZGC under the BS-03
spike**: same load, same heap, compared on pause distribution, throughput and CPU, with the decision
written down along with what it cost. Third, a **wall-clock flame graph** of the ingest path under
load, because the p99 question is a waiting question, not a CPU question.

**The three artifacts, and what each one proves:**

| Lab | Artifact | What it proves in an interview |
|---|---|---|
| Induced leak | `docs/labs/leak.md` — GC-log excerpt showing rising post-GC occupancy, MAT dominator-tree and path-to-GC-roots screenshots, the before/after code, the re-run soak | that I can go from a symptom to a line of code with a tool, not a guess |
| G1 vs ZGC | `docs/perf-report.md` — pause **distribution** (p50/p99/max, not mean), throughput, CPU, allocation rate, for both collectors on identical load | that I choose a collector on evidence and can state the cost of the choice |
| Flame graph | `docs/labs/flamegraph.md` — wall-clock and CPU graphs side by side, with the delta explained | that I know latency lives in waiting, and that I can read a profile |

**The method, which is the transferable part:**

```
1. Define the question first.        "Does ZGC reduce our p99 under the BS-03 spike, and at what CPU cost?"
2. Fix everything except one variable. Same heap, same load profile, same warmup, same JDK.
3. Warm up, then measure.            Otherwise you are benchmarking the interpreter (Q7).
4. Report the distribution.          Max and p99. A mean pause is not a latency statement.
5. Write down the COST.              ZGC's pause win is paid for in CPU and footprint. Say the price.
6. Re-run after the fix.             A fix without a re-measured baseline is a belief.
```

**Follow-ups they'll ask:**
- *"How did you know it was a leak and not just growth?"* — traffic was flat and occupancy was not.
  Two dumps an hour apart, diffed by class, showed `SessionState` instances growing without bound
  while the concurrent-session count was steady.
- *"Why not just add heap?"* — it delays the OOM and makes the eventual dump bigger. Also, the leak
  would still have been a *correctness* problem: stale session state affects the aggregates.
- *"What did the flame graph show?"* — that is where you fill in your own finding. Say it plainly,
  including if the answer was boring ("it was serialization, as expected"). A boring, verified answer
  beats an exciting, invented one.
- *"Would you run a profiler in production?"* — continuous JFR at low-overhead settings, yes, always
  on, with `dumponexit`. `settings=profile` and async-profiler: on a canary or a drained instance
  first, with a rollback plan.

> **Your story:** if you have ever diagnosed a production memory or latency problem — at Couchbase,
> JustAnswer or NetApp, in any language — put it here in the same shape: *what the symptom was, what
> signal you looked at first, what tool you reached for, what the root cause turned out to be, and
> how you verified the fix*. The language does not matter; the method does. If the real story is "we
> restarted it nightly for six months and then someone found a static map", say that — the
> self-aware version is more convincing than a heroic one. Use "I" only for what you personally did.
>
> Keep StreamPulse framed honestly: *"I built this lab to prove I can do the diagnosis end to end,"*
> not *"we had this outage in production."*

**Where in StreamPulse:** Phase 9 end to end; BS-03 (the spike that drives it), BS-02 / REQ-CC-03 and
REQ-CC-06 (the cache and counter benchmarks), Phase 8 (the dashboards that make the symptom visible in
the first place), Phase 11 (the resource limits derived from the results).

---

## Key terms

| Term | One-line meaning |
|---|---|
| **TLAB** | Thread-Local Allocation Buffer — a private slice of eden so allocation is a lock-free pointer bump. |
| **Generational hypothesis** | Most objects die young; collect the young generation often and cheaply. |
| **Allocation rate** | MB/s of new objects. Sets young-GC frequency; the first number to measure. |
| **Promotion / tenuring** | Copying a survivor into the old generation after it survives enough collections. |
| **Humongous object (G1)** | An object larger than half a G1 region; allocated in old space, a fragmentation source. |
| **Evacuation failure / to-space exhausted** | G1 had nowhere to copy survivors → fallback full GC → the multi-second pause. |
| **SATB** | Snapshot-at-the-beginning — G1's concurrent marking invariant. |
| **Load barrier / colored pointers** | ZGC's mechanism for relocating objects concurrently with the application. |
| **Safepoint** | A point where all threads stop and the heap is consistent; "time to safepoint" is part of pause time. |
| **Escape analysis / scalar replacement** | Proving an object never escapes a method, so the JIT can delete the allocation. |
| **Tiered compilation** | Interpreter → C1 (with profiling) → C2 (speculative). The source of warmup. |
| **Deoptimization** | Discarding compiled code when a speculative assumption is violated; falling back to the interpreter. |
| **Megamorphic call site** | A call site with many receiver types — cannot be devirtualized or inlined. |
| **JMH** | The only trustworthy Java microbenchmark harness: forks, warmup, `Blackhole`, `@State`. |
| **Shallow vs retained size** | Own bytes vs everything freed if this object died. Sort by retained. |
| **Dominator tree** | MAT's view answering "which single object keeps this memory alive?" |
| **NMT** | Native Memory Tracking — `jcmd VM.native_memory`, the tool for non-heap growth. |
| **JFR** | Flight Recorder — low-overhead always-on event recording; `jcmd JFR.start/dump`. |
| **Wall-clock profiling** | Sampling threads regardless of state; the right mode for a *latency* problem. |
| **OOMKilled (137)** | The kernel killed the process for exceeding the cgroup limit — not an `OutOfMemoryError`. |
| **CFS throttling** | A Kubernetes CPU *limit* descheduling your threads — including GC and JIT threads. |
| **Pinning** | A virtual thread that cannot unmount from its carrier while blocked. |

---

## Self-check (answer out loud, no notes)

1. Your pod is OOMKilled but the heap graph is flat at 60%. Name four places the memory could be, and
   the one command you would run first.
2. Given a GC log, what four things do you read, and what does "post-GC occupancy rising while traffic
   is flat" mean?
3. Why would lowering `MaxGCPauseMillis` make your pauses *worse*?
4. A latency incident with 12% CPU utilisation — which profiling mode do you use, and why is the
   other one useless here?
5. Name the five memory-leak patterns, and say what changes about the `ThreadLocal` one when you
   switch to virtual threads.
