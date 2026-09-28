# 17 — Coding round: LRU, LFU, and the BFS/DFS graph set

**Why this matters for an Event Hub role:** the prep doc names two problems outright — **Rotting
Oranges** (multi-source BFS) and **LRU Cache** — and the JD emphasises communication. That second
part is not a soft skill here: in a 45-minute round the interviewer is grading *how you think out
loud* almost as heavily as whether the code compiles. A candidate who narrates — restates the
problem, states the invariant, names the pattern, writes the signature, dry-runs an example — reads as
senior even with a small bug. A candidate who types in silence for 20 minutes and produces perfect
code reads as junior. So this file is structured differently from the rest: every problem has a
**narration script**, which is literally what you say while your hands are moving.

**Scenario anchor:** these are not detached puzzles. In StreamPulse, `kata/` is a real Gradle module
that starts in Phase 0 and runs in parallel with everything else, and two of these algorithms are
promoted into production code:
- the hand-written **LRU** becomes the title-metadata cache in `query-api` (Phase 5, **REQ-CC-03**),
  guarded by a `ReentrantReadWriteLock` and invalidated by `config.changes.v1` CDC events;
- **BFS** becomes the alert root-cause suppression walk over the service-dependency graph, and
  **topological sort** rejects cyclic rule dependencies at save time (**Phase 6**, control-plane alert
  engine; the same engine that BS-08 later extracts into its own service).

That is the single best thing you can say in a coding round: *"I've actually shipped this one — it's
the cache behind a dashboard endpoint, and the interesting part wasn't the algorithm, it was that a
cache **read** mutates the recency order, so it needs the write lock."*

---

## How to use this file

1. Read the **narration script** out loud. Not in your head — out loud. The words have to be
   available under pressure.
2. Write the solution from scratch on paper or in a plain editor, no autocomplete.
3. Then compare. If you got it right but silently, you have done half the exercise.
4. Time-box: 20 minutes for LRU/LFU, 15 for each graph problem. If you are over, the bottleneck is
   almost always that you started coding before you stated the invariant.

**The universal opening** (say this before touching the keyboard, every time):

> "Let me make sure I have the problem. *<restate it in one sentence>*. Can I check a couple of
> things: *<input size / can it be empty / are values unique / can I mutate the input>*. And can I
> assume *<the obvious constraint>*? … Good. My plan is *<pattern name>*, which gives me *<time>* time
> and *<space>* space. Let me write the signature first and then fill it in."

That is roughly 30 seconds and it changes the entire tone of the interview.

---

# Problem 1 — LRU Cache, from scratch

## Statement

Design a data structure for a **Least Recently Used** cache with a fixed capacity, supporting:

- `get(key)` → the value, or `-1` / `null` if absent. **A `get` counts as a use.**
- `put(key, value)` → insert or update. If this takes the size over capacity, evict the
  **least recently used** entry.

Both operations must be **O(1) average**. `LinkedHashMap` with `accessOrder = true` solves this in
three lines and **that is not what they are asking for** — they want the doubly linked list.

## Narration script

> "So: fixed capacity, `get` and `put` both O(1), and `get` counts as a use so it has to reorder.
>
> Two requirements pull in opposite directions. O(1) lookup by key says hash map. Maintaining a
> *recency order* with O(1) removal from the middle says linked list — and it has to be **doubly**
> linked, because when I find a node via the map I need to unlink it in O(1), which means I need its
> predecessor without walking.
>
> So the design is: `HashMap<K, Node>` for lookup, and a doubly linked list where **head is the most
> recently used and tail is the least**. `get` = map lookup, then move that node to the head. `put` =
> if present, update the value and move to head; if absent, create a node, add at head, put in the
> map, and if we are now over capacity, remove the tail node and remove its key from the map.
>
> That last bit is why the node stores its **key** as well as its value — when I evict from the tail I
> need the key to delete the map entry. That is the detail people miss and it is worth saying out
> loud.
>
> I'll use **sentinel head and tail nodes** so I never have a null check in `addFirst`/`unlink`.
> That's a few extra lines up front and it removes about five edge cases.
>
> Let me write the node class and the two list helpers first, then `get` and `put` become four lines
> each."

## Solution (Java 21)

```java
package kata.cache;

import java.util.HashMap;
import java.util.Map;

/**
 * LRU cache: HashMap for O(1) lookup + doubly linked list for O(1) recency reordering.
 *
 * Invariant maintained at all times:
 *   head.next  = most recently used
 *   tail.prev  = least recently used  (the eviction victim)
 *   map.size() == number of real nodes between the sentinels
 *
 * Single-threaded. The thread-safe variant is below (REQ-CC-03).
 */
public class LruCache<K, V> {

    /** Node holds the KEY as well as the value: eviction starts from the list and must delete from the map. */
    private static final class Node<K, V> {
        final K key;
        V value;
        Node<K, V> prev, next;
        Node(K key, V value) { this.key = key; this.value = value; }
    }

    private final int capacity;
    private final Map<K, Node<K, V>> index;

    // Sentinels: never removed, never hold data. They remove every null check from the list ops.
    private final Node<K, V> head = new Node<>(null, null);
    private final Node<K, V> tail = new Node<>(null, null);

    public LruCache(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException("capacity must be > 0");
        this.capacity = capacity;
        this.index = new HashMap<>(capacity * 2);   // room to avoid a resize at the target size
        head.next = tail;
        tail.prev = head;
    }

    /** O(1). NOTE: this MUTATES the recency order — a "read" is structurally a write. */
    public V get(K key) {
        Node<K, V> node = index.get(key);
        if (node == null) return null;
        moveToFront(node);
        return node.value;
    }

    /** O(1) amortised. */
    public void put(K key, V value) {
        Node<K, V> existing = index.get(key);
        if (existing != null) {
            existing.value = value;       // update in place — do NOT create a second node
            moveToFront(existing);
            return;
        }

        Node<K, V> node = new Node<>(key, value);
        index.put(key, node);
        addFirst(node);

        if (index.size() > capacity) {
            Node<K, V> lru = tail.prev;   // the real node just before the tail sentinel
            unlink(lru);
            index.remove(lru.key);        // ← this is why Node stores the key
        }
    }

    /** O(1). Needed in production for CDC-driven invalidation (Phase 6), not for the LeetCode form. */
    public void remove(K key) {
        Node<K, V> node = index.remove(key);
        if (node != null) unlink(node);
    }

    public int size() { return index.size(); }

    // ---- list primitives: each is unconditional because of the sentinels ----

    private void addFirst(Node<K, V> n) {
        n.prev = head;
        n.next = head.next;
        head.next.prev = n;
        head.next = n;
    }

    private void unlink(Node<K, V> n) {
        n.prev.next = n.next;
        n.next.prev = n.prev;
        n.prev = null;                    // help GC and make a double-unlink fail loudly
        n.next = null;
    }

    private void moveToFront(Node<K, V> n) {
        unlink(n);
        addFirst(n);
    }
}
```

## The thread-safe variant (REQ-CC-03) — and the discussion the interviewer wants

```java
package kata.cache;

import java.util.concurrent.locks.ReentrantReadWriteLock;

/**
 * StreamPulse query-api title-metadata cache (Phase 5, REQ-CC-03).
 *
 * THE POINT OF THIS CLASS, and the thing to say out loud:
 * a cache `get` is not a read. It moves the entry to the head of the recency list,
 * which mutates `prev`/`next` pointers on three nodes and the head sentinel. If two
 * threads did that under a shared READ lock, the list would be corrupted — you would
 * get lost entries, a cycle, or an entry that is in the map but not in the list and
 * therefore never evicted (a slow leak that only shows up in a heap dump).
 *
 * So get() takes the WRITE lock, and the ReadWriteLock buys us almost nothing over a
 * plain ReentrantLock. That is a real trade-off, not an oversight, and the three honest
 * options are:
 *   (a) write lock on get      — what we do: correct, simple, contention is the cost
 *   (b) segment the cache      — N independent LRUs by hash(key) % N; contention / N,
 *                                at the cost of approximate global LRU (each segment
 *                                evicts its own LRU, not the global one)
 *   (c) approximate recency    — record accesses into per-thread buffers and replay them
 *                                into the policy asynchronously. This is what Caffeine does,
 *                                and it is why the ADR compares against Caffeine rather than
 *                                pretending a hand-rolled cache is better.
 *
 * We ship (a) and measure it in Phase 9. If the metadata read rate ever makes the lock the
 * bottleneck, (b) is a 20-line change and (c) is "use Caffeine".
 */
public final class ThreadSafeLruCache<K, V> {

    private final ReentrantReadWriteLock lock = new ReentrantReadWriteLock();
    private final LruCache<K, V> delegate;

    public ThreadSafeLruCache(int capacity) {
        this.delegate = new LruCache<>(capacity);
    }

    public V get(K key) {
        lock.writeLock().lock();            // NOT readLock() — see the class comment
        try {
            return delegate.get(key);
        } finally {
            lock.writeLock().unlock();
        }
    }

    public void put(K key, V value) {
        lock.writeLock().lock();
        try {
            delegate.put(key, value);
        } finally {
            lock.writeLock().unlock();
        }
    }

    /** Invalidation from the config.changes.v1 CDC stream (Phase 6) — also a write. */
    public void invalidate(K key) {
        lock.writeLock().lock();
        try {
            delegate.remove(key);
        } finally {
            lock.writeLock().unlock();
        }
    }
}
```

## Complexity

| | Time | Space |
|---|---|---|
| `get` | O(1) average (hash lookup) | — |
| `put` | O(1) average, including eviction | — |
| Total | | O(capacity) — one map entry + one node per key |

Worst case for the hash map is O(n) with adversarial hash collisions; since Java 8 a bin treeifies to
O(log n). Worth one sentence if they push on "average".

## Common bugs (say these before you make them)

| Bug | Symptom |
|---|---|
| Node stores only the value | you can't remove the evicted key from the map — the map grows forever |
| No sentinels | six null checks, and one of them is wrong |
| `put` on an existing key creates a new node | duplicate nodes, size drifts, the old node leaks |
| Eviction checks `size() >= capacity` before inserting | off-by-one: the cache holds `capacity - 1` |
| `moveToFront` forgets to unlink first | the list develops a cycle; the next traversal hangs |
| Evicting `tail` instead of `tail.prev` | you unlink the sentinel and the list is destroyed |
| Thread-safe version uses the read lock on `get` | corruption under concurrency; the bug this file exists to teach |

## Follow-ups they'll ask

- *"Now make it thread-safe."* — above. Lead with *why* `get` is a write.
- *"Why not `LinkedHashMap`?"* — you can: `new LinkedHashMap<>(cap, 0.75f, true)` overriding
  `removeEldestEntry`. Say that you know it, then say why the hand-rolled version exists: it is the
  one they asked for, and in production it gives you a hook point for metrics, per-entry TTL and CDC
  invalidation that `LinkedHashMap` does not.
- *"Add a TTL."* — store an expiry timestamp on the node and treat an expired hit as a miss (lazy
  expiry). Eager expiry needs a `DelayQueue` or a sweeper, and you must then decide whether an expired
  entry still counts toward capacity.
- *"Make it LRU-K / add frequency."* — that is problem 2.
- *"What about a distributed cache?"* — different problem entirely: Redis with `maxmemory-policy
  allkeys-lru`, and now you are trading a network hop for shared state and cross-pod consistency. In
  StreamPulse the LRU is a per-pod L1 in front of Redis, and CDC events invalidate every pod's L1.
- *"How do you test it?"* — capacity enforcement, that `get` refreshes recency (put A,B,C with cap 2
  is the wrong test; put A,B, `get(A)`, put C, then assert **B** was evicted, not A), update-in-place,
  and a concurrency stress test (`LruConcurrencyStressTest`, REQ-CC-08).

---

# Problem 2 — LFU Cache

## Statement

Same contract as LRU, but evict the **Least Frequently Used** entry; ties are broken by **least
recently used among those with the minimum frequency**. `get` and `put` must be **O(1)**.

This is the "did you actually understand the LRU design or did you memorise it?" question.

## Narration script

> "LFU with O(1) is genuinely harder than LRU, so let me think about what makes it hard: I need
> 'the minimum frequency' in O(1), and frequencies change on every access.
>
> The trick is: **don't search for the minimum, track it.** I keep a `minFreq` field. The only two
> things that can change it are — a new insertion, which sets `minFreq = 1` unconditionally; and
> promoting the last remaining entry at `minFreq`, which increments it by exactly one. Frequencies
> only ever go up by one, so `minFreq` never has to jump.
>
> Data structures: `keyToNode` for O(1) lookup as before; `keyToFreq`, or store the frequency on the
> node; and `freqToList` — a map from frequency to a **doubly linked list of the keys at that
> frequency, in recency order**. That second list is just my LRU list again, one per frequency
> bucket. So LFU = LRU repeated per frequency, plus a `minFreq` pointer.
>
> On `get(k)`: look up the node, remove it from its current frequency bucket, increment its
> frequency, append it to the head of the new bucket. If the old bucket is now empty **and** it was
> `minFreq`, then `minFreq++`.
>
> On eviction: go to `freqToList.get(minFreq)`, take its **tail** — that is the least recently used
> among the least frequently used — and remove it.
>
> I'll use `LinkedHashSet` for the buckets, because it gives me O(1) add, O(1) remove-by-key and
> insertion order, which saves me writing a second linked list. In a real interview I'd say that and
> ask if they want the raw list version."

## Solution (Java 21)

```java
package kata.cache;

import java.util.HashMap;
import java.util.LinkedHashSet;
import java.util.Map;

/**
 * LFU cache with O(1) get and put.
 *
 * Structures:
 *   values   : key -> value
 *   freqs    : key -> current access count
 *   buckets  : freq -> keys at that freq, in INSERTION order (oldest first)
 *   minFreq  : the smallest freq with a non-empty bucket
 *
 * Key insight: minFreq is MAINTAINED, never searched for. It can only
 *   - reset to 1 on an insertion, or
 *   - increase by exactly 1 when the last key at minFreq is promoted.
 *
 * LinkedHashSet gives O(1) add / O(1) remove(key) / stable insertion order, which is
 * exactly the doubly-linked-list-plus-index from the LRU, already written for us.
 */
public class LfuCache<K, V> {

    private final int capacity;
    private final Map<K, V> values = new HashMap<>();
    private final Map<K, Integer> freqs = new HashMap<>();
    private final Map<Integer, LinkedHashSet<K>> buckets = new HashMap<>();
    private int minFreq = 0;

    public LfuCache(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException("capacity must be > 0");
        this.capacity = capacity;
    }

    public V get(K key) {
        V value = values.get(key);
        if (value == null) return null;
        touch(key);
        return value;
    }

    public void put(K key, V value) {
        if (values.containsKey(key)) {
            values.put(key, value);      // update in place; an update counts as a use
            touch(key);
            return;
        }

        if (values.size() >= capacity) {
            evict();
        }

        values.put(key, value);
        freqs.put(key, 1);
        buckets.computeIfAbsent(1, f -> new LinkedHashSet<>()).add(key);
        minFreq = 1;                     // a brand-new key ALWAYS resets the minimum
    }

    /** Promote a key from freq f to f+1, maintaining minFreq. */
    private void touch(K key) {
        int f = freqs.get(key);
        LinkedHashSet<K> bucket = buckets.get(f);
        bucket.remove(key);

        if (bucket.isEmpty()) {
            buckets.remove(f);
            if (minFreq == f) {
                minFreq = f + 1;         // the ONLY way minFreq increases; +1 is always enough
            }
        }

        freqs.put(key, f + 1);
        buckets.computeIfAbsent(f + 1, x -> new LinkedHashSet<>()).add(key);
    }

    /** Evict the LRU key among those with the minimum frequency. */
    private void evict() {
        LinkedHashSet<K> bucket = buckets.get(minFreq);
        K victim = bucket.iterator().next();   // insertion order → oldest at this frequency
        bucket.remove(victim);
        if (bucket.isEmpty()) buckets.remove(minFreq);
        values.remove(victim);
        freqs.remove(victim);
        // No need to fix minFreq here: put() sets it to 1 immediately afterwards.
    }
}
```

## Complexity

| | Time | Space |
|---|---|---|
| `get` | O(1) | — |
| `put` | O(1) | — |
| Total | | O(capacity) across the three maps |

## Common bugs

| Bug | Symptom |
|---|---|
| Searching for the minimum frequency | O(n) `put` — the whole point is lost |
| Forgetting `minFreq = 1` on insert | evicts the key you just added |
| Incrementing `minFreq` when a non-minimum bucket empties | evicts the wrong key later |
| `capacity == 0` not handled | NPE on the first `put` |
| An update via `put` not counting as a use | frequencies drift from reality |
| Using a plain `HashSet` for the bucket | tie-breaking by recency is lost — the LRU-within-LFU rule breaks |

## Follow-ups they'll ask

- *"What if frequencies could decrease, or age out?"* — that is **LFU with decay** / window-TinyLFU.
  Real caches do this because otherwise an item that was hot once is immortal. Caffeine uses a
  frequency sketch (Count–Min) with periodic halving. Naming that problem — *"classic LFU never
  forgets, so a burst from last Tuesday keeps an entry forever"* — is a strong signal.
- *"LRU or LFU for your cache?"* — depends on the access pattern. LRU handles recency-driven workloads
  and adapts fast to a change; LFU handles a stable hot set and resists a scan wiping the cache. For
  StreamPulse title metadata the hot set is "titles currently being watched", which *changes* when a
  live event starts, so LRU adapts better. I'd mention ARC and W-TinyLFU as the hybrids that exist
  precisely because neither is right alone.
- *"Can you do it with the linked-list version instead of `LinkedHashSet`?"* — yes, each bucket is a
  doubly linked list and each node knows its bucket. Same complexity, more code; say you'd write it
  that way if they want no `java.util` shortcuts.

---

# Problem 3 — Rotting Oranges (multi-source BFS) ★ CONFIRMED QUESTION

> This is named in the prep doc as a question that has actually been asked. Give it the most
> preparation. It is also the single best vehicle for demonstrating that you know **BFS is
> level-order** and **multi-source BFS is one BFS, not N of them.**

## Statement

You are given an `m × n` grid where each cell is:
- `0` — empty
- `1` — a fresh orange
- `2` — a rotten orange

Every minute, any fresh orange **4-directionally adjacent** to a rotten orange becomes rotten.
Return the **minimum number of minutes** until no cell has a fresh orange, or `-1` if that is
impossible.

## Narration script

> "Let me restate it: rot spreads to the four neighbours, one step per minute, from **all** rotten
> oranges simultaneously, and I want the time until everything is rotten — or `-1` if something is
> unreachable.
>
> The words 'minimum number of minutes' and 'spreads one step at a time' are the signal for **BFS**.
> BFS explores in levels, and here each level *is* one minute — that correspondence is the whole
> solution.
>
> The twist is that rot starts from **many** cells at once. The wrong instinct is to run a BFS from
> each rotten orange and take the minimum; that's O(sources × cells) and it doesn't even model the
> problem, because the sources spread *simultaneously*, not independently. The right move is
> **multi-source BFS**: seed the queue with *every* rotten cell at distance 0, and then run one
> ordinary BFS. Because they all start at level 0, the BFS naturally computes, for every cell, the
> distance to the *nearest* source. That's the trick, and it is one line of setup.
>
> So the algorithm:
> 1. One pass over the grid: push every `2` into the queue, and count the `1`s.
> 2. If `fresh == 0`, return 0 immediately — nothing to rot, and this is the edge case that catches
>    people, because otherwise you return the wrong answer or -1 on an all-empty grid.
> 3. BFS **level by level**: for each minute, drain exactly the current queue size, rot the fresh
>    neighbours, push them, decrement `fresh`.
> 4. Increment the minute counter only when a level actually produced something — otherwise you
>    return one more than the answer. I'll handle that by incrementing *before* processing and
>    returning `minutes - 1`, or by only incrementing when the queue is non-empty after a level.
>    Let me use the cleaner form: process a level, and if that level rotted anything, `minutes++`.
> 5. At the end: `fresh == 0 ? minutes : -1`. The `-1` covers fresh oranges walled off by zeros.
>
> Two implementation choices I'll state: I'll mutate the grid in place to mark visited — writing `2`
> over a `1` — which saves an O(mn) visited array; I'll mention that it destroys the input and ask if
> that's acceptable. And I'll encode coordinates as `r * cols + c` in an `int[]` queue, or just use an
> `int[]{r, c}` — I'll use the pair, it's more readable and the interviewer cares about clarity more
> than the allocation.
>
> Complexity: every cell is enqueued at most once, so **O(m × n)** time and **O(m × n)** space for the
> queue in the worst case, where the whole grid is rotten at the start."

## Solution (Java 21)

```java
package kata.graph;

import java.util.ArrayDeque;
import java.util.Deque;

public final class RottingOranges {

    private static final int EMPTY = 0, FRESH = 1, ROTTEN = 2;

    /** 4-directional neighbour offsets. Declared once, named, so the loop below reads cleanly. */
    private static final int[][] DIRECTIONS = {{-1, 0}, {1, 0}, {0, -1}, {0, 1}};

    /**
     * Multi-source BFS. The queue is seeded with EVERY initially-rotten cell, so one BFS
     * computes each fresh cell's distance to the NEAREST source — which is exactly the
     * minute it rots.
     *
     * Mutates `grid` (fresh cells are overwritten with ROTTEN as they are visited), which
     * serves as the visited-set. Say this out loud and confirm it is acceptable.
     *
     * Time  O(m*n) — each cell is enqueued at most once and scanned a constant number of times.
     * Space O(m*n) — worst case the initial queue holds the whole grid.
     */
    public static int orangesRotting(int[][] grid) {
        if (grid == null || grid.length == 0 || grid[0].length == 0) return 0;

        int rows = grid.length, cols = grid[0].length;
        Deque<int[]> queue = new ArrayDeque<>();
        int fresh = 0;

        // ---- 1. Seed: every rotten cell is a source at time 0. Count the fresh ones. ----
        for (int r = 0; r < rows; r++) {
            for (int c = 0; c < cols; c++) {
                if (grid[r][c] == ROTTEN)      queue.add(new int[]{r, c});
                else if (grid[r][c] == FRESH)  fresh++;
            }
        }

        // ---- 2. The edge case that decides the question. Nothing fresh => 0 minutes, NOT -1. ----
        if (fresh == 0) return 0;

        int minutes = 0;

        // ---- 3. Level-order BFS. One level == one minute. ----
        while (!queue.isEmpty() && fresh > 0) {
            int levelSize = queue.size();       // SNAPSHOT: cells added below belong to the NEXT minute

            for (int i = 0; i < levelSize; i++) {
                int[] cell = queue.poll();

                for (int[] d : DIRECTIONS) {
                    int nr = cell[0] + d[0];
                    int nc = cell[1] + d[1];

                    // bounds first, then state — short-circuit order matters, or you index out of range
                    if (nr < 0 || nr >= rows || nc < 0 || nc >= cols) continue;
                    if (grid[nr][nc] != FRESH) continue;          // empty, or already rotten (= visited)

                    grid[nr][nc] = ROTTEN;                        // mark visited AT ENQUEUE time
                    fresh--;
                    queue.add(new int[]{nr, nc});
                }
            }
            minutes++;    // this level rotted at least one orange (guarded by `fresh > 0` above)
        }

        // ---- 4. Any fresh orange left is unreachable: walled off by empty cells. ----
        return fresh == 0 ? minutes : -1;
    }
}
```

## Why the `fresh > 0` guard in the `while` matters

Without it, consider a grid that is already fully rotten with no fresh oranges *plus* one extra BFS
level that finds nothing: you would run one more iteration and return `1` instead of `0`. The guard
makes the loop terminate the moment there is nothing left to do, so `minutes` is exactly the number of
levels that *produced* a change. Walk the interviewer through this with the two-cell example
`[[2,1]]` (answer 1) and `[[2,2]]` (answer 0). Dry-running two tiny cases is worth more than a
paragraph of explanation.

## Dry run — say this while pointing at the grid

```
grid = [[2,1,1],        fresh cells: (0,1) (0,2) (1,0) (1,1) (2,1) (2,2)  → fresh = 6
        [1,1,0],        rotten cells: (0,0)                               → queue = [(0,0)]
        [0,1,1]]        empty cells:  (1,2) (2,0)

minute 1  levelSize=1   pop (0,0) → rot (1,0), (0,1)        fresh=4  queue=[(1,0),(0,1)]   minutes=1
minute 2  levelSize=2   pop (1,0) → rot (1,1)               fresh=3
                        pop (0,1) → rot (0,2)               fresh=2  queue=[(1,1),(0,2)]   minutes=2
minute 3  levelSize=2   pop (1,1) → rot (2,1)               fresh=1
                        pop (0,2) → nothing fresh adjacent            queue=[(2,1)]        minutes=3
minute 4  levelSize=1   pop (2,1) → rot (2,2)               fresh=0  queue=[(2,2)]         minutes=4

loop exits because fresh == 0  →  return 4
```

Point at three things while you trace: the **snapshot** of `levelSize` at the top of each minute; that
`(1,1)` was reached by `(1,0)` first and the second neighbour simply found it non-fresh; and that the
loop stops on `fresh == 0` rather than on an empty queue — `(2,2)` is still in the queue and never
needs to be processed.

> **If you lose your place mid-trace, say so:** *"Let me recount the fresh oranges — I'd rather verify
> than guess."* Catching your own slip in front of the interviewer reads as rigour. Going silent while
> you panic reads as a mistake.

## Complexity

| | |
|---|---|
| Time | **O(m × n)** — one seeding pass, and each cell enters the queue at most once |
| Space | **O(m × n)** — the queue, worst case when the entire grid starts rotten |

## Common bugs

| Bug | Symptom |
|---|---|
| Not snapshotting `queue.size()` before the inner loop | you process this minute's *and* next minute's cells as one level; the answer is 1 |
| Marking visited on **dequeue** instead of enqueue | the same cell is enqueued several times; still correct here (the `!= FRESH` check saves you) but O(4mn) and wrong in weighted variants |
| Returning `-1` when there are no oranges at all | should be `0` |
| `minutes++` unguarded | off-by-one: returns one more than the answer |
| Checking `grid[nr][nc]` before the bounds check | `ArrayIndexOutOfBoundsException` |
| Using a `Stack`/DFS | DFS does not give shortest distance; it gives *a* distance |
| 8-directional neighbours | re-read the problem — it is 4-directional |

## Follow-ups they'll ask

- *"What if rot spread diagonally too?"* — add the four diagonal offsets to `DIRECTIONS`. Nothing else
  changes. That is the payoff of naming the direction array.
- *"What if different oranges rotted at different speeds?"* — it stops being BFS and becomes
  **Dijkstra**: BFS's correctness depends on every edge having weight 1. Swap the `ArrayDeque` for a
  `PriorityQueue` ordered by time. That sentence — *"BFS is Dijkstra when all edges are 1"* — is one of
  the highest-value things you can say in a graph round.
- *"Can you do it without mutating the input?"* — a `boolean[][] visited`, or a defensive copy.
  O(mn) extra space, and I would ask which they prefer rather than assume.
- *"Return which orange rots last."* — track the last cell dequeued at the final level.
- *"How would you parallelise it for a huge grid?"* — partition by rows with halo exchange between
  partitions each round; it becomes a bulk-synchronous computation. Usually the honest answer is "the
  grid would have to be enormous before that beats a single-threaded O(mn) pass."
- *"Where have you used BFS for real?"* — StreamPulse Phase 6: the alert engine walks a service
  dependency graph (CDN → packager → player) with BFS, so when an upstream node fires an alert, every
  downstream alert within the reachable set is suppressed as a symptom of the same root cause. Same
  algorithm, and the "levels" are dependency hops.

---

# Problem 4 — Number of Islands

## Statement

Given an `m × n` grid of `'1'` (land) and `'0'` (water), return the number of islands. An island is
land connected 4-directionally, surrounded by water.

## Narration script

> "This is **connected components** on an implicit grid graph. Each cell is a vertex; edges go to the
> four neighbours that are also land.
>
> The algorithm: scan every cell; when I hit an unvisited land cell, that is a **new island**, so I
> increment the counter and then flood-fill everything reachable from it so I never count it again.
> The flood fill can be DFS or BFS — they visit the same set, only the order differs.
>
> I'll use **iterative BFS**, and I'll say why: recursive DFS is shorter, but on a 1000×1000 grid that
> is all land, the recursion depth is a million frames and you get a `StackOverflowError`. That is a
> real failure, not a theoretical one, and mentioning it unprompted is usually worth more than the
> code.
>
> I'll sink the island as I go — write `'0'` over visited land — which uses the grid itself as the
> visited set. I'll flag that it mutates the input.
>
> Time is O(m × n): every cell is visited at most once across all the flood fills, because once sunk
> it is never land again. Space is O(min(m, n)) for BFS's frontier in the worst case — the frontier of
> a BFS on a grid is bounded by the shorter dimension — or O(m × n) for DFS's stack."

## Solution (Java 21)

```java
package kata.graph;

import java.util.ArrayDeque;
import java.util.Deque;

public final class NumberOfIslands {

    private static final int[][] DIRECTIONS = {{-1, 0}, {1, 0}, {0, -1}, {0, 1}};

    /**
     * Connected components via iterative BFS flood fill.
     * Iterative on purpose: a recursive DFS on an all-land 1000x1000 grid overflows the stack.
     *
     * Time  O(m*n) — each cell is sunk at most once, so the inner BFS is amortised.
     * Space O(min(m,n)) for the BFS frontier on a grid (vs O(m*n) for a DFS stack).
     */
    public static int numIslands(char[][] grid) {
        if (grid == null || grid.length == 0) return 0;

        int rows = grid.length, cols = grid[0].length, islands = 0;

        for (int r = 0; r < rows; r++) {
            for (int c = 0; c < cols; c++) {
                if (grid[r][c] != '1') continue;

                islands++;          // an unvisited land cell can only be a NEW component
                sink(grid, r, c, rows, cols);
            }
        }
        return islands;
    }

    /** BFS flood fill: mark every cell reachable from (sr, sc) as water. */
    private static void sink(char[][] grid, int sr, int sc, int rows, int cols) {
        Deque<int[]> queue = new ArrayDeque<>();
        grid[sr][sc] = '0';                       // mark AT ENQUEUE — never at dequeue
        queue.add(new int[]{sr, sc});

        while (!queue.isEmpty()) {
            int[] cell = queue.poll();
            for (int[] d : DIRECTIONS) {
                int nr = cell[0] + d[0], nc = cell[1] + d[1];
                if (nr < 0 || nr >= rows || nc < 0 || nc >= cols) continue;
                if (grid[nr][nc] != '1') continue;
                grid[nr][nc] = '0';               // marking here prevents duplicate enqueues
                queue.add(new int[]{nr, nc});
            }
        }
    }
}
```

## Complexity

| | |
|---|---|
| Time | **O(m × n)** |
| Space | **O(min(m, n))** BFS frontier; O(m × n) if you use recursive DFS |

## Common bugs

| Bug | Symptom |
|---|---|
| Marking visited on dequeue | a cell is enqueued by several neighbours; quadratic blow-up |
| Comparing `grid[r][c] == 1` instead of `'1'` | it's a `char[][]` — compiles, never matches |
| Recursive DFS on a large grid | `StackOverflowError` |
| Forgetting that mutation destroys the input | the caller's grid is gone; ask first |

## Follow-ups they'll ask

- *"Without mutating the grid?"* — `boolean[][] visited`, O(mn) space.
- *"The grid doesn't fit in memory."* — process it in row-strips and stitch components across strip
  boundaries with **union-find**. That is the natural segue to the next follow-up.
- *"Now islands can be added one at a time (Number of Islands II)."* — union-find with path compression
  and union by rank: each added land cell starts as its own component and unions with any land
  neighbour, decrementing the count per successful union. Near-O(1) amortised per operation.
- *"Count the largest island."* — same flood fill, return its size instead of incrementing a counter.
- *"How does this relate to your project?"* — the dependency-graph walk in Phase 6 is the same
  connected-components idea on an explicit graph rather than a grid.

---

# Problem 5 — Word Ladder

## Statement

Given `beginWord`, `endWord` and a `wordList`, return the number of words in the **shortest**
transformation sequence from `beginWord` to `endWord`, where each step changes exactly one letter and
every intermediate word must be in `wordList`. Return `0` if there is no such sequence.
(The length counts both endpoints: `hit → hot → dot → dog → cog` is 5.)

## Narration script

> "'Shortest transformation sequence' — that's BFS again, on an **implicit graph** where the vertices
> are words and there is an edge between two words that differ by exactly one letter.
>
> The first question is how I find neighbours, because that decides the complexity. The naive way is,
> for each word, compare against every other word: O(N²·L). The better way is **wildcard bucketing**:
> preprocess the word list into a map from patterns like `h*t` to the list of words matching it. Then
> the neighbours of `hot` are the union of the buckets `*ot`, `h*t` and `ho*`. Building that map is
> O(N·L) patterns, and looking up a word's neighbours is O(L) bucket lookups.
>
> With L = 26-ish letters and an alphabet of 26, the alternative is to generate all `L × 25` single-letter
> mutations and test membership in a `HashSet`. That is O(L × 26) per word with no preprocessing, and
> honestly for typical constraints it is simpler and just as fast. I'll mention both and pick the
> mutation version unless the words are long.
>
> Then it's standard BFS with a level counter, starting from `beginWord` at level 1, and I remove each
> word from the unvisited set the moment I enqueue it — which is both the visited-marking and, on a
> `HashSet`, an O(1) operation.
>
> Edge case first: if `endWord` isn't in the word list at all, return 0 immediately. That's free and
> it's the case people forget.
>
> Complexity: O(N × L × 26) time with the mutation approach, O(N × L) space."

## Solution (Java 21)

```java
package kata.graph;

import java.util.ArrayDeque;
import java.util.Deque;
import java.util.HashSet;
import java.util.List;
import java.util.Set;

public final class WordLadder {

    /**
     * BFS over an implicit graph: vertices are words, edges join words differing by one letter.
     * Neighbours are generated by mutation (L * 25 candidates) and filtered by set membership,
     * which avoids the O(N^2) pairwise comparison.
     *
     * Time  O(N * L * 26) — each word is dequeued once, each generates L*26 candidates.
     * Space O(N * L)      — the unvisited set plus the queue.
     */
    public static int ladderLength(String beginWord, String endWord, List<String> wordList) {
        Set<String> unvisited = new HashSet<>(wordList);

        // Edge case: unreachable by definition. Free to check, easy to forget.
        if (!unvisited.contains(endWord)) return 0;
        unvisited.remove(beginWord);      // beginWord does not need to be in the list

        Deque<String> queue = new ArrayDeque<>();
        queue.add(beginWord);
        int level = 1;                    // the sequence length INCLUDES beginWord

        while (!queue.isEmpty()) {
            int levelSize = queue.size();  // snapshot: one level == one transformation

            for (int i = 0; i < levelSize; i++) {
                char[] chars = queue.poll().toCharArray();

                for (int pos = 0; pos < chars.length; pos++) {
                    char original = chars[pos];

                    for (char ch = 'a'; ch <= 'z'; ch++) {
                        if (ch == original) continue;
                        chars[pos] = ch;
                        String candidate = new String(chars);

                        if (candidate.equals(endWord)) return level + 1;

                        // remove() returns true only if it WAS present → membership test
                        // and visited-marking in one atomic-looking O(1) step.
                        if (unvisited.remove(candidate)) {
                            queue.add(candidate);
                        }
                    }
                    chars[pos] = original;   // restore before moving to the next position
                }
            }
            level++;
        }
        return 0;
    }
}
```

## Complexity

| | |
|---|---|
| Time | **O(N × L × 26)**, where N = words, L = word length. Each candidate string costs O(L) to build, so strictly O(N × L² × 26). |
| Space | **O(N × L)** for the set and the queue |

Be precise about that `L²`: building `new String(chars)` is O(L) and you do it `L × 26` times per word.
Saying so unprompted is a good signal.

## Common bugs

| Bug | Symptom |
|---|---|
| Not restoring `chars[pos] = original` | you mutate cumulatively and generate garbage |
| Not checking `endWord ∈ wordList` up front | correct but slower, and you may return the wrong thing on some variants |
| Starting `level` at 0 | off-by-one; the answer counts both endpoints |
| Marking visited on dequeue | the same word is enqueued many times; exponential in the worst case |
| Forgetting `unvisited.remove(beginWord)` | possible cycle back to the start |

## Follow-ups they'll ask

- *"Make it faster."* — **bidirectional BFS**: expand from both ends, always expanding the *smaller*
  frontier, and stop when they meet. Roughly halves the exponent of the search tree —
  O(b^(d/2)) instead of O(b^d) — and it is the answer they are looking for.
- *"Return the actual path, not just the length (Word Ladder II)."* — BFS to build a parent map of
  *all* shortest predecessors, then DFS backwards from `endWord` to enumerate paths. The important
  detail is that you cannot remove a word from `unvisited` as soon as you see it — you must delete a
  whole *level* at a time, or you lose alternative shortest paths.
- *"Why BFS and not DFS?"* — DFS finds *a* path, not the shortest, and it can wander into a long chain
  before finding the two-step answer.
- *"When would the wildcard-bucket preprocessing win?"* — when words are long (L ≫ 26) or when you run
  many queries against the same word list, so you amortise the preprocessing.

---

# Problem 6 — Course Schedule (topological sort / cycle detection)

> **StreamPulse tie-in:** Phase 6's control-plane rejects **cyclic alert-rule dependencies at save
> time** with exactly this algorithm, and the same graph is walked with BFS for root-cause
> suppression. So when they ask "have you used this?", you have a concrete answer: *a rule that
> depends on a rule that depends back on the first would either loop forever or never fire, so the
> API returns a 400 naming the cycle.*

## Statement

There are `numCourses` courses labelled `0..numCourses-1`, and `prerequisites[i] = [a, b]` means you
must take `b` before `a`. Return whether you can finish all courses (Course Schedule I), and/or a
valid ordering (Course Schedule II — return an empty array if impossible).

## Narration script

> "Restated as a graph problem: this is 'does a directed graph have a cycle', and the ordering version
> is 'give me a topological sort'. A topological order exists **if and only if** the graph is a DAG,
> so both questions are the same algorithm.
>
> Two standard approaches, and I'll say both before picking:
>
> **Kahn's algorithm (BFS)**: compute the in-degree of every node; seed a queue with every node of
> in-degree 0; repeatedly pop a node, append it to the order, and decrement the in-degree of each of
> its successors, enqueueing any that hit 0. At the end, if the order contains fewer than `numCourses`
> nodes, the remainder form a cycle — because every node in a cycle keeps a positive in-degree
> forever. I like this one: it is iterative, it produces the ordering for free, and the cycle
> detection falls out of a counter rather than needing separate bookkeeping.
>
> **DFS with three colours**: white = unvisited, grey = on the current recursion stack, black = fully
> explored. Hitting a **grey** node means a back edge, which means a cycle. The topological order is
> the reverse of the post-order finishing times. It's elegant but it recurses, so deep graphs can
> overflow, and the ordering has to be reversed at the end.
>
> The one thing people get wrong is the edge direction. `[a, b]` means 'b before a', so the edge goes
> **b → a** and `a`'s in-degree increases. I'll write that comment in the code so I don't flip it
> under pressure.
>
> I'll build the adjacency list with `List<List<Integer>>` sized `numCourses` — an array of lists, not
> a `HashMap`, because the nodes are dense integers.
>
> Complexity: O(V + E) time and space for both."

## Solution (Java 21) — Kahn's algorithm, returning the order

```java
package kata.graph;

import java.util.ArrayDeque;
import java.util.ArrayList;
import java.util.Deque;
import java.util.List;

public final class CourseSchedule {

    /**
     * Kahn's algorithm: BFS topological sort with cycle detection.
     *
     * prerequisites[i] = {a, b} means "b before a", so the EDGE is b -> a
     * and indegree[a]++. Getting this backwards is the #1 bug in this problem.
     *
     * Returns a valid ordering, or an EMPTY array if the graph has a cycle.
     *
     * Time  O(V + E)  Space O(V + E)
     */
    public static int[] findOrder(int numCourses, int[][] prerequisites) {

        List<List<Integer>> adjacency = new ArrayList<>(numCourses);
        for (int i = 0; i < numCourses; i++) adjacency.add(new ArrayList<>());
        int[] indegree = new int[numCourses];

        for (int[] pair : prerequisites) {
            int course = pair[0], prerequisite = pair[1];
            adjacency.get(prerequisite).add(course);   // prerequisite -> course
            indegree[course]++;
        }

        // Seed with everything that has no unmet prerequisite.
        Deque<Integer> queue = new ArrayDeque<>();
        for (int i = 0; i < numCourses; i++) {
            if (indegree[i] == 0) queue.add(i);
        }

        int[] order = new int[numCourses];
        int index = 0;

        while (!queue.isEmpty()) {
            int node = queue.poll();
            order[index++] = node;

            for (int next : adjacency.get(node)) {
                // "Remove" the edge node->next. When next has no remaining prerequisites, it's ready.
                if (--indegree[next] == 0) queue.add(next);
            }
        }

        // Anything still unplaced is in (or downstream of) a cycle: its indegree never reached 0.
        return index == numCourses ? order : new int[0];
    }

    /** Course Schedule I is the same run, reduced to a boolean. */
    public static boolean canFinish(int numCourses, int[][] prerequisites) {
        return findOrder(numCourses, prerequisites).length == numCourses;
    }
}
```

## The DFS three-colour variant — and how you report *which* rules form the cycle

```java
/**
 * StreamPulse Phase 6 flavour: it is not enough to say "cyclic". The API must return
 * a ProblemDetail naming the rules in the cycle, or the admin cannot fix it (file 02 Q3).
 * The grey/recursion stack gives you the cycle members for free.
 */
public final class RuleCycleDetector {

    private enum Colour { WHITE, GREY, BLACK }   // unvisited / on stack / done

    /** @return the rule ids forming a cycle, or an empty list if the rule graph is a DAG. */
    public static List<String> findCycle(Map<String, List<String>> dependsOn) {
        Map<String, Colour> colour = new HashMap<>();
        dependsOn.keySet().forEach(k -> colour.put(k, Colour.WHITE));

        Deque<String> stack = new ArrayDeque<>();     // the current DFS path, for reporting
        for (String node : dependsOn.keySet()) {
            if (colour.get(node) == Colour.WHITE) {
                List<String> cycle = dfs(node, dependsOn, colour, stack);
                if (!cycle.isEmpty()) return cycle;
            }
        }
        return List.of();
    }

    private static List<String> dfs(String node, Map<String, List<String>> dependsOn,
                                    Map<String, Colour> colour, Deque<String> stack) {
        colour.put(node, Colour.GREY);                // entering: on the current path
        stack.push(node);

        for (String next : dependsOn.getOrDefault(node, List.of())) {
            Colour c = colour.getOrDefault(next, Colour.WHITE);
            if (c == Colour.GREY) {
                // BACK EDGE → cycle. Everything on the stack from `next` upward is the cycle.
                List<String> cycle = new ArrayList<>();
                for (String s : stack) {
                    cycle.add(s);
                    if (s.equals(next)) break;
                }
                Collections.reverse(cycle);
                return cycle;
            }
            if (c == Colour.WHITE) {
                List<String> found = dfs(next, dependsOn, colour, stack);
                if (!found.isEmpty()) return found;
            }
            // BLACK → already fully explored, no cycle through it. Skip.
        }

        stack.pop();
        colour.put(node, Colour.BLACK);               // leaving: fully explored
        return List.of();
    }
}
```

The reason the three colours are necessary, and the thing to say: **a two-state visited set is not
enough.** With only "visited/unvisited" you cannot distinguish a *back edge* (a cycle) from a
*cross/forward edge* (a diamond, which is perfectly legal in a DAG). Grey means "on the current path";
black means "finished, and provably not part of a cycle reachable from here."

## Complexity

| | |
|---|---|
| Time | **O(V + E)** for both variants |
| Space | **O(V + E)** — adjacency list, plus O(V) for the queue / recursion stack |

## Common bugs

| Bug | Symptom |
|---|---|
| Edge direction reversed | the answer is wrong on any asymmetric input, and right on symmetric test cases — brutal |
| Using a two-state visited set for cycle detection | a diamond DAG is reported as a cycle |
| Forgetting to reset grey → black on exit | false cycles across sibling branches |
| Not handling `numCourses` with **no** prerequisites | should return `0..n-1` in any order, not empty |
| Duplicate edges in the input | in-degree counted twice; either de-duplicate or accept it (Kahn's still terminates correctly since you decrement once per stored edge) |
| Recursive DFS on a deep chain | `StackOverflowError` — prefer Kahn's in production |

## Follow-ups they'll ask

- *"Is the topological order unique?"* — only if at every step exactly one node has in-degree 0. If the
  queue ever holds two, there are multiple valid orders. If you need a *deterministic* one (and you do,
  for reproducible rule evaluation), use a `PriorityQueue` instead of an `ArrayDeque` to break ties by
  id — that is "lexicographically smallest topological order", and it costs O(V log V).
- *"Detect the cycle *and* report it."* — the three-colour DFS above; the grey stack is the cycle.
- *"Where is this in production?"* — build systems (Gradle's task graph), package managers, schema
  migrations, Kafka Streams topology ordering, and StreamPulse's alert-rule dependencies. Naming
  Gradle is nice because you are building the project with it.
- *"What if the graph changes constantly?"* — incremental topological ordering, or just re-run
  O(V + E) on each rule save, which for hundreds of rules is microseconds. Choosing the simple option
  and justifying it by scale is a senior answer.

---

# Problem 7 — Clone Graph

## Statement

Given a reference to a node in a **connected undirected** graph, return a **deep copy**. Each node
has a `val` and a `List<Node> neighbors`.

## Narration script

> "The whole problem is one idea: I need a map from **original node → its clone**, and I have to
> create the clone and put it in the map **before** I recurse into its neighbours. If I recurse first,
> a cycle sends me back to a node I haven't registered yet and I recurse forever.
>
> So: `if (map.containsKey(node)) return map.get(node);` is both my memoisation and my cycle
> protection — they are the same line. Then create the clone with the value only, put it in the map,
> and *then* loop over the neighbours, cloning each and adding it to the new node's list.
>
> DFS is the natural fit because the recursion mirrors the structure. I'll write DFS, and mention that
> BFS with an explicit queue is the same algorithm if stack depth is a concern.
>
> Edge case: a null input returns null. And a single node with no neighbours should return a new node
> with an empty list — not the same reference. I'll assert that in my head as I write.
>
> Time O(V + E) — each node cloned once, each edge traversed once from each side. Space O(V) for the
> map plus recursion."

## Solution (Java 21)

```java
package kata.graph;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public final class CloneGraph {

    public static class Node {
        public int val;
        public List<Node> neighbors;
        public Node(int val) { this.val = val; this.neighbors = new ArrayList<>(); }
    }

    /**
     * DFS with an original->clone map that doubles as the visited set.
     *
     * THE invariant: put the clone in the map BEFORE recursing into neighbours.
     * A cycle (a-b-a) then terminates, because the second visit to `a` finds it in the map.
     *
     * Time  O(V + E)   Space O(V) map + O(V) recursion depth
     */
    public static Node cloneGraph(Node node) {
        return clone(node, new HashMap<>());
    }

    private static Node clone(Node node, Map<Node, Node> seen) {
        if (node == null) return null;

        Node existing = seen.get(node);
        if (existing != null) return existing;      // memoisation AND cycle protection, one line

        Node copy = new Node(node.val);
        seen.put(node, copy);                        // register BEFORE recursing — the whole trick

        for (Node neighbor : node.neighbors) {
            copy.neighbors.add(clone(neighbor, seen));
        }
        return copy;
    }
}
```

**Iterative BFS variant**, worth knowing for the deep-graph follow-up:

```java
public static Node cloneGraphIterative(Node node) {
    if (node == null) return null;

    Map<Node, Node> seen = new HashMap<>();
    seen.put(node, new Node(node.val));
    Deque<Node> queue = new ArrayDeque<>();
    queue.add(node);

    while (!queue.isEmpty()) {
        Node current = queue.poll();
        for (Node neighbor : current.neighbors) {
            // computeIfAbsent creates the clone exactly once; the boolean tells us whether to enqueue.
            if (!seen.containsKey(neighbor)) {
                seen.put(neighbor, new Node(neighbor.val));
                queue.add(neighbor);                 // enqueue only the FIRST time we see it
            }
            seen.get(current).neighbors.add(seen.get(neighbor));
        }
    }
    return seen.get(node);
}
```

## Complexity

| | |
|---|---|
| Time | **O(V + E)** |
| Space | **O(V)** for the map, plus O(V) recursion depth (DFS) or queue (BFS) |

## Common bugs

| Bug | Symptom |
|---|---|
| Recursing before putting the clone in the map | infinite recursion on any cycle |
| Keying the map by `val` instead of by node | breaks when values are duplicated |
| Reusing the original's `neighbors` list in the clone | the "copy" shares mutable state with the original — a shallow copy wearing a deep copy's clothes |
| Not handling `null` | NPE on the trivial input |
| Assuming the graph is connected when it is not | the problem states connected; if not, you need an outer loop over all nodes |

## Follow-ups they'll ask

- *"Does `Node` need `equals`/`hashCode`?"* — no, and you should *not* add them: identity semantics are
  exactly what you want, because two distinct nodes can share a `val`. Overriding `equals` by `val`
  would silently merge them.
- *"Deep copy of an arbitrary object graph?"* — same pattern with an `IdentityHashMap`, which is the
  right structure precisely because it keys on reference identity regardless of any `equals` override.
  That is the generalisation, and it is how serialisation libraries handle cycles.
- *"Clone a linked list with random pointers?"* — identical idea, and the O(1)-space trick is to
  interleave the copies into the original list, fix the random pointers, then unweave.
- *"Where does this matter in a real system?"* — defensive copying at a service boundary so a caller
  cannot mutate your internal state, and snapshotting a mutable config graph before evaluating rules
  against it.

---

# Narration checklist

Print this. It is the actual deliverable of this file.

| # | Say it | Why it scores |
|---|---|---|
| 1 | **Restate the problem in one sentence.** | Proves you listened; catches a misread before it costs 20 minutes. |
| 2 | **Ask 2–3 clarifying questions.** Input size? Empty/null? Duplicates? Can I mutate the input? | Seniors bound the problem before solving it. |
| 3 | **State one example and the expected output**, including an edge case. | Gives you a dry-run harness for later. |
| 4 | **Name the pattern.** "This is multi-source BFS." "This is connected components." | This is the single highest-signal sentence in the round. |
| 5 | **State the invariant** before writing code. "Head is MRU, tail is LRU." "One BFS level is one minute." | Separates the people who understand from the people who remember. |
| 6 | **State the complexity you're aiming for**, and why it's optimal (or that it isn't). | Frames everything that follows. |
| 7 | **Write the signature and the data structures first**, then fill in bodies. | The interviewer can course-correct you cheaply. |
| 8 | **Narrate trade-offs as you make them.** "I'll mutate the grid as my visited set — is that OK?" | Turns a silent decision into a visible one. |
| 9 | **Dry-run a small example out loud** when you finish. | Finds off-by-one errors *before* they do. |
| 10 | **Name the edge cases you handled** and the ones you deliberately didn't. | Honesty about scope reads as maturity. |
| 11 | **Say what you'd test.** Three cases: happy path, empty, and the tricky one. | The JD is for a backend engineer, not a competitive programmer. |
| 12 | **Offer the follow-up before they ask.** "The thread-safe version is interesting because…" | Converts a coding round into a design conversation. |

**When you get stuck — the recovery script:**

> "Let me think out loud for a second. I know I need *<the invariant>*. The thing I haven't worked out
> is *<the specific gap>*. Let me try a small example on paper… "

Never go silent. Thirty seconds of silence feels like five minutes to an interviewer and gives them
nothing to help you with. Stating the gap precisely often makes them give you the hint for free —
and asking-and-receiving a hint scores far better than being stuck in silence.

**Two phrases to avoid:**
- *"This is easy"* — it never ends well.
- *"I've seen this one before"* said as if it excuses narration. Say instead: *"I've solved this
  before, so let me make sure I explain why it works rather than just typing it."*

---

# Pattern → signal grid

The recognition table. Given the phrase in the left column, reach for the pattern in the right one.

| Signal in the problem statement | Pattern | Canonical problems |
|---|---|---|
| "shortest path", "minimum steps", "fewest moves", **unweighted** | **BFS** (level-order) | Rotting Oranges, Word Ladder |
| Several starting points spreading **simultaneously** | **Multi-source BFS** — seed the queue with all of them | Rotting Oranges, "walls and gates", nearest-0 matrix |
| "shortest path" with **different edge costs** | **Dijkstra** (BFS with a `PriorityQueue`) | network delay, cheapest flights |
| "how many groups / regions / islands" | **Connected components** (DFS or BFS flood fill) | Number of Islands, friend circles |
| Components that **merge incrementally** | **Union-Find** (path compression + union by rank) | Islands II, redundant connection, accounts merge |
| "ordering", "prerequisites", "build order", "dependencies" | **Topological sort** (Kahn's BFS) | Course Schedule I/II, alien dictionary |
| "is there a cycle" in a **directed** graph | **DFS three-colour**, or Kahn's count | Course Schedule, StreamPulse rule cycles |
| "is there a cycle" in an **undirected** graph | **Union-Find**, or DFS tracking the parent | redundant connection |
| "copy / clone" a structure with cycles | **DFS + `original → copy` map, registered before recursing** | Clone Graph, copy list with random pointer |
| "explore all paths / combinations / permutations" | **Backtracking** (DFS + undo) | N-queens, subsets, word search |
| "most recently used", "evict", "capacity" | **HashMap + doubly linked list** | LRU |
| "least frequently used", "capacity", O(1) | **HashMap + freq buckets + `minFreq`** | LFU |
| "top k", "k largest", "median of a stream" | **Heap** (`PriorityQueue`), sometimes two heaps | top k frequent, find median |
| "contiguous subarray/substring with property X" | **Sliding window** (two pointers) | longest substring without repeats |
| "sorted array", "find the boundary", "minimise the maximum" | **Binary search** (sometimes on the *answer*) | koko eating bananas, ship packages |
| "overlapping subproblems", "count the ways", "min cost to reach" | **DP** — define the state first, then the transition | coin change, edit distance |
| "next greater / smaller element", "span" | **Monotonic stack** | daily temperatures, largest rectangle |
| "prefix", "autocomplete", "dictionary of words" | **Trie** | word search II, autocomplete |
| "detect a cycle in a linked list", "find the duplicate" | **Floyd's tortoise and hare** | linked list cycle, find the duplicate number |

**The two sentences that make this grid useful in the room:**

> "The phrase *'minimum number of minutes'* plus *'spreads one step at a time'* tells me BFS, because
> BFS visits in levels and each level here is one minute."

> "I'd normally reach for BFS, but the edges have different weights, so BFS's correctness argument
> breaks and I need Dijkstra."

Pattern recognition *narrated* is what a senior coding round is actually testing. The code is the
easy part.

---

## Where this lands in StreamPulse

| Kata | Promoted into | Requirement |
|---|---|---|
| LRU (+ thread-safe variant) | `query-api` title-metadata cache, invalidated by CDC | Phase 5, **REQ-CC-03** |
| LRU concurrency | `LruConcurrencyStressTest` | **REQ-CC-08** |
| LFU | ADR comparison against LRU and Caffeine for the metadata cache | Phase 5 |
| Multi-source BFS | alert root-cause suppression over the service-dependency graph (CDN → packager → player) | Phase 6 |
| Topological sort / cycle detection | rejecting cyclic alert-rule dependencies at save time, returning a ProblemDetail naming the cycle | Phase 6 |
| Number of Islands / Clone Graph | `kata/` only — kept for the coding round, with parameterized JUnit tests and a complexity README | Phase 0 → |

> **Your story:** if you have implemented a cache, a dependency resolver, or a graph traversal in
> production — a module bundler's dependency graph, a component render-order problem, a
> memoisation layer — say so *here*, in place of or alongside StreamPulse. A real frontend example is
> worth more than a portfolio one, and "I wrote the dependency-ordering logic for X" is exactly the
> same algorithm as Course Schedule.

---

## Self-check (write from scratch, no notes, 20 minutes each)

1. LRU cache with sentinels. Then explain, without looking, why the thread-safe version takes the
   **write** lock on `get`, and name the two alternatives.
2. Rotting Oranges. Then explain why you snapshot `queue.size()`, and what changes if the oranges rot
   at different speeds.
3. Course Schedule II with Kahn's algorithm. Then say which direction the edge goes for
   `prerequisites[i] = [a, b]`, and how you would return a *deterministic* ordering.
4. Clone Graph. Then explain the one line that is both the memoisation and the cycle protection, and
   why you must not override `equals` on the node.
5. Recite the narration checklist's first six steps from memory. If you cannot, the algorithms will
   not save you.
