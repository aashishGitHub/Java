# 06 — Data Layer: JPA/Hibernate, Transactions, HikariCP, Flyway, SQL vs NoSQL

**Why this matters for an Event Hub role:** the streaming path gets all the glamour, but the
control plane is where the money and the correctness live — tenants, rules, quotas, billing, audit.
That is boring CRUD right up until it isn't: an N+1 that turns a 100ms page into 4 seconds, a
`REQUIRES_NEW` that silently self-deadlocks a 10-connection pool, an audit row that disappears with
the rollback it was supposed to survive, a migration that takes an `ACCESS EXCLUSIVE` lock on the
hottest table during a live event. Interviewers probe this layer because it is where "I've used
Spring Data" and "I understand the database" separate.

**Scenario anchor — BS-05 (REQ-DL-01…08):** tenants are billed monthly on ingested events and
retained GB. Finance needs a heavy aggregate report in < 2s. The ops console lists
tenants → apps → titles and is slow (N+1). Every attempted rule change must be audited *even when
the change itself is rejected*. Two admins must not over-allocate the same quota pool. And the same
transaction that changes a rule must write the outbox row that Debezium picks up (BS-08 / Phase 6).

---

## Q1. Walk me through the JPA entity states and what the persistence context actually is.

**30-second answer:** The persistence context is a unit-of-work cache — a `Map` from entity identity
(class + primary key) to the managed instance, plus a snapshot of each entity's loaded state. An
entity is in one of four states: **transient** (just `new`-ed, no identity, unknown to the context),
**managed/persistent** (attached to an open context, changes tracked), **detached** (has an identity,
but the context that loaded it is closed), and **removed** (scheduled for a `DELETE` at flush). The
consequence people miss: for a *managed* entity you never call `save()`. You mutate the object and
Hibernate issues the `UPDATE` at flush time via dirty checking. `save()` on a managed entity is a
no-op that hides what is really happening.

**Deeper:**

```
            new Entity()
                 │
            TRANSIENT ──persist()──▶ MANAGED ──remove()──▶ REMOVED ──flush()──▶ DELETE
                 ▲                    │   ▲                                      
                 │                    │   │                                      
                 │        tx commit / │   │ merge()                              
                 │        em.close()  ▼   │                                      
                 └────────────────  DETACHED                                     
                                       (mutations here are ignored)
```

**Dirty checking, mechanically:** when an entity is loaded, Hibernate stores a *loaded state*
snapshot — an `Object[]` of the property values as they came off the ResultSet — in the
`EntityEntry` beside the managed instance. At flush, `DefaultFlushEntityEventListener` compares the
current property values against that snapshot field by field and builds an `UPDATE` for the
differing columns (or for all columns, depending on `@DynamicUpdate`). This is why:

- Holding 50,000 entities in one persistence context is slow *and* memory-hungry: you pay for two
  copies of everything and an O(n) dirty check on every flush.
- A batch job should `flush()` + `clear()` periodically, or bypass JPA entirely (Q6).

**First-level cache (L1):** the persistence context *is* the L1 cache. It is per-`EntityManager`, so
in Spring, per-transaction. Two `findById(7)` calls in the same transaction give you the **same Java
object** and one SQL statement. That guarantee is `==`, not `equals` — this is repeatable-read
semantics *at the application level*, independent of the database isolation level.

| Cache | Scope | Enabled by default | StreamPulse use |
|---|---|---|---|
| L1 / persistence context | one transaction | yes, always on, cannot be disabled | free identity map |
| L2 (`@Cacheable`, e.g. Ehcache/Infinispan) | SessionFactory / app-wide | **no** | not used — we invalidate via `config.changes.v1` CDC instead, so caching is explicit and observable |
| Query cache | app-wide, keyed by JPQL + params | no | not used; it is easy to get wrong and easy to make slower |

**Code:**

```java
// StreamPulse control-plane. All four states in one method.
@Transactional
public void renameTenant(UUID id, String newName) {

    Tenant fresh = new Tenant("kidsplus");   // TRANSIENT: no id, not in the context

    Tenant t = tenantRepo.findById(id)       // MANAGED: in L1, loaded-state snapshot taken
                         .orElseThrow();

    t.setName(newName);                      // no save() call — dirty checking will emit the UPDATE

    Tenant again = tenantRepo.findById(id).orElseThrow();
    assert t == again;                       // SAME instance: L1 identity map, zero extra SQL

    // On commit: flush → dirty check → UPDATE tenant SET name=? WHERE id=? AND version=?
}   // context closes here → `t` becomes DETACHED; mutating it now changes nothing
```

**Follow-ups they'll ask:**
- *"`save()` vs `merge()` vs `persist()`?"* — `persist()` attaches a transient instance and returns
  void (the same object becomes managed). `merge()` copies the state of a *detached* instance onto a
  managed one and **returns the managed copy** — the object you passed in stays detached, which is
  the single most common merge bug. Spring Data's `save()` picks between them based on
  `isNew()` (null/zero id, or `Persistable.isNew()`).
- *"Why is `getReference()`/`getOne` different from `findById`?"* — it returns a lazy proxy without
  hitting the database, which is what you want when you only need to set a foreign key. Touching any
  non-id property triggers a load, or `EntityNotFoundException` if the row is gone.
- *"Is the persistence context thread-safe?"* — no. An `EntityManager` is single-threaded by
  contract. Spring injects a thread-bound proxy, which is why handing a managed entity to another
  thread (a `CompletableFuture` in BS-02) is a bug waiting for a lazy load.

**Pitfalls / red flags:** calling `repo.save(managedEntity)` "to be safe", or not being able to say
what makes an `UPDATE` happen without a `save()`.

**Where in StreamPulse:** BS-05 / REQ-DL-01 (admin CRUD), REQ-DL-06 (outbox written in the same
unit of work).

---

## Q2. When does Hibernate actually send SQL to the database? Explain flush modes.

**30-second answer:** Almost never when you call the repository method. Writes are queued in the
action queue and sent at **flush**, which happens (a) at transaction commit, (b) before a query that
Hibernate thinks overlaps the pending changes, or (c) when you call `flush()` yourself. That is
`FlushModeType.AUTO`, the default. `COMMIT` mode flushes only at commit, so a query can read stale
data that your own transaction has already changed in memory. The practical takeaways are: the SQL
you see in the log is not in the order you wrote it, insert ordering is Hibernate's choice, and a
`ConstraintViolationException` surfaces at commit, far from the line that caused it.

**Deeper:**

Flush does three things, in this order (`ActionQueue` execution order, which is fixed and *not* your
call order):

```
1. OrphanRemovalAction
2. EntityInsertAction        ← all inserts, in the order entities were persisted
3. EntityUpdateAction        ← dirty-check results
4. CollectionRemoveAction
5. CollectionUpdateAction
6. CollectionRecreateAction
7. EntityDeleteAction        ← all deletes LAST
```

That last line is the classic trap: `delete(x); save(y);` where `y` reuses `x`'s unique key will
fail on the unique constraint, because the insert runs before the delete. The fix is an explicit
`flush()` between them.

| Mode | Flush at commit | Flush before queries | Use when |
|---|---|---|---|
| `AUTO` (JPA default) | yes | yes, if the query's tables overlap pending changes | default; correct-by-default |
| `COMMIT` | yes | no | read-heavy blocks where you *know* nothing pending is relevant; a micro-optimisation with a correctness cliff |
| `ALWAYS` (Hibernate-only) | yes | before every query, no overlap analysis | debugging |
| `MANUAL` (Hibernate-only) | no | no | long conversational flows; you own every flush |

**One detail worth saying out loud:** the overlap analysis for `AUTO` works for **JPQL/HQL and
Criteria**, because Hibernate can see which entities the query touches. It does **not** work for a
**native SQL query** unless you declare the affected tables (via the query's synchronized-space
setting). So a native query can silently read pre-flush data.

**Code:**

```java
@Transactional
void reallocateQuota(UUID from, UUID to, long amount) {
    quotaRepo.deleteByTenantId(from);          // queued as a DELETE action
    quotaRepo.save(new Quota(to, amount));     // queued as an INSERT action

    // ❌ without this, the INSERT is executed BEFORE the DELETE (action queue ordering)
    //    and violates uq_quota_tenant.
    entityManager.flush();
}
```

```yaml
# StreamPulse local profile: see the SQL, with parameters, and count it.
spring:
  jpa:
    properties:
      hibernate:
        generate_statistics: true      # enables the Statistics MBean used by the N+1 test (Q4)
logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.orm.jdbc.bind: TRACE   # Hibernate 6 parameter logging category
```

**Follow-ups they'll ask:**
- *"Does `flush()` commit?"* — no. Flush writes the SQL; commit ends the database transaction. After
  a flush without commit, another connection still cannot see the rows (at READ COMMITTED).
- *"Why did my `@Transactional(readOnly = true)` speed things up?"* — it sets the Hibernate flush
  mode to `MANUAL` (so no dirty checking / no automatic flush) and sets the JDBC connection
  read-only, which some databases and poolers use to route to a replica. On Postgres it also
  disallows writes at the server side.
- *"Where does batching come in?"* — `hibernate.jdbc.batch_size`, plus
  `order_inserts`/`order_updates` so the batches actually group. Batching is disabled by default and
  is silently defeated by `IDENTITY` id generation, because Hibernate must execute each insert to
  learn the id. Use a sequence with a pooled optimiser instead.

**Pitfalls / red flags:** believing `save()` issues SQL immediately, or not being able to explain why
a constraint violation appeared "on the wrong line".

**Where in StreamPulse:** BS-05 / REQ-DL-06 (outbox row and entity flushed in the same unit of work
so Debezium can never see one without the other).

**Verify:** the Hibernate 6 parameter-logging category (`org.hibernate.orm.jdbc.bind`) differs from
Hibernate 5's (`org.hibernate.type.descriptor.sql`). Confirm against the version you pin.

---

## Q3. Lazy vs eager loading, and how do you deal with `LazyInitializationException`?

**30-second answer:** `LazyInitializationException` means you touched an uninitialised proxy after
the persistence context that owned it closed — typically in a controller or a Jackson serializer,
after the `@Transactional` service method returned. The fix is **not** to make the association eager
and it is **not** open-session-in-view. The fix is to decide *at query time* what you need — a fetch
join or an `@EntityGraph` — and to map to a DTO inside the transaction so nothing lazy escapes the
service layer. Changing `fetch = LAZY` to `EAGER` "fixes" one screen and pessimises every other query
that touches that entity, forever.

**Deeper:**

Defaults, which interviewers do ask for verbatim:

| Association | JPA default fetch |
|---|---|
| `@ManyToOne` | **EAGER** |
| `@OneToOne` | **EAGER** |
| `@OneToMany` | LAZY |
| `@ManyToMany` | LAZY |
| `@Basic` / `@Lob` | EAGER |

So the sensible baseline is: **mark every `@ManyToOne` and `@OneToOne` as `LAZY` explicitly**, then
fetch what you need per query. An eager `@ManyToOne` is a hidden join on *every* query that loads the
owning entity, including the ones that never look at the parent.

A `@OneToOne` on the **non-owning** side cannot be proxied lazily in the general case — Hibernate must
run a query just to know whether to hand you a proxy or `null`. That is a genuine "why is there an
extra SELECT" answer, and the workarounds (shared primary key, bytecode enhancement with
`@LazyToOne`) are worth naming.

**Why open-session-in-view is a trap.** Spring Boot keeps `spring.jpa.open-in-view=true` by default
and logs a warning about it. It holds the `EntityManager` — and therefore a **pooled JDBC
connection** — open for the whole HTTP request, including view rendering and JSON serialization. On a
slow client or a slow serializer you hold a connection you are not using, and your pool (Q11) is
sized for database work, not for response streaming. It also turns every accidental lazy access in
the web layer into a silent extra query that nobody reviews.

```yaml
spring:
  jpa:
    open-in-view: false      # StreamPulse: off everywhere. Fetch deliberately or fail loudly.
```

The point of turning it off is that it makes the mistake **fail in a test** instead of degrading
production quietly.

**Code:**

```java
// ❌ BROKEN: the association is lazy, the DTO mapping happens outside the transaction.
@GetMapping("/v1/admin/tenants/{id}")
public TenantDto get(@PathVariable UUID id) {
    Tenant t = service.find(id);          // @Transactional ended when this returned
    return new TenantDto(t.getName(),
                         t.getApps().size());   // LazyInitializationException
}

// ✅ FIXED: the fetch decision and the mapping both live inside the transaction.
@Transactional(readOnly = true)
public TenantDto find(UUID id) {
    Tenant t = tenantRepo.findByIdWithApps(id).orElseThrow(TenantNotFound::new);
    return new TenantDto(t.getName(), t.getApps().size());   // still managed here
}

public interface TenantRepository extends JpaRepository<Tenant, UUID> {

    @Query("select t from Tenant t left join fetch t.apps where t.id = :id")
    Optional<Tenant> findByIdWithApps(@Param("id") UUID id);
}
```

The strongest version of this answer is: **never return an entity from a controller**. Use a DTO (a
record) or a Spring Data projection. That kills `LazyInitializationException`, stops your JSON
contract from being coupled to your schema, and prevents a lazy `toString()` in a log statement from
issuing queries.

**Follow-ups they'll ask:**
- *"What about `Hibernate.initialize()`?"* — it works inside a transaction and is fine for a one-off,
  but it is a per-object patch on a per-query problem.
- *"DTO projection vs fetch join?"* — a constructor expression / interface projection
  (`select new com...TenantDto(t.name, count(a)) …`) does not build entities at all: no persistence
  context bloat, no dirty checking, no lazy proxies. For read-only screens it is strictly better.
- *"What does `@Transactional(readOnly = true)` buy on a read path?"* — no dirty check, no automatic
  flush, a read-only JDBC connection, and it documents intent.

**Pitfalls / red flags:** "I set `FetchType.EAGER`" as the answer, or leaving `open-in-view` on and
not knowing it is on.

**Where in StreamPulse:** BS-05 / REQ-DL-03 (the tenant tree); `open-in-view: false` is set in the
base `application.yml` and there is an ArchUnit rule that controllers must not reference `..domain.entity..`.

---

## Q4. How do you *detect* an N+1 problem — not fix it, detect it?

**30-second answer:** You assert on the query count in a test. Hibernate's `Statistics` exposes
`getPrepareStatementCount()`, so I write a test that seeds N parents, calls the endpoint, and asserts
the statement count is ≤ a constant that does not grow with N. That turns "this got slow" into a
build failure. In production the detection is different: p99 latency that scales with result-set
size, a span count in the trace that scales with rows, and `pg_stat_statements` showing one tiny
query with an enormous `calls` count. Eyeballing the SQL log works once; a counted test works forever.

**Deeper:**

Three layers of detection, all of which I'd mention:

| Layer | Tool | What it catches |
|---|---|---|
| Test | Hibernate `Statistics` query counter, or a datasource proxy (`datasource-proxy`, `p6spy`) | regressions, at build time, per endpoint |
| Trace | OTel JDBC instrumentation — 201 child spans under one HTTP span | production, per request, with the exact statement |
| Database | `pg_stat_statements`: high `calls`, low `mean_exec_time`, huge `total_exec_time` | the aggregate cost across all callers |

The "2N+1" shape worth naming: 1 query for tenants, N for each tenant's apps, N more for each app's
titles. With 200 tenants × 20 apps that is 4,201 round trips. Each is maybe 0.3 ms of database time
and 0.5 ms of network — so ~3.5 seconds of pure latency with an idle database and an idle CPU. That
arithmetic (200 + 200×20 + 1, times round-trip time) is the thing to say out loud, because it shows
you understand N+1 is a **latency** problem, not a database-load problem.

**Code — the regression test (REQ-DL-03):**

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = NONE)      // real Postgres via Testcontainers
@DisplayName("REQ-DL-03: tenant tree executes a bounded number of statements")
class TenantTreeQueryCountTest {

    @Autowired EntityManager em;
    @Autowired TenantQueryService service;

    private Statistics stats() {
        return em.getEntityManagerFactory()
                 .unwrap(SessionFactory.class)
                 .getStatistics();               // needs hibernate.generate_statistics=true
    }

    @ParameterizedTest
    @ValueSource(ints = {1, 10, 200})            // the point: the assertion does NOT scale with N
    void boundedQueryCount(int tenantCount) {
        seed(tenantCount, 20, 50);               // tenants × apps × titles
        stats().clear();

        service.tenantTree();

        assertThat(stats().getPrepareStatementCount())
            .as("statement count must not grow with tenant count")
            .isLessThanOrEqualTo(3);
    }
}
```

Keep the **broken** version alive as a documented lab (the plan calls for exactly this): a test
annotated `@Disabled("N+1 lab — see docs/labs/n-plus-one.md")` that asserts the count *does* grow.
Being able to show the before and after is a much better interview artifact than the fix alone.

**Follow-ups they'll ask:**
- *"Why not just look at the SQL log?"* — you will, once, while debugging. It does not survive the
  next refactor. And in a service with 40 endpoints nobody reads 4,000 log lines.
- *"Does `Statistics` cost anything in production?"* — `generate_statistics` adds bookkeeping on
  every session. I enable it in `local` and in tests; in production I rely on tracing and
  `pg_stat_statements`. Measure before leaving it on.
- *"What if the N+1 is in a different layer?"* — same technique, different counter: a WireMock
  request counter for HTTP, a DynamoDB SDK metric publisher for `GetItem` counts. N+1 is a pattern,
  not a JPA feature.

**Pitfalls / red flags:** "I'd notice it in the logs." Seniors automate the noticing.

**Where in StreamPulse:** BS-05 / REQ-DL-03; `TenantTreeQueryCountTest`, `docs/labs/n-plus-one.md`.

**Verify:** the exact `Statistics` accessor names (`getPrepareStatementCount()`,
`getQueryExecutionCount()`, `getEntityLoadCount()`) against your Hibernate version before quoting
them in code.

---

## Q5. Now fix it. What are the options and what does each one cost you?

**30-second answer:** Four real options. `JOIN FETCH` (or `@EntityGraph`, the declarative form of the
same thing) collapses it to one query, but it multiplies rows — a cartesian product — and Hibernate
throws `MultipleBagFetchException` if you try it on two `List` collections at once. `@BatchSize` /
`hibernate.default_batch_fetch_size` turns N queries into N/batch queries with an `IN (…)`, keeping
the row count linear — this is my default for deep trees. `@Fetch(SUBSELECT)` re-runs the original
query as a subselect, which is two queries regardless of N but re-executes the (possibly expensive)
parent query. And the fifth option, which is often the right one for a read-only screen: don't load
entities at all — project straight into a DTO with a hand-written query.

**Deeper:**

| Fix | Queries | Row explosion | Pagination safe | Multiple collections | Best for |
|---|---|---|---|---|---|
| `JOIN FETCH` / `@EntityGraph` | 1 | **yes** — parent rows × child rows | **no** — `HHH000104`: Hibernate paginates in memory | one `List` only (`MultipleBagFetchException`); `Set`s work but still cartesian | one collection, small fan-out, no paging |
| `@BatchSize(size = n)` | 1 + ⌈N/n⌉ | no | yes | any number | deep trees, paginated lists — **default choice** |
| `@Fetch(FetchMode.SUBSELECT)` | 2 | no | yes | any number | one big collection over a large, already-paid-for parent query |
| DTO projection / native SQL | 1 | controlled by your SQL | yes | n/a | read-only screens and reports |

**The cartesian-product problem, concretely.** 200 tenants, each with 20 apps and 50 titles. A single
`join fetch t.apps join fetch t.titles` returns 200 × 20 × 50 = **200,000 rows** to transport and
deduplicate in memory, to produce 200 + 4,000 + 10,000 = 14,200 objects. You traded 4,201 round trips
for one query that moves 14× more data than exists. That is why `MultipleBagFetchException` is
arguably a feature: Hibernate refuses the version of this that it cannot even deduplicate correctly
(a `List`/bag has no identity semantics, so duplicates would be *kept*).

**The pagination trap.** `JOIN FETCH` plus `Pageable` makes Hibernate log
`HHH000104: firstResult/maxResults specified with collection fetch; applying in memory` — it loads
**every** matching row and paginates in Java. On a big table that is an OOM, not a slow query. The
correct shape is two steps: page over the parent ids, then batch-fetch the children for that page.

**Code — the three fixes side by side:**

```java
// Option A — @EntityGraph: declarative fetch plan on the repository method.
//            Same SQL as a fetch join; keeps the JPQL clean; composable per use case.
public interface TenantRepository extends JpaRepository<Tenant, UUID> {

    @EntityGraph(attributePaths = {"apps"})
    @Query("select t from Tenant t where t.active = true")
    List<Tenant> findActiveWithApps();
}

// Option B — @BatchSize: what StreamPulse actually uses for the 3-level tree (REQ-DL-03).
//            1 query for tenants + ⌈200/50⌉ = 4 for apps + ⌈4000/50⌉ = 80 for titles.
//            Row count stays linear; pagination still works.
@Entity
class Tenant {
    @OneToMany(mappedBy = "tenant", fetch = FetchType.LAZY)
    @BatchSize(size = 50)                 // org.hibernate.annotations.BatchSize
    private List<App> apps = new ArrayList<>();
}
```

```yaml
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 50     # global default; beats annotating every collection
```

```java
// Option C — the one I'd actually ship for a read-only console page.
//            No entities, no persistence context, no lazy anything. One query, exact columns.
record TenantTreeRow(UUID tenantId, String tenantName, UUID appId, String appName,
                     UUID titleId, String titleName) {}

List<TenantTreeRow> rows = jdbcClient.sql("""
        SELECT t.id, t.name, a.id, a.name, ti.id, ti.name
        FROM   tenant t
        LEFT   JOIN app   a  ON a.tenant_id = t.id
        LEFT   JOIN title ti ON ti.app_id  = a.id
        WHERE  t.active
        ORDER  BY t.name, a.name, ti.name
        """)
    .query(TenantTreeRow.class)
    .list();
// Assemble the tree in Java — you know the shape, and it is one round trip. (REQ-DL-02 reasoning.)
```

**Follow-ups they'll ask:**
- *"Why does `Set` avoid `MultipleBagFetchException` but not the cartesian product?"* — a `Set` has
  identity semantics so Hibernate can deduplicate the rows; the database still returns all of them.
  You fixed the exception, not the transport cost.
- *"`@EntityGraph` FETCH vs LOAD?"* — `FETCH` (the default) treats every attribute *not* in the graph
  as LAZY. `LOAD` leaves attributes outside the graph at their mapped fetch type, so eager
  `@ManyToOne`s still load. `FETCH` is the predictable one.
- *"What about `distinct`?"* — in Hibernate 6, duplicate parent references from a fetch join are
  removed by default, so `select distinct` is no longer needed for that purpose (and adding it may
  push a real `DISTINCT` into the SQL, which costs a sort). Behaviour differs from Hibernate 5, so
  know which you are on.

**Pitfalls / red flags:** naming only `JOIN FETCH`, and not knowing the pagination or
multiple-collection consequences. Or "I'd add a second-level cache" — that hides an N+1 behind a
cache you now have to invalidate.

**Where in StreamPulse:** BS-05 / REQ-DL-03; the ADR records why `@BatchSize` beat fetch join here.

**Verify:** the Hibernate 6 default-distinct behaviour and the exact `MultipleBagFetchException`
conditions against your pinned version.

---

## Q6. When would you drop the ORM and write SQL?

**30-second answer:** When the work is set-based rather than object-based. JPA is optimised for
loading a small object graph, mutating it, and writing it back — the transactional write model.
The billing report is the opposite: aggregate ten million rows, group, window, return one row per
tenant. There are no entities in that answer, so an entity mapper only gets in the way. In
StreamPulse the admin CRUD is Spring Data JPA and the billing report is `JdbcClient` with
hand-written SQL, and the ADR records the benchmark. The general rule: **JPA for the write model,
SQL for the read model** — which is just CQRS said in one sentence.

**Deeper:**

| | JPA / Hibernate | `JdbcClient` / `JdbcTemplate` | Raw JDBC | jOOQ |
|---|---|---|---|---|
| Sweet spot | small graphs, CRUD, dirty checking, optimistic locking | reports, bulk writes, window functions, DB-specific SQL | driver-level tuning, streaming huge result sets | type-safe dynamic SQL |
| Identity map / L1 | yes | no | no | no |
| Caching, lazy loading | yes | no | no | no |
| Dialect portability | high | low (you wrote the SQL) | low | medium |
| Surprise factor | high (flush timing, N+1, cascades) | low — what you wrote is what runs | none | low |
| Boilerplate | low | low (`JdbcClient`), medium (`JdbcTemplate`) | high | low |

`JdbcClient` is the fluent API Spring added on top of `JdbcTemplate`/`NamedParameterJdbcTemplate`;
it is a façade, not a new driver, so it shares the same `DataSource`, the same `@Transactional`, and
the same connection as your JPA code when both run in one transaction. That last point matters: you
can mix them inside a single transaction, but if you write via `JdbcClient` and then read via JPA in
the same transaction, JPA's L1 may serve you a stale object — flush and/or clear across the boundary.

**Code — REQ-DL-02, the billing report:**

```java
@Repository
class BillingReportRepository {

    private final JdbcClient jdbc;      // Spring Framework 6.1+ / Boot 3.2+ — see Verify

    /**
     * Monthly usage per tenant, with the month-over-month delta computed in the database.
     * Written in SQL because:
     *   - it is a set operation over ~10M usage_hourly rows; there are no entities involved;
     *   - LAG() has no JPQL equivalent (Hibernate 6 supports some window functions in HQL,
     *     but the readable version is the SQL one);
     *   - the query plan is the thing being tuned, and hiding it behind a mapper helps nobody.
     * Target: < 2s (BS-05 NFR). EXPLAIN ANALYZE output is committed in the ADR.
     */
    List<TenantUsage> monthlyUsage(YearMonth month) {
        return jdbc.sql("""
            SELECT tenant_id,
                   SUM(event_count)                                    AS events,
                   SUM(bytes_stored) / 1024.0 / 1024 / 1024            AS gb_stored,
                   SUM(event_count) - LAG(SUM(event_count)) OVER (
                       PARTITION BY tenant_id ORDER BY date_trunc('month', bucket_start)
                   )                                                   AS delta_vs_prev_month
            FROM   usage_hourly
            WHERE  bucket_start >= :from AND bucket_start < :to
            GROUP  BY tenant_id, date_trunc('month', bucket_start)
            """)
            .param("from", month.atDay(1).atStartOfDay())
            .param("to",   month.plusMonths(1).atDay(1).atStartOfDay())
            .query(TenantUsage.class)     // record-style row mapping
            .list();
    }
}
```

The honest framing for the interview: *"I wrote the same query both ways, measured, and chose. The
JPQL version worked; it was harder to read and I could not control the plan. The number is in the
ADR."* Measured beats dogmatic.

**Follow-ups they'll ask:**
- *"Doesn't mixing two data-access styles hurt maintainability?"* — it is one extra idiom, scoped to
  the read model, and each is used where it is strongest. The alternative — bending JPQL until it
  produces the plan you want — is worse to maintain because the mapping between what you wrote and
  what runs is no longer obvious.
- *"How do you avoid SQL injection here?"* — named parameters only; string concatenation of user
  input is banned by review and, for the dynamic-filter cases, the column/direction is validated
  against an allow-list enum before it touches the SQL.
- *"What about bulk updates?"* — `@Modifying @Query("update …")` or plain SQL, and remember it
  **bypasses the persistence context**: entities already loaded are now stale, and `@Version` is not
  bumped unless you bump it yourself. Use `@Modifying(clearAutomatically = true, flushAutomatically = true)`
  or do bulk work in its own transaction.
- *"Streaming a huge result set?"* — that is the raw-JDBC case: `setFetchSize`, autocommit off on
  Postgres (otherwise the driver buffers the whole result), and a cursor. JPA's `Stream<T>` still
  builds entities.

**Pitfalls / red flags:** "ORMs are bad" or "always use JPA". Both answers are a refusal to think
about the access pattern.

**Where in StreamPulse:** BS-05 / REQ-DL-01 (JPA write model) vs REQ-DL-02 (JDBC read model);
`BillingReportIT` seeds 10M rows and captures `EXPLAIN ANALYZE`.

**Verify:** `JdbcClient` exists from Spring Framework 6.1 (Boot 3.2). On an older version, use
`NamedParameterJdbcTemplate`. Also confirm which window functions your Hibernate version supports in
HQL before claiming JPQL cannot do it.

---

## Q7. Go through the transaction propagation types. What actually happens with `REQUIRES_NEW`?

**30-second answer:** Seven of them. `REQUIRED` is the default and is what you want 95% of the time:
join the caller's transaction, or start one. `REQUIRES_NEW` **suspends** the caller's transaction and
starts a genuinely independent one — and the part people miss is that this takes a **second physical
connection from the pool**, held simultaneously with the first. That is how you self-deadlock a small
pool: with `maximumPoolSize = 10`, ten concurrent requests each holding an outer connection and
waiting for an inner one will sit there until `connectionTimeout` fires. `NESTED` is different again:
one connection, a JDBC **savepoint**, so the inner work can roll back without killing the outer — but
if the outer rolls back, the inner goes with it.

**Deeper:**

| Propagation | If a transaction exists | If none exists | Connections used |
|---|---|---|---|
| `REQUIRED` (default) | join it | start one | 1 |
| `REQUIRES_NEW` | **suspend** it, start a new one | start one | **2** (concurrently) |
| `NESTED` | savepoint inside it | start one (behaves like `REQUIRED`) | 1 |
| `SUPPORTS` | join it | run **non-transactionally** | 0 or 1 |
| `NOT_SUPPORTED` | suspend it, run non-transactionally | run non-transactionally | 1 suspended |
| `MANDATORY` | join it | throw `IllegalTransactionStateException` | 1 |
| `NEVER` | throw | run non-transactionally | 0 |

**REQUIRED vs REQUIRES_NEW vs NESTED, mechanically:**

```
REQUIRED                REQUIRES_NEW                    NESTED
┌───────────┐           ┌───────────┐                   ┌─────────────────┐
│ outer tx  │           │ outer tx  │ ──suspended──┐    │ outer tx        │
│  ┌──────┐ │           └───────────┘              │    │  SAVEPOINT s1   │
│  │inner │ │  same          conn #1               │    │   ┌──────────┐  │
│  └──────┘ │  conn      ┌───────────┐             │    │   │ inner    │  │
└───────────┘            │ inner tx  │  conn #2 ◀──┘    │   └──────────┘  │
 one commit,             │  COMMITS  │  independently   │  ROLLBACK TO s1 │ ← inner only
 one rollback            └───────────┘                  └─────────────────┘
                                                         one conn, one commit
```

- `REQUIRED`: there is **one** physical transaction. An inner rollback marks the *whole* thing
  rollback-only, and the outer commit then throws `UnexpectedRollbackException`. That is the
  "I caught the exception, why did everything still roll back?" question.
- `REQUIRES_NEW`: two physical transactions, two connections, two commits. The inner one is durable
  even if the outer rolls back — which is the entire point for the audit row (REQ-DL-04). The cost is
  the second connection and the fact that you now have **no atomicity** across the two.
- `NESTED`: one connection, savepoints. Outer rollback still discards everything. Support depends on
  the transaction manager and the driver — `DataSourceTransactionManager` supports it;
  `JpaTransactionManager` requires `setNestedTransactionAllowed(true)` and the JPA provider to
  cooperate. Check before designing around it.

**Code — REQ-DL-04, and the pool hazard in the same breath:**

```java
@Service
class RuleService {
    private final AuditService audit;                 // separate bean → the proxy is crossed (file 01, Q9)

    @Transactional                                    // REQUIRED: conn #1
    public void update(RuleUpdate cmd) {
        audit.record(cmd);                            // borrows conn #2, commits, returns it
        validate(cmd);                                // throws → conn #1 rolls back
        ruleRepo.save(cmd.toEntity());
    }
}

@Service
class AuditService {
    /**
     * REQUIRES_NEW because the audit must survive the caller's rollback (REQ-DL-04).
     * POOL BUDGET: this method holds a SECOND connection for its whole duration while the
     * caller still holds the first. With maximumPoolSize=10, 10 concurrent update() calls
     * would all be waiting for an 11th connection that cannot exist → connectionTimeout.
     * Mitigations: keep the inner transaction tiny (one INSERT, no network calls), and size
     * the pool for peak *nesting depth* × concurrency, not just concurrency (Q11).
     */
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void record(RuleUpdate cmd) {
        auditRepo.save(AuditRecord.attempted(cmd));
    }
}
```

```java
@Test
@DisplayName("REQ-DL-04: audit row survives the rule rollback")
void auditSurvivesRollback() {
    assertThatThrownBy(() -> ruleService.update(invalidRule()))
        .isInstanceOf(ValidationException.class);

    assertThat(ruleRepo.findAll()).isEmpty();      // rule rolled back
    assertThat(auditRepo.findAll()).hasSize(1);    // audit committed independently
}
```

**Follow-ups they'll ask:**
- *"Is there a cheaper way to get the audit row?"* — yes, and I'd consider it: publish the audit as
  an application event and persist it after the transaction resolves
  (`@TransactionalEventListener(phase = AFTER_COMPLETION)`), or write it to the outbox. `REQUIRES_NEW`
  is the simplest thing that works; it is not free.
- *"What does `UnexpectedRollbackException` mean?"* — an inner participating transaction marked the
  shared transaction rollback-only, and then the outer tried to commit. Your catch block did not undo
  the mark.
- *"When is `MANDATORY` useful?"* — on a repository-layer method that must never be called outside a
  transaction. It converts a silent autocommit-per-statement bug into a startup-time-loud failure.
- *"Does `@Transactional` on a `private` method work?"* — no. Proxy-based AOP; it is not even
  advisable. Same family as the self-invocation gotcha.

**Pitfalls / red flags:** listing the seven propagation types from memory but not knowing
`REQUIRES_NEW` takes a second connection. That single fact is the senior signal here.

**Where in StreamPulse:** BS-05 / REQ-DL-04; `AuditRequiresNewTest`, `SelfInvocationGotchaTest`, and
the HikariCP lab in REQ-DL-07.

**Verify:** `NESTED` support for `JpaTransactionManager` in your Spring version, and savepoint
support in your JDBC driver.

---

## Q8. Isolation levels — what anomaly does each one prevent, and what does Postgres actually do?

**30-second answer:** Four levels, three classic anomalies. READ UNCOMMITTED allows dirty reads;
READ COMMITTED prevents dirty reads but allows non-repeatable reads and phantoms; REPEATABLE READ
prevents non-repeatable reads; SERIALIZABLE prevents phantoms and everything else. The important
practical point is that **Postgres does not implement these literally** — it is MVCC, so readers
never block writers, READ UNCOMMITTED behaves as READ COMMITTED, and REPEATABLE READ is snapshot
isolation which already prevents phantoms. That is why READ COMMITTED is the practical default: under
MVCC it is cheap, it never blocks a reader, and the anomalies it leaves are better solved with
explicit locking or an optimistic version column than with a global isolation level.

**Deeper:**

| Level | Dirty read | Non-repeatable read | Phantom read | Lost update |
|---|---|---|---|---|
| READ UNCOMMITTED | possible (not in PG) | possible | possible | possible |
| READ COMMITTED | no | possible | possible | possible |
| REPEATABLE READ | no | no | possible per the standard; **not in Postgres** | no (PG aborts with 40001) |
| SERIALIZABLE | no | no | no | no |

The anomalies, stated so they cannot be confused:

- **Dirty read** — you read a row another transaction has written but not committed.
- **Non-repeatable read** — you read the *same row* twice and get different values, because someone
  committed in between.
- **Phantom read** — you run the *same query* twice and the **set of rows** changes, because someone
  inserted or deleted a matching row.
- **Lost update** — two transactions read the same row, both compute from it, both write; one
  write is silently discarded. This is the one that actually bites in application code, and note it
  is not in the standard's anomaly list.

**Postgres MVCC in three sentences.** Every row version carries `xmin` (the transaction that created
it) and `xmax` (the transaction that deleted/superseded it). A `SELECT` takes a snapshot — a set of
transaction ids considered visible — and reads the row version visible to that snapshot; an `UPDATE`
writes a **new row version** rather than overwriting, so readers never wait for writers and writers
never wait for readers. The cost is dead tuples, which `VACUUM`/autovacuum reclaims, and table bloat
if autovacuum can't keep up — which is the real operational consequence of a long-running
transaction, because it holds back the oldest visible snapshot and blocks cleanup for the whole
database.

**Where the levels differ in Postgres specifically:**

- **READ COMMITTED**: a new snapshot is taken **per statement**. So a multi-statement read can see
  the database move under it. An `UPDATE … WHERE` that finds a row locked will wait, then
  **re-evaluate the WHERE clause against the new version** ("EPQ" / EvalPlanQual) — which is a real
  source of surprise.
- **REPEATABLE READ**: one snapshot for the **whole transaction**, so it is true snapshot isolation
  and phantoms cannot appear. A conflicting update aborts your transaction with
  `could not serialize access due to concurrent update` (SQLSTATE 40001) — so your application
  **must** have a retry path.
- **SERIALIZABLE**: snapshot isolation plus Serializable Snapshot Isolation (SSI) — Postgres tracks
  read/write dependencies and aborts one transaction of any cycle, also with 40001. No extra locks
  taken, but more aborts under contention, so again: retry, with backoff and a cap.

**Code:**

```java
/**
 * REQ-DL-05 — monthly quota allocation. Two admins must not over-allocate the same pool.
 *
 * Option A (chosen): READ COMMITTED + SELECT ... FOR UPDATE. Predictable: the second
 *   transaction blocks on the row lock, then reads the post-commit value. No retry loop
 *   to test, no 40001 handling, and the contention is on one row of one tenant.
 * Option B: SERIALIZABLE + retry on SQLSTATE 40001. Fewer locks, but every caller needs
 *   idempotent retry logic and the failure mode is an abort under load — worse for a
 *   human-facing admin action.
 */
@Transactional(isolation = Isolation.READ_COMMITTED)
public void allocate(UUID tenantId, long amount) {
    QuotaPool pool = quotaRepo.lockForUpdate(tenantId);      // SELECT ... FOR UPDATE
    if (pool.remaining() < amount) {
        throw new QuotaExceededException(tenantId, amount, pool.remaining());
    }
    pool.consume(amount);                                    // dirty checking writes it
}
```

```java
public interface QuotaRepository extends JpaRepository<QuotaPool, UUID> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)                    // → SELECT ... FOR UPDATE
    @Query("select q from QuotaPool q where q.tenantId = :id")
    QuotaPool lockForUpdate(@Param("id") UUID id);
}
```

**Follow-ups they'll ask:**
- *"Why not just set SERIALIZABLE everywhere?"* — because you inherit a retryable failure on every
  write path, and retry-on-abort must be idempotent and bounded or it becomes a retry storm. It is a
  correct answer that moves the complexity into every caller.
- *"Does a higher isolation level fix a lost update?"* — REPEATABLE READ and above in Postgres do
  (by aborting), READ COMMITTED does not. But the application-level fix — `@Version` or
  `FOR UPDATE` — is more targeted and gives a better error message.
- *"What breaks with a long-running transaction?"* — it pins the oldest snapshot, autovacuum can't
  reclaim dead tuples, tables and indexes bloat, plans degrade. Also, in Postgres, it consumes a
  connection the whole time. This is another reason `open-in-view` is off (Q3).
- *"MySQL differences?"* — InnoDB's default is REPEATABLE READ, not READ COMMITTED, and it uses
  **gap locks / next-key locks** in RR to prevent phantoms, which means it takes locks Postgres does
  not and can deadlock in places Postgres would not.

**Pitfalls / red flags:** reciting the anomaly table without knowing that Postgres's REPEATABLE READ
already prevents phantoms, or not knowing that SERIALIZABLE means "be ready to retry".

**Where in StreamPulse:** BS-05 / REQ-DL-05; `QuotaAllocationConcurrencyIT` runs N concurrent
allocations and asserts the pool is never over-drawn and the total is conserved.

---

## Q9. Optimistic or pessimistic locking — how do you choose?

**30-second answer:** Optimistic (`@Version`) when conflicts are rare and the user can be told to
retry — human-scale editing, like the admin UI updating an alert rule. The row is not locked; the
`UPDATE` carries `AND version = ?`, and a zero-row update means someone else won, which surfaces as
`OptimisticLockException` and, in my API, a 409 with a ProblemDetail. Pessimistic (`SELECT … FOR
UPDATE`) when conflicts are *likely* and a retry loop would just thrash — quota allocation, counter
decrements, and above all **queue-style workloads**, where `FOR UPDATE SKIP LOCKED` is the pattern
that lets N workers pull disjoint rows from one table without blocking each other.

**Deeper:**

| | Optimistic (`@Version`) | Pessimistic (`FOR UPDATE`) |
|---|---|---|
| Lock held | none | row lock, for the rest of the transaction |
| Detects conflict | at write time | prevents it at read time |
| Failure mode | `OptimisticLockException` → 409 → user retries | blocked writer, then success; or lock timeout |
| Scales with contention | badly (livelock: everyone retries and loses) | predictably (serialised, bounded by lock wait) |
| Works across a detached/HTTP round trip | **yes** — the version travels in the DTO | no — you cannot hold a DB lock across a user think-time |
| Cost | one extra column, one extra WHERE predicate | connection held longer, deadlock risk if lock order varies |

**Optimistic across an HTTP round trip is the killer feature.** The admin GETs a rule at version 4,
edits it for two minutes, PUTs it back with `version: 4`. If someone else saved in between, the row
is at version 5 and your update matches zero rows. You cannot do that with a pessimistic lock — no
sane system holds a database row lock for two minutes of human thinking. Practically, expose the
version as an HTTP `ETag` and require `If-Match`, so the optimistic check is part of the API contract
rather than a hidden column.

**`FOR UPDATE SKIP LOCKED`** is the one to know cold, because it turns a relational table into a
usable work queue and it is exactly the outbox-relay pattern (file 07):

```sql
-- N poller instances run this concurrently. Each gets a DISJOINT batch, nobody blocks.
-- Without SKIP LOCKED, instance 2 would WAIT for instance 1's rows instead of taking others.
-- NOWAIT would instead error immediately — useful when you want to fail fast, not skip.
UPDATE outbox
SET    status = 'SENDING', claimed_at = now(), claimed_by = :instanceId
WHERE  id IN (
    SELECT id FROM outbox
    WHERE  status = 'PENDING'
    ORDER  BY created_at          -- preserves ordering as far as a concurrent claimer can
    LIMIT  100
    FOR UPDATE SKIP LOCKED        -- Postgres 9.5+, MySQL 8.0+
)
RETURNING id, aggregate_id, event_type, payload;
```

Note the ordering caveat I'd volunteer: `SKIP LOCKED` with multiple workers gives you *approximate*
FIFO, not strict global ordering. If you need per-key ordering, claim by a partition of the key space
(one worker per hash bucket) rather than racing on one queue.

**Code:**

```java
@Entity
class AlertRule {
    @Id private UUID id;

    @Version                       // long is safest; a timestamp @Version has clock-skew edges
    private long version;          // REQ-DL-01

    private String expression;
}

// The UPDATE Hibernate emits:
//   UPDATE alert_rule SET expression=?, version=5 WHERE id=? AND version=4
// 0 rows affected  →  OptimisticLockException
```

```java
@RestControllerAdvice
class ConflictHandler {
    /** REQ-DL-01: optimistic conflicts become a 409 with an RFC 7807 body, not a 500. */
    @ExceptionHandler(ObjectOptimisticLockingFailureException.class)
    ProblemDetail onConflict(ObjectOptimisticLockingFailureException e) {
        var pd = ProblemDetail.forStatus(HttpStatus.CONFLICT);
        pd.setType(URI.create("https://streampulse.dev/problems/stale-write"));
        pd.setTitle("Rule was modified by someone else");
        pd.setDetail("Reload the rule and reapply your change.");
        return pd;
    }
}
```

**Follow-ups they'll ask:**
- *"`PESSIMISTIC_READ` vs `PESSIMISTIC_WRITE` vs `PESSIMISTIC_FORCE_INCREMENT`?"* — read is a shared
  lock (`FOR SHARE`), write is exclusive (`FOR UPDATE`), force-increment bumps the `@Version` even
  when you only read, which is how you make a parent "change" when only a child changed.
- *"`OPTIMISTIC_FORCE_INCREMENT` — why?"* — same idea for aggregates: editing a child should conflict
  with a concurrent edit of the aggregate root.
- *"What about deadlocks with `FOR UPDATE`?"* — take locks in a **consistent order** (sort by id
  before locking, exactly like the BS-02 quota-transfer lab), set a `lock_timeout`, and expect
  SQLSTATE 40P01 in your error handling.
- *"Does `@Version` work if someone updates the row with raw SQL?"* — no, unless that SQL bumps the
  version column too. Bulk `@Modifying` updates have the same hole.

**Pitfalls / red flags:** "I'd use `synchronized`" (wrong process, wrong machine, wrong decade), or
not knowing `SKIP LOCKED` exists when asked how to build a queue on Postgres.

**Where in StreamPulse:** BS-05 / REQ-DL-01 (optimistic on rules), REQ-DL-05 (pessimistic on quota),
and the outbox relay in Phase 6 (`SKIP LOCKED`).

---

## Q10. Which exceptions roll back a Spring transaction?

**30-second answer:** By default, only `RuntimeException` and `Error`. A **checked** exception does
**not** roll back — the transaction commits with the partial work in it. That default surprises
people, and it silently ships corrupt data. Two fixes: throw unchecked exceptions from your service
layer (my preference, because a checked exception in a business method is usually the wrong design
anyway), or declare `@Transactional(rollbackFor = Exception.class)`. The second thing to know: a
rollback triggered inside a `REQUIRED` inner method marks the whole shared transaction rollback-only,
so catching the exception in the caller does **not** save you — the commit then throws
`UnexpectedRollbackException`.

**Deeper:**

```
throw inside @Transactional
        │
        ├── RuntimeException / Error ──────────────▶ rollback            (default)
        ├── checked Exception ────────────────────▶ COMMIT               (default — the trap)
        ├── listed in rollbackFor ────────────────▶ rollback
        └── listed in noRollbackFor ──────────────▶ commit

caught by the CALLER, but the inner tx was REQUIRED and already marked rollback-only
        └──▶ outer commit throws UnexpectedRollbackException
```

The rule is "most specific match wins": with `rollbackFor = Exception.class` and
`noRollbackFor = QuotaWarning.class`, a `QuotaWarning` commits and everything else rolls back.

The `TransactionSynchronizationManager` hooks are the other half of this question, because they are
how you avoid the dual-write bug without an outbox in the simple cases:

```java
// Publish to Kafka only if the transaction actually committed.
// Still NOT atomic (the process can die between commit and publish) — which is exactly
// why StreamPulse uses the outbox + CDC instead (file 07). Know the difference.
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
void onRuleChanged(RuleChangedEvent e) { publisher.publish(e); }
```

**Code — the bug and the fix:**

```java
// ❌ BROKEN: IOException is checked → Spring COMMITS the half-written rule change.
@Transactional
public void importRules(MultipartFile f) throws IOException {
    for (RuleCsvRow row : parse(f)) {         // may throw IOException halfway through
        ruleRepo.save(row.toRule());
    }
}

// ✅ FIXED (option 1, preferred): translate at the boundary; the service layer throws unchecked.
@Transactional
public void importRules(MultipartFile f) {
    try {
        for (RuleCsvRow row : parse(f)) ruleRepo.save(row.toRule());
    } catch (IOException e) {
        throw new RuleImportFailedException("could not read rule CSV", e);   // RuntimeException
    }
}

// ✅ FIXED (option 2): keep the checked exception, widen the rollback rule explicitly.
@Transactional(rollbackFor = Exception.class)
public void importRules(MultipartFile f) throws IOException { … }
```

**Follow-ups they'll ask:**
- *"Why is the default like that?"* — it mirrors the EJB convention: checked exceptions were meant to
  be recoverable business outcomes the caller handles, so committing was the "expected outcome" path.
  Reasonable in 1999; a footgun now.
- *"How do you force a rollback without throwing?"* —
  `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()`. Use sparingly; it is
  invisible at the call site.
- *"Does the rollback undo things outside the database?"* — no. A Kafka publish, an S3 PUT, an email
  are all already gone. That asymmetry is the dual-write problem and the reason for the outbox.
- *"Class-level vs method-level `@Transactional`?"* — the most specific annotation wins; method
  overrides class. Note Spring's own rule that `@Transactional` on an interface method is not
  detected with CGLIB proxies — put it on the class.

**Pitfalls / red flags:** not knowing the checked-exception default. It is a five-second question with
a big signal.

**Where in StreamPulse:** BS-05 / REQ-DL-06 — `OutboxAtomicityIT` asserts that on rollback *neither*
the entity nor the outbox row exists, which only holds if the rollback rules are right.

---

## Q11. How do you size a HikariCP pool, and what breaks when you get it wrong?

**30-second answer:** You measure. The commonly cited starting point — `connections ≈ (cores × 2) +
effective spindles`, from the PostgreSQL wiki via HikariCP's docs — is a **heuristic**, not a law, and
it is about the *database server's* cores, not your app pod's. The counter-intuitive part is that a
bigger pool is usually **slower**: past the point where the database can do useful parallel work, more
connections mean more context switching, more lock contention and more memory per backend, so
throughput flattens and latency rises. I size from the little's-law shape — concurrent in-flight
queries = arrival rate × query duration — then load-test and watch pool wait time, not pool size.
And I budget for nesting: every `REQUIRES_NEW` (Q7) doubles a request's connection demand.

**Deeper — the settings that actually matter:**

| Property | Default (verify) | What it means | How I set it |
|---|---|---|---|
| `maximumPoolSize` | 10 | hard cap on concurrent connections | from load test; also `replicas × maxPoolSize` must stay under Postgres `max_connections`, minus a reserve |
| `minimumIdle` | = max | idle connections kept warm | leave equal to max for a steady-load service; a fixed pool avoids churn |
| `connectionTimeout` | 30 s | how long `getConnection()` waits before throwing | **lower it** (2–5 s). Failing fast turns pool exhaustion into a visible 503 instead of a pile-up |
| `maxLifetime` | 30 min | retire a connection this old, even if healthy | set **below** any infra timeout (DB `idle_in_transaction_session_timeout`, a proxy/NAT/firewall idle kill). HikariCP's guidance is a margin of at least ~30 s |
| `idleTimeout` | 10 min | retire idle connections above `minimumIdle` | only relevant if `minimumIdle < maximumPoolSize` |
| `leakDetectionThreshold` | 0 (off) | log a stack trace when a connection is held longer than this | on in `local` and CI (e.g. 20 s). Points straight at the method that leaked |
| `validationTimeout`, `keepaliveTime` | — | liveness of pooled connections | matters behind a NAT that silently drops idle TCP |

**Little's law, shown rather than asserted:** if the service handles 400 req/s and each request spends
15 ms inside the database, the average number of *simultaneously active* database operations is
400 × 0.015 = **6**. A pool of 10 is already generous; a pool of 100 adds queueing inside Postgres
rather than in your app, where you cannot see it. The number to alarm on is
`hikaricp.connections.pending` and the `hikaricp.connections.acquire` timer — if acquisition time is
near zero, the pool is big enough regardless of what the formula said.

**Pool exhaustion with virtual threads — the modern version of this question.** Virtual threads remove
the thread pool as an implicit concurrency limiter. With platform threads and a 200-thread Tomcat
pool, at most 200 requests could ever ask for a connection. With
`spring.threads.virtual.enabled=true`, 20,000 concurrent requests each spawn a virtual thread, and
all 20,000 pile onto `getConnection()`. The pool is still 10. So:

1. You need an **explicit** admission-control limit — a `Semaphore`, a Resilience4j `Bulkhead`, or a
   rate limiter — in front of the database, because the thread pool is no longer doing that job for
   you.
2. A low `connectionTimeout` becomes essential: with 20,000 waiters and a 30 s timeout you have built
   a 30-second-deep queue that guarantees every client times out first.
3. Blocking inside a virtual thread is fine — that is the design — but the **pinning** question
   matters: a virtual thread that blocks inside a `synchronized` block pins its carrier thread. Check
   whether the pool and driver versions you pin still use `synchronized` on the hot path, and run
   with pinning diagnostics during the load test.

**Code:**

```yaml
# StreamPulse control-plane. Numbers are a STARTING POINT to be replaced by Phase 9 measurements.
spring:
  datasource:
    hikari:
      maximum-pool-size: 16          # heuristic start: 8-core db → (8*2)+0 = 16. Then measure.
      minimum-idle: 16               # fixed-size pool: no churn under spiky load
      connection-timeout: 3000       # fail fast → 503 with Retry-After, not a 30s stall
      max-lifetime: 1500000          # 25 min, comfortably under the 30-min idle killer in front of PG
      leak-detection-threshold: 20000    # local/CI only; logs the holder's stack trace
      pool-name: control-plane-pool      # shows up in the Micrometer tag and in thread dumps
  threads:
    virtual:
      enabled: true                  # verify the property name for your Boot version
```

```java
/**
 * REQ-DL-07 lab. With virtual threads there is no thread pool bounding DB concurrency,
 * so we bound it explicitly. The semaphore is sized ABOVE the pool (some headroom for
 * fairness) but far below the request concurrency, so queueing happens here — where we
 * can measure and shed it — rather than inside HikariCP's handoff queue.
 */
@Component
class DbAdmissionControl {
    private final Semaphore permits = new Semaphore(24, true);

    <T> T call(Supplier<T> work) {
        if (!permits.tryAcquire(/* timeout */ 200, TimeUnit.MILLISECONDS)) {
            throw new ServiceOverloadedException();   // → 503 + Retry-After
        }
        try { return work.get(); } finally { permits.release(); }
    }
}
```

**Follow-ups they'll ask:**
- *"Symptoms of pool exhaustion?"* — `SQLTransientConnectionException: … request timed out after
  30000ms`, `hikaricp.connections.pending` climbing, p99 latency that looks like a step function at
  exactly `connectionTimeout`, and threads parked in `getConnection` in the thread dump.
- *"Pool per service or shared?"* — per service instance, always. Then do the multiplication:
  `pods × maxPoolSize + migrations + admin tools ≤ max_connections − superuser_reserved_connections`.
  At high pod counts this is where PgBouncer in transaction mode earns its place — with the caveat
  that transaction pooling breaks session-level features (prepared statements without protocol
  support, advisory locks, `SET`).
- *"`maxLifetime` — why bother if connections are healthy?"* — because something upstream (a NAT
  gateway, a load balancer, the DB's own timeout) will kill them at an unpredictable moment, and you
  want the pool to retire them on *your* schedule and not hand a dead socket to a request.
- *"How does this interact with `REQUIRES_NEW`?"* — a request that nests one level needs two
  connections at once. Effective capacity is `maxPoolSize / max nesting depth`. Say this; it links
  back to Q7.

**Pitfalls / red flags:** quoting the formula as if it were physics, or "we set it to 100 to be safe".

**Where in StreamPulse:** BS-05 / REQ-DL-07; pool metrics on the Grafana dashboard (Phase 8), the
virtual-thread exhaustion lab, `leakDetectionThreshold` on in `local`.

**Verify:** all HikariCP defaults above, and the `spring.threads.virtual.enabled` property name, for
the exact versions you pin. Also verify the current state of `synchronized`-related pinning in your
HikariCP and JDBC driver versions — this has been changing.

---

## Q12. How do you change the schema of a busy table without downtime?

**30-second answer:** Flyway for the mechanics, **expand/contract** for the strategy. Every migration
must be backward-compatible with the *currently running* version of the application, because during a
rolling deploy both versions are live at once. So a rename becomes: add the new column, backfill it
in batches, dual-write from the app, switch reads, stop writing the old column, and only in a later
release drop it. Five deploys instead of one, and no downtime. The second half is knowing which DDL
takes a blocking lock: on Postgres, `CREATE INDEX` must be `CONCURRENTLY`, and every `ALTER TABLE`
needs a `lock_timeout` so it queues rather than blocking the world behind it.

**Deeper:**

**Expand / contract, as a release plan:**

```
release 1  EXPAND   ALTER TABLE alert_rule ADD COLUMN expression_v2 text;   -- nullable, no default
                    (old code ignores it; new code not deployed yet)
release 2  MIGRATE  app dual-writes expression AND expression_v2
                    background job backfills expression_v2 in batches of 5k, with a sleep
release 3  SWITCH   reads come from expression_v2 (feature flag, instant rollback)
release 4  STOP     app stops writing expression
release 5  CONTRACT ALTER TABLE alert_rule DROP COLUMN expression;
```

Each step is independently reversible, and no step requires both versions of the app to agree.

**The Postgres lock table you should be able to sketch:**

| Operation | Lock | Blocks reads | Notes |
|---|---|---|---|
| `ADD COLUMN` (nullable, no default) | ACCESS EXCLUSIVE, **brief** | momentarily | catalogue-only, no rewrite |
| `ADD COLUMN … DEFAULT <constant>` | ACCESS EXCLUSIVE, brief | momentarily | no table rewrite since PG 11 (stored as a "fast default") |
| `ADD COLUMN … NOT NULL` with no default | ACCESS EXCLUSIVE + **full scan/rewrite** | yes, for the duration | avoid; add nullable, backfill, then add the constraint |
| `SET NOT NULL` | ACCESS EXCLUSIVE + scan | yes | cheaper if a matching `CHECK … NOT VALID` was already validated |
| `CREATE INDEX` | SHARE — blocks **writes** | no | use `CONCURRENTLY` |
| `CREATE INDEX CONCURRENTLY` | does not block reads or writes | no | two passes, slower, **cannot run inside a transaction** |
| `ALTER TABLE … TYPE` | ACCESS EXCLUSIVE + rewrite | yes | treat as expand/contract, never in-place on a big table |
| `DROP COLUMN` | ACCESS EXCLUSIVE, brief | momentarily | logically hidden; space reclaimed by later rewrites |

The subtle killer: an `ACCESS EXCLUSIVE` request **queues behind** existing readers and **every
subsequent query queues behind it**. So a 10-second `ALTER` waiting on one long `SELECT` stalls the
whole table for as long as that `SELECT` runs. Hence:

```sql
SET lock_timeout = '3s';   -- fail the migration rather than build a queue of blocked queries
```

**Code — Flyway conventions in StreamPulse:**

```sql
-- db/migration/V17__add_expression_v2.sql   (EXPAND — safe with old pods still running)
SET lock_timeout = '3s';
ALTER TABLE alert_rule ADD COLUMN expression_v2 text;   -- nullable: old INSERTs still work
```

```sql
-- db/migration/V18__index_expression_v2.sql
-- CREATE INDEX CONCURRENTLY cannot run inside a transaction, so this migration must be
-- marked as non-transactional for Flyway. Verify the exact mechanism for your Flyway version.
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_alert_rule_expr_v2
    ON alert_rule (tenant_id) WHERE expression_v2 IS NOT NULL;
```

Flyway rules I'd state as non-negotiable:
- **Versioned migrations are immutable.** Flyway stores a checksum in `flyway_schema_history`; editing
  a shipped migration fails validation on every environment that already ran it. Fix forward with a
  new `V<n+1>`.
- `R__` repeatable migrations (views, functions) re-run whenever their checksum changes — good for
  read-model views, wrong for anything with data.
- **No `DROP` in the same release that stops using the thing.** Rollback needs the old column.
- `flyway.validate-on-migrate` stays on; `clean` is disabled outside `local`
  (Flyway has a `cleanDisabled` setting that defaults to on in recent versions — verify).
- Run migrations from **one** place (an init container or a dedicated job), not from N concurrently
  starting pods. Flyway does take a lock, but a single runner makes the failure mode obvious.

**Follow-ups they'll ask:**
- *"How do you back-fill 200 million rows?"* — batched by primary-key range, each batch its own
  transaction, throttled with a sleep, resumable via a checkpoint table, and watching replication lag.
  One giant `UPDATE` bloats the table, holds a long transaction (blocking vacuum — see Q8) and cannot
  be paused.
- *"What about renaming a column?"* — you don't. Rename is expand/contract. A literal `RENAME` breaks
  every pod still running the old code, for the whole rollout window.
- *"How does this interact with Debezium?"* — a schema change flows into the CDC stream and can break
  downstream consumers. That is precisely why the outbox table's schema is fixed and generic
  (`aggregate_id`, `type`, `payload jsonb`): the *business* schema can evolve without touching the
  CDC contract. That's file 07.
- *"Rollback of a migration?"* — Flyway Community has no `undo`. The real answer is that expand/contract
  makes rollback a *deploy* rollback, not a schema rollback. Every safe migration is additive.

**Pitfalls / red flags:** "Flyway handles it", or proposing a `RENAME COLUMN` on a live table.

**Where in StreamPulse:** BS-05 / REQ-DL-01 (Flyway-managed schema), Phase 6; the outbox schema in
Phase 6 is deliberately CDC-stable.

**Verify:** the PG-11 fast-default behaviour, `cleanDisabled` defaults, and how your Flyway version
marks a migration as non-transactional for `CREATE INDEX CONCURRENTLY`.

---

## Q13. SQL or NoSQL? Walk me through how you'd actually decide.

**30-second answer:** I start from the access patterns, not the data model. Write down every query the
system must serve, with its latency budget and its cardinality — then pick the store that answers them
without a scan. Relational wins when the queries are open-ended, ad-hoc and relational (the billing
report joins usage, tenants and contracts, and finance will invent a new question next month).
DynamoDB wins when the patterns are **known, few, and keyed** — give me tenant + metric + time range
and I'll design a partition key that answers it in single-digit milliseconds at any scale. In
StreamPulse both are true at once, which is why the control plane is Postgres and the aggregate
history is DynamoDB. That is CQRS with two engines, not indecision.

**Deeper — the StreamPulse decision table:**

| Data | Access pattern | Store | Why |
|---|---|---|---|
| Tenants, apps, titles, rules | ad-hoc joins, small volume, strong consistency, transactions | **Postgres** | it's the write model; relationships and constraints are the point |
| Windowed aggregates (history) | `tenant + metric + time range`, huge volume, known query | **DynamoDB** single-table | predictable latency at scale, TTL for expiry, no join needed |
| Latest window (live dashboard) | point read by key, sub-ms, disposable | **Redis** | it's a cache; recomputable from Kafka |
| Raw events (cold) | rare, analytical, full scans | **S3 NDJSON**, `tenant=/dt=/hour=` | cheapest durable bytes; Athena/Spark when needed |
| Alert history | heterogeneous shape per rule type, append-heavy, query by tenant+time | **Mongo** (optional profile) or **Postgres JSONB** | REQ-DL-08 compares them — see below |

**DynamoDB single-table design, the parts interviewers probe:**

```
PK = TENANT#sportsnow#METRIC#rebuffer_ratio     ← partition key: determines the physical partition
SK = WINDOW#2026-09-28T14:30:00Z                ← sort key: enables range queries within a partition

Query(PK = "TENANT#sportsnow#METRIC#rebuffer_ratio",
      SK between "WINDOW#2026-09-28T00:00" and "WINDOW#2026-09-28T23:59")
      → one partition, one sequential read, no scan, cost ∝ bytes returned
```

- **GSI** (global secondary index) has its **own** partition key and sort key, its own capacity, and
  is **eventually consistent** — you cannot do a strongly consistent read on a GSI. StreamPulse uses
  one for "by title across tenants". **LSI** shares the table's partition key, must be created with
  the table, and brings a 10 GB per-partition-key limit; I avoid LSIs for that reason.
- **Hot partitions**: a partition key with skewed traffic (e.g. `PK = tenantId` when `streamflix` is
  40% of traffic) concentrates load on one physical partition. Adaptive capacity absorbs some of this
  now, but the design fix is to widen the key — include the metric, or add a write-shard suffix
  `#0..#9` and fan the read across shards. This is the same reasoning as choosing a Kafka partition
  key, and saying so connects the two halves of the system.
- **Conditional writes for idempotency** — the single most useful DynamoDB feature for an event
  pipeline: `ConditionExpression = "attribute_not_exists(PK)"` makes the write succeed exactly once;
  a duplicate delivery gets `ConditionalCheckFailedException`, which the consumer treats as success.
  That is how at-least-once delivery becomes effectively-once **without** any distributed transaction.
- **Capacity modes**: on-demand (pay per request, instant scaling, more expensive per request) vs
  provisioned + auto-scaling (cheaper at steady state, but auto-scaling reacts in minutes, so a live
  sports spike throttles first). For a spiky telemetry workload I'd start on-demand, measure the
  steady-state floor, then consider provisioned with a healthy baseline for the predictable part.
- Hard limits worth knowing: 400 KB per item, 1 MB per Query/Scan page, `BatchWriteItem` 25 items.

**MongoDB vs Postgres JSONB (REQ-DL-08):**

| | MongoDB | Postgres JSONB |
|---|---|---|
| Schema flexibility | native; per-document shape | native; `jsonb` column beside typed columns |
| Indexing inside the document | rich (compound, multikey, wildcard, partial) | GIN (`jsonb_path_ops` for containment), or a B-tree on an **expression**/generated column |
| Joins with relational data | `$lookup`, limited | ordinary SQL joins — the whole point |
| Transactions | multi-document since 4.0, with caveats | full ACID, same transaction as your relational writes |
| Operational surface | another database to run, back up, secure, monitor | none — it's already there |

My default is **JSONB until proven otherwise**: the alert history has to join tenants and rules
anyway, and "one fewer datastore" is a real operational saving. The honest case for Mongo is a team
that already runs it, or document sizes/aggregation-pipeline needs that JSONB handles awkwardly. In
StreamPulse Mongo exists behind an optional profile specifically so the ADR can compare them with
measurements rather than opinions.

**Postgres vs MySQL, the short version:**

| | Postgres | MySQL (InnoDB) |
|---|---|---|
| Default isolation | READ COMMITTED | REPEATABLE READ |
| MVCC implementation | row versions in the heap; `VACUUM` reclaims | undo log; purge thread; clustered PK index |
| Phantom prevention in RR | snapshot isolation (no gap locks) | next-key/gap locks — more locking, different deadlocks |
| JSON | `jsonb` (binary, indexable with GIN) | `JSON` (needs generated columns for real indexing) |
| `SKIP LOCKED` | 9.5+ | 8.0+ |
| Extensibility | extensions (PostGIS, `pg_stat_statements`, TimescaleDB) | fewer |
| Clustered storage | heap + separate indexes (index-only scans need the visibility map) | PK-clustered — secondary index lookups hit the PK index |

**Follow-ups they'll ask:**
- *"Why not put everything in Postgres?"* — for this volume you could go a long way, especially with
  partitioning or TimescaleDB, and I'd say so rather than cargo-culting NoSQL. DynamoDB earns its
  place here because the aggregate store is write-heavy, append-only, keyed, and needs TTL expiry —
  and because operationally it is someone else's problem at 3 a.m.
- *"How do you keep two stores consistent?"* — you don't, in the ACID sense. One store is the source
  of truth (Postgres for config, Kafka for events) and every other store is a **derived projection**
  that can be rebuilt by replaying the log. That is the answer, and it is file 07.
- *"What about a `Scan`?"* — a Scan means your key design failed for that access pattern. Add a GSI,
  or accept that this query belongs in S3/Athena, not DynamoDB.

**Pitfalls / red flags:** choosing by fashion ("NoSQL scales"), or by data shape alone. The access
pattern and its latency budget decide.

**Where in StreamPulse:** BS-05 / REQ-DL-08 + Phase 4 (`PK=TENANT#t#METRIC#m`, `SK=WINDOW#ts`,
conditional writes); the ADR "Postgres vs DynamoDB vs Mongo for time-series aggregates".

**Verify:** DynamoDB limits (400 KB item, 1 MB page, 25-item batch) and the current adaptive-capacity
behaviour against the AWS docs — these have changed over time.

---

## Q14. Walk me through StreamPulse's data layer end to end. What did you choose and why?

*(This is the applied question — the one that turns the previous thirteen answers into evidence.)*

**30-second answer:** The control plane is a deliberate split. **Writes** — tenants, apps, titles,
alert rules — are Spring Data JPA over Postgres with Flyway migrations and `@Version` optimistic
locking, because that side is small, relational and needs transactions. **Reads** that are reports —
the monthly billing aggregate over ten million `usage_hourly` rows — are `JdbcClient` with hand-written
SQL and window functions, because there are no entities in a `GROUP BY`. The ops console tree is an
N+1 lab: I kept the broken version as a disabled test and fixed the real one with `@BatchSize`, with a
Hibernate-`Statistics` assertion that the statement count stays ≤ 3 regardless of tenant count. Audit
uses `REQUIRES_NEW` so a rejected rule change is still recorded — and I sized the pool knowing that
costs a second connection. Quota allocation uses `SELECT … FOR UPDATE` at READ COMMITTED rather than
SERIALIZABLE, because I'd rather block for 2 ms than write retry logic into every admin action. And
every rule change writes an outbox row **in the same transaction**, which is what makes the Debezium
CDC path correct rather than best-effort.

**The decisions, and what each one bought:**

| Decision | Alternative rejected | Why |
|---|---|---|
| JPA for writes, `JdbcClient` for reports | all-JPA | a `GROUP BY` over 10M rows has no entities; the plan is the thing being tuned (REQ-DL-02) |
| `@BatchSize(50)` for the tenant tree | `join fetch` | fetch join breaks pagination and a 3-level fetch is a 200k-row cartesian product (REQ-DL-03) |
| `REQUIRES_NEW` for audit | outbox / `AFTER_COMMIT` event | simplest thing that satisfies "audit the rejected change"; cost is one extra connection, budgeted in the pool (REQ-DL-04) |
| `FOR UPDATE` at READ COMMITTED | SERIALIZABLE + retry | contention is on one row; blocking is cheaper and more predictable than teaching every caller to retry 40001 (REQ-DL-05) |
| Outbox row in the business transaction | publish after commit | after-commit publishing loses the event if the process dies in the gap — this is the dual-write bug (REQ-DL-06, file 07) |
| Postgres + DynamoDB + Redis + S3 | one store | four access patterns with four latency budgets; Kafka is the source of truth so every projection is rebuildable (REQ-DL-08) |
| `open-in-view: false`, DTOs at the boundary | default Boot config | makes lazy-loading mistakes fail in tests, and stops a connection being held through JSON serialization |

**The tests that make it evidence, not a claim:**

| Test | What it proves |
|---|---|
| `TenantTreeQueryCountTest` | statement count does not grow with tenant count (REQ-DL-03) |
| `BillingReportIT` | < 2s over 10M seeded rows; `EXPLAIN ANALYZE` captured in the ADR (REQ-DL-02) |
| `AuditRequiresNewTest` / `SelfInvocationGotchaTest` | the audit row survives the rollback — and doesn't, if you self-invoke (REQ-DL-04) |
| `QuotaAllocationConcurrencyIT` | N concurrent allocations never over-draw; total conserved (REQ-DL-05) |
| `OutboxAtomicityIT` | rollback leaves neither entity nor outbox row (REQ-DL-06) |
| HikariCP lab | pool exhaustion under virtual threads, and the semaphore that fixes it (REQ-DL-07) |

**Follow-ups they'll ask:**
- *"What would you change at 10× scale?"* — partition `usage_hourly` by month (declarative
  partitioning, so old partitions detach instead of `DELETE`), move the billing read to a replica, and
  pre-aggregate daily rollups so the monthly report reads thousands of rows instead of millions. The
  admin write model wouldn't change at all — it doesn't scale with traffic, it scales with tenants.
- *"What's the weakest part?"* — `REQUIRES_NEW` for audit. It works, it is simple, and it couples the
  audit's availability to the same pool. If audit volume grew I'd move it to the outbox and let CDC
  carry it, which also makes it replayable.
- *"How do you know the pool size is right?"* — I don't, from the formula. I know from
  `hikaricp.connections.pending` and the acquire timer under Gatling load, and both are on the Grafana
  dashboard.

> **Your story:** if you have tuned a connection pool, killed an N+1, run a zero-downtime migration or
> chosen between Postgres and a document store in real work — at Couchbase, JustAnswer or NetApp — put
> it here in this shape: what was slow or wrong, how you *measured* it, what you changed, what the
> number became. Keep "I built" for what you personally wrote. StreamPulse is a portfolio project;
> present it as one: *"I rebuilt this in Java to prove the pattern end to end."*

**Where in StreamPulse:** BS-05 end to end; REQ-DL-01…08; Phase 6.

---

## Key terms

| Term | One-line meaning |
|---|---|
| **Persistence context** | The unit-of-work identity map + loaded-state snapshots; also the L1 cache. |
| **Dirty checking** | Comparing an entity's current state to its loaded snapshot at flush to generate the `UPDATE`. |
| **Flush** | Writing queued SQL to the connection; happens at commit, before overlapping queries, or on demand. Not a commit. |
| **`LazyInitializationException`** | Touching an uninitialised proxy after the persistence context closed. |
| **N+1** | One query for the parents, one per parent for the children; a latency problem, not a load problem. |
| **`MultipleBagFetchException`** | Hibernate refusing to fetch-join two `List` collections at once (undeduplicable cartesian product). |
| **`@BatchSize`** | Load lazy associations in `IN (…)` chunks — the pagination-safe N+1 fix. |
| **`JdbcClient`** | Spring's fluent JDBC façade (6.1+); same `DataSource` and transaction as JPA. |
| **`REQUIRES_NEW`** | Suspend the caller's transaction and take a **second connection** for an independent one. |
| **`NESTED`** | One connection, a JDBC savepoint; inner rollback survives, outer rollback does not. |
| **MVCC** | Row versions with `xmin`/`xmax` and per-snapshot visibility; readers never block writers. `VACUUM` reclaims. |
| **Snapshot isolation** | Postgres REPEATABLE READ: one snapshot per transaction, so no phantoms; conflicts abort with 40001. |
| **SSI** | Postgres SERIALIZABLE: dependency tracking that aborts one transaction of any unserialisable cycle. |
| **`@Version`** | Optimistic locking column; the `UPDATE … AND version = ?` that returns 0 rows on conflict. |
| **`FOR UPDATE SKIP LOCKED`** | Row-level claim that skips rows others hold — how you build a queue on a table. |
| **`leakDetectionThreshold`** | HikariCP setting that logs the stack trace of whoever held a connection too long. |
| **Expand / contract** | Additive schema change, backfill, switch, then remove — the zero-downtime migration shape. |
| **Single-table design** | One DynamoDB table with composite `PK`/`SK` encoding several entity types and access patterns. |
| **Conditional write** | A DynamoDB write guarded by a `ConditionExpression`; how at-least-once becomes effectively-once. |

---

## Self-check (answer out loud, no notes)

1. An entity is managed and you change a field but never call `save()`. What SQL runs, when, and how
   did Hibernate know what changed?
2. You have a 3-level tree and a paginated list. Why is `JOIN FETCH` the wrong fix, and what is the
   right one?
3. Exactly how many connections does one request hold when a `REQUIRED` method calls a `REQUIRES_NEW`
   method — and what does that do to `maximumPoolSize`?
4. Postgres REPEATABLE READ: which anomaly does it prevent that the SQL standard says it needn't, and
   what new failure mode does it introduce?
5. Rename a column on a 200-million-row table with zero downtime. List the releases.
