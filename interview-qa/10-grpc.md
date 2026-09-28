# 10 — gRPC: Protobuf, Streaming, Deadlines, Status Codes, and the Kubernetes Problem

**Why this matters for an Event Hub role:** high-volume telemetry from devices is exactly where JSON
over HTTP/1.1 stops being free. gRPC is the standard answer, and it is also on your resume through the
NetApp Kubernetes/gRPC microservice — so expect both "explain gRPC" *and* "walk me through how you
built X." The senior differentiators in this topic are three: schema evolution rules, deadline
propagation, and knowing why gRPC breaks naive Kubernetes load balancing.

**Scenario anchor — BS-09 (smart-TV streaming ingest):** a smart-TV OEM SDK keeps a long-lived
connection and emits a heartbeat every 10s per device. HTTP batch overhead — TLS handshakes, JSON
parsing — dominates device CPU and battery, so the OEM asks for a binary streaming protocol. The
constraint that makes it interesting: both protocols must share one `IngestUseCase`, proven by a test
asserting identical Kafka records (REQ-GR-02).

---

## Q1. What is gRPC, and when would you choose it over REST?

**30-second answer:** gRPC is an RPC framework: you define services and messages in a `.proto` file,
generate typed client and server code, and it runs over HTTP/2 with Protobuf binary encoding. I choose
it for **internal service-to-service** calls where I control both ends and want a schema-enforced
contract, low serialization cost, and real streaming. I stay with REST for public APIs, anything a
browser calls directly, and anything where human-readable debugging and broad tooling matter more than
bytes. The honest framing is that it's not "gRPC is faster" — it's that a generated contract and
bidirectional streaming are things REST doesn't give you, and binary encoding is a bonus.

**Deeper:**

| | REST/JSON | gRPC | GraphQL |
|---|---|---|---|
| Contract | OpenAPI (optional, often drifts) | `.proto`, **required**, codegen enforces it | schema, required |
| Encoding | text JSON | binary Protobuf | text JSON |
| Transport | usually HTTP/1.1 | HTTP/2 required | HTTP/1.1+ |
| Streaming | SSE (server→client only), WebSocket (separate protocol) | **native, all four directions** | subscriptions (usually WS) |
| Browser | native | needs gRPC-Web + a proxy | native |
| Debuggability | curl | needs `grpcurl` + reflection | GraphiQL |
| Best at | public APIs, browser clients | internal mesh, high-volume, streaming | client-shaped aggregation |

The **contract-is-mandatory** property is the one worth leading with. An OpenAPI spec is a document
people forget to update; a `.proto` is the thing the code is generated *from*, so drift is impossible
by construction. In StreamPulse the proto lives in `libs/contracts` alongside the Avro schemas, and a
breaking-change check runs in CI (REQ-GR-01).

**Code:**

```protobuf
// libs/contracts/src/main/proto/streampulse/telemetry/v1/ingest.proto
syntax = "proto3";
package streampulse.telemetry.v1;          // versioned package → v2 can coexist during migration

service TelemetryIngest {
  // BS-09: long-lived device connection. Bidi so we can ack durable sequence numbers back.
  rpc StreamEvents (stream TelemetryEvent) returns (stream Ack);

  // Parity with the REST batch endpoint, for clients that cannot hold a connection open.
  rpc SendBatch (Batch) returns (BatchAck);
}

message TelemetryEvent {
  string  event_id   = 1;   // client-generated UUID → idempotency key (REQ-GR-02)
  string  tenant_id  = 2;
  string  session_id = 3;   // Kafka partition key
  int64   ts_millis  = 4;
  EventType type     = 5;
  oneof payload {           // exactly one of these is set — the discriminated union
    Heartbeat  heartbeat  = 10;
    Buffering  buffering  = 11;
    ErrorEvent error      = 12;
  }
}
```

**Follow-ups they'll ask:**
- *"Is gRPC actually faster than REST?"* — for the serialization and header cost, measurably yes; for
  a call dominated by a 50ms database query, irrelevant. I'd quote my own benchmark or say I'd measure
  it, not a number from a blog post.
- *"What do you lose?"* — curl-ability, browser support, caching by intermediaries, and easy debugging
  by anyone who doesn't have the proto. `grpcurl` + server reflection recovers some of it.
- *"Why HTTP/2 specifically?"* — streaming and multiplexing need it; HTTP/1.1 can't carry N concurrent
  logical streams on one connection without head-of-line blocking.

**Pitfalls / red flags:** "gRPC is faster than REST" as the whole answer. The senior answer leads with
the contract and streaming, and is honest that binary encoding is often the smallest win.

**Where in StreamPulse:** BS-09 / REQ-GR-01; `libs/contracts`.

---

## Q2. How does Protobuf encode data on the wire, and why can you never reuse a field number?

**30-second answer:** Each field is encoded as a **tag plus a value**, where the tag packs the field
number and a 3-bit wire type into a varint. The field *name* is never transmitted — only the number.
That's the whole reason the format is compact, and also the whole reason field numbers are sacred: if
you delete field 5 and later reuse 5 for a different type, an old client's bytes get decoded into the
new field, silently. You get corrupt data, not a parse error. The fix is `reserved`, which makes the
compiler refuse to let anyone reuse the number.

**Deeper:**

```
message on the wire:  [tag][value][tag][value]...
tag = (field_number << 3) | wire_type       ← both packed into one varint
```

Two practical consequences:

1. **Field numbers 1–15 cost one byte** for the tag; 16–2047 cost two. So put your highest-frequency
   fields in 1–15. In `TelemetryEvent` above, the fields on every single heartbeat (`event_id`,
   `tenant_id`, `session_id`, `ts_millis`, `type`) take 1–5; the payload variants — only one of which
   is present per message — start at 10. At StreamPulse's event volume that byte-per-field is not
   nothing, and it is a good detail to mention unprompted.
2. **Unknown fields are preserved** (in proto3, since 3.5) rather than discarded, so a proxy that
   parses and re-serializes a message doesn't silently drop fields it doesn't know about.

**Code:**

```protobuf
message TelemetryEvent {
  reserved 6, 7;                  // ← retired fields: the compiler now REFUSES to reuse these
  reserved "device_model";        // retired name too (matters for JSON mapping)

  string event_id = 1;
  // ...
}
```

**Follow-ups they'll ask:**
- *"What are the wire types?"* — varint (ints, bools, enums), 64-bit, length-delimited (strings,
  bytes, embedded messages, packed repeated), and 32-bit. The two deprecated group types you can
  ignore.
- *"Why is a negative `int32` expensive?"* — varint encodes it as if unsigned, so a negative number
  takes the full 10 bytes. Use `sint32`/`sint64`, which zig-zag encode. A genuinely good detail.
- *"How big is a Protobuf message vs the JSON?"* — usually substantially smaller, mostly because field
  names aren't transmitted. I'd measure rather than quote a ratio.

**Pitfalls / red flags:** not knowing that names aren't on the wire — it's the fact that explains both
the compactness and the renaming rules in Q3.

**Verify:** the exact unknown-field-retention behaviour by proto3 version, and the one-byte tag range,
against current Protobuf docs before you quote them.

---

## Q3. What counts as a breaking change to a `.proto`, and what doesn't?

**30-second answer:** Safe: **adding** a new field with a fresh number, and **renaming** a field —
because the name isn't on the wire. Breaking: **changing a field's number**, **changing its type**
incompatibly, and **reusing a retired number**. Deleting a field is safe on the wire but you must
`reserve` the number so nobody reuses it later. The rule I apply is that the proto is an append-only
log of field numbers. In StreamPulse this is enforced mechanically in CI, not by review — humans are
bad at remembering which numbers are retired.

**Deeper:**

| Change | Wire-safe? | Notes |
|---|---|---|
| Add a field with a new number | ✅ | Old readers ignore it (and preserve it as unknown) |
| Rename a field | ✅ on the wire | ❌ for JSON mapping and for generated code — it's a **source**-breaking change |
| Delete a field | ✅ *if* you `reserved` the number | Without `reserved`, a future reuse silently corrupts |
| Change field number | ❌ | The field simply vanishes for the other side |
| `int32` → `int64` | ✅ (both varint) | Value may truncate on old readers |
| `int32` → `string` | ❌ | Different wire type; garbage or parse failure |
| Add a value to an `enum` | ⚠️ | Old readers see an unrecognized value — handle the default/unknown case |
| Move a field into a `oneof` | ❌ | Changes presence semantics |

The enum row is the one that bites in practice. Always define the zero value as an explicit
`*_UNSPECIFIED`, and always handle it:

```protobuf
enum EventType {
  EVENT_TYPE_UNSPECIFIED = 0;   // proto3 requires a zero value; make it mean "unknown", not a real type
  EVENT_TYPE_HEARTBEAT   = 1;
  EVENT_TYPE_BUFFERING   = 2;
  EVENT_TYPE_ERROR       = 3;
}
```

A server that treats `UNSPECIFIED` as "reject with `INVALID_ARGUMENT`" stays correct when a newer
client sends an enum value it has never heard of — it degrades to a clear error instead of a wrong
branch.

**Follow-ups they'll ask:**
- *"How do you enforce this?"* — a breaking-change checker in CI that diffs the proto against the
  committed baseline (the `buf` toolchain is the common choice). The point is that it's automated.
- *"How does this compare to Avro compatibility modes?"* — Avro resolves by *name* with reader/writer
  schema resolution and a registry; Protobuf resolves by *number* with no registry needed. StreamPulse
  uses both: Avro on Kafka (BACKWARD compatibility in Schema Registry) and Protobuf on the gRPC edge.
  Being able to contrast them is a strong signal.
- *"What if you genuinely must break it?"* — new package version (`v2`), run both, migrate clients,
  retire `v1`. Same as REST.

**Where in StreamPulse:** BS-09 / REQ-GR-01 (lint + breaking-change check in CI).

**Verify:** the `buf` CLI's current command names and whether it's still the standard tool.

---

## Q4. What are the four RPC types, and when do you use each?

**30-second answer:** Unary (one request, one response — the REST equivalent); server streaming (one
request, many responses — a live feed or a large result set); client streaming (many requests, one
response — bulk upload); and bidirectional streaming (both, independently, over one connection). The
key thing about bidi is that the two directions are **not** lock-stepped — the server can send acks
whenever it likes while the client keeps pushing. That's exactly what BS-09 needs: devices push
heartbeats continuously, and the server periodically acks the last durable sequence number.

**Deeper:**

| Type | Shape | StreamPulse use | REST equivalent |
|---|---|---|---|
| Unary | 1 → 1 | `SendBatch` for clients that can't hold a connection | `POST /v1/events:batch` |
| Server streaming | 1 → N | live metric feed to an internal consumer | SSE |
| Client streaming | N → 1 | bulk historical backfill from a device that was offline | chunked upload |
| Bidirectional | N ↔ N | **`StreamEvents`** — the smart-TV path | WebSocket |

**Code** — the bidi server side, which is where the interesting mechanics live:

```java
@GrpcService                                    // Spring gRPC starter — VERIFY the annotation name
public class TelemetryIngestService extends TelemetryIngestGrpc.TelemetryIngestImplBase {

    private final IngestUseCase ingest;         // REQ-GR-02: the SAME port the REST adapter calls

    @Override
    public StreamObserver<TelemetryEvent> streamEvents(StreamObserver<Ack> responseObserver) {
        // Return an observer the runtime feeds as messages arrive.
        return new StreamObserver<>() {
            private long lastDurableSeq = 0;

            @Override
            public void onNext(TelemetryEvent proto) {
                // Honour client cancellation / deadline expiry — don't keep working for a gone caller.
                if (Context.current().isCancelled()) {
                    responseObserver.onError(Status.CANCELLED
                            .withDescription("client cancelled").asRuntimeException());
                    return;
                }
                var result = ingest.accept(ProtoMapper.toDomain(proto));   // shared domain logic
                lastDurableSeq = result.sequence();

                // Ack independently of the inbound flow — this is what "bidirectional" buys us.
                responseObserver.onNext(Ack.newBuilder().setDurableSeq(lastDurableSeq).build());
            }

            @Override
            public void onError(Throwable t) {
                // Client died or the connection broke. No response is possible; just release resources.
                log.warn("stream failed after seq {}", lastDurableSeq, t);
            }

            @Override
            public void onCompleted() {
                responseObserver.onCompleted();   // half-close the response side
            }
        };
    }
}
```

**Follow-ups they'll ask:**
- *"What happens if the client disconnects mid-stream?"* — `onError` fires on the server with a
  `CANCELLED` or transport status; the `Context` is cancelled so downstream work can be abandoned. The
  events already published to Kafka stay published — which is why the ack carries a durable sequence
  number the client can resume from.
- *"Is a stream ordered?"* — yes, within a single stream, messages are delivered in order. Across
  streams, no ordering exists.
- *"Bidi vs two separate streams?"* — bidi shares one HTTP/2 stream and one lifecycle, so cancellation
  and deadlines apply to both directions at once.

**Pitfalls / red flags:** describing bidi as request-response ping-pong. The independence of the two
directions is the point.

**Where in StreamPulse:** BS-09 / REQ-GR-01, REQ-GR-02.

**Verify:** the Spring gRPC integration's annotation and starter names — there have been both a
community `grpc-spring-boot-starter` and newer official Spring support; confirm which you're using.

---

## Q5. What does HTTP/2 give gRPC, and does it really eliminate head-of-line blocking?

**30-second answer:** HTTP/2 gives multiplexing — many independent logical streams over one TCP
connection — plus HPACK header compression and per-stream flow control. That removes **application-level**
head-of-line blocking, which is the HTTP/1.1 problem where one slow response blocks the connection.
But it does **not** remove **transport-level** head-of-line blocking: it's still one TCP connection, so
a single lost packet stalls every stream on it until retransmission. Only HTTP/3 over QUIC fixes that.
That nuance is usually the follow-up, so I say it up front.

**Deeper:**

```
HTTP/1.1                             HTTP/2
─────────────────────────            ─────────────────────────
conn 1: [req A ......][req B]        conn 1: stream 1 ═╗
conn 2: [req C .....]                        stream 3 ═╬═ interleaved frames
conn 3: [req D ...]                          stream 5 ═╝
(browser opens ~6 connections)       (one connection, N streams)
 ↑ A blocks B on the same conn        ↑ no app-level HoL … but one TCP loss stalls all
```

Flow control exists at **two levels**: per-stream and per-connection, each with a window that the
receiver advertises and replenishes with `WINDOW_UPDATE` frames. This is the mechanism underneath
Q9's application-level backpressure — the `request(n)` you call in your handler ultimately governs
whether those windows get replenished.

**Follow-ups they'll ask:**
- *"Why does gRPC need TLS?"* — it doesn't strictly (h2c, cleartext HTTP/2, exists), but many
  intermediaries only negotiate HTTP/2 via ALPN over TLS, so in practice TLS is the reliable path.
- *"What is HPACK and why care?"* — header compression with a shared dynamic table. It matters
  enormously for BS-09's workload: a heartbeat every 10s per device is a tiny payload with repetitive
  headers, so uncompressed headers would dominate the bytes.
- *"Is there a downside to one connection?"* — yes, and it's the Kubernetes load-balancing problem in
  Q11.

**Pitfalls / red flags:** claiming HTTP/2 "solves head-of-line blocking" flat out. The transport-level
caveat is exactly what a senior interviewer is listening for.

---

## Q6. Explain deadlines, and how they differ from a client-side timeout.

**30-second answer:** A timeout is local — *my* client gives up after N seconds, but the server keeps
working, burning capacity for a result nobody will read. A gRPC **deadline** is an absolute point in
time that travels **with the request** in its metadata, so every hop downstream knows when the whole
operation becomes pointless and can abandon its work. That's the difference that matters under load:
deadlines stop a slow dependency from converting into wasted work across the entire call chain. The
rule I follow is that every gRPC call has a deadline — an unset deadline means infinity, which is
never what you want.

**Deeper:**

The propagation is the point:

```
client (deadline = now + 2s)
  └─▶ ingest-gateway   ~0.3s used, 1.7s remaining ─ propagated
        └─▶ control-plane  ~0.2s used, 1.5s remaining ─ propagated
              └─▶ database call bounded by the REMAINING budget, not a fresh 2s
```

Without propagation, each hop starts its own 2s timer and the worst-case total is 2s × depth. With it,
the total is bounded by the original 2s, and a hop that sees an already-expired deadline fails
immediately instead of starting work.

**Code:**

```java
// CLIENT: always set a deadline. withDeadlineAfter returns a NEW stub — stubs are immutable.
var ack = ingestStub
        .withDeadlineAfter(2, TimeUnit.SECONDS)
        .sendBatch(batch);

// SERVER: honour cancellation for anything long-running.
@Override
public void sendBatch(Batch req, StreamObserver<BatchAck> obs) {
    Context ctx = Context.current();
    for (var event : req.getEventsList()) {
        if (ctx.isCancelled()) {                     // deadline expired OR client cancelled
            obs.onError(Status.CANCELLED.asRuntimeException());
            return;                                  // stop burning capacity for a dead call
        }
        ingest.accept(ProtoMapper.toDomain(event));
    }
    obs.onCompleted();
}
```

**Follow-ups they'll ask:**
- *"What status does the caller see when a deadline expires?"* — `DEADLINE_EXCEEDED`.
- *"Is a deadline retry-safe?"* — careful: `DEADLINE_EXCEEDED` is ambiguous. The server may have
  completed the work and the response was lost. That's precisely why BS-09's ingest is idempotent on
  `event_id` — retry safety comes from idempotency, never from the status code.
- *"How do you pick the value?"* — from the SLO downward, not from the observed latency upward. If the
  caller's budget is 2s, no downstream deadline may exceed what's left of it.

**Pitfalls / red flags:** treating deadline and timeout as synonyms. The propagation property is the
entire answer.

**Where in StreamPulse:** BS-09 / REQ-GR-03; `GrpcDeadlineCancellationTest`.

---

## Q7. Which gRPC status codes matter, and which are safe to retry?

**30-second answer:** The retry question splits them into three groups. **Retryable:** `UNAVAILABLE`
(transient, the canonical retry-with-backoff case) and `RESOURCE_EXHAUSTED` (retry, but only after
backing off — it means you're the problem). **Never retry:** `INVALID_ARGUMENT`, `NOT_FOUND`,
`PERMISSION_DENIED`, `UNAUTHENTICATED`, `UNIMPLEMENTED` — retrying is guaranteed to fail identically.
**Ambiguous, needs idempotency:** `DEADLINE_EXCEEDED`, `ABORTED`, `INTERNAL` — the work may or may not
have happened. The trio people confuse is `FAILED_PRECONDITION` vs `ABORTED` vs `UNAVAILABLE`, and
they're distinguished exactly by retry semantics.

**Deeper:**

| Code | Meaning | Retry? |
|---|---|---|
| `OK` | success | — |
| `INVALID_ARGUMENT` | the request is malformed regardless of state | **never** |
| `FAILED_PRECONDITION` | system state is wrong; fix the state first | **not until state changes** |
| `ABORTED` | concurrency conflict (e.g. transaction abort) | **yes, at a higher level** |
| `UNAVAILABLE` | transient; service down or overloaded | **yes, with backoff** |
| `RESOURCE_EXHAUSTED` | quota or rate limit hit | yes, after backoff — honour any retry hint |
| `DEADLINE_EXCEEDED` | budget expired; outcome unknown | only if the operation is idempotent |
| `UNAUTHENTICATED` / `PERMISSION_DENIED` | no valid credentials / not allowed | never (refresh token, then a *new* call) |
| `UNIMPLEMENTED` | method doesn't exist | never — it's a version mismatch |
| `INTERNAL` | server bug | no; alert instead |

BS-09's mapping (REQ-GR-03):

```java
// Domain exceptions → status codes. This mapping IS the API contract for a gRPC service.
Status toStatus(Throwable t) {
    return switch (t) {
        case ValidationException e -> Status.INVALID_ARGUMENT.withDescription(e.getMessage());
        case QuotaExceededException e -> Status.RESOURCE_EXHAUSTED
                .withDescription("tenant quota exceeded; retry after " + e.retryAfter());
        case BackpressureException e -> Status.UNAVAILABLE      // the in-flight semaphore is full
                .withDescription("ingest saturated");
        case TenantNotOnboardedException e -> Status.FAILED_PRECONDITION
                .withDescription("tenant is not onboarded");     // retrying won't help until it is
        default -> Status.INTERNAL;                              // never leak internals in the message
    };
}
```

Two details worth saying unprompted: put machine-readable retry hints in **trailers** (the
`google.rpc.RetryInfo` message in error details) rather than asking clients to parse your description
string; and never put an internal exception message in the description, because it's attacker-visible.

**Follow-ups they'll ask:**
- *"`RESOURCE_EXHAUSTED` vs `UNAVAILABLE` for backpressure?"* — `RESOURCE_EXHAUSTED` when the caller
  has exceeded *their* quota (their problem, don't hammer); `UNAVAILABLE` when *we're* saturated
  (retry against another instance is reasonable). StreamPulse uses both, deliberately.
- *"How does this map to HTTP status codes?"* — there's a documented mapping (roughly
  `INVALID_ARGUMENT`→400, `UNAUTHENTICATED`→401, `PERMISSION_DENIED`→403, `NOT_FOUND`→404,
  `RESOURCE_EXHAUSTED`→429, `UNAVAILABLE`→503); verify it rather than reciting from memory.

**Where in StreamPulse:** BS-09 / REQ-GR-03; `GrpcStatusMappingTest`.

**Verify:** the exact error-details types (`google.rpc.RetryInfo`, `ErrorInfo`) and the canonical
gRPC↔HTTP code mapping table.

---

## Q8. How do interceptors work, and what do you use them for?

**30-second answer:** Interceptors are gRPC's middleware — the equivalent of servlet filters. A
`ServerInterceptor` sees every incoming call before it reaches your service, and a `ClientInterceptor`
wraps every outgoing call. I use them for the three cross-cutting concerns you always need: authentication
(read and validate the JWT from metadata), tracing (extract or inject W3C trace context so a trace
survives the hop), and metrics. Keeping them out of the service methods is what lets the gRPC adapter
stay a thin adapter over the shared use case.

**Code:**

```java
/** REQ-GR-05. JWT from metadata → validated → put in Context for the service to read. */
class JwtServerInterceptor implements ServerInterceptor {

    private static final Metadata.Key<String> AUTH =
            Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER);
    static final Context.Key<TenantId> TENANT = Context.key("tenant");

    @Override
    public <Q, P> ServerCall.Listener<Q> interceptCall(
            ServerCall<Q, P> call, Metadata headers, ServerCallHandler<Q, P> next) {

        String bearer = headers.get(AUTH);
        TenantId tenant;
        try {
            tenant = jwtVerifier.verify(bearer).tenantId();
        } catch (JwtException e) {
            // Close the call here — the service method is never invoked.
            call.close(Status.UNAUTHENTICATED.withDescription("invalid token"), new Metadata());
            return new ServerCall.Listener<>() {};      // no-op listener
        }
        // Context is immutable and propagates to the handler (and to child calls).
        Context ctx = Context.current().withValue(TENANT, tenant);
        return Contexts.interceptCall(ctx, call, headers, next);
    }
}
```

**Follow-ups they'll ask:**
- *"Ordering?"* — interceptors form a chain and order matters: authenticate before you do expensive
  work, but start the trace span *outside* auth so failed auth still appears in traces.
- *"Why `Context` rather than a `ThreadLocal`?"* — gRPC calls can hop threads, and `Context` is designed
  to propagate correctly across the async boundaries and into child calls (including deadline
  propagation). A raw `ThreadLocal` leaks or loses the value.
- *"How does this relate to Spring Security?"* — you can bridge the gRPC `Context` into Spring's
  `SecurityContext`, but the interceptor is the authoritative place for gRPC. Verify the current
  Spring Security gRPC integration before claiming a specific approach.

**Where in StreamPulse:** BS-09 / REQ-GR-05; tracing interceptor feeds the Phase 8 OTel pipeline.

---

## Q9. How does backpressure work on a gRPC stream?

**30-second answer:** By default the runtime auto-requests messages, so a fast client can outrun a slow
handler and the messages queue in memory — that's the OOM you don't see coming. The fix is manual flow
control: disable auto-request and explicitly call `request(n)` when you're ready for more, which stops
replenishing the HTTP/2 flow-control window and pushes the pressure back to the client's TCP buffer.
On the sending side you check `isReady()` and use an `onReadyHandler` rather than writing blindly. In
BS-09 I tie the inbound `request(n)` to the same in-flight semaphore the REST path uses, so both
protocols apply identical backpressure (REQ-GR-04).

**Deeper:**

```
client pushes ──▶ [HTTP/2 flow-control window] ──▶ server handler
                            ▲                            │
                            └── replenished only when ────┘
                                the handler calls request(n)
```

This is the same principle as `CallerRunsPolicy` in BS-02 and the 429 + `Retry-After` on the REST path:
**never buffer without a bound**. The three honest options when you can't keep up are slow the producer,
shed load explicitly, or drop with a policy — buffering unboundedly just relocates the failure to the
heap.

**Code:**

```java
@Override
public StreamObserver<TelemetryEvent> streamEvents(StreamObserver<Ack> responseObserver) {
    var serverObserver = (ServerCallStreamObserver<Ack>) responseObserver;
    serverObserver.disableAutoRequest();          // take manual control of inbound demand

    var handler = new StreamObserver<TelemetryEvent>() {
        @Override public void onNext(TelemetryEvent e) {
            ingest.accept(ProtoMapper.toDomain(e));
            serverObserver.request(1);            // ask for exactly one more: strict 1-in-1-out
        }
        @Override public void onError(Throwable t) { /* ... */ }
        @Override public void onCompleted() { serverObserver.onCompleted(); }
    };

    serverObserver.request(1);                    // prime the pump
    return handler;
}
```

**Follow-ups they'll ask:**
- *"Why not just request more at once?"* — you can (`request(n)` with a larger n gives pipelining), and
  it's a throughput/memory trade. The point is that *you* choose the bound.
- *"What about the response direction?"* — check `serverObserver.isReady()` before `onNext`, and
  register `setOnReadyHandler` to resume. Writing while not ready is what fills the outbound buffer.
- *"How does this compare to Reactor backpressure?"* — same concept, different API: `request(n)` in
  Reactive Streams *is* this. Good place to link to file 02.

**Where in StreamPulse:** BS-09 / REQ-GR-04.

**Verify:** `ServerCallStreamObserver.disableAutoRequest()`, `isReady()`, `setOnReadyHandler()` method
names against your grpc-java version — this API has changed over time (an older `disableAutoInboundFlowControl()`
existed).

---

## Q10. Does gRPC have built-in retries?

**30-second answer:** Yes — gRPC supports a declarative retry policy (and hedging) configured through
**service config**, so you specify max attempts, backoff, and which status codes are retryable without
writing retry code. It also enforces a retry *throttle* so a failing backend doesn't get hammered.
That said, I'm careful about layering: if I have gRPC retries **and** Resilience4j retries **and** a
client-side loop, one failure becomes dozens of calls. I pick one layer and disable the others.

**Deeper:** hedging is the interesting one — instead of waiting for failure, it sends the request to
another backend after a delay and takes the first response. It cuts tail latency, but **only for
idempotent operations**, because you are deliberately issuing duplicate requests. BS-09's ingest is
idempotent on `event_id`, so hedging would actually be safe there; a payment call would not be.

The retry-amplification arithmetic is worth having ready: 3 retries at each of 3 layers is up to
3 × 3 × 3 = 27 calls from one user action. That's how a partial outage becomes a full one.

**Follow-ups they'll ask:**
- *"Retry vs hedging?"* — retry reacts to failure; hedging acts pre-emptively against slowness. Hedging
  costs extra load on every slow call.
- *"Which layer should own retries?"* — the one closest to the failure that knows whether the operation
  is safe to repeat. In StreamPulse that's Resilience4j at the adapter, because it also has the
  circuit breaker (see file 05 on why Retry wraps CircuitBreaker).

**Where in StreamPulse:** BS-09; interacts with BS-04 / REQ-RS-01.

**Verify:** the gRPC service-config JSON schema (`retryPolicy`, `hedgingPolicy` field names,
`retryableStatusCodes`) and whether retries are enabled by default in your grpc-java version.

---

## Q11. You deploy a gRPC service on Kubernetes and one pod gets all the traffic. Why?

**30-second answer:** Because a Kubernetes `Service` load-balances **connections**, not **requests** —
it's L4. gRPC opens one long-lived HTTP/2 connection and multiplexes every request over it, so once a
client picks a pod, every subsequent request goes to that same pod for the life of the connection. Add
a pod and it gets nothing, because no new connections are being made. The fixes are all about moving
balancing to L7: client-side load balancing against a headless service, a proxy like Envoy, or a
service mesh. This is the single most common gRPC-on-Kubernetes surprise and it's worth naming before
they ask.

**Deeper:**

```
❌ ClusterIP Service (L4):
   client ──one HTTP/2 conn──▶ [kube-proxy picks pod A ONCE] ──▶ pod A  (100% of requests)
                                                                  pod B  (idle)
                                                                  pod C  (idle)

✅ headless Service + client-side LB:
   client resolves ALL pod IPs, keeps a conn to each, round-robins REQUESTS across them
```

| Fix | How | Trade-off |
|---|---|---|
| Client-side LB | headless Service (`clusterIP: None`) + `round_robin` policy in the channel | client must handle endpoint changes; fine for internal Java clients |
| L7 proxy (Envoy/nginx) | proxy terminates HTTP/2 and balances per-request | extra hop, extra component |
| Service mesh (Istio/Linkerd) | sidecar does it transparently | operationally heavy; only worth it if you want the mesh anyway |
| Periodic reconnect | `MAX_CONNECTION_AGE` on the server forces rebalancing | crude, but genuinely effective and cheap |

That last row is underrated: setting a max connection age on the **server** makes clients periodically
reconnect and thus re-balance, which mitigates the problem without any new infrastructure. Naming it
signals you've actually operated this, not just read about it.

**Follow-ups they'll ask:**
- *"Does this affect autoscaling?"* — badly. HPA scales up, the new pods stay idle, CPU stays high on
  the original pods, and it scales up again. You get a cost spike with no relief.
- *"How would you detect it?"* — per-pod request-rate metrics. A flat line on new pods after a scale-up
  is the signature (ties to file 11).
- *"Does the same apply to HTTP/2 REST or database connections?"* — yes to HTTP/2 REST, and connection
  pools have an analogous stickiness. Any long-lived connection over an L4 balancer has this shape.

**Pitfalls / red flags:** not knowing this at all — it's the question that separates "used gRPC in a
tutorial" from "ran gRPC in production." Given your NetApp Kubernetes/gRPC bullet, expect it.

> **Your story:** if you hit connection-stickiness, uneven pod load, or a mesh rollout at NetApp, this
> is where it goes — what the symptom looked like, how it was diagnosed, which fix was chosen and why.
> Keep the claim scoped to what you personally did.

**Where in StreamPulse:** BS-09 / REQ-GR-07; Phase 11.

---

## Q12. Can a browser call gRPC directly?

**30-second answer:** No. Browser JavaScript can't control HTTP/2 frames at the level gRPC needs, so
you use **gRPC-Web**, which is a different wire protocol requiring a translating proxy (Envoy has a
built-in filter). And gRPC-Web doesn't support client streaming or bidirectional streaming — only unary
and server streaming. That limitation is exactly why StreamPulse's dashboard uses **SSE** from the
WebFlux query-api rather than gRPC: gRPC is the device-to-backend and service-to-service protocol,
SSE is the browser protocol.

**Deeper:**

| | gRPC | gRPC-Web | SSE |
|---|---|---|---|
| Browser native | ❌ | needs a proxy | ✅ |
| Client streaming | ✅ | ❌ | ❌ |
| Bidi streaming | ✅ | ❌ | ❌ |
| Server streaming | ✅ | ✅ | ✅ |
| Human-debuggable | ❌ | ❌ | ✅ (it's text) |

StreamPulse's protocol map, which is a good thing to be able to draw:

```
smart TVs ──gRPC bidi──▶ ingest-gateway ──▶ Kafka ──▶ ... ──▶ query-api ──SSE──▶ browser dashboard
other apps ──REST batch──▶ ingest-gateway
internal services ──gRPC unary──▶ each other
```

**Follow-ups they'll ask:**
- *"Why not WebSocket for the dashboard?"* — SSE is one-directional, which is all a dashboard needs; it
  works over plain HTTP, reconnects automatically with `Last-Event-ID`, and is far easier to operate
  through proxies. See file 07.
- *"Isn't maintaining three protocols a cost?"* — it would be, if each had its own logic. They don't:
  all three are adapters over the same use case (Q13 in file 01). That's the payoff of hexagonal.

**Where in StreamPulse:** Phase 5 (SSE) vs BS-09 (gRPC).

---

## Q13. How do you test a gRPC service?

**30-second answer:** Three layers. The use case gets plain unit tests with no gRPC at all — that's the
whole point of keeping the adapter thin. The adapter gets an **in-process server**, which exercises the
real gRPC machinery (interceptors, status mapping, streaming) without opening a socket. Then one
integration test with Testcontainers Kafka proves the whole path. The test I care most about in BS-09 is
the **parity test**: feed identical input through the REST and gRPC adapters and assert the Kafka
records are byte-identical. That's what proves the second protocol didn't fork the business logic.

**Code:**

```java
/** REQ-GR-02: the two protocols must be indistinguishable downstream. */
@Test
void restAndGrpcProduceIdenticalKafkaRecords() {
    var event = sampleEvent("evt-1");

    restClient.postBatch(List.of(event));
    var viaRest = kafkaConsumer.pollOne(RAW_TOPIC);

    grpcStub.sendBatch(Batch.newBuilder().addEvents(toProto(event)).build());
    var viaGrpc = kafkaConsumer.pollOne(RAW_TOPIC);

    assertThat(viaGrpc.key()).isEqualTo(viaRest.key());          // same partition key
    assertThat(viaGrpc.value()).isEqualTo(viaRest.value());      // same Avro payload
}
```

**Follow-ups they'll ask:**
- *"How do you test a deadline?"* — set a very short deadline against a deliberately slow handler and
  assert `DEADLINE_EXCEEDED` on the client plus `Context.isCancelled()` observed on the server. Both
  halves matter: the client seeing the error isn't proof the server stopped working.
- *"How do you test streaming?"* — a `StreamObserver` test double collecting emissions, with `awaitility`
  for the async assertions rather than `Thread.sleep`.
- *"Contract testing for gRPC?"* — the proto plus the CI breaking-change check *is* most of it; that's
  a real advantage over REST, where you need Pact to get the same guarantee.

**Where in StreamPulse:** BS-09 / REQ-GR-02; `GrpcIngestIT`, `ProtocolParityTest`.

**Verify:** grpc-java's test helpers — `GrpcCleanupRule` is a JUnit 4 `TestRule`; confirm the current
recommended JUnit 5 approach (typically managing `InProcessServerBuilder`/`InProcessChannelBuilder`
lifecycles yourself) before relying on a specific class name.

---

## Q14. Walk me through how you added gRPC ingest to StreamPulse without duplicating logic.

*(The applied question — this is where it becomes evidence rather than knowledge.)*

**30-second answer:** The REST path already existed, so the design constraint was that gRPC must not
fork the business logic. Because the service was hexagonal, both adapters call the same `IngestUseCase`
port — validation, idempotency, partition-key selection and publishing all live in the domain, and the
adapters only translate wire formats and map errors. The gRPC-specific work was four things: the proto
contract with a CI breaking-change check; a bidi `StreamEvents` method with manual flow control tied to
the *same* in-flight semaphore as REST, so backpressure is identical; status-code mapping from domain
exceptions; and a JWT interceptor. Then I benchmarked both protocols and wrote down the trade-off
rather than assuming gRPC won. The test I'd point at is `ProtocolParityTest` — identical input, identical
Kafka records.

**The design decisions and why:**

| Decision | Why | What I'd reconsider |
|---|---|---|
| Bidi, not client streaming | devices need durable-sequence acks to resume after a drop | client streaming if acks weren't needed |
| Backpressure shared with REST | one saturation policy, not two that drift | — |
| Idempotency on `event_id`, not on the protocol | makes `DEADLINE_EXCEEDED` retries safe | — |
| gRPC for devices, SSE for browsers | gRPC-Web can't do bidi anyway | — |
| Client-side LB / max connection age | L4 Service would pin all devices to one pod | a mesh, if we already ran one |

**Follow-ups they'll ask:**
- *"Was gRPC actually worth it?"* — that's what the benchmark is for (REQ-GR-06). The honest answer is
  that it depends on the payload size and connection pattern, and for a 10s heartbeat from millions of
  devices the header compression and connection reuse matter more than the payload encoding. I'd give
  the measured numbers, not a guess.
- *"What would you do differently?"* — I'd question whether two ingest protocols are worth the
  maintenance at all. The parity test makes it safe, but "supported protocols" is a long-term cost, and
  I'd want a real client demanding it before adding one.
- *"How does this interact with the K8s deployment?"* — directly; see Q11. Long-lived device connections
  are exactly the workload that breaks L4 balancing.

> **Your story:** your NetApp bullet is a Kubernetes/gRPC microservice with a 99.98% uptime SLA. The
> two things to have ready cold are the **mechanism** (what the service did, what the gRPC contract
> looked like, how clients connected and were balanced) and **what kept the SLA there** — which ties to
> file 11 on error budgets and file 13 on probes, PDBs and rolling updates. Use "I built" only for what
> you personally wrote.

**Where in StreamPulse:** BS-09 end to end; REQ-GR-01…07; Phase 2.

---

## Key terms

| Term | One-line meaning |
|---|---|
| **Wire type** | The 3-bit code packed into the tag that says how to decode the following bytes. |
| **Field number** | The permanent identity of a field on the wire; names are never transmitted. |
| **`reserved`** | Declares retired field numbers/names so the compiler refuses to let anyone reuse them. |
| **`oneof`** | A discriminated union — at most one member is set; setting one clears the others. |
| **Deadline** | An absolute time that travels with the request and propagates to every downstream hop. |
| **`Context`** | gRPC's request-scoped, immutable carrier for deadline, cancellation and values. |
| **Trailers** | Metadata sent *after* the response body — where status and error details live. |
| **Service config** | Declarative per-method policy (retries, hedging, timeouts) delivered to the client. |
| **Hedging** | Pre-emptively sending duplicate requests to cut tail latency; idempotent operations only. |
| **gRPC-Web** | A browser-compatible variant requiring a proxy; no client or bidi streaming. |
| **Headless Service** | A K8s Service with `clusterIP: None` that returns all pod IPs, enabling client-side LB. |
| **HPACK** | HTTP/2 header compression — the reason tiny, frequent messages are cheap. |

---

## Self-check (answer out loud, no notes)

1. Why can renaming a Protobuf field be safe on the wire but still break your build?
2. State the difference between a client timeout and a gRPC deadline in one sentence, then say why it
   matters under load.
3. Which three status codes are ambiguous about whether the work happened, and what makes retrying
   them safe anyway?
4. A gRPC service on Kubernetes sends all traffic to one pod after a scale-up. Explain the cause and
   name three fixes.
5. What single test proves that adding gRPC didn't fork your business logic, and what exactly does it
   assert?
