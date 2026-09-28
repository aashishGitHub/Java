# 03 — Java Concurrency: memory model, locks, pools, `CompletableFuture`, virtual threads, deadlock

**Why this matters for an Event Hub role:** a telemetry hub is a concurrency problem wearing a
business costume. Every interesting property of it — throughput, tail latency, whether a pod falls
over under a spike — is decided by how work is handed between threads and what happens when one side
is slower than the other. The interview reflects that: they will ask *"which pool does
`supplyAsync` use?"* not because they care about trivia, but because the wrong answer means you have
shipped a service whose entire fan-out shares a pool sized `cores - 1` with everything else in the
JVM. And in 2026 there is a second layer: you now have to justify **thread pools versus virtual
threads**, and be clear about what virtual threads do *not* fix.

**Scenario anchor — BS-02:** the Title Command Center fetches five panels in parallel with per-panel
timeouts (REQ-CC-01/02) from a named executor, backed by a hand-written LRU under a read-write lock
(REQ-CC-03); `sink-writer` offloads S3 uploads to a bounded `ThreadPoolExecutor` with
`CallerRunsPolicy` so a slow S3 slows Kafka consumption instead of OOM-ing (REQ-CC-04), with a
virtual-thread + `Semaphore` alternative behind a flag (REQ-CC-05); per-tenant counters use `LongAdder`
in a `ConcurrentHashMap` (REQ-CC-06); and there is a **deliberate deadlock lab** —
`POST /admin/quota/transfer` locking `from` then `to`, reproduced with a `CyclicBarrier`, fixed with
lock ordering plus `tryLock(timeout)`, with the thread dump committed to `docs/labs/deadlock.md`
(REQ-CC-07, REQ-CC-08).

---

## Q1. What does the Java Memory Model actually guarantee, and what does `volatile` do?

**30-second answer:** The JMM does not describe *when* a write becomes visible — it defines a partial
order called **happens-before**, and its only promise is: if action A happens-before action B, then
B sees everything A did. If there is no happens-before edge between two threads, the compiler, the
JIT and the CPU are all free to reorder, cache in a register, or delay a write indefinitely, and there
is no wall-clock argument you can make about it. `volatile` creates two of those edges: a write to a
volatile field happens-before every subsequent read of that field, and it also publishes *everything
the writing thread did before it*. What `volatile` does **not** give you is atomicity — `count++` on a
volatile field is still read-modify-write and still loses updates. My rule: `volatile` for a flag or a
reference you publish; `Atomic*` or a lock for anything you compound.

**Deeper:**

The happens-before edges worth being able to list on demand:

| Edge | Reads on the right see writes on the left |
|---|---|
| Program order within one thread | trivially |
| `volatile` write → `volatile` read of the same field | **yes** |
| `synchronized` unlock → subsequent lock of the same monitor | yes |
| `Lock.unlock()` → subsequent `lock()` | yes |
| `Thread.start()` → everything in the started thread | yes |
| everything in a thread → another thread's successful `join()` | yes |
| `Future`/`CompletableFuture` completion → anything that observes the result | yes |
| Writing to a `final` field in a constructor → any thread that sees the *safely published* object | yes |
| **Nothing** between two unsynchronised threads | **no** — this is where bugs live |

The failure a whiteboard can show:

```
Thread A                              Thread B
data = 42;                            while (!ready) { }      // may loop FOREVER:
ready = true;   // non-volatile       use(data);              // the JIT can hoist `ready` out
                                                              // of the loop into a register
```

Make `ready` volatile and both problems vanish — the loop re-reads, and the `data = 42` write is
published by the volatile write's release semantics. That second half is the part people miss:
`volatile` on `ready` fixes the visibility of `data` too, because of the ordering edge.

`final` fields are a special case worth naming: the JMM guarantees that any thread seeing a *correctly
published* object sees its `final` fields fully initialised, with no extra synchronisation. That is
why immutable value objects (Java records) are automatically safe to share, and it is the single
cheapest concurrency strategy available: **don't have mutable shared state.**

**Code:**

```java
/**
 * StreamPulse S3BufferFlusher (BS-01 / REQ-LC-01).
 * `running` is volatile because the shutdown hook thread writes it and the flusher
 * thread reads it. Without volatile the flusher can spin forever on a cached `true`.
 */
class S3BufferFlusher implements SmartLifecycle {

    private volatile boolean running;          // flag: volatile is exactly right
    private final LongAdder flushedBatches = new LongAdder();   // counter: volatile is NOT enough

    @Override public void stop() {
        running = false;                       // happens-before the reader's next read
    }

    void flushLoop() {
        while (running) {                      // re-read every iteration
            flushOne();
            flushedBatches.increment();        // compound operation → needs an atomic, not volatile
        }
    }
}
```

**Follow-ups they'll ask:**
- *"Is `volatile` a lock?"* — no. It gives ordering and visibility for a single variable, with no
  mutual exclusion and no atomicity for compound actions.
- *"Does `volatile` cost anything?"* — it emits memory fences, so it prevents some JIT optimisations
  and, on some architectures, forces cache coherence traffic. Negligible on a flag; measurable in a
  tight loop, which is why `LongAdder` exists (Q3).
- *"Explain double-checked locking."* — the classic broken idiom is fixed by making the field
  `volatile`; without it, another thread can see a non-null reference to a partially-constructed
  object. In practice I'd use a static holder class or an `enum` singleton and not write DCL at all.
- *"What is 'safe publication'?"* — handing an object to another thread through a channel that
  creates a happens-before edge: a volatile field, a final field, a `static` initialiser, a concurrent
  collection, or a lock. Publishing through a plain field is not safe.

**Pitfalls / red flags:** "volatile makes it thread-safe", or explaining visibility purely as "caches"
— the compiler and the JIT reorder too, and on x86 the hardware is the *least* of your problems.

**Where in StreamPulse:** `SmartLifecycle` running flags (BS-01), the LRU's `volatile` size counter,
and the deliberate visibility-bug lab in `kata/`.

---

## Q2. `synchronized`, `ReentrantLock`, `ReadWriteLock`, `StampedLock` — when do you use which?

**30-second answer:** `synchronized` is the default: it is the simplest thing that works, the JVM
optimises it well, and it cannot leak — the monitor is released even if the block throws.
I reach for `ReentrantLock` when I need something `synchronized` cannot express: a **timed** or
**interruptible** acquisition (`tryLock(timeout)` is the fix in the StreamPulse deadlock lab), fairness,
multiple condition variables, or the ability to lock in one method and unlock in another.
`ReentrantReadWriteLock` when reads massively outnumber writes and reads are long enough that
concurrent readers actually buy something. `StampedLock` when even that is too expensive and I want an
**optimistic read** — read without a lock, then validate that nobody wrote during the read and fall
back if they did. The order of preference is also the order of danger: `StampedLock` is not reentrant
and has no `Condition` support, so it is easy to deadlock against yourself.

**Deeper:**

| | `synchronized` | `ReentrantLock` | `ReentrantReadWriteLock` | `StampedLock` |
|---|---|---|---|---|
| Reentrant | yes | yes | yes | **no** |
| Timed / interruptible acquire | no | **yes** | yes | yes |
| Fairness option | no | yes (costly) | yes | no |
| Multiple conditions | one (`wait`/`notify`) | **many** (`newCondition()`) | yes | no `Condition` |
| Concurrent readers | no | no | **yes** | yes + optimistic |
| Optimistic (lock-free) read | no | no | no | **yes** (`tryOptimisticRead` + `validate`) |
| Release on exception | automatic | **you must `finally`** | you must `finally` | you must `finally` |
| Virtual threads (JDK 21) | **pins the carrier** while blocked | does not pin | does not pin | does not pin |

That last row is new and it is the one that changes real code: on **JDK 21**, a virtual thread that
blocks inside `synchronized` pins its carrier thread. So on a virtual-thread service, hot paths that
block under `synchronized` should be converted to `ReentrantLock`. **Verify for your JDK** — I believe
JDK 24 (JEP 491) removed this limitation, which would make `synchronized` safe again, but do not state
that as fact without checking the release you run.

**`ReadWriteLock`'s trap, which is REQ-CC-03 exactly:** a read-write lock only helps if reads are truly
read-only. An LRU cache's `get()` **mutates the access order** — it moves the entry to the head of the
list. So `get()` is a *writer*, and a naive `readLock()` around it is a data race that corrupts the
linked list. Your three honest options: take the write lock on read (simple, correct, and the reason a
read-write lock buys you nothing here); segment the cache so contention is spread; or accept
approximate recency (which is what Caffeine does — it records accesses into per-thread ring buffers
and replays them into the policy asynchronously). Being able to say *"so I chose the write lock, and
here is what I'd measure before going further"* is the senior answer.

**Code:**

```java
/**
 * REQ-CC-03 — the thread-safe LRU that backs the Command Center's metadata panel.
 * Deliberately takes the WRITE lock on get(), because get() reorders the list.
 * The alternative designs are documented in the ADR; this one is correct and simple,
 * and Phase 9 measures whether the contention actually matters at our read rate.
 */
public final class ThreadSafeLruCache<K, V> {

    private final ReentrantReadWriteLock lock = new ReentrantReadWriteLock();
    private final LinkedHashMapLikeLru<K, V> delegate;   // HashMap + doubly linked list (file 17, problem 1)

    public V get(K key) {
        lock.writeLock().lock();          // NOT readLock(): a hit mutates recency order
        try {
            return delegate.get(key);
        } finally {
            lock.writeLock().unlock();    // finally, always — this is what synchronized gives free
        }
    }

    /** A pure read that does NOT touch recency — this one legitimately uses the read lock. */
    public boolean containsWithoutTouching(K key) {
        lock.readLock().lock();
        try {
            return delegate.containsKeyWithoutReorder(key);
        } finally {
            lock.readLock().unlock();
        }
    }
}
```

```java
/**
 * StampedLock, for contrast: the per-title "current concurrent viewers" gauge,
 * read constantly by the dashboard and written once per 10s window.
 * Optimistic read = zero contention in the common case.
 */
class ViewerGauge {
    private final StampedLock sl = new StampedLock();
    private long concurrent;
    private Instant windowStart;

    Snapshot read() {
        long stamp = sl.tryOptimisticRead();          // no lock taken — just a version stamp
        long c = concurrent;
        Instant w = windowStart;                      // may be torn/inconsistent — that's expected
        if (!sl.validate(stamp)) {                    // a writer intervened → discard and retry properly
            stamp = sl.readLock();
            try { c = concurrent; w = windowStart; }
            finally { sl.unlockRead(stamp); }
        }
        return new Snapshot(c, w);
    }

    void write(long c, Instant w) {
        long stamp = sl.writeLock();
        try { concurrent = c; windowStart = w; }
        finally { sl.unlockWrite(stamp); }
    }
}
```

**Follow-ups they'll ask:**
- *"Why must you copy the fields into locals in the optimistic read?"* — because between reading them
  and validating, a writer may change them; the locals are the candidate answer you then *validate*.
  Reading the fields again after `validate` defeats the whole mechanism.
- *"Is `synchronized` slow?"* — not usually. The JVM has biased/thin-lock optimisations for
  uncontended cases (**verify which are still present in your JDK — biased locking was deprecated and
  removed**), and under contention every option is slow. Measure before replacing it.
- *"What's `Condition` for?"* — `await`/`signal` with multiple wait sets: a bounded buffer with
  separate `notFull` and `notEmpty` conditions wakes only the threads that can make progress, where
  `wait`/`notifyAll` wakes everyone. Usually I'd just use a `BlockingQueue` (Q5).
- *"Lock upgrade?"* — `ReentrantReadWriteLock` cannot upgrade read→write (it deadlocks); it can
  downgrade write→read. `StampedLock` has `tryConvertToWriteLock`, which may fail and must be handled.

**Pitfalls / red flags:** a `lock()` without a `try/finally`, "a read-write lock is always better for
read-heavy data" (it is not — the read lock still has coordination cost and can starve writers), or
not knowing an LRU's `get` is a write.

**Where in StreamPulse:** REQ-CC-03 (LRU), REQ-CC-07 (`tryLock` in the quota-transfer fix), Phase 9
(pinning lab with `synchronized` under virtual threads).

**Verify:** the `synchronized`-pins-carrier behaviour is JDK-version-dependent (JEP 491); biased
locking removal is also JDK-version-dependent.

---

## Q3. `AtomicLong` vs `LongAdder` vs `synchronized` — what are the trade-offs?

**30-second answer:** `AtomicLong` is a single memory location updated by a CAS retry loop, so under
heavy contention every thread is fighting for the same cache line and the retry loop burns CPU —
throughput can actually *fall* as you add cores. `LongAdder` trades space and read cost for write
throughput: it keeps a base value plus an array of per-thread-ish cells, each on its own cache line, so
contending threads mostly hit different cells; `sum()` walks them. So: **`LongAdder` for write-heavy
counters you read rarely** — which is exactly a per-tenant ingest counter scraped once every 15
seconds by Prometheus. `AtomicLong` when you need the *current exact value* atomically as part of a
decision, like a sequence number or `compareAndSet`-based state. `synchronized` only if the update is
compound enough that no atomic expresses it.

**Deeper:**

| | `AtomicLong` | `LongAdder` | `synchronized` counter |
|---|---|---|---|
| Mechanism | CAS retry loop on one field | striped cells, CAS per cell | monitor |
| Write throughput under contention | degrades | **scales** | worst |
| Read (`get`/`sum`) | O(1), exact, atomic | O(cells), **not an atomic snapshot** | O(1) |
| Memory | 1 long | base + cell array (grows with contention) | 1 long |
| Supports CAS / `getAndSet` / `updateAndGet` | **yes** | no | n/a |
| Right for | sequences, state machines, IDs | metrics, statistics | compound updates |

The "not an atomic snapshot" point is the one to volunteer: `LongAdder.sum()` walks the cells while
other threads are still incrementing, so the value is a valid count at *no single instant*. For a
metric that is completely fine. For "have we hit the quota exactly?" it is not — and that is why
StreamPulse uses `LongAdder` for the Prometheus gauge (REQ-CC-06) and Bucket4j/Redis for the actual
enforcement (REQ-RS-04). Two different tools because they answer two different questions.

**CAS and ABA, briefly, because it is the standard follow-up:** compare-and-swap reads a value,
computes a new one, and swaps only if the location still holds the original. The ABA problem is that
"still holds A" does not mean "was never changed" — it could have gone A→B→A. For a counter that is
irrelevant; for a lock-free stack of nodes it is a real bug, and the fix is a version stamp
(`AtomicStampedReference`) or not writing lock-free data structures by hand.

**Code:**

```java
/**
 * REQ-CC-06 — per-tenant ingest counters at a design target of 50k ev/s.
 * Write path: millions of increments. Read path: one Prometheus scrape per 15s.
 * That ratio is the entire argument for LongAdder.
 */
@Component
public class TenantIngestCounters {

    private final ConcurrentHashMap<String, LongAdder> counters = new ConcurrentHashMap<>();
    private final MeterRegistry meters;

    public void recordAccepted(String tenantId, int events) {
        // computeIfAbsent: atomic per bin, and the mapping function registers the gauge exactly once.
        counters.computeIfAbsent(tenantId, this::newCounter).add(events);
    }

    private LongAdder newCounter(String tenantId) {
        LongAdder adder = new LongAdder();
        // Micrometer holds a weak reference to the object and calls the function on scrape.
        meters.gauge("ingest.events.accepted", Tags.of("tenant", tenantId), adder, LongAdder::sum);
        return adder;
    }
}
```

```java
// AtomicLong where the VALUE ITSELF drives a decision — a monotonic SSE event id (Phase 5).
private final AtomicLong sequence = new AtomicLong();
String nextEventId() {
    return Instant.now().toEpochMilli() + "-" + sequence.incrementAndGet();   // exact, atomic, unique
}

// updateAndGet for a compound update with no lock — the function MUST be pure and side-effect free,
// because it can be retried an arbitrary number of times.
private final AtomicReference<WindowState> state = new AtomicReference<>(WindowState.empty());
void advance(Instant ts) {
    state.updateAndGet(s -> s.windowStart().isBefore(ts.minus(WINDOW)) ? s.roll(ts) : s.touch(ts));
}
```

**Follow-ups they'll ask:**
- *"Where does the JMH benchmark come in?"* — REQ-CC-06 has one comparing `AtomicLong`, `LongAdder`
  and `synchronized` under contention. I would not quote a number I have not run; the *expected shape*
  is that `LongAdder` pulls ahead as thread count exceeds core count, and I'd present it as a measured
  curve, not a rule.
- *"False sharing — what is it?"* — two unrelated variables on the same 64-byte cache line, so
  updating one invalidates the other's line on every core. `LongAdder`'s cells are padded
  (`@Contended`) specifically to avoid it.
- *"`AtomicInteger.incrementAndGet` vs `getAndIncrement`?"* — same operation, different return; use
  `getAndIncrement` when you want the pre-value (array index allocation), `incrementAndGet` for a
  running count.
- *"Is `Micrometer`'s `Counter` enough?"* — usually yes, and it may well use a `LongAdder`-like
  accumulator internally. The reason StreamPulse holds its own `LongAdder` is that the value is also
  used in-process for a fast quota *hint*; verify Micrometer's internals rather than assuming.

**Pitfalls / red flags:** using `LongAdder` for something that needs `compareAndSet`, or claiming
`LongAdder` is "always faster" — it is worse for the read-heavy case and uses more memory.

**Where in StreamPulse:** REQ-CC-06 (`LongAdder` gauges + JMH), Phase 5 (`AtomicLong` SSE sequence),
Phase 8 (the gauges land on the Grafana RED dashboard).

---

## Q4. How does `ConcurrentHashMap` work, and what is the trap with `computeIfAbsent`?

**30-second answer:** Since Java 8 it is an array of bins with no global lock: an insert into an
*empty* bin is a single CAS with no locking at all, and only when a bin is already occupied does it
`synchronized` on that bin's head node. So contention is per-bin, and throughput scales with table
size rather than thread count. Bins that get long (8 entries, once the table is at least 64) convert
from a linked list to a red-black tree, so a bad `hashCode` degrades to O(log n) instead of O(n).
Reads are lock-free — `Node.val` and `next` are volatile — which is why `get` never blocks. The trap
is that the atomic methods (`compute`, `computeIfAbsent`, `merge`) hold the **bin lock** while running
your lambda: if that lambda touches the same map you can deadlock or get an `IllegalStateException`
about a recursive update, and if it is slow you have serialised every other key in that bin.

**Deeper:**

```
table[]  ─┬─▶ bin 0: null                     insert → CAS the node in, NO LOCK
          ├─▶ bin 1: Node(k,v) → Node(k,v)    insert → synchronized (binHead) { walk/append }
          ├─▶ bin 2: TreeBin (red-black)      ← after TREEIFY_THRESHOLD=8 with table.length >= 64
          └─▶ …
resize:   cooperative — threads that find a ForwardingNode help migrate the table
```

The concurrency contract people misstate:

| Operation | Atomic? |
|---|---|
| `get`, `put`, `remove`, `putIfAbsent`, `replace` | **yes** |
| `compute`, `computeIfAbsent`, `computeIfPresent`, `merge` | **yes**, and the function runs under the bin lock |
| `containsKey(k)` then `put(k,v)` | **no** — that is check-then-act, the classic race |
| `size()`, iteration | **weakly consistent** — a snapshot of nothing in particular; iterators never throw `ConcurrentModificationException` but may reflect concurrent updates |

So `computeIfAbsent` is both the fix for the check-then-act race *and* a foot-gun. The rule I state out
loud: **the mapping function must be short, pure, non-blocking, and must not touch this map.**
Building a `new LongAdder()` is fine. Calling a database, doing an HTTP request, or looking up another
key in the same map is not.

**Code:**

```java
// ✅ correct: atomic, exactly one LongAdder per tenant no matter how many threads race
counters.computeIfAbsent(tenantId, t -> new LongAdder()).add(events);

// ❌ broken: check-then-act. Two threads can both see null and both create a counter;
//    one of them then increments an adder that is not in the map, and those events vanish.
LongAdder a = counters.get(tenantId);
if (a == null) {
    a = new LongAdder();
    counters.put(tenantId, a);
}
a.add(events);

// ❌ broken differently: expensive work under the bin lock.
//    Every other key hashing to this bin is now blocked behind an HTTP call.
meta.computeIfAbsent(titleId, id -> controlPlaneClient.fetchTitle(id));   // 150ms under a lock

// ✅ the load-outside-the-lock pattern when the value is expensive to build
V v = meta.get(titleId);
if (v == null) {
    V loaded = controlPlaneClient.fetchTitle(titleId);   // slow work, no lock held
    v = meta.putIfAbsent(titleId, loaded);               // last writer may lose — that's fine here
    if (v == null) v = loaded;
}
// If a duplicate load is NOT acceptable, store a CompletableFuture<V> as the value instead:
//   futures.computeIfAbsent(id, i -> CompletableFuture.supplyAsync(() -> load(i), loader))
// The lambda is now cheap (it just creates a future), and exactly one load happens per key.
```

That last comment is the pattern worth knowing cold — it is how a real cache does single-flight
loading, and it is what Caffeine's `AsyncLoadingCache` does for you.

**Follow-ups they'll ask:**
- *"Why is `size()` weakly consistent?"* — because maintaining an exact size would need a contended
  counter, defeating the point. It uses a striped counter (the same idea as `LongAdder`) and sums it.
- *"Is `ConcurrentHashMap` a drop-in for `Collections.synchronizedMap`?"* — functionally almost, but
  `synchronizedMap` locks the whole map (so compound operations can be made atomic by locking it
  externally) while `ConcurrentHashMap` cannot be externally locked. Also `ConcurrentHashMap` forbids
  null keys and values — deliberately, because `get` returning null would be ambiguous between
  "absent" and "mapped to null".
- *"How do you evict from it?"* — it has no eviction. That is precisely why StreamPulse has a
  hand-written LRU (REQ-CC-03) rather than "just a `ConcurrentHashMap`", and why Caffeine exists.
- *"`ConcurrentSkipListMap`?"* — when you need *sorted* concurrent access with range queries; O(log n)
  instead of O(1), lock-free. That is the right structure for a time-ordered ring buffer if you were
  not using Redis.

**Pitfalls / red flags:** `if (!map.containsKey(k)) map.put(k, v)` in "thread-safe" code, or blocking
I/O inside `computeIfAbsent`.

**Where in StreamPulse:** REQ-CC-06 (`ConcurrentHashMap<TenantId, LongAdder>`), Phase 5
(`TenantSinkRegistry`, one `Sinks.Many` per tenant via `computeIfAbsent`), and the LRU that exists
*because* `ConcurrentHashMap` has no eviction policy.

---

## Q5. Producer/consumer with a `BlockingQueue` — which implementation, and why does the choice matter?

**30-second answer:** `BlockingQueue` is the hand-off primitive: `put` blocks when full, `take` blocks
when empty, so flow control is built into the data structure and neither side needs a lock or a
condition variable. The choice of implementation *is* the backpressure policy.
`ArrayBlockingQueue` is bounded by construction — which is what I want, because a bounded queue is the
only kind that applies backpressure. `LinkedBlockingQueue` is optionally bounded and defaults to
`Integer.MAX_VALUE`, which is how "we used a queue" becomes "we OOM under load".
`SynchronousQueue` has zero capacity — every `put` waits for a `take` — which is the right choice when
you want a thread pool to grow rather than queue. `PriorityBlockingQueue` is unbounded, so it can
never apply backpressure, and that is usually a surprise.

**Deeper:**

| Implementation | Capacity | Locking | Use it for |
|---|---|---|---|
| `ArrayBlockingQueue` | **fixed, required** | one lock, two conditions | bounded hand-off — the default choice |
| `LinkedBlockingQueue` | optional (default unbounded) | two locks (head/tail) → higher throughput | high-throughput bounded queues; **always pass a capacity** |
| `SynchronousQueue` | 0 | — | direct hand-off; makes `ThreadPoolExecutor` grow to `max` immediately |
| `LinkedTransferQueue` | unbounded | lock-free, supports `transfer()` | producer wants to wait until consumed |
| `PriorityBlockingQueue` | **unbounded** | one lock | ordering by priority, no backpressure |
| `DelayQueue` | unbounded | one lock | scheduled/retry-after-delay work |

This table is not trivia — it is the hidden half of Q7. `Executors.newFixedThreadPool(n)` uses an
**unbounded** `LinkedBlockingQueue`, so a fixed pool never rejects, never applies backpressure, and
simply accumulates tasks until the heap dies. That single fact is why StreamPulse constructs its
`ThreadPoolExecutor` by hand (REQ-CC-04).

**Code:**

```java
/**
 * kata/ — the bounded producer/consumer, written twice.
 * Version 1: BlockingQueue does all the coordination.
 */
class S3UploadPipeline {

    // Bounded ON PURPOSE. 200 pending uploads ≈ the backpressure budget; past that the
    // producer blocks, which slows Kafka consumption, which is the behaviour we want (REQ-CC-04).
    private final BlockingQueue<UploadTask> queue = new ArrayBlockingQueue<>(200);
    private volatile boolean running = true;

    void submit(UploadTask t) throws InterruptedException {
        queue.put(t);              // BLOCKS when full → backpressure, not OOM
    }

    /** Non-blocking variant when the caller must not be stalled (e.g. a Netty event loop). */
    boolean offerOrReject(UploadTask t) {
        return queue.offer(t);     // false → caller decides: 429, drop, or run inline
    }

    void consumerLoop() {
        while (running || !queue.isEmpty()) {
            try {
                // Poll with a timeout rather than take(), so shutdown doesn't hang forever.
                UploadTask t = queue.poll(1, TimeUnit.SECONDS);
                if (t != null) s3.upload(t);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();   // ALWAYS restore the flag
                return;
            }
        }
    }
}
```

```java
/**
 * Version 2: the same thing with wait/notify, to show you know what the queue is doing.
 * Note the WHILE loop around wait() — spurious wakeups are real and `if` is a bug.
 */
class HandRolledBoundedBuffer<T> {
    private final Deque<T> items = new ArrayDeque<>();
    private final int capacity;
    private final Object lock = new Object();

    void put(T t) throws InterruptedException {
        synchronized (lock) {
            while (items.size() == capacity) lock.wait();   // WHILE, not IF
            items.addLast(t);
            lock.notifyAll();                               // notifyAll: producers and consumers share one wait set
        }
    }

    T take() throws InterruptedException {
        synchronized (lock) {
            while (items.isEmpty()) lock.wait();
            T t = items.removeFirst();
            lock.notifyAll();
            return t;
        }
    }
}
// With ReentrantLock you'd use two Conditions (notFull / notEmpty) and signal() instead of
// signalAll(), waking only a thread that can actually proceed. That is the real argument
// for ReentrantLock over synchronized here.
```

**Follow-ups they'll ask:**
- *"Why `notifyAll` and not `notify`?"* — with a single wait set holding both producers and consumers,
  `notify` can wake the wrong kind and the system stalls. Two `Condition`s let you use `signal`
  safely.
- *"How do you shut a consumer loop down cleanly?"* — a volatile flag plus a poison-pill element, or
  `poll(timeout)` plus interruption. Never `Thread.stop`. And always restore the interrupt flag.
- *"What's the difference between `put`/`offer`/`add`?"* — `put` blocks, `offer` returns false (or has
  a timed form), `add` throws `IllegalStateException`. Picking the wrong one silently changes your
  overload behaviour.
- *"Is `ConcurrentLinkedQueue` a `BlockingQueue`?"* — no. It is unbounded and non-blocking, so
  `poll()` returns null instead of waiting, and there is no backpressure. Different tool.

**Pitfalls / red flags:** an unbounded queue anywhere on a path that can be overwhelmed, `if` instead
of `while` around `wait()`, or swallowing `InterruptedException` without restoring the flag.

**Where in StreamPulse:** REQ-CC-04 (the `ArrayBlockingQueue(200)` in the S3 offload executor),
`kata/` (bounded producer/consumer two ways), Phase 4 (`sink-writer`).

---

## Q6. `CountDownLatch`, `CyclicBarrier`, `Semaphore`, `Phaser` — what is each for?

**30-second answer:** `CountDownLatch` is a **one-shot gate**: N counts down to zero and it can never
be reset — "wait until all five panels have returned" or, inverted, "hold 100 threads at the line and
release them all at once" for a stress test. `CyclicBarrier` is a **rendezvous that resets**: N threads
each call `await` and none proceed until all have arrived, then it rearms for the next round —
which is exactly how you reproduce a deadlock deterministically. `Semaphore` is a **permit counter**
bounding concurrent access to a resource; it is not a gate, it is a limit, and it is the thing you
still need when you use virtual threads. `Phaser` is the flexible generalisation: dynamic
registration, multiple phases, and an `onAdvance` hook — useful when the number of participants
changes between rounds, and usually overkill.

**Deeper:**

| | Reusable | Who waits | Count changes | Typical use |
|---|---|---|---|---|
| `CountDownLatch` | **no** | any thread, on `await()` | fixed at construction | wait-for-N-to-finish; start-all-at-once |
| `CyclicBarrier` | **yes** | the *participants* themselves | fixed | iterative simulations, deterministic race reproduction |
| `Semaphore` | yes | acquirers, when out of permits | permits can be added/removed | **bounding concurrency** |
| `Phaser` | yes | participants | **dynamic** register/deregister | variable-party multi-phase work |

Two details that separate a real answer from a memorised one:
1. `CyclicBarrier` breaks: if one participant is interrupted or times out, the barrier enters a broken
   state and every other participant gets `BrokenBarrierException`. That is a feature — it stops the
   others from waiting forever — but you must handle it.
2. `Semaphore` permits are not owned by a thread. Thread A can `acquire` and thread B can `release`;
   nothing checks. That flexibility is also how you leak permits, which is why every `acquire` needs a
   `finally { release(); }`.

**Code — the two testing idioms that make concurrency tests deterministic (REQ-CC-08):**

```java
/**
 * Idiom 1 — CountDownLatch as a starting gun.
 * Without this, threads created in a loop start staggered and the race you're
 * trying to prove usually doesn't happen.
 */
@Test
@DisplayName("REQ-CC-08: 100 concurrent increments lose nothing")
void noLostUpdates() throws Exception {
    int threads = 100, perThread = 1_000;
    var start = new CountDownLatch(1);          // the gun
    var done  = new CountDownLatch(threads);    // the finish line
    var counters = new TenantIngestCounters();

    try (var pool = Executors.newVirtualThreadPerTaskExecutor()) {
        for (int i = 0; i < threads; i++) {
            pool.submit(() -> {
                start.await();                                       // everyone waits here
                for (int j = 0; j < perThread; j++) counters.recordAccepted("sportsnow", 1);
                done.countDown();
                return null;
            });
        }
        start.countDown();                                           // …and go
        assertThat(done.await(10, TimeUnit.SECONDS)).isTrue();
    }
    assertThat(counters.total("sportsnow")).isEqualTo(threads * (long) perThread);
}
```

```java
/**
 * Idiom 2 — CyclicBarrier to force a crossed-lock deadlock deterministically (REQ-CC-07).
 * Both threads must be holding their FIRST lock before either tries the SECOND.
 * A plain sleep makes this flaky; a barrier makes it reproducible.
 */
@Test
void v1TransferDeadlocksUnderCrossedTransfers() throws Exception {
    var barrier = new CyclicBarrier(2);
    var service = new QuotaTransferServiceV1(quotas);   // locks `from` then `to` — the bug

    Thread t1 = Thread.ofPlatform().start(() ->
            service.transferWithRendezvous("streamflix", "sportsnow", 100, barrier));
    Thread t2 = Thread.ofPlatform().start(() ->
            service.transferWithRendezvous("sportsnow", "streamflix", 100, barrier));

    t1.join(2_000);
    t2.join(2_000);

    // Prove it, don't assume it: ask the JVM.
    long[] deadlocked = ManagementFactory.getThreadMXBean().findDeadlockedThreads();
    assertThat(deadlocked).isNotNull();          // v2 (lock ordering) makes this null
}
```

**Follow-ups they'll ask:**
- *"Latch or barrier for 'wait for all workers to finish'?"* — latch. The waiter is not a
  participant, and you never need to reuse it.
- *"Isn't `CompletableFuture.allOf` the modern answer?"* — for asynchronous composition, yes. Latches
  are for thread-level coordination and, above all, for *tests*. In production code a latch is often a
  sign you should be using a future or an executor's `awaitTermination`.
- *"What does a `Semaphore` give you over a fixed thread pool?"* — decoupling *concurrency limit* from
  *thread count*. With virtual threads there is no meaningful thread count any more, so the semaphore
  is the only limit left (Q9, REQ-CC-05).
- *"`Exchanger`?"* — a two-party swap point; real but rarely the right tool. Worth naming to show you
  know the package.

**Pitfalls / red flags:** a concurrency test with `Thread.sleep(100)` in it. That test is flaky by
construction and an interviewer will notice.

**Where in StreamPulse:** REQ-CC-07/08 (`QuotaTransferDeadlockTest`, `LruConcurrencyStressTest`),
REQ-CC-05 (the `Semaphore(16)` in front of the virtual-thread uploader), Phase 2 (in-flight semaphore
→ 429).

---

## Q7. Explain every `ThreadPoolExecutor` parameter, and how you size a pool.

**30-second answer:** Core size, maximum size, keep-alive, the queue, the thread factory and the
rejection handler — and the single most misunderstood thing is the **order** in which they interact.
A new task goes to a new thread only while the pool is below `corePoolSize`; after that it goes into
the **queue**; only when the queue is **full** does the pool grow toward `maximumPoolSize`; and only
when the queue is full *and* the pool is at max does the rejection handler fire. So with an unbounded
queue the pool never exceeds core size and `maximumPoolSize` is decoration — which is exactly what
`Executors.newFixedThreadPool` gives you. Sizing: for CPU-bound work, roughly the number of cores; for
I/O-bound work the classic starting point is `cores × (1 + wait/compute)`. I treat that as a
*starting point to measure from*, never a result, because the real constraint is almost always the
downstream resource — the connection pool, the partner's rate limit — not the CPU.

**Deeper:**

```
submit(task)
   │
   ├─ poolSize < corePoolSize ?  ──yes──▶ create a new thread, run it
   │                              no
   ├─ queue.offer(task) ok ?     ──yes──▶ queued; a free worker picks it up
   │                              no (queue FULL)
   ├─ poolSize < maximumPoolSize?──yes──▶ create a new thread, run it
   │                              no
   └─ RejectedExecutionHandler.rejectedExecution(task, executor)
```

| Parameter | What it controls | The mistake |
|---|---|---|
| `corePoolSize` | threads kept alive even when idle | setting it to max "to be safe" |
| `maximumPoolSize` | ceiling, **only reachable when the queue is full** | assuming it's reached under load with an unbounded queue |
| `keepAliveTime` | how long non-core idle threads survive | ignoring it (`allowCoreThreadTimeOut(true)` lets core threads expire too) |
| `workQueue` | the buffer, and therefore the backpressure policy (Q5) | unbounded |
| `threadFactory` | **names**, daemon status, uncaught-exception handler, priority | leaving threads called `pool-3-thread-7` and then reading a thread dump |
| `handler` | what happens at saturation | `AbortPolicy` (the default) with no plan for the exception |

Rejection policies, and what each one *means* as a product decision:

| Policy | Behaviour | Says to the caller |
|---|---|---|
| `AbortPolicy` (default) | throws `RejectedExecutionException` | "fail fast — you handle it" |
| `CallerRunsPolicy` | the **submitting thread** executes the task | "**backpressure** — you're now the worker, so you can't submit more" |
| `DiscardPolicy` | silently drops | "this data is worthless" (rarely true, and it's silent) |
| `DiscardOldestPolicy` | drops the head of the queue, retries | "freshest wins" — a latest-value policy |
| custom | e.g. spool to disk, publish to a DLQ, increment a metric and 429 | the honest option for most services |

**`CallerRunsPolicy` is the whole of REQ-CC-04** and it is worth explaining as a mechanism, not a
setting: when S3 is slow, the queue fills; the next upload is executed *on the Kafka listener thread*;
that thread is therefore not polling Kafka; consumption slows; the lag metric rises and the autoscaler
sees it. The backpressure propagates all the way to the broker with no extra code. The cost is that a
listener thread can be occupied for as long as one upload takes, which is why the upload itself needs
a timeout — otherwise you have converted "slow S3" into "stalled consumer group" and a rebalance.
Naming that cost is what makes the answer senior.

**Code:**

```java
/**
 * REQ-CC-04 — sink-writer's S3 offload pool.
 * Every parameter is a decision, and the comment says which one.
 */
@Bean("s3UploadExecutor")
ThreadPoolExecutor s3UploadExecutor(MeterRegistry meters) {
    ThreadPoolExecutor ex = new ThreadPoolExecutor(
            4,                                        // core: steady-state parallel uploads
            16,                                       // max: burst headroom, bounded by S3 client conns
            60, TimeUnit.SECONDS,                     // keepAlive: shed burst threads after a minute
            new ArrayBlockingQueue<>(200),            // BOUNDED — this is the backpressure budget
            new CustomizableThreadFactory("s3-upload-"),  // named threads → readable thread dumps
            new ThreadPoolExecutor.CallerRunsPolicy()); // saturation → the listener thread uploads

    // Instrument it, or you will be guessing in the incident.
    ExecutorServiceMetrics.monitor(meters, ex, "s3-upload");   // queue depth, active, completed, rejected
    return ex;
}
```

```java
/**
 * The two sizing heuristics, written down so they are treated as starting points.
 *
 *   CPU-bound:  threads ≈ cores                        (+1 to cover the occasional page fault)
 *   I/O-bound:  threads ≈ cores × (1 + waitTime/serviceTime)
 *
 * Example arithmetic (derived here, NOT a measured StreamPulse number):
 *   8 cores, an S3 PUT that waits ~180ms and costs ~20ms of CPU
 *   → 8 × (1 + 180/20) = 8 × 10 = 80 threads
 * ...which is 5× our max of 16. That gap is the real lesson: the binding constraint is
 * not CPU, it's the S3 client's connection pool and the account's request rate. Size to
 * the SCARCEST downstream resource, then measure (Phase 9).
 */
```

**Follow-ups they'll ask:**
- *"Why not `Executors.newCachedThreadPool()`?"* — it uses a `SynchronousQueue` and an unbounded
  maximum, so under a burst it creates an unbounded number of platform threads. It is a denial of
  service against yourself.
- *"How do you shut one down?"* — `shutdown()` (stop accepting, finish queued), then
  `awaitTermination(timeout)`, then `shutdownNow()` (interrupt and return the unstarted tasks). In
  Spring, `ThreadPoolTaskExecutor` with `setWaitForTasksToCompleteOnShutdown(true)` and an await
  timeout, wired into the BS-01 `SmartLifecycle` drain.
- *"How do you tell from a metric that a pool is undersized?"* — queue depth trending up and staying
  up, rejection count non-zero, or (with `CallerRunsPolicy`) submit latency rising on the caller. All
  three are on the Phase 8 dashboard.
- *"Where do exceptions from `execute` vs `submit` go?"* — `execute` lets them reach the thread's
  uncaught-exception handler; `submit` captures them inside the `Future`, so if you never call `get()`
  the exception is **silently swallowed**. That is one of the most common real production bugs in this
  area.

**Pitfalls / red flags:** "we used a thread pool" with no bound anywhere, or quoting
`cores × (1 + wait/compute)` as if it settles the question.

**Where in StreamPulse:** REQ-CC-04 (S3 offload), REQ-CC-01 (a separate named `commandCenterExecutor`,
deliberately *not* the common pool — Q8), Phase 8 (pool metrics), Phase 9 (measured sizing).

---

## Q8. Walk me through `CompletableFuture` — composition, error handling, timeouts, and which pool it uses.

**30-second answer:** `CompletableFuture` is a promise plus a composition algebra. `thenApply` is
`map` — a synchronous transform of the value. `thenCompose` is `flatMap` — your function returns
another future, and without it you get `CompletableFuture<CompletableFuture<T>>`. `thenCombine` joins
two independent futures with a bi-function; `allOf` waits for many. On the error side, `exceptionally`
recovers, `handle` sees both outcomes, and `whenComplete` observes without changing. `orTimeout`
fails the future after a duration and `completeOnTimeout` substitutes a default instead — which is
exactly the per-panel degradation in the Command Center. And the answer they are actually fishing for:
**`supplyAsync` without an executor runs on `ForkJoinPool.commonPool()`**, whose parallelism is
`availableProcessors() - 1` and which is shared with parallel streams and every library in the JVM. One
blocking call in there starves everything else. So I always pass a named executor.

**Deeper:**

| Method | Signature shape | Use |
|---|---|---|
| `thenApply(f)` | `T → U` | synchronous transform (`map`) |
| `thenCompose(f)` | `T → CompletionStage<U>` | chain another async call (`flatMap`) |
| `thenCombine(other, f)` | `(T, U) → V` | join two **independent** futures |
| `thenAccept` / `thenRun` | consume / ignore | side effects at the end |
| `allOf(cfs…)` | → `CompletableFuture<Void>` | wait for all; **you then `join()` each one for its value** |
| `anyOf(cfs…)` | → `CompletableFuture<Object>` | first to finish wins (hedged requests) |
| `exceptionally(f)` | `Throwable → T` | recover with a fallback value |
| `handle(bi)` | `(T, Throwable) → U` | see both, transform either |
| `whenComplete(bi)` | `(T, Throwable) → void` | observe, do not transform; the result passes through |
| `orTimeout(d, unit)` | | complete **exceptionally** with `TimeoutException` after `d` |
| `completeOnTimeout(v, d, unit)` | | complete **normally** with `v` after `d` |

**The threading rules, stated precisely** — this is the part that separates levels:

1. `supplyAsync(sup)` → `ForkJoinPool.commonPool()`. Its parallelism is
   `Runtime.availableProcessors() - 1`; on a 1-core container that is **0**, and the JVM falls back to
   a new thread per task. Both extremes are surprising in production.
2. `supplyAsync(sup, executor)` → your executor. **Always do this.**
3. `thenApply(f)` (no `Async`) runs `f` on *whichever thread completed the previous stage*, or on the
   calling thread if the previous stage is already complete. So the same line of code can run on three
   different threads on three different days. If `f` is expensive, that thread — possibly a Netty event
   loop or a Kafka listener — is now doing your work.
4. `thenApplyAsync(f)` → common pool. `thenApplyAsync(f, executor)` → your executor. Same rule.

**The common-pool hazard, concretely:** the common pool is also used by `parallelStream()`. If your
Command Center fan-out and someone's `list.parallelStream()` are both on it, and one of your five
panels makes a blocking HTTP call, you have consumed a slot in a pool sized `cores - 1` that the whole
JVM shares. On an 8-core box that is 7 slots. Seven concurrent slow panels and the JVM's parallelism is
gone. (ForkJoin has a `ManagedBlocker` mechanism that can compensate, but relying on it is not a
design.)

**Code:**

```java
/**
 * REQ-CC-01 / REQ-CC-02 — the Title Command Center fan-out, blocking edition.
 * (query-api is WebFlux, so the shipped version uses Mono.zip; this is the MVC +
 *  virtual-threads variant that the Phase 9 benchmark compares against.)
 */
@Service
public class CommandCenterService {

    private final Executor panels;          // NAMED executor — never the common pool
    private final Duration budget;          // REQ-CC-02, default 150ms, per-panel configurable

    public CommandCenterDto load(String titleId) {

        // Each panel: its own timeout, its own fallback. One slow panel degrades ONE panel.
        CompletableFuture<Panel> viewers = panel("viewers",
                () -> redis.liveViewers(titleId));
        CompletableFuture<Panel> trend = panel("trend",
                () -> dynamo.trend24h(titleId));
        CompletableFuture<Panel> meta = panel("meta",
                () -> lruTitleCache.get(titleId));       // REQ-CC-03
        CompletableFuture<Panel> alerts = panel("alerts",
                () -> redis.activeAlerts(titleId));
        CompletableFuture<Panel> errors = panel("errors",
                () -> redis.topErrorCodes(titleId));

        // allOf completes when all five settle. Because each already has a fallback,
        // this NEVER fails — the page always renders.
        CompletableFuture.allOf(viewers, trend, meta, alerts, errors).join();

        // join() after allOf is non-blocking in practice: they're all done.
        return new CommandCenterDto(viewers.join(), trend.join(), meta.join(),
                                    alerts.join(), errors.join());
    }

    private CompletableFuture<Panel> panel(String name, Supplier<Object> fetch) {
        return CompletableFuture
                .supplyAsync(() -> Panel.ok(name, fetch.get()), panels)   // ← explicit executor
                .completeOnTimeout(Panel.degraded(name), budget.toMillis(), TimeUnit.MILLISECONDS)
                .exceptionally(ex -> {                                     // failure ≠ timeout
                    meters.counter("commandcenter.panel.degraded", "panel", name).increment();
                    log.warn("panel {} failed", name, ex);
                    return Panel.degraded(name);
                });
    }
}
```

```java
// thenApply vs thenCompose — the type tells you which you need.
CompletableFuture<TitleMeta> meta   = fetchTitle(id);                 // CF<TitleMeta>

CompletableFuture<String>    name   = meta.thenApply(TitleMeta::name);         // T → U      : map
CompletableFuture<Artwork>   art    = meta.thenCompose(m -> fetchArt(m.artId()));// T → CF<U> : flatMap
// meta.thenApply(m -> fetchArt(m.artId()))  would be CF<CF<Artwork>> — the classic nesting bug.

// thenCombine — two INDEPENDENT calls, joined. Both run concurrently.
CompletableFuture<PageDto> page = meta.thenCombine(fetchViewers(id), PageDto::new);
```

```java
// completeOnTimeout vs orTimeout — a one-word difference with opposite semantics.
cf.orTimeout(150, MILLISECONDS);                      // → TimeoutException on the error path
cf.completeOnTimeout(Panel.degraded(), 150, MILLISECONDS);  // → a normal value, no exception
// REQ-CC-02 wants a DEGRADED panel, not a failed page, so completeOnTimeout is the primitive.
// Caveat worth saying: neither one CANCELS the underlying work. The slow call is still
// running and still holding a thread/connection. If that matters, the cancellation has to
// come from the client's own timeout (WebClient responseTimeout, JDBC queryTimeout).
```

**Follow-ups they'll ask:**
- *"How do you get the values out of `allOf`?"* — `allOf` returns `CompletableFuture<Void>`, so you
  join each original future afterwards (as above), or `Stream.of(cfs).map(CompletableFuture::join)`
  *after* the `allOf` completes. Joining before the `allOf` serialises them and destroys the point.
- *"What does `join` throw versus `get`?"* — `join` throws unchecked `CompletionException`; `get`
  throws checked `ExecutionException` and `InterruptedException`. Both wrap the original cause, so
  always unwrap before logging or you lose the real exception in the message.
- *"Does `cancel(true)` interrupt the running task?"* — for `CompletableFuture`, **no**: it completes
  the future exceptionally but does not interrupt the supplier. That surprises people coming from
  `FutureTask`.
- *"Is this just worse `Mono.zip`?"* — they are the same shape. The differences: Reactor gives
  backpressure and cancellation propagation, `CompletableFuture` is in the JDK with no dependency and
  composes with blocking code. For a five-way fan-out with no streaming, either is fine; I'd match
  whatever the service already is.
- *"Which pool does `thenApply` run on if the future is already complete?"* — the **calling** thread.
  Worth knowing because it means a chain assembled after completion executes entirely inline.

**Pitfalls / red flags:** `supplyAsync` with no executor; `thenApply` doing blocking work; catching
nothing, so a failed panel takes down the whole page; or calling `get()` inside a loop and turning a
parallel fan-out into a sequential one.

**Where in StreamPulse:** REQ-CC-01, REQ-CC-02 (`CommandCenterTimeoutTest` stubs a 2s panel and
asserts p95 < 250ms with that panel `DEGRADED`), Phase 2 (async Kafka sends), Phase 5.

**Verify:** `ForkJoinPool.commonPool()` parallelism is `availableProcessors() - 1` and is influenced
by `-Djava.util.concurrent.ForkJoinPool.common.parallelism` — confirm the behaviour on your JDK,
especially inside a CPU-limited container.

---

## Q9. Virtual threads: what actually happens, and what are the rules?

**30-second answer:** A virtual thread is a `Thread` scheduled by the JVM onto a small pool of
**carrier** platform threads (a `ForkJoinPool` sized to the core count by default). When it blocks on
something the JDK has instrumented — socket I/O, `ReentrantLock`, `BlockingQueue`, `Thread.sleep` —
the JVM **unmounts** it: the continuation's stack is copied to the heap and the carrier moves on to
another virtual thread. That is why a million of them are affordable: each costs a heap-allocated
stack that grows on demand, not a fixed OS stack. Three rules follow. **Don't pool them** — they are
designed to be created per task, and a pool reintroduces the limit you were removing. **Don't use them
for CPU-bound work** — the carrier pool is core-sized, so there is nothing to gain. And **you still
need a `Semaphore`**, because removing the thread limit does not create capacity in the database, the
partner API, or S3; it just moves the queue somewhere worse.

**Deeper:**

```
  1,000,000 virtual threads          ForkJoinPool of carriers (≈ #cores)
  ┌──────────────────────┐            ┌──────────────┐
  │ vt-1 (RUNNABLE) ─────┼──mounted──▶│ carrier-1    │
  │ vt-2 (parked on I/O) │            │ carrier-2    │
  │ vt-3 (parked)        │            │ …            │
  │ …                    │            └──────────────┘
  └──────────────────────┘
       stacks live on the HEAP, copied in/out on mount/unmount
```

**Pinning** — a virtual thread that *cannot* unmount and therefore occupies its carrier while blocked:

| Cause | Status |
|---|---|
| Blocking inside a `synchronized` block/method | pins on **JDK 21**; I believe JEP 491 (JDK 24) fixed this — **verify for your JDK** |
| Blocking inside a native frame / JNI / a foreign function call | still pins |
| `Object.wait()` | pinned in early versions — **verify** |

Detection on JDK 21 is `-Djdk.tracePinnedThreads=full`, which prints a stack trace when a pinned
thread blocks. **That flag's status changed in later JDKs in favour of a JFR event
(`jdk.VirtualThreadPinned`) — verify which mechanism your JDK offers.** The Phase 9 lab exists
precisely to run this and record what it actually printed.

**Why pooling virtual threads is wrong**, in one line each:
- The point of a pool is to amortise expensive thread creation. Virtual threads are cheap to create,
  so there is nothing to amortise.
- A pool of size N reimposes a concurrency limit of N — the exact thing you switched to virtual
  threads to remove — but now in a place nobody will think to look.
- `ThreadLocal`s persist across tasks in a pooled thread, so a pool reintroduces leakage between
  unrelated requests (Q12).

**What a virtual thread does *not* fix:** the downstream. Ten thousand virtual threads each taking a
HikariCP connection means 9,990 of them queue on the pool, and connection-acquisition timeouts look
like a database outage in your dashboards. REQ-DL-07 has this as an explicit lab. The correct shape is
"unbounded threads, explicitly bounded *resources*", and that is what the semaphore expresses.

**Code:**

```java
/**
 * REQ-CC-05 — the virtual-thread alternative to the ThreadPoolExecutor in Q7,
 * selected by a property so Phase 9 can benchmark both.
 *
 * Note what replaced what:
 *   ThreadPoolExecutor(core, max, queue)  →  unlimited threads + Semaphore(16)
 * The semaphore is now the ONLY bound. Make it explicit and configurable, and
 * name it after the resource it protects — not after the pool it replaced.
 */
@Bean
@ConditionalOnProperty(name = "streampulse.s3.uploader", havingValue = "virtual")
S3Uploader virtualThreadUploader(S3Client s3, UploadProperties props) {

    ExecutorService exec = Executors.newVirtualThreadPerTaskExecutor();   // one VT per task
    Semaphore permits = new Semaphore(props.maxConcurrentUploads());      // 16: S3 client conns

    return batch -> CompletableFuture.runAsync(() -> {
        try {
            // acquire BEFORE doing the work; this is the backpressure point.
            // With no timeout this queues unboundedly in the semaphore's wait set —
            // so use the timed form and translate a failure into a real decision.
            if (!permits.tryAcquire(props.acquireTimeout().toMillis(), TimeUnit.MILLISECONDS)) {
                throw new UploadRejectedException("S3 uploader saturated");   // → spool or 429
            }
            try {
                s3.putObject(batch.request(), batch.body());
            } finally {
                permits.release();          // finally, always
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new CompletionException(e);
        }
    }, exec);
}
```

```java
// ❌ the anti-pattern the interviewer is checking for
var wrong = Executors.newFixedThreadPool(200, Thread.ofVirtual().factory());
//           ^ a bounded pool OF virtual threads: you pay the complexity and keep the limit

// ✅ create per task; the try-with-resources close() waits for all submitted tasks
try (var exec = Executors.newVirtualThreadPerTaskExecutor()) {
    for (var task : tasks) exec.submit(task);
}   // ExecutorService is AutoCloseable since Java 19 — close() == shutdown + awaitTermination
```

**Follow-ups they'll ask:**
- *"Do `ThreadLocal`s work?"* — yes, and that is a problem: a million virtual threads each with a
  `ThreadLocal` holding a buffer is a million buffers. `ScopedValue` is the intended replacement
  (immutable, scoped, shared with child tasks). **Verify whether `ScopedValue` is preview or final in
  your JDK** — it moved through several releases.
- *"How do you name them for a thread dump?"* — `Thread.ofVirtual().name("s3-upload-", 0).factory()`.
  Unnamed virtual threads make a dump unreadable, and `jcmd Thread.dump_to_file -format=json` is the
  tool that can actually dump millions of them.
- *"Does Spring use them automatically?"* — with `spring.threads.virtual.enabled=true` Boot switches
  the servlet container's request handling and several task executors to virtual threads.
  **Verify the property and its exact scope for your Boot version.**
- *"So is the thread pool dead?"* — no. Pools are still the right tool for **CPU-bound** work (you
  want exactly core-many) and for anywhere you want an explicit, observable concurrency bound with a
  queue and a rejection policy. `CallerRunsPolicy` has no virtual-thread equivalent.

**Pitfalls / red flags:** "virtual threads mean we don't need to think about limits any more." The
opposite is true — they *delete* the accidental limit that thread pools were providing, so every limit
now has to be deliberate.

**Where in StreamPulse:** Phase 2 (`ingest-gateway` on virtual threads + in-flight semaphore),
REQ-CC-05 (the alternative uploader + ADR), Phase 9 (pinning detection lab, benchmark),
REQ-DL-07 (pool exhaustion with virtual threads).

**Verify:** JEP 491's delivery version, `-Djdk.tracePinnedThreads` vs the JFR pinning event,
`ScopedValue`'s preview status, and `spring.threads.virtual.enabled` — all moved recently.

---

## Q10. How do deadlocks happen, how do you prevent them, and how do you prove one is happening?

**30-second answer:** Four conditions must hold simultaneously — Coffman's: **mutual exclusion**,
**hold and wait**, **no preemption**, and **circular wait**. Break any one and deadlock is impossible.
In practice the one you break is circular wait, by imposing a **global lock ordering**: if every
thread acquires locks in the same total order, a cycle cannot form. The belt-and-braces addition is
`tryLock` with a timeout, which breaks "no preemption" — a thread that cannot get the second lock
releases the first, backs off with jitter, and retries. To prove it, you do not guess: `jstack` (or
`jcmd Thread.print`) prints a "Found one Java-level deadlock" section naming the threads and the
monitors, and programmatically `ThreadMXBean.findDeadlockedThreads()` returns the thread IDs. In
StreamPulse this is a deliberate lab: v1 of the quota transfer deadlocks under crossed transfers, and
the thread dump is committed to the repo.

**Deeper:**

| Coffman condition | How you'd break it | Practical? |
|---|---|---|
| Mutual exclusion | use immutable data or lock-free structures | sometimes — the best fix when it works |
| Hold and wait | acquire all locks atomically, or none | hard; needs a lock manager |
| No preemption | `tryLock(timeout)` — give up and release | **yes**, and it also bounds the damage |
| Circular wait | **global lock ordering** | **yes** — the standard fix |

The crossed-transfer deadlock, as a picture:

```
T1: transfer(streamflix → sportsnow)     T2: transfer(sportsnow → streamflix)
    lock(streamflix)   ✔                     lock(sportsnow)   ✔
    lock(sportsnow)    ✖ waits for T2        lock(streamflix)  ✖ waits for T1
                    └──────── circular wait ────────┘
```

Note this is *not* a Java-specific problem — the identical bug exists in SQL, where two transactions
update the same two rows in opposite order and the database detects the cycle and kills one as a
deadlock victim. Being able to say "same shape, and Postgres solves it by *detecting* rather than
*preventing*" shows you understand the class of problem, and it connects directly to REQ-DL-05.

**Code — the bug and the fix, which is REQ-CC-07:**

```java
// ❌ v1 — locks in argument order. Deadlocks under crossed transfers.
@Service
class QuotaTransferServiceV1 {
    void transfer(String from, String to, long amount) {
        synchronized (bucketFor(from)) {          // T1 holds streamflix, wants sportsnow
            synchronized (bucketFor(to)) {        // T2 holds sportsnow,  wants streamflix
                bucketFor(from).withdraw(amount);
                bucketFor(to).deposit(amount);
            }
        }
    }
}
```

```java
/**
 * ✅ v2 — two independent fixes applied together (REQ-CC-07).
 *
 *  1. LOCK ORDERING by tenantId: every thread acquires the lexicographically smaller
 *     id first, so a cycle cannot form. This alone makes deadlock impossible.
 *  2. tryLock WITH TIMEOUT: defence in depth. If an ordering bug is ever reintroduced,
 *     the system degrades to a failed request with a metric, not a hung thread pool.
 *     ReentrantLock, not synchronized, because synchronized has no timed acquire
 *     (and on JDK 21 it also pins virtual threads — Q9).
 */
@Service
class QuotaTransferServiceV2 {

    private final ConcurrentHashMap<String, ReentrantLock> locks = new ConcurrentHashMap<>();

    void transfer(String from, String to, long amount) throws InterruptedException {
        if (from.equals(to)) return;

        // 1. Total order on the lock identity. Any deterministic comparator works —
        //    tenantId here; System.identityHashCode with a tie-breaker if there is no natural key.
        String first  = from.compareTo(to) < 0 ? from : to;
        String second = from.compareTo(to) < 0 ? to   : from;

        ReentrantLock l1 = locks.computeIfAbsent(first,  k -> new ReentrantLock());
        ReentrantLock l2 = locks.computeIfAbsent(second, k -> new ReentrantLock());

        // 2. Timed acquisition. Note the careful unwinding on partial failure.
        if (!l1.tryLock(200, TimeUnit.MILLISECONDS)) {
            throw new QuotaBusyException(first);
        }
        try {
            if (!l2.tryLock(200, TimeUnit.MILLISECONDS)) {
                throw new QuotaBusyException(second);      // l1 released by the outer finally
            }
            try {
                bucketFor(from).withdraw(amount);          // invariant: total quota is conserved
                bucketFor(to).deposit(amount);             // asserted by REQ-CC-08's stress test
            } finally {
                l2.unlock();
            }
        } finally {
            l1.unlock();
        }
    }
}
```

**Diagnosing it on a running JVM — the sequence I'd actually run:**

```bash
jcmd -l                                  # find the pid
jcmd <pid> Thread.print > dump.txt       # or: jstack <pid>
grep -A 40 "Found one Java-level deadlock" dump.txt
```

```
Found one Java-level deadlock:
=============================
"http-nio-8080-exec-3":
  waiting to lock monitor 0x... (object 0x...a1, a QuotaBucket),
  which is held by "http-nio-8080-exec-7"
"http-nio-8080-exec-7":
  waiting to lock monitor 0x... (object 0x...b2, a QuotaBucket),
  which is held by "http-nio-8080-exec-3"
```

Two caveats worth volunteering, because they are what an experienced person knows:
- `jstack` detects deadlocks on **monitors and `java.util.concurrent.Lock`s**. It does **not** detect
  a logical livelock, a `Semaphore` permit starvation, or a thread-pool self-deadlock (a task in the
  pool waiting on another task in the same pool). For those, the dump still helps but you read it
  yourself.
- Thread-pool self-deadlock is the one people miss. `REQUIRES_NEW` nested transactions exhausting a
  small connection pool (file 01 Q9) is the same bug in another costume.

**Follow-ups they'll ask:**
- *"What if there's no natural ordering key?"* — `System.identityHashCode`, with a tie-breaker lock
  for the rare collision (the classic "account transfer" solution from *Java Concurrency in Practice*).
- *"Livelock and starvation?"* — livelock: threads keep changing state in response to each other and
  make no progress (two `tryLock` loops retrying in lockstep — fixed with **randomised** backoff, the
  same jitter argument as REQ-RS-01). Starvation: a thread never gets the lock; fixed with a fair lock,
  at a throughput cost.
- *"Can you deadlock without locks?"* — yes: a bounded queue with producers and consumers on the same
  pool, a `CountDownLatch` nobody counts down, two `CompletableFuture`s each joining the other.
- *"How do you prevent this class of bug in CI?"* — the stress test in REQ-CC-08 (crossed transfers
  under a `CyclicBarrier`, asserting `findDeadlockedThreads() == null` and that total quota is
  conserved), plus a code review rule that nested lock acquisition needs a documented order.

**Pitfalls / red flags:** "we just made the lock bigger" (a single global lock does prevent deadlock —
by destroying concurrency), or knowing the four conditions by name but not which one your fix breaks.

**Where in StreamPulse:** REQ-CC-07 (`QuotaTransferDeadlockTest`, `docs/labs/deadlock.md` with the real
dump), REQ-DL-05 (the database-level twin: `SELECT … FOR UPDATE` ordering), Phase 9.

---

## Q11. What is a race condition, and how is it different from a visibility problem?

**30-second answer:** A **visibility** problem is one thread not seeing another's write at all — the
JMM question from Q1, fixed with `volatile`, a lock, or safe publication. A **race condition** is a
correctness bug that depends on *timing*: the classic form is **check-then-act**, where you test a
condition and then act on it, and between the two another thread invalidates the test. `if (!map.containsKey(k)) map.put(k, v)`
is the canonical example, and note that every individual operation there is thread-safe — the
composition is not. That is the point I make: *thread-safe components do not compose into a
thread-safe system*. The fix is either to make the compound operation atomic (`putIfAbsent`,
`computeIfAbsent`, a CAS loop, a conditional write in the database) or to hold a lock across the whole
sequence.

**Deeper:**

The three shapes, with the StreamPulse example of each:

| Shape | Example | Fix |
|---|---|---|
| Check-then-act | "is this `eventId` already processed? no → process it" | `SET NX EX` in Redis (one round trip, atomic) |
| Read-modify-write | `counter = counter + 1` | `LongAdder` / `AtomicLong` / `UPDATE … SET n = n + 1` |
| Multi-step invariant | withdraw from A, deposit to B — total must be conserved | a lock spanning both, or a DB transaction |

**Idempotency as a race-condition strategy** is the distributed-systems version of this answer, and it
is the one that matters most for an event hub. StreamPulse's ingest dedup is exactly a check-then-act
race across *processes*, where no JVM lock can help:

```java
// ❌ check-then-act across processes: two pods can both see "absent" and both publish.
if (redis.get(eventId) == null) {
    redis.set(eventId, "1", ttl);
    kafka.send(event);            // duplicate published
}

// ✅ one atomic operation. SET NX returns whether WE were the one who set it.
Boolean weWon = redis.opsForValue().setIfAbsent(key(eventId), "1", ttl);
if (Boolean.TRUE.equals(weWon)) {
    kafka.send(event);
}
// Honest caveat to state out loud: this is still not exactly-once. If the pod dies between
// the SET and the send, the event is dropped. Ordering it the other way (send, then SET)
// gives at-least-once instead, which combined with idempotent CONSUMERS is the design
// StreamPulse chose — correctness comes from the consumer, not from this line.
```

The DynamoDB twin is a conditional write (`attribute_not_exists(PK)`), and the Postgres twin is a
unique constraint or `INSERT … ON CONFLICT DO NOTHING`. Same pattern, three storage engines: **make the
store enforce the invariant, because it is the only thing all the processes share.**

**Code — a race you can demonstrate in a test:**

```java
@Test
@DisplayName("REQ-CC-08: check-then-act loses updates; computeIfAbsent does not")
void checkThenActLosesUpdates() throws Exception {
    var broken = new HashMap<String, LongAdder>();   // deliberately not concurrent
    var fixed  = new ConcurrentHashMap<String, LongAdder>();
    var start  = new CountDownLatch(1);
    var done   = new CountDownLatch(64);

    try (var pool = Executors.newVirtualThreadPerTaskExecutor()) {
        for (int i = 0; i < 64; i++) {
            pool.submit(() -> {
                start.await();
                fixed.computeIfAbsent("t", k -> new LongAdder()).increment();
                done.countDown();
                return null;
            });
        }
        start.countDown();
        done.await(5, TimeUnit.SECONDS);
    }
    assertThat(fixed.get("t").sum()).isEqualTo(64);
    // The `broken` map is left in the test as the lab: run it enough times and the count
    // is short, or you get a corrupted HashMap (an infinite loop on resize, historically).
}
```

**Follow-ups they'll ask:**
- *"Is a race condition always a bug?"* — no. A benign race is one where any interleaving gives an
  acceptable result — e.g. two threads both computing and caching the same immutable value. Being able
  to identify those stops you over-synchronising.
- *"How do you find races that only show up in production?"* — stress tests with a `CountDownLatch`
  starting gun, run many iterations; `-XX:+UseZGC`/different hardware to change interleavings; jcstress
  for the genuinely subtle JMM ones; and, most practically, invariant assertions (total quota
  conserved) rather than output assertions.
- *"TOCTOU in the file system / S3?"* — same shape: "does this key exist? no → write it". The fix is a
  conditional put, not a check.

**Pitfalls / red flags:** conflating the two ("we added `volatile` to fix the race"), or believing that
using `ConcurrentHashMap` makes the *code* thread-safe rather than the *map*.

**Where in StreamPulse:** Phase 2 (Redis `SET NX EX` dedup, plus a concurrency test firing duplicate
batches in parallel and asserting exactly one publish), REQ-DL-05 (`SELECT … FOR UPDATE`),
REQ-CC-07/08.

---

## Q12. `ThreadLocal` — what is it for, and how does it leak?

**30-second answer:** `ThreadLocal` gives each thread its own copy of a value, and it is how most
framework context works — `RequestContextHolder` for `@RequestScope`, SLF4J's MDC, Spring Security's
`SecurityContextHolder`, the transaction-bound EntityManager. The leak comes from its
implementation: each `Thread` holds a `ThreadLocalMap` whose **keys are weak references to the
`ThreadLocal` object but whose values are strong**. So if the thread outlives the value — which is
exactly what a thread *pool* guarantees — the value stays reachable until that thread dies or
something calls `remove()`. In an application server the classic consequence is a redeployed webapp
whose entire classloader is pinned by one `ThreadLocal` value on a pooled thread. The rule is: set it
in a `try`, `remove()` it in the `finally`, always, or use the closeable wrappers frameworks provide.

**Deeper:**

```
Thread (pooled, lives for the JVM's lifetime)
  └─ ThreadLocalMap
       └─ Entry:  key = WeakReference<ThreadLocal>   value = STRONG reference ──▶ your object
                  ↑ collectible                                                   ↑ NOT collectible
                    if the ThreadLocal itself is GC'd, the entry becomes a "stale entry";
                    the map clears some of those opportunistically on later get/set/remove,
                    but that is best-effort — it is not a substitute for remove().
```

Three distinct problems, worth separating:
1. **Memory leak** — the value never released; the heap dump shows it under `Thread` →
   `threadLocals`.
2. **Data leakage between requests** — worse than a leak and a genuine security bug: request 2, served
   by the same pooled thread, sees request 1's tenant because nobody cleared it. In a multi-tenant
   system this is a cross-tenant data exposure.
3. **Doesn't propagate** — the value does not follow work handed to another thread, which is why
   MDC/tenant context disappears inside `CompletableFuture.supplyAsync` and inside a reactive chain.
   `InheritableThreadLocal` only copies at thread *creation*, so it does nothing for a pooled thread
   that already exists.

**Code:**

```java
/**
 * StreamPulse tenant context (used by the MDC filter and the @TenantScoped aspect).
 * The try/finally is not optional — with virtual threads the leak is smaller (the thread
 * dies with the request) but the FILTER is the same code that runs on Tomcat's pooled
 * platform threads in control-plane, where it very much matters.
 */
public final class TenantContextHolder {

    private static final ThreadLocal<String> CURRENT = new ThreadLocal<>();

    public static String current() {
        String t = CURRENT.get();
        if (t == null) throw new IllegalStateException("no tenant bound to this thread");
        return t;
    }

    /** try-with-resources so the remove() cannot be forgotten. */
    public static Scope bind(String tenantId) {
        CURRENT.set(tenantId);
        return CURRENT::remove;              // Scope extends AutoCloseable
    }

    public interface Scope extends AutoCloseable { @Override void close(); }
}

// usage — the remove() is structural, not a discipline
try (var ignored = TenantContextHolder.bind(jwt.getClaimAsString("tenant"))) {
    chain.doFilter(req, res);
}
```

```java
/**
 * Propagating context across a thread hand-off (REQ-CC-01's fan-out).
 * Capture on the caller, re-bind on the worker. Do NOT pass the ThreadLocal itself.
 */
String tenant = TenantContextHolder.current();          // capture on the request thread
String traceId = MDC.get("traceId");

CompletableFuture.supplyAsync(() -> {
    try (var ignored = TenantContextHolder.bind(tenant);
         var ignored2 = MDC.putCloseable("traceId", traceId)) {
        return redis.liveViewers(titleId);
    }
}, panels);
// Alternative: wrap the Executor once so every submitted task does this automatically —
// a TaskDecorator in Spring, or Micrometer's ContextSnapshot/context-propagation library.
// Verify the current context-propagation API names for your versions.
```

**Follow-ups they'll ask:**
- *"Do virtual threads change this?"* — the *leak* mostly goes away, because a virtual thread is
  created per task and dies with it. The *propagation* problem does not. And a million virtual threads
  each holding a `ThreadLocal` value is a new memory shape to think about — which is part of why
  `ScopedValue` was designed (immutable, bounded to a dynamic scope, inherited by structured child
  tasks). **Verify its preview/final status for your JDK.**
- *"How do you find one in a heap dump?"* — Eclipse MAT, dominator tree, look for `Thread` →
  `threadLocals` → `ThreadLocalMap` → `Entry[]` with large retained size. Phase 9 has this as a
  deliberate lab: induce the leak, find it, fix it.
- *"Why is a `static` `ThreadLocal` the recommended form?"* — a per-instance `ThreadLocal` in a
  short-lived object creates a new key per instance, so the map grows with stale entries. `static
  final` is one key for the lifetime of the class.
- *"How does Spring propagate the security context to `@Async`?"* — a `DelegatingSecurityContextExecutor`
  / `TaskDecorator`; the same capture-and-rebind pattern, provided by the framework.

**Pitfalls / red flags:** `ThreadLocal.set()` with no matching `remove()`, or assuming context follows
the work to another thread.

**Where in StreamPulse:** `libs/platform` MDC/tenant filter, Phase 9 (the induced leak found via heap
dump — the plan lists "static map / ThreadLocal" as the two induced leaks), REQ-LC-04 (request scope,
which is a `ThreadLocal` underneath).

---

## Q13. How do you test concurrent code so the test is not flaky?

**30-second answer:** By removing the timing from the test. Three techniques do almost all of it.
**A latch as a starting gun** so N threads genuinely run simultaneously instead of staggering — a
`for` loop that starts threads is not a concurrency test. **A barrier for deterministic interleaving**
when I need a specific order of events, like the crossed-lock deadlock. And **assert invariants, not
outputs** — "total quota is conserved", "no counter increment was lost", "exactly one publish happened"
— because an invariant holds under every interleaving whereas an expected output usually only holds
under one. Then I run it many iterations, and if the failure is subtle enough to be a memory-model
question I reach for jcstress rather than pretending a unit test will find it. And `Thread.sleep` never
appears in the assertion path: `Awaitility` with a polling condition and a timeout replaces it.

**Deeper:**

| Technique | Removes | Example in StreamPulse |
|---|---|---|
| `CountDownLatch` starting gun | stagger between thread starts | `LruConcurrencyStressTest` |
| `CyclicBarrier` rendezvous | non-determinism in *which* interleaving | `QuotaTransferDeadlockTest` (REQ-CC-07) |
| Invariant assertions | dependence on a specific interleaving | total quota conserved (REQ-CC-08) |
| `Awaitility.await().atMost(…).until(…)` | `sleep`-based waits | waiting for a Kafka offset to advance |
| `ThreadMXBean.findDeadlockedThreads()` | "it hung, probably a deadlock" guessing | the deadlock assertion |
| Repeat the test (`@RepeatedTest`) | one-shot luck | the dedup race (Phase 2) |
| jcstress | JMM-level reordering questions | the `volatile` visibility lab |
| Testcontainers + Toxiproxy | fake latency assumptions | `S3OffloadBackpressureIT` |

The thing that makes these tests *valuable* rather than decorative is that they encode a requirement:
`@DisplayName("REQ-CC-08: …")` ties the test to the spec, which is the StreamPulse convention.

**Code:**

```java
/**
 * REQ-CC-08 — the invariant test. Notice there is no expectation about ORDER,
 * only about what must be true afterwards no matter what order happened.
 */
@RepeatedTest(20)
@DisplayName("REQ-CC-08: crossed quota transfers conserve total quota and never deadlock")
void transfersConserveTotalQuota() throws Exception {
    var quotas = new QuotaBuckets(Map.of("streamflix", 10_000L, "sportsnow", 10_000L));
    var service = new QuotaTransferServiceV2(quotas);
    long totalBefore = quotas.total();

    int threads = 32, transfersPerThread = 300;
    var start = new CountDownLatch(1);
    var done  = new CountDownLatch(threads);

    try (var pool = Executors.newVirtualThreadPerTaskExecutor()) {
        for (int i = 0; i < threads; i++) {
            boolean forward = i % 2 == 0;       // half the threads transfer the other way: crossed
            pool.submit(() -> {
                start.await();
                for (int j = 0; j < transfersPerThread; j++) {
                    try {
                        if (forward) service.transfer("streamflix", "sportsnow", 1);
                        else         service.transfer("sportsnow", "streamflix", 1);
                    } catch (QuotaBusyException ignored) {
                        // tryLock timeout is an ACCEPTABLE outcome; a hang is not.
                    }
                }
                done.countDown();
                return null;
            });
        }
        start.countDown();
        assertThat(done.await(20, TimeUnit.SECONDS))
                .as("no thread is stuck — a false here means a deadlock or livelock")
                .isTrue();
    }

    assertThat(quotas.total()).isEqualTo(totalBefore);                     // the invariant
    assertThat(ManagementFactory.getThreadMXBean().findDeadlockedThreads()).isNull();
}
```

**Follow-ups they'll ask:**
- *"Doesn't `@RepeatedTest(20)` just hide flakiness?"* — it makes an intermittent failure *more*
  likely to surface in CI, which is the opposite of hiding it. What hides flakiness is a retry-on-fail
  plugin, and I would not use one on concurrency tests.
- *"How do you test that something is *not* concurrent — e.g. that exactly one publish happened?"* —
  fire N duplicate batches in parallel against a real Redis (Testcontainers) and assert
  `verify(publisher, times(1))`. That is Phase 2's dedup test and it is the strongest kind: it can
  only pass if the atomicity is real.
- *"Why Testcontainers rather than an embedded fake?"* — because the behaviour under test *is* the
  real Redis/Kafka semantics (`SET NX` atomicity, rebalance behaviour). A fake tests your fake.
- *"How do you keep these fast?"* — separate `integrationTest` source set, so the unit suite stays in
  seconds and the slow ones run on their own Gradle task and in CI.

**Pitfalls / red flags:** `Thread.sleep(500); assertThat(...)`. It is both slow and unreliable, and it
is the single clearest signal that someone has not really written concurrent tests.

**Where in StreamPulse:** `LruConcurrencyStressTest`, `QuotaTransferDeadlockTest`,
`TenantContextConcurrencyTest` (REQ-LC-04), `S3OffloadBackpressureIT` (REQ-CC-04), and the Phase 2
duplicate-batch dedup test.

---

## Q14. Take me through concurrency in StreamPulse end to end — where did you use what, and why?

*(The applied question. Everything above becomes evidence here.)*

**30-second answer:** Five places, each with a different tool because they are different problems.
**One**: the Command Center fans out to five panels with `CompletableFuture.supplyAsync` on a *named*
executor — never the common pool — with `completeOnTimeout` per panel, so one slow dependency degrades
one panel instead of the page. **Two**: `sink-writer` offloads S3 uploads to a bounded
`ThreadPoolExecutor` with `CallerRunsPolicy`, so when S3 is slow the Kafka listener thread does the
upload itself, consumption slows, lag rises and the autoscaler reacts — backpressure with no extra
code. **Three**: per-tenant counters are `LongAdder` in a `ConcurrentHashMap` via `computeIfAbsent`,
because the write rate is enormous and the read rate is one Prometheus scrape per 15 seconds.
**Four**: the title-metadata LRU is guarded by a `ReentrantReadWriteLock` and takes the *write* lock on
a read, because a cache hit mutates recency order — which is a real design discussion, not a mistake.
**Five**: the quota-transfer deadlock lab — v1 locks in argument order and deadlocks under crossed
transfers, v2 fixes it with lock ordering plus `tryLock(timeout)`, with the thread dump in the repo.
And the through-line: `ingest-gateway` runs on **virtual threads**, so the accidental limit a thread
pool used to give me is gone, and every limit — the in-flight semaphore, the upload permits, the
connection pool — had to become explicit.

**The map, as one table:**

| Where | Problem | Tool | Why not the alternative | REQ |
|---|---|---|---|---|
| Command Center fan-out | 5 sequential calls ≈ 600ms, target 250ms p95 | `supplyAsync` on a named executor + `allOf` + `completeOnTimeout` | common pool is shared JVM-wide and `cores-1` wide | CC-01, CC-02 |
| Title metadata | repeated lookups of a small hot set | hand-written LRU + `ReentrantReadWriteLock` | `ConcurrentHashMap` has no eviction; Caffeine compared in an ADR | CC-03 |
| S3 upload offload | slow S3 stalls Kafka listener threads | bounded `ThreadPoolExecutor` + `ArrayBlockingQueue(200)` + `CallerRunsPolicy` | an unbounded queue turns latency into OOM | CC-04 |
| S3 upload, variant | benchmark the modern shape | virtual threads + `Semaphore(16)` | pooling virtual threads reimposes the limit | CC-05 |
| Per-tenant counters | 50k ev/s of increments, read every 15s | `LongAdder` in `ConcurrentHashMap` | `AtomicLong` contends on one cache line | CC-06 |
| Quota transfer | two locks, crossed order | lock ordering by tenantId + `tryLock(200ms)` | a single global lock kills concurrency | CC-07 |
| Ingest admission | virtual threads removed the natural bound | in-flight `Semaphore` → 429 + `Retry-After` | queueing the excess just moves the failure | Phase 2 |
| Graceful drain | ordered stop, flush before close | `SmartLifecycle` phases + a `volatile` running flag | `@PreDestroy` has no ordering guarantee | LC-01/02 |

**The one design decision I would defend hardest:** `CallerRunsPolicy` on the S3 pool. It looks like a
hack — "the caller does the work?" — but it is the only rejection policy that turns saturation into
*backpressure* instead of into loss or unbounded memory. The chain is: queue full → listener thread
uploads → listener not polling → consumer lag rises → KEDA scales `sink-writer` on lag (Phase 11) → lag
recovers. Every link is observable on the Phase 8 dashboard. The cost, which I would state without
being asked, is that a listener thread can be blocked for one upload's duration, so the upload needs
its own timeout or a stalled consumer triggers a rebalance — and that is exactly why REQ-KT-04 exists
(`max.poll.records × worst-case processing time < max.poll.interval.ms`).

**How it is verified:** `CommandCenterTimeoutTest` (a stubbed 2s panel; asserts the page still meets
p95 and that panel reports `DEGRADED`), `S3OffloadBackpressureIT` (Toxiproxy injects 1s of S3 latency;
asserts consumer lag stays under the threshold and nothing is dropped), `QuotaTransferDeadlockTest`
(v1 times out, v2 completes 10k crossed transfers with quota conserved), `LruConcurrencyStressTest`,
and a JMH report for `AtomicLong` vs `LongAdder` vs `synchronized`.

> **Your story:** you have a real concurrency-under-load story somewhere in 18 years — a
> browser-side race, a queue that grew unbounded, a caching layer that served one user's data to
> another. Put it here in the same shape: what broke, what the mechanism was, what you changed, how
> you proved it. Keep "I built" for what you personally wrote, and present StreamPulse as what it is —
> *"I built this to prove the pattern end to end."* Never describe BS-02 as an incident you lived
> through.

**Follow-ups they'll ask:**
- *"Why not make the whole thing reactive and get backpressure for free?"* — `query-api` is reactive,
  for exactly that reason (file 02 Q7). `sink-writer` is not, because Spring Kafka's batch listener
  with manual acks gives me an offset-commit story I can explain, and `CallerRunsPolicy` gives me the
  same backpressure with none of the reactive learning curve on a component that is mostly I/O
  plumbing. Different problems, different tools.
- *"What would you change if you rewrote it today on JDK 25+?"* — honestly: if JEP 491 has landed in
  the JDK I'm on, the `synchronized`-to-`ReentrantLock` conversions become unnecessary, and
  `ScopedValue` replaces the `ThreadLocal` context plumbing. **Both are things I'd verify against the
  release notes before saying so in an interview.**
- *"What is the first metric you'd look at in an incident?"* — thread-pool queue depth and rejection
  count, consumer lag, and connection-pool wait time. All three are on one Grafana row, and together
  they tell you which side of the hand-off is the bottleneck.

**Where in StreamPulse:** BS-02 end to end; REQ-CC-01…08; Phases 4, 5, 9 + `kata/`.

---

## Key terms

| Term | One-line meaning |
|---|---|
| **Happens-before** | The JMM's partial order; without an edge, one thread's write may never be seen by another. |
| **`volatile`** | Visibility + ordering for one field. No atomicity, no mutual exclusion. |
| **Safe publication** | Handing an object to another thread through a channel that creates a happens-before edge. |
| **CAS** | Compare-and-swap: the hardware primitive behind every `Atomic*` and lock-free structure. |
| **ABA** | A location returns to its old value, so CAS succeeds although the state changed. |
| **`LongAdder`** | Striped counter: fast contended writes, `sum()` is not an atomic snapshot. |
| **False sharing** | Unrelated variables on one cache line, so each write invalidates the other core's copy. |
| **`computeIfAbsent`** | Atomic get-or-create; the lambda runs **under the bin lock**, so keep it cheap and non-recursive. |
| **Bounded queue** | The only kind that applies backpressure; unbounded is a slower OOM. |
| **`CallerRunsPolicy`** | Saturation → the submitter executes the task → the submitter stops submitting. Backpressure. |
| **Core vs max pool size** | Max is only reached once the **queue is full** — with an unbounded queue it is unreachable. |
| **`thenApply` vs `thenCompose`** | `map` vs `flatMap`; the second avoids `CompletableFuture<CompletableFuture<T>>`. |
| **`orTimeout` vs `completeOnTimeout`** | Fail after d, vs substitute a default after d. Neither cancels the underlying work. |
| **`ForkJoinPool.commonPool()`** | `supplyAsync`'s default: `cores - 1` wide, shared JVM-wide. Don't put I/O on it. |
| **Carrier thread** | The platform thread a virtual thread is mounted on. |
| **Pinning** | A virtual thread that can't unmount — classically inside `synchronized` on JDK 21. |
| **Coffman conditions** | Mutual exclusion, hold-and-wait, no preemption, circular wait — break one, no deadlock. |
| **Lock ordering** | A global total order on lock acquisition; the standard fix for circular wait. |
| **Check-then-act** | The canonical race shape; fixed by making the compound operation atomic. |
| **`ThreadLocal` leak** | Weak key, **strong value**, pooled thread → the value is retained until `remove()`. |

---

## Self-check (answer out loud, no notes)

1. `ready = true` after `data = 42`, both non-volatile. Name both things that can go wrong, and say
   why making only `ready` volatile fixes both.
2. A `ThreadPoolExecutor` with core 4, max 16 and an unbounded queue is under heavy load. How many
   threads are running, and why?
3. Which pool does `CompletableFuture.supplyAsync(sup)` use, how wide is it, and what else in the JVM
   shares it?
4. Your LRU cache uses a `ReentrantReadWriteLock`. Why does `get()` take the **write** lock, and what
   are your two alternatives?
5. Name the four Coffman conditions, say which one lock ordering breaks and which one `tryLock` breaks,
   and describe the command you would run to prove a deadlock on a live JVM.
