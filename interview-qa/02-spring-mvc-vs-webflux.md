# 02 — Spring MVC vs WebFlux: request flow, Reactor, backpressure, SSE, virtual threads

**Why this matters for an Event Hub role:** an event hub is two systems glued together — a
*write* side that must absorb bursts without falling over, and a *read* side that must push
freshly-computed numbers to hundreds of open browser tabs and keep pushing them for hours. The write
side is a classic Spring MVC problem (filters, validation, error contracts, thread pools). The read
side is a streaming problem (one connection per viewer, held open, fed from a Kafka listener). The JD
says *"Spring MVC"* explicitly and the prep doc flags it as an asked topic, so you need the servlet
machinery cold; and the moment you say "SSE", you will be asked *"why not WebFlux everywhere?"* and
*"why not virtual threads instead?"* Those two questions are where seniority shows, because the
honest answer is **"it depends, and here is the measurement I ran."**

**Scenario anchor — BS-02 (Title Command Center) + Phase 5 (query-api):** one page per title during a
live event, five panels fetched in parallel (REQ-CC-01) with per-panel timeouts (REQ-CC-02), plus a
live SSE feed `GET /v1/stream/metrics` fed by a Kafka listener through a `Sinks.Many` multicast, with
`onBackpressureLatest`, a 15s heartbeat and `Last-Event-ID` resume from a Redis ring buffer. The
ingest side (`ingest-gateway`, Phase 2) stays Spring MVC **on virtual threads** — deliberately, as a
contrast, with a Gatling benchmark ADR comparing the two.

---

## Q1. Walk me through what happens between a request hitting Tomcat and your `@RestController` method running.

**30-second answer:** The connector accepts the socket and hands the request to a Tomcat worker
thread. It runs the **servlet filter chain** — that is container-level, so Spring Security, CORS,
request logging and MDC setup all live here. The chain ends at `DispatcherServlet`, which is the
front controller. It asks each `HandlerMapping` (usually `RequestMappingHandlerMapping`) which
handler matches the path/method/headers, gets back a `HandlerExecutionChain` — handler plus
interceptors — runs `preHandle` on each interceptor, then finds a `HandlerAdapter` to invoke it.
The adapter resolves every method argument through `HandlerMethodArgumentResolver`s (that is where
`@RequestBody` Jackson deserialization and `@Valid` run), calls my method, and passes the return
value to a `HandlerMethodReturnValueHandler` — for `@ResponseBody` that is
`RequestResponseBodyMethodProcessor`, which does content negotiation and picks an
`HttpMessageConverter`. Exceptions go to `HandlerExceptionResolver`s, and
`ExceptionHandlerExceptionResolver` is the one that finds my `@ControllerAdvice`. Then
`postHandle`, render, `afterCompletion`, and the thread goes back to the pool.

**Deeper:**

```
socket ──▶ Tomcat acceptor ──▶ worker thread (exec-N) from the connector pool
   │
   ▼
FilterChain                       ← servlet spec; sees EVERY request, even 404s and static resources
   ├─ CharacterEncodingFilter
   ├─ SecurityFilterChain         (Spring Security is ONE filter delegating to an inner chain)
   ├─ RequestContextFilter        ← binds RequestContextHolder (this is what makes @RequestScope work)
   └─ …your OncePerRequestFilter  (traceId → MDC)
   │
   ▼
DispatcherServlet.doDispatch()
   1. checkMultipart
   2. getHandler(request)          → HandlerMapping[]   → HandlerExecutionChain(handler, interceptors[])
   3. getHandlerAdapter(handler)   → RequestMappingHandlerAdapter
   4. interceptor.preHandle()      (in order; if one returns false, dispatch stops here)
   5. adapter.handle()
        ├─ HandlerMethodArgumentResolver[]  @RequestBody → HttpMessageConverter → Jackson
        │                                   @Valid → LocalValidatorFactoryBean
        ├─ YOUR CONTROLLER METHOD
        └─ HandlerMethodReturnValueHandler[] → content negotiation → converter → response body
   6. interceptor.postHandle()     (skipped if the handler threw)
   7. processDispatchResult()      → HandlerExceptionResolver[] if there was an exception
   8. interceptor.afterCompletion() (ALWAYS runs, like a finally block)
```

Two things worth saying unprompted because they are the usual follow-ups:

1. **`HandlerMapping` is plural and ordered.** `RequestMappingHandlerMapping` (annotations),
   `RouterFunctionMapping` (functional endpoints), `SimpleUrlHandlerMapping`,
   `ResourceHandlerMapping` (static files). First match wins, and "no mapping" is what produces the
   404 path.
2. **Error handling has two completely different layers.** An exception thrown *inside* the dispatch
   is handled by `HandlerExceptionResolver` → your `@ControllerAdvice`. An exception thrown in a
   **filter**, before `DispatcherServlet`, is not — it propagates to the container, which does an
   `ERROR` dispatch to `/error` (`BasicErrorController` in Boot). That is exactly why a Spring
   Security 401 does not hit your `@ExceptionHandler`: it is produced in the filter chain, and you
   customise it with an `AuthenticationEntryPoint` instead. Getting this right is what makes the
   error envelope consistent across the whole API (Q3).

**Code:** the ingest-gateway filter that puts `traceId` and `tenantId` on every log line.

```java
/**
 * StreamPulse ingest-gateway (Phase 2).
 * OncePerRequestFilter, not Filter: guarantees one execution per request even across
 * FORWARD/INCLUDE/ASYNC dispatches — with async request processing, a plain Filter's
 * doFilter can be re-entered on the ERROR/ASYNC dispatch and you get duplicate log context.
 */
@Component
@Order(Ordered.HIGHEST_PRECEDENCE + 20)   // after tracing instrumentation, before security
public class RequestContextLoggingFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest req,
                                    HttpServletResponse res,
                                    FilterChain chain) throws ServletException, IOException {
        String traceId = Optional.ofNullable(req.getHeader("traceparent"))
                                 .map(RequestContextLoggingFilter::extractTraceId)
                                 .orElseGet(() -> UUID.randomUUID().toString());
        try (var ignored1 = MDC.putCloseable("traceId", traceId)) {
            chain.doFilter(req, res);
        }
        // MDC is thread-bound. With ASYNC dispatch or @Async the value does NOT follow the
        // work to another thread — see file 03 Q9 and the context-propagation note in Q11.
    }
}
```

**Follow-ups they'll ask:**
- *"Where does `@RequestScope` get its request from?"* — `RequestContextHolder`, a `ThreadLocal`
  populated by `RequestContextFilter` / `DispatcherServlet`. Which is also why a request-scoped bean
  blows up on a `CompletableFuture` thread (file 01 Q3).
- *"How does content negotiation decide JSON vs XML?"* — `ContentNegotiationStrategy`: `Accept`
  header by default in Boot; path extension and `?format=` are off by default in recent versions.
  Then `HttpMessageConverter.canWrite(type, mediaType)` picks the converter.
- *"What is `HandlerExecutionChain`?"* — the handler plus the interceptors that matched its path;
  `DispatcherServlet` uses it to run `preHandle`/`postHandle`/`afterCompletion` in the right order
  (`afterCompletion` runs in reverse order of `preHandle`, and only for interceptors whose
  `preHandle` succeeded).

**Pitfalls / red flags:** "the DispatcherServlet routes to the controller" and nothing else. Name
`HandlerMapping`, `HandlerAdapter`, argument resolvers, return-value handlers, and the
filter-vs-dispatch split for exceptions.

**Where in StreamPulse:** `ingest-gateway` (Phase 2) and `control-plane` (Phase 6) are both MVC;
`RequestContextLoggingFilter` lives in `libs/platform` so every service gets it (REQ-LC-07).

**Verify:** default content-negotiation behaviour (path extension / `favorParameter`) changed across
Spring versions — confirm for the Boot version you pin.

---

## Q2. Filter vs `HandlerInterceptor` vs `@ControllerAdvice` — which do you use for what?

**30-second answer:** A **filter** is servlet-level: it sees every request including ones that never
reach Spring, it can wrap the request and response objects, and it is the only one of the three that
can modify the raw byte stream. An **interceptor** is Spring-MVC-level: it runs inside
`DispatcherServlet`, so it knows the resolved `HandlerMethod` — which means it can read annotations on
the controller method and make decisions per endpoint. `@ControllerAdvice` is not a pipeline stage at
all: it is a way to contribute `@ExceptionHandler`, `@InitBinder` and `@ModelAttribute` methods across
controllers. My rule: cross-cutting infrastructure that must apply to *everything* → filter;
behaviour that depends on *which handler was selected* → interceptor; error-to-response mapping →
`@ControllerAdvice`.

**Deeper:**

| | Filter | `HandlerInterceptor` | `@ControllerAdvice` |
|---|---|---|---|
| Layer | Servlet container | Spring MVC | Spring MVC |
| Registered via | `@Component` + `@Order`, or `FilterRegistrationBean` | `WebMvcConfigurer.addInterceptors` | `@ControllerAdvice` bean |
| Sees requests that 404 / are rejected by security | **yes** | no | no |
| Knows the target `HandlerMethod` (and its annotations) | no | **yes** | n/a |
| Can wrap request/response, modify the body stream | **yes** | no | no |
| Can short-circuit | yes (don't call `chain.doFilter`) | yes (`preHandle` → `false`) | n/a |
| Runs on exceptions from filters | yes (if it's upstream) | no | **no** |
| Ordering | `@Order` / registration order | registration order | `@Order` between advices |
| WebFlux equivalent | `WebFilter` | *none* — use `WebFilter` or `HandlerFilterFunction` | same annotation works |

The WebFlux row matters for StreamPulse: `query-api` is WebFlux, so any interceptor logic from the MVC
services has to be re-expressed as a `WebFilter`. That is a genuine cost of running two stacks and a
good thing to volunteer when they ask "why not WebFlux everywhere?"

**Ordering trap worth naming:** interceptor ordering is *registration order* for `preHandle` and
**reverse** for `afterCompletion`. Filter ordering with `@Order` is lowest-value-first. Mixing
`@Order` on a `@Component` filter with a `FilterRegistrationBean` that also sets an order is a classic
source of "my filter runs after security and the tenant is null".

**Code:** the same requirement — per-tenant quota — at two layers, and why the interceptor wins.

```java
/**
 * Interceptor, not filter: the decision depends on an annotation on the handler method
 * (@QuotaCost(units = 5)), which only exists once the handler has been resolved.
 * A filter would have to re-derive the route by string matching — brittle.
 */
@Component
class QuotaInterceptor implements HandlerInterceptor {

    private final DistributedQuota quota;    // Bucket4j + Redis, REQ-RS-04

    @Override
    public boolean preHandle(HttpServletRequest req, HttpServletResponse res, Object handler) {
        if (!(handler instanceof HandlerMethod hm)) return true;   // static resources etc.

        QuotaCost cost = hm.getMethodAnnotation(QuotaCost.class);
        if (cost == null) return true;

        String tenant = TenantContextHolder.current();
        if (!quota.tryConsume(tenant, cost.units())) {
            // Don't hand-write JSON here; throw and let @ControllerAdvice own the error shape (Q3).
            throw new QuotaExceededException(tenant, quota.retryAfter(tenant));
        }
        return true;
    }
}

@Configuration
class WebConfig implements WebMvcConfigurer {
    @Override public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(quotaInterceptor).addPathPatterns("/v1/**");
    }
}
```

**Follow-ups they'll ask:**
- *"Why throw instead of writing the 429 directly in `preHandle`?"* — because an exception goes
  through `ExceptionHandlerExceptionResolver` and gets the same `ProblemDetail` envelope as every
  other error. Writing to the response by hand gives you two error formats in one API.
- *"When would you still use a filter for this?"* — if the quota must also cover requests that never
  reach a handler (unmapped paths, or you want to bill 404s), or if you need it in front of Spring
  Security.
- *"How do you make `@ControllerAdvice` apply to only some controllers?"* —
  `@ControllerAdvice(basePackages = ...)`, `assignableTypes = ...`, or `annotations = ...`.
- *"Does an interceptor see the request body?"* — not without consuming it. The body stream is
  single-read; if you need it, wrap the request in a `ContentCachingRequestWrapper` in a **filter**.
  That is a real reason to use a filter.

**Pitfalls / red flags:** doing authentication in an interceptor (too late — it runs after the
security filter chain has already decided), or reading `request.getInputStream()` in a filter and then
being surprised the controller gets an empty body.

**Where in StreamPulse:** `ingest-gateway` — security filter chain (Keycloak JWT), tenant MDC filter,
quota interceptor, platform `@RestControllerAdvice`. BS-04 / REQ-RS-04.

---

## Q3. How do you design the error contract for a public API?

**30-second answer:** RFC 7807 `application/problem+json`, which Spring 6 supports first-class via
the `ProblemDetail` class. Every error — validation, quota, not-found, upstream failure — comes out of
one `@RestControllerAdvice` with the same shape: `type`, `title`, `status`, `detail`, `instance`, plus
custom properties. I make `type` a stable URI that identifies the error *class*, because that is the
field clients are supposed to branch on; `detail` is human prose and I treat it as unstable. I never
leak stack traces or internal identifiers, and I always attach the `traceId` so support can jump
straight to the trace. The part people forget is that errors produced in the **filter chain** (401 from
security, 413 payload-too-large from the container) bypass `@ControllerAdvice`, so I wire those
separately or they become the one inconsistent response in the API.

**Deeper:**

```
{
  "type":     "https://errors.streampulse.dev/quota-exceeded",   ← stable, documented, branch on this
  "title":    "Ingest quota exceeded",                           ← short, stable
  "status":   429,
  "detail":   "Tenant sportsnow consumed 50000 of 50000 events in the current window",
  "instance": "/v1/events:batch",                                ← the URI that failed
  "traceId":  "4bf92f3577b34da6a3ce929d0e0e4736",                ← custom property
  "retryAfterSeconds": 12
}
```

Spring 6 gives you three building blocks and they are worth naming precisely:

| Piece | What it is |
|---|---|
| `ProblemDetail` | A plain value holder for the RFC 7807 fields plus a `Map` of extensions. |
| `ErrorResponse` | An interface: "I can render myself as a `ProblemDetail` with a status and headers." |
| `ErrorResponseException` | A ready-made `RuntimeException` implementing `ErrorResponse`, so you can throw a fully-formed problem. |
| `ResponseEntityExceptionHandler` | The base class that already maps Spring's own exceptions (`MethodArgumentNotValidException`, `HttpMessageNotReadableException`, …) to `ProblemDetail`. Extend it rather than re-implementing those. |

**Code:**

```java
/**
 * libs/platform — one advice, inherited by every StreamPulse service (REQ-LC-07).
 * Extending ResponseEntityExceptionHandler means Spring's own 20-odd exceptions
 * already produce ProblemDetail; we only add our domain ones.
 */
@RestControllerAdvice
public class PlatformExceptionHandler extends ResponseEntityExceptionHandler {

    private static final URI BASE = URI.create("https://errors.streampulse.dev/");

    @ExceptionHandler(QuotaExceededException.class)
    ProblemDetail onQuota(QuotaExceededException ex) {
        var pd = ProblemDetail.forStatusAndDetail(HttpStatus.TOO_MANY_REQUESTS, ex.getMessage());
        pd.setType(BASE.resolve("quota-exceeded"));
        pd.setTitle("Ingest quota exceeded");
        pd.setProperty("retryAfterSeconds", ex.retryAfter().toSeconds());
        pd.setProperty("traceId", MDC.get("traceId"));
        return pd;
    }

    /**
     * Hook provided by the base class: runs for EVERY exception it handles, so the
     * traceId is attached once instead of in 20 handlers. Signature is version-sensitive
     * — see Verify.
     */
    @Override
    protected ProblemDetail createProblemDetail(Exception ex, HttpStatusCode status,
                                                String defaultDetail, String detailMessageCode,
                                                Object[] detailMessageArguments, WebRequest request) {
        ProblemDetail pd = super.createProblemDetail(ex, status, defaultDetail,
                                                     detailMessageCode, detailMessageArguments, request);
        pd.setProperty("traceId", MDC.get("traceId"));
        return pd;
    }
}
```

And the bit that is usually missing — the security errors:

```java
@Bean
SecurityFilterChain api(HttpSecurity http, HandlerExceptionResolver resolver) throws Exception {
    return http
        .oauth2ResourceServer(o -> o.jwt(Customizer.withDefaults())
            // Route the filter-chain 401 back through the SAME advice, so the body shape matches.
            .authenticationEntryPoint((req, res, ex) -> resolver.resolveException(req, res, null, ex)))
        .build();
}
```

**Follow-ups they'll ask:**
- *"Do you need `@RestControllerAdvice`, or is there a switch?"* — Boot can turn on
  `ProblemDetail` responses for Spring's built-in exceptions with a property
  (`spring.mvc.problemdetails.enabled`, and a WebFlux twin). You still need an advice for your own
  exceptions. **Verify the exact property names for your Boot version.**
- *"How do you version the error contract?"* — the `type` URI is the contract. Adding properties is
  backward-compatible; changing a `type` is a breaking change and needs a new URI.
- *"How do you stop this leaking internals?"* — never put the exception message of an *unexpected*
  exception into `detail`. Map anything unhandled to a generic 500 with the `traceId`, and log the
  real message server-side.
- *"Same shape in WebFlux?"* — yes, `ProblemDetail` is in `spring-web`, and WebFlux has its own
  `ResponseEntityExceptionHandler` in the reactive package. The unhandled-exception path differs
  (`WebExceptionHandler` / `ErrorWebExceptionHandler` instead of `BasicErrorController`).

**Pitfalls / red flags:** a bespoke `{"success": false, "error": "..."}` envelope invented per
service, or a 200 response with an error body inside it.

**Where in StreamPulse:** `libs/platform` error model (REQ-LC-07), REQ-DL-01 (409 on optimistic-lock
conflict), REQ-RS-03 (429 + `Retry-After`), Phase 1 (the OpenAPI spec documents each `type` URI).

**Verify:** the `createProblemDetail(...)` signature and the `problemdetails.enabled` property keys
differ across Spring 6.x / Boot 3.x minor versions — check the version you pin.

---

## Q4. How does validation work in Spring, and where do you draw the line between validation and business rules?

**30-second answer:** Bean Validation (Jakarta) via `@Valid` on the `@RequestBody`, which the argument
resolver runs before my method body executes; a failure becomes
`MethodArgumentNotValidException` and my advice turns it into a `ProblemDetail` with a per-field
`errors` array. `@Validated` is the Spring annotation and does two extra things `@Valid` cannot:
validation **groups**, and method-level validation on `@RequestParam`/`@PathVariable` when placed on
the class. The line I draw: Bean Validation is for **structural, context-free** rules — "not blank",
"between 1 and 500", "matches this pattern". Anything that needs a lookup, another entity, or a policy
decision is a domain rule and lives in the domain service, because a constraint validator that opens
a database connection is a repository in disguise and it is untestable and unorderable.

**Deeper:**

The three exception types and where each comes from is the follow-up that trips people:

| Where the constraint is | Exception | Typical status |
|---|---|---|
| `@Valid @RequestBody Dto` | `MethodArgumentNotValidException` (a `BindException`) | 400 |
| `@Validated` on the **class**, constraint on a `@RequestParam`/`@PathVariable` | Spring 6.1+: `HandlerMethodValidationException`; earlier: `ConstraintViolationException` | 400 |
| Inside a `@Validated` service bean (method validation via AOP) | `ConstraintViolationException` | 500 unless mapped |
| WebFlux `@Valid @RequestBody` | `WebExchangeBindException` | 400 |

Nested and collection validation needs `@Valid` **on the field/element**, not just the top level —
`List<@Valid EventDto>` or `@Valid List<EventDto>` depending on what you mean. Missing that is the
most common "why didn't my validation fire".

**Code:** the ingest batch DTO (REQ-INGEST: ≤500 events per request).

```java
// Java 21 record + Jakarta validation. Immutable, no getters to write, no Lombok.
public record IngestBatch(
        @NotBlank @Size(max = 64) String tenantId,
        @NotEmpty @Size(max = 500)                        // REQ: batch cap, enforced structurally
        List<@Valid @NotNull TelemetryEventDto> events    // @Valid on the ELEMENT — cascades
) {}

public record TelemetryEventDto(
        @NotBlank String eventId,                          // idempotency key (Phase 2)
        @NotBlank String sessionId,                        // Kafka partition key
        @NotNull EventType type,
        @NotNull @PastOrPresent Instant ts,                // no events from the future
        @Valid Payload payload
) {}
```

```java
@RestController
@RequestMapping("/v1/events")
class IngestController {

    private final IngestUseCase ingest;      // domain port — no Spring types inside (file 01 Q10)

    @PostMapping(value = ":batch", consumes = APPLICATION_JSON_VALUE)
    ResponseEntity<BatchAccepted> ingest(
            @Valid @RequestBody IngestBatch batch,                       // structural validation
            @RequestHeader(value = "Idempotency-Key", required = false) String idemKey) {

        // Business rules live HERE, not in a ConstraintValidator:
        //   - is this tenant active?         (needs a lookup)
        //   - is titleId owned by tenant?    (needs a cross-entity check)
        //   - is the tenant over quota?      (needs distributed state)
        return ResponseEntity.accepted().body(ingest.accept(batch.toCommand(idemKey)));
    }
}
```

Field-level error rendering, once, in the platform advice:

```java
@Override
protected ResponseEntity<Object> handleMethodArgumentNotValid(
        MethodArgumentNotValidException ex, HttpHeaders headers,
        HttpStatusCode status, WebRequest request) {

    ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
    pd.setType(URI.create("https://errors.streampulse.dev/validation-failed"));
    pd.setTitle("Request validation failed");
    pd.setProperty("errors", ex.getBindingResult().getFieldErrors().stream()
            .map(fe -> Map.of("field", fe.getField(), "message", fe.getDefaultMessage()))
            .toList());
    return ResponseEntity.badRequest().body(pd);
}
```

**Follow-ups they'll ask:**
- *"When have you written a custom constraint?"* — a `@ValidTenantId` that checks the format
  (`^[a-z][a-z0-9-]{2,31}$`). Format, not existence. Existence is a lookup.
- *"Validation groups — a real use?"* — create vs update: `id` must be null on create and non-null on
  update. `@Validated(OnUpdate.class)` on the handler, `@NotNull(groups = OnUpdate.class)` on the
  field.
- *"How do you validate a payload whose shape depends on `type`?"* — a Jackson polymorphic type
  (`@JsonTypeInfo`/`@JsonSubTypes`) per event type, each with its own constraints, so validation
  stays declarative. In StreamPulse the Avro union in `libs/contracts` is the source of truth and the
  DTO is generated from it.
- *"Does `@Valid` work on a `Mono<Dto>` in WebFlux?"* — yes for `@RequestBody`; the validation happens
  after the body is decoded, and the failure surfaces as `WebExchangeBindException` in the reactive
  chain, not as a thrown exception on the calling thread. That difference matters for how you test it.

**Pitfalls / red flags:** a `ConstraintValidator` with an `@Autowired` repository; or validation logic
duplicated in the controller *and* the service because nobody decided where the line is.

**Where in StreamPulse:** Phase 2 ingest DTOs; Phase 6 admin CRUD (REQ-DL-01) uses groups for
create/update; `@WebMvcTest` slice tests assert the 400 body shape.

**Verify:** Spring 6.1 changed method-parameter validation to throw `HandlerMethodValidationException`
rather than `ConstraintViolationException` — confirm the behaviour in your version before claiming it.

---

## Q5. What is a `Mono`/`Flux` really, and what is the difference between a cold and a hot publisher?

**30-second answer:** A `Mono`/`Flux` is not a running computation — it is a **recipe**. Building the
chain is *assembly time* and nothing executes; work starts only when something subscribes, which is
*subscription time*. That is why a `Flux` with no subscriber does literally nothing, and why "I called
the method but the HTTP request never went out" is the classic first Reactor bug. **Cold** means each
subscriber triggers its own independent execution from the start — a `WebClient` call is cold, two
subscribers means two HTTP requests. **Hot** means the source is producing regardless of subscribers,
and a late subscriber sees only what arrives after it joins. A Kafka listener pushing into a
`Sinks.Many` is hot, and in StreamPulse that is exactly the right model: a dashboard that connects at
14:05 wants the live numbers, not a replay of the day.

**Deeper:**

```
ASSEMBLY TIME                         SUBSCRIPTION TIME                RUNTIME
Flux.just(1,2,3)                      subscriber.onSubscribe(s)        s.request(n)
   .map(this::enrich)      ──build──▶ chain of Subscribers built   ──▶ onNext × n
   .filter(...)                       BOTTOM-UP (downstream first)      onComplete / onError
   .flatMap(...)
   ↑ nothing has run yet              ↑ this is when subscribeOn picks the thread
```

The signal contract is the whole of Reactive Streams: `onSubscribe` once, then **0..N `onNext`**, then
**at most one** terminal `onComplete` or `onError`. Nothing after a terminal signal. Every operator you
write has to respect that.

| | Cold | Hot |
|---|---|---|
| Starts producing | on each subscribe | independently of subscribers |
| Two subscribers get | two independent executions | the same stream (multicast) |
| Examples | `Flux.just`, `WebClient.retrieve()`, `Mono.fromCallable` | `Sinks.Many.multicast()`, `.share()`, `.publish().refCount()`, `.replay(n)` |
| Late subscriber sees | everything, from the start | only what comes next (or the last *n* with `replay`) |

Turning cold into hot is `share()` (multicast + refCount), `cache()` / `replay(n)` (multicast +
replay), or `publish().autoConnect(k)`. Turning hot into "warm" is `replay(n)` — which is what you'd
use if you wanted new dashboards to immediately see the last window instead of a blank chart.

**Code:**

```java
// COLD: two subscribers → two HTTP calls to control-plane.
Mono<TitleMeta> cold = webClient.get().uri("/titles/{id}", id).retrieve().bodyToMono(TitleMeta.class);
cold.subscribe();   // request #1
cold.subscribe();   // request #2  ← surprise if you expected caching

// WARM: one call, result cached for 60s — this is the poor-man's LRU before we wrote a real one.
Mono<TitleMeta> warm = cold.cache(Duration.ofSeconds(60));

// HOT: the StreamPulse SSE fan-out (Phase 5, REQ-CC-01 read side).
// One Kafka listener feeds N browser subscribers. Nobody re-reads Kafka per browser.
private final Sinks.Many<MetricAggregate> live =
        Sinks.many().multicast().onBackpressureBuffer(256, false);   // see Q8 for the args

@KafkaListener(topics = "telemetry.agg.v1")
void onAggregate(MetricAggregate agg) {
    // tryEmitNext returns a RESULT, it does not throw. Ignoring it is how you lose events silently.
    Sinks.EmitResult r = live.tryEmitNext(agg);
    if (r.isFailure()) {
        meterRegistry.counter("sse.emit.failed", "reason", r.name()).increment();
    }
}
```

**Follow-ups they'll ask:**
- *"What is the difference between `map` and `flatMap`?"* — `map` is a synchronous 1:1 transform;
  `flatMap` takes a function returning a `Publisher`, subscribes to each inner publisher (up to
  `concurrency`, default 256) and **merges** them, so ordering is not preserved. Use `concatMap` when
  order matters, `flatMapSequential` when you want concurrency *and* ordered output.
- *"`doOnNext` vs `map`?"* — `doOnNext` is a side-effect hook that must not change the element; using
  it to mutate is how you get a chain nobody can reason about.
- *"What's the reactive equivalent of a `finally`?"* — `doFinally(signalType -> …)`, which also tells
  you *why* it ended, including `CANCEL`. For SSE that is how you decrement your "connected clients"
  gauge when a browser tab closes.
- *"Why does my `subscribe()` swallow errors?"* — the no-arg `subscribe()` uses a default error
  consumer that just logs via `Operators.onErrorDropped`. Always supply an error consumer, or better,
  return the publisher and let the framework subscribe.

**Pitfalls / red flags:** calling `.block()` inside a reactive chain, or `.subscribe()` inside a
controller method and returning `void` — you've just fired and forgotten with no backpressure, no
error propagation and no cancellation.

**Where in StreamPulse:** `query-api` (Phase 5). The `Sinks.Many` fan-out is REQ-CC-01's read side and
the mechanism behind `GET /v1/stream/metrics`.

---

## Q6. Explain WebFlux's threading model. How can it serve more concurrency with fewer threads, and what breaks it?

**30-second answer:** Netty runs a small fixed pool of **event-loop threads** — by default roughly one
per core — and each one handles many connections by reacting to readiness events rather than blocking
on a socket. Concurrency is bounded by *memory and file descriptors*, not by thread count, so 10,000
idle SSE connections cost 10,000 socket buffers instead of 10,000 platform-thread stacks. The absolute
rule is that **you must never block an event-loop thread**: one `Thread.sleep`, one JDBC call, one
`.block()` and you have stalled every connection pinned to that loop, not just yours. If I genuinely
must call something blocking I move it with `subscribeOn(Schedulers.boundedElastic())` — an elastic
pool sized in the hundreds, explicitly for legacy blocking work — and I keep BlockHound wired into the
test build so a blocking call fails the test instead of production.

**Deeper:**

```
                  ┌──────────────── Netty event loop group (≈ 1 thread per core) ────────────────┐
  10k sockets ───▶│ reactor-http-nio-1  ▸ conn 1, 42, 97 …   (selector: who is readable/writable) │
                  │ reactor-http-nio-2  ▸ conn 2, 43, 98 …                                        │
                  └───────────────────────────────────────────────────────────────────────────────┘
                            │ hands the decoded request into your chain, ON THAT SAME THREAD
                            ▼
                  your Flux chain ── stays on the event loop unless an operator switches it
                            │
                            ├── subscribeOn(boundedElastic())  → the SOURCE subscription moves
                            └── publishOn(parallel())          → everything DOWNSTREAM of here moves
```

`subscribeOn` vs `publishOn` — say it precisely, this is a standard question:

| | Affects | Position in the chain |
|---|---|---|
| `subscribeOn(s)` | where the **subscription** (and therefore the source's work) happens | anywhere; the one closest to the source wins, and it affects everything up to the next `publishOn` |
| `publishOn(s)` | where **downstream** operators and `onNext` run | positional — everything after it moves |

The built-in schedulers:

| Scheduler | Sized | For |
|---|---|---|
| `Schedulers.parallel()` | ≈ number of cores | short CPU-bound work |
| `Schedulers.boundedElastic()` | large cap (documented as a multiple of cores) with a bounded task queue | **blocking** I/O you cannot avoid |
| `Schedulers.immediate()` | — | run on the calling thread |
| `Schedulers.single()` | 1 | serialising work |
| `Schedulers.fromExecutor(e)` | your pool | bridging to an existing executor |

Note the Reactor 3.6-era addition of a *virtual-thread-backed* `boundedElastic` behaviour when the
`reactor.schedulers.defaultBoundedElasticOnVirtualThreads` system property is enabled — worth knowing
exists, but **verify the property name and status in the Reactor version you pin**; it has moved.

**Code:** detecting the mistake instead of hoping.

```java
// build.gradle.kts — test-only agent that throws when a blocking call happens on a non-blocking thread
// testImplementation("io.projectreactor.tools:blockhound-junit-platform:<version>")

// The mistake BlockHound catches:
// ❌ a JDBC lookup inside a WebFlux handler
@GetMapping("/v1/titles/{id}")
Mono<TitleMeta> broken(@PathVariable String id) {
    return Mono.just(jdbcTemplate.queryForObject(SQL, mapper, id));   // BLOCKS the event loop
}

// ✅ if the blocking call is unavoidable, isolate it and bound it
@GetMapping("/v1/titles/{id}")
Mono<TitleMeta> fixed(@PathVariable String id) {
    return Mono.fromCallable(() -> jdbcTemplate.queryForObject(SQL, mapper, id))
               .subscribeOn(Schedulers.boundedElastic())   // moves the blocking work off the loop
               .timeout(Duration.ofMillis(150))            // REQ-CC-02 per-panel budget
               .onErrorResume(TimeoutException.class, e -> Mono.just(TitleMeta.degraded(id)));
}
```

> **Your story:** if you have ever debugged a thread-starvation incident — Node's event loop blocked by
> a sync call, or a Java pool exhausted by a slow downstream — this is where it goes. The shape is the
> same in both runtimes and the interviewer will recognise it. Keep it to what you personally did.

**Follow-ups they'll ask:**
- *"How many event-loop threads, exactly?"* — Reactor Netty defaults to
  `max(availableProcessors(), 4)` as I understand it, overridable with
  `reactor.netty.ioWorkerCount`. **Verify the default and the property name** before quoting a number.
- *"What happens if you block one anyway?"* — every connection assigned to that loop stalls. Under
  load the symptom is bimodal latency: most requests fast, a subset catastrophically slow. That
  signature is the tell.
- *"How is the context (MDC, security, tenant) carried across threads?"* — not by `ThreadLocal`.
  Reactor has the `Context` / `ContextView` attached to the subscription, and Micrometer's
  context-propagation library bridges `ThreadLocal`s into it. This is a real cost of reactive code and
  a fair point in the virtual-threads comparison (Q11).
- *"Can WebFlux run on Tomcat?"* — yes, on any Servlet 3.1+ async container; Netty is only the
  default. The programming model is the same; the threading details differ.

**Pitfalls / red flags:** "WebFlux is faster." It is not faster per request — it is *cheaper per
concurrent connection*. Saying "faster" and being unable to define the axis is a senior-level miss.

**Where in StreamPulse:** `query-api` (Phase 5); BlockHound is in the `test-conventions` Gradle plugin
so every reactive module gets it. Phase 5's benchmark ADR is the evidence.

**Verify:** Reactor Netty's default worker count, the `boundedElastic` cap formula, and the
virtual-thread scheduler property — all version-sensitive.

---

## Q7. What is backpressure, and what do you actually do when a consumer is slower than the producer?

**30-second answer:** Backpressure in Reactive Streams is not a vague concept, it is the
`Subscription.request(n)` call: the subscriber tells the publisher how many elements it is ready for,
and a compliant publisher never sends more. That works when the source can be slowed — a database
cursor, a file, another reactive service. It does **not** work when the source is genuinely hot and
external, like a Kafka listener pushing into a sink or a browser on hotel wifi: you cannot tell a live
event stream to slow down. At that point backpressure becomes a *policy* choice and you must pick one
of four: buffer (bounded — unbounded buffering is just a slower OOM), drop, keep only the latest, or
error the subscriber. For a live metrics dashboard the correct answer is **latest**, because a
dashboard showing a 40-second-old value is worse than one that skipped six frames.

**Deeper:**

| Strategy | Operator | Semantics | When it's right |
|---|---|---|---|
| Buffer | `onBackpressureBuffer(n, onOverflow, strategy)` | queue up to `n`, then apply overflow strategy | short bursts, bounded, you need every element |
| Drop newest | `onBackpressureDrop(consumer)` | discard arriving items the subscriber can't take | lossy telemetry, sampling |
| Latest | `onBackpressureLatest()` | keep only the most recent unconsumed item | **live gauges/dashboards** — REQ-CC-01 read side |
| Error | `onBackpressureError()` | terminate with `MissingBackpressureException` | you want to fail loudly and reconnect |
| Rate-shape | `sample(Duration)`, `limitRate(n)`, `conflate`-style windows | emit at most one per interval | charts that redraw at 4 fps anyway |

`limitRate(n)` is the underrated one: it does not drop anything, it just makes the downstream request
in chunks of `n` with a 75% replenish threshold by default, which stops a `flatMap` from pulling an
unbounded prefetch out of a database.

**Where it silently doesn't apply:** `Flux.interval`, `Sinks` with an unbounded buffer, and anything
you built with `Flux.create(sink, OverflowStrategy.IGNORE)`. In those cases the "backpressure" is
whatever your heap can hold.

**Code:** the actual SSE endpoint shape.

```java
/**
 * StreamPulse query-api, Phase 5.
 * Chain reads bottom-up for backpressure: the browser is the slow subscriber, so
 * everything upstream of onBackpressureLatest is protected from it.
 */
@GetMapping(value = "/v1/stream/metrics", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
Flux<ServerSentEvent<MetricAggregate>> stream(@AuthenticationPrincipal Jwt jwt) {

    String tenant = jwt.getClaimAsString("tenant");

    return sinkRegistry.forTenant(tenant).asFlux()          // hot, multicast (Q8)
            .filter(agg -> agg.tenantId().equals(tenant))   // defence in depth
            .onBackpressureLatest()                         // slow browser → skip frames, never queue
            .sample(Duration.ofMillis(250))                 // the UI redraws 4×/s; sending 50/s is waste
            .map(this::toSse)
            .mergeWith(heartbeat())                         // Q9
            .doOnCancel(() -> connections.decrementAndGet())
            .doOnSubscribe(s -> connections.incrementAndGet());
}
```

**Follow-ups they'll ask:**
- *"Buffer or latest — how do you choose?"* — ask what the consumer does with element *k* once
  element *k+1* exists. If nothing (a gauge), latest. If it accumulates (a ledger, a transcript),
  buffer — and if the buffer can overflow, you need a different transport, because you are describing
  a queue, not a stream.
- *"`onBackpressureBuffer` unbounded — why is that a trap?"* — it converts a latency problem into an
  OOM. It also hides the problem until the heap dump.
- *"Does HTTP give you backpressure for free?"* — TCP flow control gives you *some*: if the browser
  stops reading, the socket buffer fills, Netty stops writing, and the write side of your `Flux`
  applies backpressure upward. That is real, but it is coarse and it is why you still need an explicit
  strategy at the fan-out point.
- *"What is the ingest-side equivalent?"* — a bounded in-flight `Semaphore` in front of the Kafka
  producer that returns **429 + `Retry-After`** when saturated (Phase 2). Same idea, expressed in HTTP
  instead of Reactive Streams: push the problem back to the client rather than queueing it.

**Pitfalls / red flags:** treating backpressure as "add a bigger buffer". The senior answer names the
policy choice and ties it to what the consumer actually needs.

**Where in StreamPulse:** Phase 5 SSE; REQ-CC-04 (`CallerRunsPolicy` is the thread-pool expression of
the same idea — see file 03 Q7); Phase 2's 429 backpressure.

---

## Q8. How do you fan one Kafka listener out to hundreds of SSE subscribers?

**30-second answer:** One `Sinks.Many` per tenant, held in a registry, created lazily and multicast.
The Kafka listener calls `tryEmitNext` once per aggregate; every browser subscribed to that tenant's
`asFlux()` gets it. The important design decisions are: *multicast* (not unicast, because there are
many subscribers, and not replay-all, because a new dashboard does not want the day's history);
a **bounded** `onBackpressureBuffer`; checking the `EmitResult` instead of assuming success; and
cleaning up sinks for tenants with no subscribers so the registry is not a memory leak. And I would
say explicitly that I am *not* using reactor-kafka — a plain Spring Kafka listener feeding a sink is
simpler and keeps the consumer's offset management in ordinary, well-understood code.

**Deeper:**

The `Sinks` API is a small decision tree, and knowing it is the difference between "I've used
Reactor" and "I understand Reactor":

```
Sinks.many()
   ├─ .unicast()    → exactly one subscriber ever; buffers everything before it arrives
   ├─ .multicast()  → N subscribers; by default emits only what arrives AFTER each subscribes
   └─ .replay()     → N subscribers; new ones get history: .all(), .latest(), .limit(n), .limit(duration)
        then: .onBackpressureBuffer(bufferSize, autoCancel)
              .directAllOrNothing() / .directBestEffort()   ← no buffer at all
Sinks.one()  → a Mono-shaped, single-value sink
```

`multicast().onBackpressureBuffer()` has a behaviour people get bitten by: with the default
`autoCancel = true`, when the **last** subscriber leaves, the sink terminates and later subscribers get
nothing. For a long-lived per-tenant fan-out you want `autoCancel = false`, or you re-create the sink
on demand.

`tryEmitNext` returns a `Sinks.EmitResult` — `OK`, `FAIL_OVERFLOW`, `FAIL_NON_SERIALIZED`,
`FAIL_TERMINATED`, `FAIL_ZERO_SUBSCRIBER`, `FAIL_CANCELLED`. `FAIL_NON_SERIALIZED` exists because sinks
require serialized emission (the Reactive Streams spec forbids concurrent `onNext`); if two Kafka
listener threads emit to the same sink concurrently, one gets that result. The fixes are: a single
listener thread per sink, `emitNext(value, EmitFailureHandler.busyLooping(Duration))`, or your own
lock. **Verify `busyLooping`'s availability and signature in your Reactor version.**

**Code:**

```java
/**
 * Phase 5 — per-tenant SSE fan-out registry.
 * One Kafka partition-assignment feeds N browsers. Memory cost is O(tenants), not O(browsers).
 */
@Component
public class TenantSinkRegistry {

    private static final int BUFFER = 256;   // bounded: ~256 aggregates ≈ a few seconds of slack

    // computeIfAbsent is atomic per bin; the mapping function must be cheap and must not
    // touch this same map (see file 03 Q4).
    private final Map<String, Sinks.Many<MetricAggregate>> sinks = new ConcurrentHashMap<>();

    public Sinks.Many<MetricAggregate> forTenant(String tenantId) {
        return sinks.computeIfAbsent(tenantId, t ->
                Sinks.many()
                     .multicast()
                     .onBackpressureBuffer(BUFFER, /* autoCancel */ false));  // survives 0 subscribers
    }

    /** Called from the Spring Kafka listener thread — one thread per partition, so emission is serialized. */
    public void publish(MetricAggregate agg) {
        Sinks.Many<MetricAggregate> sink = sinks.get(agg.tenantId());
        if (sink == null) return;                       // nobody watching this tenant: cheapest path
        Sinks.EmitResult result = sink.tryEmitNext(agg);
        if (result.isFailure()) {
            meters.counter("sse.emit.failed", "tenant", agg.tenantId(),
                           "reason", result.name()).increment();
        }
    }

    /** Scheduled reaper: drop sinks nobody is subscribed to, so a churny tenant list isn't a leak. */
    @Scheduled(fixedDelay = 60_000)
    void reap() {
        sinks.entrySet().removeIf(e -> e.getValue().currentSubscriberCount() == 0);
    }
}
```

**Follow-ups they'll ask:**
- *"What if you have three query-api pods and the browser is on pod 2, but the Kafka partition is
  assigned to pod 1?"* — this is the real design question. Options: (a) every pod consumes **all**
  partitions of `telemetry.agg.v1` using a unique consumer group per pod (fan-out by broadcast — the
  topic is small and aggregated, so this is affordable and it is what StreamPulse does); (b) route
  browsers to the pod that owns the partition (sticky routing — operationally painful); (c) put a
  Redis pub/sub hop in between. I'd pick (a) and justify it with the topic's volume.
- *"Why not reactor-kafka?"* — its maintenance status is unclear to me, and a Spring Kafka listener
  gives me batch listeners, manual acks, `@RetryableTopic` and a rebalance story I can explain. The
  sink is a two-line bridge. I'd rather own the bridge than the consumer.
- *"Ordering guarantees to the browser?"* — per sink, emission is serialized, so order is preserved
  for a given tenant as long as one thread emits. Across tenants there is no ordering and none is
  needed.
- *"What about `Flux.create` instead of `Sinks`?"* — `Flux.create` is per-subscriber (cold), so you'd
  get one bridge per browser. `Sinks.Many` is the multicast primitive; that's the difference.

**Pitfalls / red flags:** ignoring the `EmitResult`, using an unbounded buffer, or leaking one sink per
*connection* instead of per tenant.

**Where in StreamPulse:** Phase 5 `query-api`; the plan explicitly records "Spring Kafka listener →
Reactor `Sinks.Many` for SSE fan-out, no reactor-kafka" as a decision.

**Verify:** `Sinks.EmitFailureHandler.busyLooping(Duration)` availability, and the exact `autoCancel`
default for `multicast().onBackpressureBuffer(...)` in your Reactor version.

---

## Q9. Explain how Server-Sent Events work at the wire level, and what you have to get right in production.

**30-second answer:** SSE is a plain HTTP/1.1 (or HTTP/2) GET whose response never ends:
`Content-Type: text/event-stream`, no `Content-Length`, chunked, and the body is a sequence of
UTF-8 text blocks separated by a blank line, with `data:`, `event:`, `id:` and `retry:` fields. The
browser's `EventSource` reconnects automatically when the connection drops and sends back the last
`id:` it saw as the `Last-Event-ID` request header — so **resume is a protocol feature you just have
to implement the server half of**. The three things that bite you in production are: intermediaries
(nginx, ALBs, corporate proxies) buffering the response, which makes the stream appear dead — fixed
with `X-Accel-Buffering: no` and disabling proxy buffering; idle timeouts killing a quiet stream —
fixed with a periodic heartbeat; and the HTTP/1.1 six-connections-per-origin browser limit, which
HTTP/2 multiplexing removes.

**Deeper:**

The wire format, exactly:

```
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
X-Accel-Buffering: no          ← nginx: do not buffer this response

: heartbeat                    ← a COMMENT (line starting with ':'), ignored by EventSource
                               ← blank line terminates the block

event: metric
id: 1727520000123-7
data: {"titleId":"tt-9001","concurrent":18422,"window":"2026-09-28T10:00:00Z"}

event: metric
id: 1727520000487-7
data: {"titleId":"tt-9001","concurrent":18510,"window":"2026-09-28T10:00:10Z"}

retry: 3000                    ← tells the browser to wait 3s before reconnecting
```

Rules that matter:
- A multi-line payload is multiple `data:` lines; the client joins them with `\n`.
- `id:` is what comes back as `Last-Event-ID`. Make it **meaningful** — in StreamPulse it is
  `epochMillis-partition`, so the server can seek.
- A comment line (`: anything`) is the cheapest heartbeat: it keeps the socket and any idle timers
  alive without the client's `onmessage` firing.
- `EventSource` is **GET only** and cannot set custom headers — so the token goes in a cookie or a
  query parameter, or you use a `fetch`-based polyfill. This is a real design constraint and a good
  thing to raise before they do.

**Reconnect and resume, end to end:**

```
browser                          query-api                         Redis ring buffer
   │  GET /v1/stream/metrics        │                                  │
   │───────────────────────────────▶│  subscribe to tenant sink        │
   │◀─── id: 1727520000487-7 ───────│                                  │
   │        ✂ connection drops       │                                  │
   │  (waits `retry` ms + jitter)   │                                  │
   │  GET  Last-Event-ID: …487-7 ──▶│  ZRANGEBYSCORE (487, +inf) ─────▶│  replay the gap
   │◀─── missed events, then live ──│  then concat with the live sink  │
```

**Code:**

```java
/**
 * Phase 5. Heartbeat + resume. Two details:
 *  - the heartbeat is a COMMENT event, so the browser's onmessage does not fire
 *  - resume concatenates the replayed gap BEFORE the live stream, so ordering holds
 */
@GetMapping(value = "/v1/stream/metrics", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
Flux<ServerSentEvent<MetricAggregate>> stream(
        @AuthenticationPrincipal Jwt jwt,
        @RequestHeader(value = "Last-Event-ID", required = false) String lastEventId) {

    String tenant = jwt.getClaimAsString("tenant");

    Flux<ServerSentEvent<MetricAggregate>> replay =
            (lastEventId == null)
                ? Flux.empty()
                : ringBuffer.since(tenant, lastEventId)      // bounded: last 5 min in a Redis ZSET
                            .map(this::toSse);

    Flux<ServerSentEvent<MetricAggregate>> live =
            sinks.forTenant(tenant).asFlux()
                 .onBackpressureLatest()
                 .map(this::toSse);

    Flux<ServerSentEvent<MetricAggregate>> heartbeat =
            Flux.interval(Duration.ofSeconds(15))
                .map(i -> ServerSentEvent.<MetricAggregate>builder()
                                         .comment("hb")        // ": hb\n\n" on the wire
                                         .build());

    // concat(replay, live) preserves order; mergeWith(heartbeat) interleaves keep-alives.
    return Flux.concat(replay, live)
               .mergeWith(heartbeat)
               .doFinally(sig -> log.debug("SSE closed tenant={} signal={}", tenant, sig));
}

private ServerSentEvent<MetricAggregate> toSse(MetricAggregate a) {
    return ServerSentEvent.<MetricAggregate>builder(a)
            .id(a.windowStart().toEpochMilli() + "-" + a.partition())  // resumable cursor
            .event("metric")
            .retry(Duration.ofSeconds(3))                              // client reconnect delay hint
            .build();
}
```

**Follow-ups they'll ask:**
- *"Why a bounded ring buffer and not full replay?"* — because unbounded resume is a different
  product (that's Kafka). Five minutes covers a wifi blip and a pod rollout; beyond that the client
  does a fresh `GET /v1/metrics?from=…` snapshot and then re-attaches. I'd state that fallback
  explicitly, because "resume from anywhere" is a promise you cannot keep cheaply.
- *"What if the client reconnects to a different pod?"* — it works, because the ring buffer is in
  Redis (shared) and every pod consumes all partitions. Sticky sessions are not required. That is a
  deliberate design choice.
- *"How do you do SSE in Spring MVC, not WebFlux?"* — `SseEmitter` (or `ResponseBodyEmitter`), on top
  of servlet async. It works, but every open stream holds an async context, and with **virtual
  threads** the cost profile changes significantly — which is exactly the Phase 5 benchmark.
- *"What happens on server shutdown?"* — complete the sinks so clients get a clean `onComplete`, and
  let `EventSource` reconnect to the next pod. Wire it into the `SmartLifecycle` drain from BS-01 so
  it happens *before* the Kafka listeners stop.

**Pitfalls / red flags:** no heartbeat (streams die silently behind proxies after 60s), no `id:` (no
resume possible), or trying to send custom auth headers from `EventSource`.

**Where in StreamPulse:** Phase 5 (`GET /v1/stream/metrics`), Phase 7 (`useEventSource` React hook with
jittered backoff), BS-02 read side.

**Verify:** `ServerSentEvent.Builder#comment` and `#retry` exist in Spring's reactive codec package —
confirm the exact builder methods for your Spring version.

---

## Q10. SSE vs WebSocket vs long-polling — how do you choose?

**30-second answer:** SSE when the data flows **one way, server to client**, and the payload is text —
which is exactly a metrics dashboard. It is plain HTTP, so it goes through every proxy, CDN and
corporate firewall, it has automatic reconnect and resume built into the browser, and the server side
is a `Flux`. WebSocket when you need **bidirectional, low-latency** exchange — chat, collaborative
editing, a game — and you are willing to pay for it: a separate protocol after an upgrade handshake,
your own heartbeat/reconnect/resume logic, and load balancers that must be configured for it.
Long-polling only as a fallback for environments where neither works. For StreamPulse the dashboard is
strictly read-only, so SSE is the right call, and I'd add that the browser HTTP/1.1 connection limit
is the one argument *against* it — solved by serving over HTTP/2.

**Deeper:**

| | SSE | WebSocket | Long-polling |
|---|---|---|---|
| Direction | server → client only | full duplex | server → client (simulated) |
| Protocol | HTTP; `text/event-stream` | `ws://`/`wss://` after HTTP Upgrade | HTTP |
| Payload | UTF-8 text only (base64 your binary) | text **or** binary frames | anything |
| Auto reconnect | **built into `EventSource`** | you write it | you write it |
| Resume after gap | `Last-Event-ID`, built in | you design it | you design it |
| Custom request headers | **no** (EventSource limitation) | limited (subprotocol/query) | yes |
| Proxy / CDN friendliness | high (it's just HTTP) | medium — needs Upgrade support | highest |
| Browser connection limit | 6 per origin on HTTP/1.1; **not** on HTTP/2 | ~unlimited | 6 per origin |
| Server cost per client | 1 open response | 1 open socket + protocol state | repeated request churn |
| Spring API | `Flux<ServerSentEvent<T>>` / `SseEmitter` | `WebSocketHandler`, STOMP | `DeferredResult` |
| Latency | ~one TCP RTT for the push | same | +1 poll interval |

**When I'd switch StreamPulse to WebSocket:** if the dashboard became interactive in a way that needs
a low-latency client→server channel — e.g. the operator drags a time window and the server
re-subscribes the stream. Today that is a separate REST call, which is fine, so the extra protocol
buys nothing. Saying *"and here is the trigger that would change my mind"* is the answer they want.

**Follow-ups they'll ask:**
- *"Doesn't HTTP/2 make SSE strictly better?"* — it removes the six-connection limit, yes. It does not
  make it bidirectional. It also introduces head-of-line blocking at the TCP layer that HTTP/3 fixes.
- *"What about gRPC server streaming?"* — great for service-to-service (StreamPulse uses gRPC for TV
  ingest, BS-09), poor for browsers without grpc-web plus a proxy. Browser reach is the deciding
  factor.
- *"Scaling either of them to 100k clients?"* — the shape is the same: the connections are cheap
  (memory/fds), the fan-out is the problem. You need a per-node multicast (the `Sinks.Many` registry),
  a shared source every node can read (Kafka broadcast), and a plan for what a reconnect storm does
  when a node dies — which is why the `retry:` hint and client-side **jittered** backoff matter.
- *"Does SSE work through your company's proxy?"* — the honest answer is "usually, and the failure
  mode is buffering, and the fix is `X-Accel-Buffering: no` plus a heartbeat." Knowing the failure
  mode is the signal.

**Pitfalls / red flags:** "WebSocket because it's real-time." Both are real-time; the question is
directionality, payload type, and operational cost.

**Where in StreamPulse:** Phase 5 (SSE), Phase 7 (`useEventSource` with reconnect/backoff), Phase 12
(the activity feed uses the same SSE channel). The ticket-booking Go project used WebSocket — so you
have a genuine comparison to draw on.

---

## Q11. Java 21 virtual threads versus reactive — when does each actually win?

**30-second answer:** Virtual threads solve the *same problem* as reactive — not burning an OS thread
while waiting on I/O — but they solve it **underneath** the programming model instead of on top of it.
With virtual threads I keep ordinary blocking code: try/catch works, stack traces are real, a debugger
steps through it, `ThreadLocal` and MDC work, and a junior can read it. Reactive gives me two things
virtual threads do not: **explicit backpressure** as part of the type, and **composition operators**
for streams over time — merge, window, sample, timeout, retry-with-backoff. So my rule is: a
request/response service that fans out to several blocking calls → Spring MVC on virtual threads,
every time. A **streaming** endpoint where a fast producer meets slow consumers — SSE, a Kafka
fan-out, anything where I must express "drop frames rather than queue" — → reactive, because
`onBackpressureLatest` has no idiomatic virtual-thread equivalent. StreamPulse deliberately runs both
and benchmarks them, which is the honest version of this answer.

**Deeper:**

| | Spring MVC + virtual threads | WebFlux / Reactor |
|---|---|---|
| Programming model | blocking, imperative | declarative, callback-composed |
| Debugging | real stack traces, breakpoints work | assembly-vs-runtime stack split, `checkpoint()`/`onOperatorDebug` |
| `ThreadLocal` / MDC / `@RequestScope` | works | needs Reactor `Context` + context-propagation |
| Backpressure | **none built in** — you need a `Semaphore`, bounded queue, or 429 | first-class (`request(n)` + strategies) |
| Stream-over-time operators | none (you write loops) | `window`, `buffer`, `sample`, `merge`, `retryWhen` |
| Blocking a thread | fine, that's the point | catastrophic (Q6) |
| Cost per concurrent request | a virtual thread's stack, grown on demand on the heap | a subscription's worth of objects |
| Library requirements | JDBC is fine | needs R2DBC / reactive Redis / WebClient all the way down |
| The hard failure mode | **pinning** (see below) and unbounded fan-out to a downstream | one blocking call stalls an event loop |
| Team cost | near zero | real — the whole team must learn it |

**The pinning caveat, stated precisely:** in **JDK 21**, a virtual thread that blocks inside a
`synchronized` block or inside a native/foreign call cannot unmount from its carrier thread — it
*pins* it. Enough pinned carriers and you have a `ForkJoinPool`-sized thread pool again, with none of
the benefits. The mitigations on 21 are: replace `synchronized` with `ReentrantLock` on paths that
block, and run with `-Djdk.tracePinnedThreads=full` in test to find them. **I believe JDK 24
(JEP 491) removed the `synchronized` pinning limitation, and newer JDKs surface pinning via a JFR
event rather than that system property — verify both against the JDK you are actually running.**

**The other thing people miss:** virtual threads remove the *thread* limit, not the *downstream*
limit. If 10,000 virtual threads each open a JDBC connection, you have not fixed anything — you have
moved the queue from the thread pool to the connection pool, where it fails worse. That is why
REQ-CC-05 pairs `Executors.newVirtualThreadPerTaskExecutor()` with a `Semaphore(16)`: the semaphore is
now the *only* thing bounding concurrency, so it has to be explicit.

**Code:**

```java
// ingest-gateway (Phase 2): Spring MVC, blocking code, virtual threads underneath.
// application.yml
//   spring:
//     threads:
//       virtual:
//         enabled: true        ← VERIFY this key for your Boot version

@PostMapping("/v1/events:batch")
ResponseEntity<BatchAccepted> ingest(@Valid @RequestBody IngestBatch batch) {
    // Plain blocking code on a virtual thread. Readable, debuggable, MDC works.
    // The bound is NOT the thread pool any more — it's this semaphore (REQ-CC-05 rationale).
    if (!inFlight.tryAcquire()) {
        throw new OverloadedException(Duration.ofSeconds(1));   // → 429 + Retry-After
    }
    try {
        return ResponseEntity.accepted().body(ingest.accept(batch.toCommand()));
    } finally {
        inFlight.release();
    }
}
```

```java
// Structured concurrency is the piece that makes the virtual-thread version competitive
// with CompletableFuture fan-out for the Command Center (REQ-CC-01).
// NOTE: StructuredTaskScope was a PREVIEW API in JDK 21 and its API changed in later releases.
// Verify the exact shape for your JDK before using it in an answer.
```

**Follow-ups they'll ask:**
- *"Can you use virtual threads with WebFlux?"* — you can run `boundedElastic` on virtual threads
  (there is a Reactor property for it — verify the name), which makes the "unavoidable blocking call"
  escape hatch cheaper. But the event loops themselves stay platform threads and must stay unblocked.
- *"Would you migrate an existing WebFlux service to virtual threads?"* — only with a measured reason.
  The rewrite risk is real and reactive code that already works is not a problem. New services are
  where the choice is free.
- *"Do virtual threads make everything faster?"* — no. For CPU-bound work they are neutral-to-worse
  (same carrier pool, extra scheduling). They help *concurrency of blocking I/O*, nothing else.
- *"What did your benchmark show?"* — say the honest thing: *"I ran it in Phase 9 with Gatling and
  wrote it up in an ADR; here is the shape of the result and the caveats of a single-laptop
  benchmark."* Never quote a number you have not measured.

> **Your story:** you have lived the equivalent trade-off on the frontend — callback/Promise-based
> concurrency versus `async/await`. The argument that async/await won because it is *readable and
> debuggable*, not because it is faster, is exactly the virtual-threads argument. That analogy lands
> well with backend interviewers and it is genuinely yours.

**Pitfalls / red flags:** "virtual threads make WebFlux obsolete." They remove one of its three
reasons to exist (cheap concurrency), not the other two (backpressure and stream composition).

**Where in StreamPulse:** `ingest-gateway` Phase 2 (MVC + virtual threads), `query-api` Phase 5
(WebFlux), Phase 5 benchmark ADR comparing the same endpoint on both, Phase 9 pinning lab
(REQ-CC-05).

**Verify:** `spring.threads.virtual.enabled` (Boot 3.2+), `-Djdk.tracePinnedThreads` status, JEP 491's
delivery version, and `StructuredTaskScope`'s current API — all changed recently.

---

## Q12. `RestTemplate`, `WebClient`, `RestClient` — which do you use and why?

**30-second answer:** `WebClient` for reactive code and anywhere I want streaming responses;
`RestClient` for blocking code, because it gives me `WebClient`'s fluent API and modern configuration
with synchronous semantics, so on virtual threads I get simple imperative code without dragging
Reactor into a service that does not need it. `RestTemplate` is maintenance-mode: still supported, not
getting new features, and I would not start new code with it. The practical detail behind all three is
that **the client is not the interesting part — the configuration is**: connection pool sizing,
connect and read timeouts (never infinite), retry policy and where it sits relative to the circuit
breaker, and whether the error branch is an exception or a status you have to inspect.

**Deeper:**

| | `RestTemplate` | `WebClient` | `RestClient` |
|---|---|---|---|
| Style | blocking, template | non-blocking, reactive; `.block()` available | blocking, fluent |
| Since | Spring 3 | Spring 5 | **Spring 6.1 / Boot 3.2** — verify |
| Streaming response | no | **yes** (`bodyToFlux`, SSE client) | limited |
| Use in WebFlux | never | yes | no (it blocks) |
| Use on virtual threads | works | works but pulls in Reactor | **best fit** |
| Error default | throws on 4xx/5xx (`HttpStatusCodeException`) | **does not throw** until `.retrieve()...onStatus` or `toEntity`; `.exchange` gives full control | throws on 4xx/5xx by default, `onStatus` to customise |

Two configuration points I would raise unprompted:

1. **Timeouts are not optional.** A default `WebClient` has no response timeout. StreamPulse has an
   ArchUnit rule for this (REQ-RS-06: "no HTTP client without a timeout") because it is the single
   most common cause of a cascading outage.
2. **Ordering versus resilience decorators.** Retry must sit *outside* the circuit breaker so that
   retried failures count toward opening it (REQ-RS-07); if retry is inside, the breaker sees one
   logical call and never trips.

**Code:**

```java
/**
 * query-api → control-plane (Phase 5). Reactive, because this result feeds a Mono
 * that is combined with four other panels (REQ-CC-01) and each has its own budget (REQ-CC-02).
 */
@Bean
WebClient controlPlaneClient(WebClient.Builder builder, ControlPlaneProperties props) {
    HttpClient http = HttpClient.create()
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 500)
            .responseTimeout(Duration.ofMillis(props.readTimeoutMs()));   // never unbounded

    return builder
            .baseUrl(props.baseUrl())
            .clientConnector(new ReactorClientHttpConnector(http))
            .defaultStatusHandler(HttpStatusCode::is5xxServerError,
                    r -> Mono.error(new UpstreamUnavailableException("control-plane")))
            .build();
}

Mono<TitleMeta> titleMeta(String id) {
    return controlPlane.get().uri("/v1/titles/{id}", id)
            .retrieve()
            .bodyToMono(TitleMeta.class)
            .timeout(Duration.ofMillis(150))                       // per-panel budget
            .onErrorResume(e -> Mono.just(TitleMeta.degraded(id))); // REQ-CC-02 DEGRADED panel
}
```

```java
/**
 * control-plane → partner webhook (Phase 6, BS-07). Blocking service, virtual threads,
 * so RestClient keeps the code imperative — and the resilience decorators are explicit.
 */
RestClient partner = RestClient.builder()
        .baseUrl(partnerProps.url())
        .requestFactory(clientHttpRequestFactoryWithTimeouts())   // connect + read timeouts
        .build();
```

**Follow-ups they'll ask:**
- *"HTTP interface clients?"* — Spring 6 lets you declare an interface with `@GetExchange`/`@PostExchange`
  and generate a proxy (`HttpServiceProxyFactory`), backed by `WebClient` or `RestClient`. It gives you
  a typed, mockable port, which fits hexagonal nicely. **Verify the factory/adapter class names for
  your version** — they were refined in 6.1/6.2.
- *"How do you test an HTTP client?"* — WireMock for the real wire behaviour (timeouts, 503s, slow
  responses — which is what BS-04 needs), `MockWebServer` as a lighter option, and
  `MockRestServiceServer` for `RestTemplate`. For `WebClient`, an `ExchangeFunction` stub for pure unit
  tests.
- *"Connection pool sizing?"* — Reactor Netty's pool is per-host with a max-connections and a pending
  acquire timeout; when it is exhausted the symptom is a timeout in *acquire*, not in the call, and
  people misdiagnose it as a slow server. Worth knowing the metric name to look at.

**Pitfalls / red flags:** `WebClient` with `.block()` inside a WebFlux handler; or any HTTP client with
no timeout configured.

**Where in StreamPulse:** `query-api` → control-plane and Redis/DynamoDB adapters (Phase 5);
`control-plane` → partner webhooks (BS-07, with the BS-04 resilience stack around it).

**Verify:** `RestClient` requires Spring Framework 6.1 / Boot 3.2 or later; HTTP-interface client class
names changed across 6.x minors.

---

## Q13. How do you test each of these layers?

**30-second answer:** Slices, not `@SpringBootTest`, for everything that can be sliced. `@WebMvcTest`
boots only the web layer — the controller, the converters, the argument resolvers, my
`@ControllerAdvice` and the security filter chain — and I drive it with `MockMvc`, mocking the use-case
port. `@WebFluxTest` is the reactive twin with `WebTestClient`. For the Reactor chains themselves I use
`StepVerifier`, which asserts the actual signal sequence — the `onNext`s, then `onComplete` or
`onError` — and `StepVerifier.withVirtualTime` so a 15-second heartbeat test runs in milliseconds
instead of taking 15 seconds. Then, and only then, Testcontainers integration tests for the things a
slice cannot prove: real Kafka, real Redis, real serialization.

**Deeper:**

| Goal | Tool | What it proves | Boot cost |
|---|---|---|---|
| Controller mapping, validation, error body, security rules | `@WebMvcTest` + `MockMvc` | the HTTP contract | fast (web layer only) |
| Same, reactive | `@WebFluxTest` + `WebTestClient` | the HTTP contract | fast |
| Operator chain semantics | `StepVerifier` | exact signal sequence, timing, errors | none |
| Time-dependent operators (`interval`, `timeout`, `sample`) | `StepVerifier.withVirtualTime` | behaviour without wall-clock waiting | none |
| SSE end to end | `WebTestClient` returning `FluxExchangeResult<T>` + `StepVerifier` | headers, event framing, ids | fast |
| Kafka → sink → SSE | Testcontainers + `@SpringBootTest` | the wiring and serialization | slow — few of these |
| Frontend ↔ API contract | Spring Cloud Contract / Pact | the two sides agree | medium |

**Code:**

```java
// 1. Slice test: the HTTP contract, no Kafka, no Redis, no Netty.
@WebMvcTest(IngestController.class)
class IngestControllerTest {

    @Autowired MockMvc mvc;
    @MockitoBean IngestUseCase ingest;      // @MockBean in older Boot — VERIFY for your version

    @Test
    @DisplayName("REQ-INGEST: a batch over 500 events is rejected as a ProblemDetail 400")
    void rejectsOversizedBatch() throws Exception {
        mvc.perform(post("/v1/events:batch")
                    .contentType(APPLICATION_JSON)
                    .content(batchJson(501)))
           .andExpect(status().isBadRequest())
           .andExpect(content().contentTypeCompatibleWith("application/problem+json"))
           .andExpect(jsonPath("$.type").value("https://errors.streampulse.dev/validation-failed"))
           .andExpect(jsonPath("$.errors[0].field").value("events"));

        verifyNoInteractions(ingest);       // it never reached the domain — validation did its job
    }
}
```

```java
// 2. StepVerifier: backpressure policy, asserted rather than assumed.
@Test
@DisplayName("REQ-CC-01 read side: a slow subscriber sees the LATEST value, not a queue of stale ones")
void slowSubscriberGetsLatest() {
    Sinks.Many<Integer> sink = Sinks.many().multicast().onBackpressureBuffer(8, false);

    StepVerifier.create(sink.asFlux().onBackpressureLatest(), /* initial request */ 0)
            .then(() -> { sink.tryEmitNext(1); sink.tryEmitNext(2); sink.tryEmitNext(3); })
            .thenRequest(1)                       // subscriber finally asks for one
            .expectNext(3)                        // ← latest, 1 and 2 were dropped
            .thenCancel()
            .verify(Duration.ofSeconds(1));
}
```

```java
// 3. Virtual time: a 15s heartbeat asserted in ~0ms.
@Test
void heartbeatEvery15Seconds() {
    StepVerifier.withVirtualTime(() -> Flux.interval(Duration.ofSeconds(15)).take(3))
            .expectSubscription()
            .expectNoEvent(Duration.ofSeconds(15)).expectNext(0L)
            .expectNoEvent(Duration.ofSeconds(15)).expectNext(1L)
            .expectNoEvent(Duration.ofSeconds(15)).expectNext(2L)
            .verifyComplete();
}
```

```java
// 4. SSE contract: the headers and the framing, not just the payload.
@Test
void sseEndpointEmitsMetricEventsWithResumableIds() {
    FluxExchangeResult<ServerSentEvent<String>> result = webTestClient.get()
            .uri("/v1/stream/metrics")
            .accept(MediaType.TEXT_EVENT_STREAM)
            .exchange()
            .expectStatus().isOk()
            .expectHeader().contentTypeCompatibleWith(MediaType.TEXT_EVENT_STREAM)
            .returnResult(new ParameterizedTypeReference<ServerSentEvent<String>>() {});

    StepVerifier.create(result.getResponseBody())
            .assertNext(e -> {
                assertThat(e.event()).isEqualTo("metric");
                assertThat(e.id()).matches("\\d+-\\d+");    // resumable cursor format
            })
            .thenCancel()
            .verify(Duration.ofSeconds(5));
}
```

**Follow-ups they'll ask:**
- *"Why not just `@SpringBootTest` for everything?"* — a full context per test class is seconds, not
  milliseconds, and it hides which layer broke. I keep a handful of full-context tests as smoke tests
  and push everything else down.
- *"Does `@WebMvcTest` include security?"* — the security filter chain is auto-configured in the
  slice, which is what you want, and `@WithMockUser` / `SecurityMockMvcRequestPostProcessors.jwt()`
  drive it. If you forget, every test gets a 401 and people "fix" it by disabling security — which
  means the authorization rules are then untested.
- *"How do you test the Kafka-listener-to-sink bridge without Kafka?"* — call the listener method
  directly with a constructed record and assert on the sink's `asFlux()` with `StepVerifier`. The
  Kafka part is proven once, separately, with Testcontainers.
- *"How do you keep `StepVerifier` tests from hanging CI?"* — always `verify(Duration)`, never the
  no-argument `verify()`, on anything that could be infinite.

**Pitfalls / red flags:** asserting on an SSE stream with `Thread.sleep`, or a reactive test that
passes because nothing ever subscribed.

**Where in StreamPulse:** Phase 2 (`@WebMvcTest`), Phase 5 (`@WebFluxTest`, `StepVerifier`, the SSE
integration test), Phase 7 (contract test between `query-api` and `web/`). The `test-conventions`
Gradle plugin gives every module JUnit 5, AssertJ, Mockito, Testcontainers and BlockHound.

**Verify:** `@MockitoBean` replaced `@MockBean` in recent Boot/Spring Framework versions — check which
one your version wants.

---

## Q14. Walk me through the StreamPulse Command Center end to end — why MVC on one side and WebFlux on the other?

*(This is the applied question. It is the one that converts everything above into evidence.)*

**30-second answer:** Two endpoints, two different problems, two deliberate choices. The **write**
side, `POST /v1/events:batch`, is Spring MVC on virtual threads: it is request/response, it does
blocking work (Redis dedup, a Kafka send), and the code is much easier to read and debug imperatively.
Its concurrency bound is an explicit in-flight `Semaphore` that returns 429 with `Retry-After` when
saturated — because virtual threads remove the thread limit, so the limit has to become explicit. The
**read** side is WebFlux for two specific reasons: 10,000 long-lived SSE connections are cheap on an
event loop, and — the real reason — I need `onBackpressureLatest` and `sample`, which are
stream-over-time operators with no natural blocking equivalent. The Command Center page itself fans
out to five panels in parallel with per-panel timeouts, so a slow DynamoDB panel degrades that one
panel instead of the page. And I benchmarked both stacks on the same endpoint in Phase 9 rather than
arguing about it.

**Flow of control — `GET /v1/titles/{id}/command-center` (REQ-CC-01, REQ-CC-02):**

```
browser
  │  GET /v1/titles/tt-9001/command-center
  ▼
query-api (WebFlux)  ── WebFilter: JWT → tenant → Reactor Context (NOT ThreadLocal)
  │
  ├─ Mono A  live viewers        ← reactive Redis            budget 150ms
  ├─ Mono B  24h trend           ← DynamoDB async client     budget 150ms
  ├─ Mono C  title metadata      ← LRU cache → control-plane budget 150ms   (REQ-CC-03)
  ├─ Mono D  active alerts       ← Redis                     budget 150ms
  └─ Mono E  top error codes     ← Redis ZSET                budget 150ms
  │
  Mono.zip(A,B,C,D,E)  — all five subscribed concurrently; total ≈ max(latencies), not the sum
  │        each has .timeout(budget).onErrorResume(→ Panel.degraded())
  ▼
CommandCenterDto { panels: [ {status: OK|DEGRADED, data|null}, … ] }   → 200, always
```

The design decision worth defending: **a degraded panel is a 200, not a 500.** An operator during a
live match needs the four panels that work far more than they need a correct HTTP status on the one
that does not. The degradation is visible in the payload (`status: "DEGRADED"`) and in a metric
(`commandcenter.panel.degraded{panel}`), so it is not silent.

**Flow of data — the live stream:**

```
telemetry.agg.v1 (Kafka)
   │  @KafkaListener, one consumer group PER POD  → every pod sees every partition
   ▼
TenantSinkRegistry.publish()  → Sinks.Many (multicast, bounded buffer, per tenant)
   │                          → Redis ZSET ring buffer (last 5 min, for Last-Event-ID resume)
   ▼
GET /v1/stream/metrics  → asFlux() → onBackpressureLatest → sample(250ms) → SSE, id = ts-partition
   │                                  ▲ 15s comment heartbeat merged in
   ▼
browser EventSource → auto-reconnect with Last-Event-ID → server replays the gap from Redis, then live
```

**Why the two stacks, in a table I'd draw on the whiteboard:**

| | `ingest-gateway` (write) | `query-api` (read) |
|---|---|---|
| Shape | request/response, short | long-lived streams |
| Stack | Spring MVC + virtual threads | WebFlux |
| Concurrency bound | explicit `Semaphore` → 429 + `Retry-After` | `request(n)` + `onBackpressureLatest` |
| Why not the other one | reactive buys nothing here and costs debuggability | 10k blocking threads for 10k idle SSE streams, and no drop-frames operator |
| The evidence | Phase 9 Gatling run, ADR with the numbers | same benchmark, same ADR |

**Follow-ups they'll ask:**
- *"Isn't running two stacks a cost?"* — yes, and I'd name it: two mental models, two testing
  approaches, interceptors don't port to `WebFilter`, and context propagation works differently. In a
  product team I'd need a strong reason. Here the reason is that the two sides genuinely have
  different shapes — and for the portfolio project, being able to argue both is the point.
- *"Could the whole thing be MVC + virtual threads, SSE included?"* — technically yes, with
  `SseEmitter`. What I'd lose is the backpressure vocabulary: I'd have to hand-roll "drop frames when
  the client is slow", which is a bounded queue plus a discard policy per connection. That is a
  reimplementation of `onBackpressureLatest` with more bugs. That trade is the honest answer.
- *"What happens to open SSE streams during a rolling deploy?"* — BS-01's `SmartLifecycle` drain
  completes the sinks first, clients get `onComplete`, `EventSource` reconnects with jittered backoff
  to a surviving pod and resumes from `Last-Event-ID`. The gap is bounded by the Redis ring buffer.
  The jitter matters: without it, every dashboard reconnects in the same 100ms window.
- *"Where's the LRU cache in this?"* — panel C. It is the hand-written `HashMap` + doubly-linked-list
  LRU from `kata/` promoted into production code, guarded by a `ReentrantReadWriteLock`, invalidated
  by `config.changes.v1` CDC events. The interesting bit is that a *read* mutates the access order, so
  it needs the write lock — which is a genuine design discussion, covered in file 17 problem 1 and
  REQ-CC-03.

> **Your story:** the reconnect/backoff/resume half of this is *your* territory — you have built
> real-time frontends. Say how you handled reconnect storms, exponential backoff with jitter, and what
> the UI shows while disconnected. Backend interviewers rarely get a candidate who can argue both ends
> of the socket, and that is a genuine differentiator for you. Keep "I built" for what you personally
> wrote.

**Where in StreamPulse:** BS-02 / REQ-CC-01, REQ-CC-02, REQ-CC-03; Phase 5 (query-api, SSE, LRU,
benchmark ADR); Phase 7 (`useEventSource`); Phase 9 (the measured comparison).
Tests: `CommandCenterTimeoutTest`, the `@WebFluxTest` SSE slice, `StepVerifier` backpressure tests.

---

## Key terms

| Term | One-line meaning |
|---|---|
| **`DispatcherServlet`** | MVC's front controller: mapping → adapter → argument resolvers → return-value handlers. |
| **`HandlerMapping` / `HandlerAdapter`** | "Which handler?" and "how do I invoke it?" — the two pluggable halves of dispatch. |
| **`OncePerRequestFilter`** | A filter guaranteed to run once per request across FORWARD/ASYNC/ERROR dispatches. |
| **`HandlerInterceptor`** | MVC-level hook that knows the resolved `HandlerMethod`; no WebFlux equivalent. |
| **`ProblemDetail` / RFC 7807** | The standard `application/problem+json` error body: `type`, `title`, `status`, `detail`, `instance`. |
| **Assembly vs subscription time** | Building the chain vs starting it; nothing runs until something subscribes. |
| **Cold vs hot publisher** | Per-subscriber execution vs a shared stream a late subscriber joins mid-flight. |
| **`subscribeOn` / `publishOn`** | Where the *source* runs vs where *downstream* operators run. |
| **Backpressure** | `Subscription.request(n)` — and, when the source can't slow down, a buffer/drop/latest/error policy. |
| **`onBackpressureLatest`** | Keep only the newest unconsumed element — the right default for live gauges. |
| **`Sinks.Many`** | A programmatic push entry point into Reactor; `multicast` is the fan-out primitive. |
| **`EmitResult`** | The return of `tryEmitNext`; ignoring it is how emissions are lost silently. |
| **SSE** | `text/event-stream` over HTTP: `data:`/`event:`/`id:`/`retry:`, comment heartbeats, `Last-Event-ID` resume. |
| **`Last-Event-ID`** | The header `EventSource` sends on reconnect, carrying the last `id:` it saw. |
| **Pinning (virtual threads)** | A virtual thread that cannot unmount from its carrier — classically inside `synchronized` on JDK 21. |
| **`StepVerifier`** | Asserts the exact signal sequence of a `Mono`/`Flux`, with virtual time for timers. |
| **BlockHound** | Test-time agent that throws when blocking happens on a non-blocking thread. |

---

## Self-check (answer out loud, no notes)

1. An exception is thrown in a servlet filter. Does your `@ControllerAdvice` see it? Why does that
   matter for the API's error contract?
2. Draw the SSE wire format for one event, and say which field enables resume and how.
3. A `Flux` feeding a slow browser: name the four backpressure policies, and justify which one a
   live metrics dashboard should use.
4. Under what conditions does a virtual thread pin its carrier, and what is the second limit that
   virtual threads do *not* remove?
5. In StreamPulse, why is the write side MVC and the read side WebFlux — and what is the one operator
   you would have to hand-roll if the read side were MVC too?
