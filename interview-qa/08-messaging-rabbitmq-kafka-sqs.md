# 08 — Messaging: RabbitMQ Internals, and the Broker Decision Table

**Why this matters for an Event Hub role:** "we use Kafka" is not an architecture, it's a default. The
senior signal is being able to say *why* a broker fits a problem and where it stops fitting — and the
fastest way to demonstrate that is to know a second broker properly. RabbitMQ is the right second
broker to know, because it is Kafka's opposite on almost every axis: a **smart broker with dumb
consumers** (routing, per-message acknowledgement, per-message TTL, priorities, redelivery tracking)
versus Kafka's **dumb broker with smart consumers** (an immutable log, offsets owned by the client,
replay for free). Once you can articulate that one sentence, the whole decision table falls out of it.

**Scenario anchor — BS-07 (REQ-MQ-01…07):** broadcast partners want to subscribe to *subsets* of
alerts — "all CRITICAL for `sportsnow`", "any CDN alert for any tenant". Each partner has its own
endpoint and its own reliability. Modelling that in Kafka means a topic per partner (which explodes)
or every partner consuming everything and filtering (wasteful, and no per-message retry). Routing by
pattern, per-message acks, per-partner retry and a dead-letter queue are precisely RabbitMQ's
strengths, so the alert engine publishes to a **topic exchange** and each partner gets a durable queue
bound with its own pattern. Phase 6. Secondary anchor: **BS-04 / REQ-RS-03, REQ-RS-05** — partner
calls go through the rate limiter and per-tenant bulkhead, and the rate limiter's fallback *is* the
RabbitMQ queue.

> **Version note:** RabbitMQ 3.8 added quorum queues, 3.9 added streams, and classic **mirrored**
> queues were deprecated and then removed in the 4.x line. Several defaults and queue-type
> capabilities changed with it. **Verify** the queue types, capabilities and defaults against the
> RabbitMQ version you actually run. AWS service quotas quoted later are documented values as I
> recall them — **verify in the current AWS docs** before quoting a number in a design review.

---

## Q1. Explain RabbitMQ's model: exchanges, queues, bindings, routing keys.

**30-second answer:** In AMQP a publisher never publishes to a queue — it publishes to an **exchange**
with a **routing key**. The exchange holds **bindings** to queues, and its type decides how the
routing key is matched against those bindings. Queues hold the messages; consumers consume from
queues. That indirection is the whole point: the publisher knows nothing about who is listening, and
you can add a subscriber by declaring a queue and a binding — **no publisher change, no deploy**. In
StreamPulse that's REQ-MQ-02 literally: `POST /v1/partners/{id}/subscriptions {pattern: "..."}`
declares a durable queue and binds it, and the alert engine is unaware anything changed.

**Deeper:**

```
                                  ┌────── binding: "alert.sportsnow.CRITICAL.*" ──▶ [ partner.acme ]
                                  │
publisher ──▶ [ exchange alerts.x ]──── binding: "alert.*.*.cdn" ─────────────────▶ [ partner.globo ]
   routing key:  type=topic       │
   alert.sportsnow.CRITICAL.cdn   └────── binding: "alert.#" ───────────────────────▶ [ audit.all ]

  ⚠ this one message matched THREE bindings → RabbitMQ places a COPY in each queue.
    Queues are independent: acking in one has no effect on the others.
```

The pieces, precisely:

| Concept | What it is | Notes |
|---|---|---|
| **Exchange** | a routing function | has a type (direct/topic/fanout/headers), durability, and an `alternate-exchange` |
| **Routing key** | a string set by the publisher | matched per exchange type; ignored entirely by `fanout` |
| **Binding** | exchange → queue (or exchange → exchange) + a binding key | many-to-many; a message can be copied to many queues |
| **Queue** | an ordered buffer with consumers | durable/transient, exclusive, auto-delete, plus arguments (`x-*`) |
| **Vhost** | a namespace for all of the above | the isolation unit — separate permissions, separate everything |
| **Channel** | a lightweight session on a TCP connection | acks, prefetch and confirms are **per channel**; not thread-safe |
| **Default exchange** | the nameless `""` direct exchange | every queue is implicitly bound to it by its own name — this is why "publish to a queue" *appears* to work |

**Durability is three independent things**, and getting one of them wrong is the classic RabbitMQ data
loss:
1. **Durable exchange** — survives a broker restart.
2. **Durable queue** — the queue definition survives a restart.
3. **Persistent message** (`delivery_mode=2`) — the *message* is written to disk.

A persistent message on a transient queue dies with the queue. A non-persistent message on a durable
queue dies with the broker. You need all three, plus publisher confirms (Q3), for "the broker has it".
And even then: a persistent message is written to disk but the confirm may arrive before fsync unless
the queue type guarantees otherwise — which is one of the arguments for quorum queues (Q7).

**Code:** REQ-MQ-01/02, declared with Spring AMQP:

```java
@Configuration
class RabbitTopology {

    /** REQ-MQ-01. Durable topic exchange — the single publication point for every alert. */
    @Bean
    TopicExchange alertsExchange() {
        return ExchangeBuilder.topicExchange("alerts.x")
                .durable(true)
                // Unroutable messages (no binding matched) land here instead of vanishing.
                // Pairs with the mandatory flag (Q3) — belt and braces.
                .alternate("alerts.unrouted.x")
                .build();
    }

    /**
     * REQ-MQ-02. Declared at RUNTIME when a partner registers, not at startup —
     * that's the whole point of the exchange indirection.
     */
    @Service
    static class PartnerSubscriptionService {

        private final RabbitAdmin admin;

        void subscribe(String partnerId, String pattern) {
            var queue = QueueBuilder.durable("partner." + partnerId)
                    .deadLetterExchange("partner.dlx")                  // Q6
                    .deadLetterRoutingKey("partner." + partnerId + ".dlq")
                    .quorum()                                           // Q7
                    .build();

            admin.declareQueue(queue);
            admin.declareBinding(BindingBuilder.bind(queue)
                    .to(new TopicExchange("alerts.x"))
                    .with(pattern));        // e.g. "alert.sportsnow.CRITICAL.*"
            // Declarations are IDEMPOTENT — re-declaring identical definitions is a no-op.
            // Re-declaring with DIFFERENT arguments is a channel-level error (406 PRECONDITION_FAILED),
            // which is a real operational trap on deploys that change queue arguments.
        }
    }
}
```

**Follow-ups they'll ask:**
- *"What happens to a message that matches no binding?"* — it is **silently discarded** by default.
  Two defences: the `mandatory` flag (the broker returns it to the publisher, Q3) and an
  `alternate-exchange` on the exchange (the broker routes it to a catch-all). REQ-MQ-03 requires the
  first; I'd use both.
- *"Can an exchange bind to another exchange?"* — yes, exchange-to-exchange bindings, which let you
  build routing trees (e.g. a fanout that feeds several topic exchanges). Powerful and easy to make
  unreadable.
- *"What's a channel and why does it matter?"* — a multiplexed session on one TCP connection.
  Channels are **not thread-safe**, so one channel per thread; Spring AMQP's `CachingConnectionFactory`
  handles this with a channel cache, and a too-small `channelCacheSize` under high concurrency
  silently serialises your publishers.

**Pitfalls / red flags:** saying "publish to a queue". The exchange indirection *is* RabbitMQ's model,
and skipping it means you don't have the mental model.

**Where in StreamPulse:** BS-07 / REQ-MQ-01, REQ-MQ-02; Phase 6; `PartnerRoutingIT`.

---

## Q2. The four exchange types — when do you use each?

**30-second answer:** **Direct** matches the routing key exactly — a simple work-queue or a
command-to-one-handler. **Topic** matches the routing key against a dotted pattern with wildcards
(`*` = exactly one word, `#` = zero or more words) — this is the flexible one and what StreamPulse
uses. **Fanout** ignores the routing key entirely and copies to every bound queue — broadcast/cache
invalidation. **Headers** matches on message header attributes instead of the routing key, with
`x-match=all|any` — useful when your routing dimensions aren't naturally hierarchical, but slower and
much rarer. My rule: start with topic unless you specifically need fanout's broadcast or direct's
simplicity; headers only when the routing criteria genuinely can't be flattened into a dotted string.

**Deeper:**

| Type | Matches on | Wildcards | Cost | Use for |
|---|---|---|---|---|
| `direct` | routing key **==** binding key | none | cheapest | work queues, command routing, log severity levels |
| `topic` | routing key vs dotted pattern | `*` = one word, `#` = zero or more | cheap | **hierarchical subscription** — BS-07 |
| `fanout` | nothing — copies to all bound queues | n/a | cheapest | broadcast, cache invalidation, "every service needs this" |
| `headers` | message headers, `x-match=all` or `any` | n/a | slowest | multi-dimensional, non-hierarchical routing |

**The StreamPulse routing key design (REQ-MQ-01)** — worth explaining as a *design*, because the key
structure is what determines which subscriptions are expressible:

```
routing key:   alert . <tenant> . <severity> . <category>
example:       alert . sportsnow . CRITICAL . cdn

  most general ────────────────────────────▶ most specific
  Order matters: the hierarchy must go general → specific, because `*` matches ONE word
  positionally and `#` is most useful as a suffix.

binding pattern                       matches
────────────────────────────────────────────────────────────────────────────
alert.sportsnow.CRITICAL.*            every CRITICAL sportsnow alert, any category
alert.sportsnow.#                     everything for sportsnow
alert.*.CRITICAL.*                    every CRITICAL alert, any tenant, any category
alert.*.*.cdn                         every CDN alert for any tenant   (REQ-MQ-02's 2nd example)
alert.#                               everything (the audit queue)
alert.*.CRITICAL.cdn                  exactly critical CDN alerts
```

Note what you **cannot** express: "CRITICAL or HIGH but not for kidsplus". Topic patterns have no
negation and no OR across positions. The options are two bindings to the same queue (OR is fine —
bind the queue twice; RabbitMQ delivers **one copy per queue** even if several bindings match, so
there's no duplication), or a headers exchange, or filtering in the consumer. That limitation is worth
naming: it's exactly the sort of thing that looks fine on a whiteboard and bites in month three.

Also worth knowing: **`#` matches zero or more words**, so `alert.#` matches `alert` itself.
And a common gotcha — `*` does not match across dots, so `alert.*` does **not** match
`alert.sportsnow.CRITICAL.cdn`.

**Code:**

```java
/**
 * REQ-MQ-01. Building the routing key IS the routing design. Keep it total and stable:
 * a null or free-text segment here silently breaks every partner's binding.
 */
String routingKey(AlertEvent a) {
    return String.join(".",
            "alert",
            a.tenantId(),                       // sportsnow | streamflix | kidsplus
            a.severity().name(),                // CRITICAL | HIGH | WARN
            a.category().name().toLowerCase()); // cdn | packager | player | origin
    // ⚠ Validate these at the source: a tenant id containing a '.' creates a phantom level
    //    and quietly stops matching every existing binding. Enforce [a-z0-9-]+ in the contract.
}
```

```java
// Fanout, for the case where routing would be pointless — every service must invalidate.
// (StreamPulse actually does this over a COMPACTED KAFKA TOPIC instead — config.changes.v1 —
//  because a new/restarted service must be able to REBUILD its whole cache from history,
//  which a fanout exchange cannot give it. That contrast is the Q12 answer in miniature.)
@Bean FanoutExchange cacheInvalidation() {
    return new FanoutExchange("cache.invalidate.x", true, false);
}
```

**Follow-ups they'll ask:**
- *"Why not one queue per severity instead of patterns?"* — because the subscription dimension is
  per *partner*, and each partner wants a different slice. Queue-per-slice would mean every partner's
  requirements are baked into the topology at deploy time. Pattern binding moves it to runtime, which
  is REQ-MQ-02's requirement.
- *"Does a message matching two bindings to the same queue get delivered twice?"* — no, once per
  queue. Two bindings to two *different* queues means two copies, one per queue.
- *"How many bindings can an exchange have?"* — a lot, but topic routing is a trie walk per message,
  so very large binding sets cost CPU per publish. With hundreds of partners it's fine; with hundreds
  of thousands you'd rethink the model.

**Pitfalls / red flags:** confusing `*` and `#`, or not knowing fanout ignores the routing key.

**Where in StreamPulse:** BS-07 / REQ-MQ-01, REQ-MQ-02; `PartnerRoutingIT` asserts that *only*
matching queues receive a given alert.

---

## Q3. Publisher confirms vs transactions, and what does the `mandatory` flag do?

**30-second answer:** Both answer "did the broker take responsibility for my message?", but only one
is usable at volume. **AMQP transactions** (`tx.select`/`tx.commit`) are synchronous and block per
message — the throughput cost is severe. **Publisher confirms** are the asynchronous alternative: you
put the channel in confirm mode, the broker sends `basic.ack` (or `basic.nack`) per message with a
delivery tag, and you keep publishing meanwhile. That's the right mechanism, and REQ-MQ-03 requires
it. The **`mandatory` flag** answers a *different* question: confirms tell you the broker accepted the
message, but a message that matched **no binding** is accepted and then discarded — so a confirm alone
does not mean "someone will receive this". `mandatory=true` makes the broker **return** an unroutable
message to the publisher (a `basic.return`), which is how you detect a partner whose queue was never
declared. You need both: confirm = "the broker has it", return = "but nobody is bound to receive it".

**Deeper:**

```
publish ──▶ [ exchange ] ──▶ routed to ≥ 1 queue ──▶ basic.ack (confirm)      ✅ safe
                        │
                        └──▶ matched NO binding
                               ├─ mandatory=false → silently DROPPED, and you still get an ACK ⚠
                               └─ mandatory=true  → basic.return to publisher, THEN an ack

nack (basic.nack) = the broker could NOT take responsibility (internal error, disk alarm).
                    Rare, and the case people forget to handle.
```

| | Transactions (`tx.*`) | Publisher confirms |
|---|---|---|
| Mode | synchronous, blocking | asynchronous, pipelined |
| Throughput | poor | high |
| Granularity | per commit (batch) | per message (delivery tag), with a `multiple` flag |
| Tells you | committed to the broker | ack/nack per message |
| Routing failure | not covered | not covered — that's `mandatory` |
| Verdict | essentially never, for messaging | **the default** |

**The ordering subtlety that catches people:** when a message is unroutable and `mandatory` is set,
the broker sends the `basic.return` **before** the `basic.ack`. So a naive `ConfirmCallback` that
treats "acked" as success will mark a returned message as delivered. Spring AMQP handles the
correlation for you if you use `CorrelationData` with a `ReturnsCallback` — and the important
practical rule is: **don't clear your outbox row on the confirm alone; clear it on "confirmed and not
returned".**

**Confirms are also asynchronous relative to your business transaction**, which is the deeper design
point and why REQ-MQ-03 says "keep the alert in the outbox for retry". If you publish inside a
database transaction and the confirm arrives after commit, you have a dual-write problem: the DB says
"sent" and the broker may nack. The only clean solution is the **transactional outbox** — write the
alert row and the outbox row in one DB transaction, and have a separate publisher drain the outbox and
delete rows only on a positive confirm. That's exactly BS-05 / REQ-DL-06 and file 07's topic, and
naming the connection is a strong senior signal.

**Code:**

```yaml
spring:
  rabbitmq:
    publisher-confirm-type: correlated   # 'correlated' carries YOUR CorrelationData into the
                                         # callback so you can identify WHICH message was acked.
                                         # 'simple' gives you waitForConfirms() blocking semantics.
    publisher-returns: true              # enable basic.return delivery
    template:
      mandatory: true                    # REQ-MQ-03 — required for returns to happen at all
```

```java
/**
 * REQ-MQ-03. Confirms + mandatory + outbox. The outbox row is the source of truth;
 * the callbacks only DELETE it. If the process dies between publish and confirm,
 * the row survives and the sweeper republishes — at-least-once with an idempotent
 * consumer downstream (dedup on alertId, see the Kafka file Q7).
 */
@Configuration
class PublisherConfirms {

    PublisherConfirms(RabbitTemplate template, OutboxRepository outbox, MeterRegistry meters) {

        template.setConfirmCallback((correlation, ack, cause) -> {
            String alertId = ((AlertCorrelation) correlation).alertId();
            if (ack) {
                outbox.markConfirmed(alertId);     // NOT deleted yet — a return may still arrive
            } else {
                // NACK: the broker refused responsibility (disk alarm, internal error).
                // Leave the outbox row; the sweeper retries with backoff.
                meters.counter("rabbit.publish.nack").increment();
                log.error("Broker nacked alert {}: {}", alertId, cause);
            }
        });

        // ReturnsCallback fires for UNROUTABLE messages — i.e. no partner queue is bound
        // to this pattern. This is a CONFIGURATION bug (a partner registered but the queue
        // was never declared), so it must be loud, not a silent drop.
        template.setReturnsCallback(returned -> {
            meters.counter("rabbit.publish.unroutable",
                           "routingKey", returned.getRoutingKey()).increment();
            log.error("Unroutable alert, routingKey={} replyText={}",
                      returned.getRoutingKey(), returned.getReplyText());
            outbox.markUnroutable(returned.getMessage().getMessageProperties().getCorrelationId());
        });
    }
}
```

```java
/**
 * The publish itself. Persistent delivery mode + durable queue + durable exchange +
 * confirm = the full durability chain. Miss any link and the message is at risk.
 */
void publish(AlertEvent alert) {
    rabbitTemplate.convertAndSend(
            "alerts.x",
            routingKey(alert),
            alert,
            message -> {
                message.getMessageProperties()
                       .setDeliveryMode(MessageDeliveryMode.PERSISTENT);   // delivery_mode=2
                message.getMessageProperties().setMessageId(alert.alertId());
                return message;
            },
            new AlertCorrelation(alert.alertId()));     // → arrives in the ConfirmCallback
}
```

**Follow-ups they'll ask:**
- *"Does an ack mean the message is on disk?"* — for a persistent message on a durable queue, the
  broker acks after it considers the message safe, but the exact fsync semantics depend on the queue
  type and version. Quorum queues give a stronger, Raft-replicated guarantee (Q7). **Verify** the
  precise durability contract for your version before promising it to anyone.
- *"How do you handle the async confirm without leaking memory?"* — you're tracking outstanding
  messages somewhere. If that's an in-memory map, it grows during a broker slowdown. The outbox *is*
  the bounded, durable version of that map — another reason to use it.
- *"Publisher confirms vs Kafka's `acks=all`?"* — the same idea at the same place in the design: "has
  the broker taken durable responsibility?". Kafka layers idempotence on top (PID + sequence numbers)
  so a retry after a lost ack doesn't duplicate; RabbitMQ has no equivalent producer-side dedup, so a
  republish after a lost confirm **does** create a duplicate and the consumer must dedup. That's a
  genuine, concrete difference between the two brokers.

**Pitfalls / red flags:** using transactions for throughput-sensitive publishing, or believing a
confirm means someone will receive the message.

**Where in StreamPulse:** BS-07 / REQ-MQ-03, `PublisherConfirmIT`; links to BS-05 / REQ-DL-06
(outbox).

**Verify:** `publisher-confirm-type` values, `setReturnsCallback` vs the deprecated
`setReturnCallback`, and fsync semantics per queue type in your RabbitMQ version.

---

## Q4. Consumer acknowledgement: ack, nack, reject, requeue. What does `redelivered` mean?

**30-second answer:** RabbitMQ tracks delivery **per message**, which is the fundamental difference
from Kafka's per-partition offset. `basic.ack(deliveryTag, multiple)` says "done, delete it".
`basic.reject(deliveryTag, requeue)` rejects one message; `basic.nack(deliveryTag, multiple, requeue)`
is the RabbitMQ extension that can reject many at once. The `requeue` flag is the decision that
matters: `requeue=true` puts the message back for redelivery, `requeue=false` **dead-letters** it if a
DLX is configured and discards it otherwise. The `redelivered` flag on a delivery tells the consumer
"this message has been delivered before" — it's a hint, not a count, so you cannot build a retry limit
from it. Blind `requeue=true` on a permanent failure is the classic RabbitMQ incident: an infinite
hot loop where one poison message is redelivered thousands of times per second.

**Deeper:**

```
consumer receives delivery (deliveryTag=42, redelivered=false)
   │
   ├─ success ──────────────▶ basic.ack(42)                      message deleted
   │
   ├─ transient failure ────▶ basic.nack(42, requeue=true)       redelivered, redelivered=true
   │                                                              ⚠ immediately, with NO delay →
   │                                                                 busy-loop if it always fails
   │
   ├─ permanent failure ────▶ basic.nack(42, requeue=false)      → DLX if configured
   │                                                              → DISCARDED if not ⚠
   │
   └─ consumer dies / channel closes ──────────────────────────▶ automatically requeued
                                                                  (redelivered=true on next delivery)
```

| Mechanism | Scope | Notes |
|---|---|---|
| `basic.ack(tag, multiple)` | one, or all up to the tag | `multiple=true` is the batching optimisation |
| `basic.reject(tag, requeue)` | exactly one | the original AMQP method |
| `basic.nack(tag, multiple, requeue)` | one or many | RabbitMQ extension; the one you'll actually use |
| `redelivered` flag | per delivery | a **boolean hint**, not a counter; can be true even on the first delivery to *this* consumer |
| `x-delivery-limit` | per queue (quorum queues) | a real, broker-enforced redelivery counter → DLX |

**The `redelivered` trap** is worth stating precisely because it sounds like it solves the problem and
doesn't: it tells you the broker *thinks* this message may have been delivered before (e.g. a previous
consumer died without acking). It does not tell you how many times, and it's not reliable as a retry
counter. If you need "give up after 3 attempts" you have three real options: a per-message header you
increment yourself when you republish (Q6's TTL retry chain), Spring AMQP's stateful retry interceptor
(Q9), or — cleanest — a **quorum queue with `x-delivery-limit`**, where the broker counts and
dead-letters for you.

**Acknowledge modes in Spring AMQP:**

| Mode | Behaviour | Use when |
|---|---|---|
| `AUTO` (default) | the container acks if the listener returns normally, nacks if it throws | most cases; still gives at-least-once |
| `MANUAL` | your method takes a `Channel` + delivery tag and acks explicitly | you need to ack after an external durable write |
| `NONE` | no acks at all — fire and forget, broker deletes on delivery | at-most-once; metrics, logs, things you can lose |

REQ-MQ-04 specifies **manual ack** for partner delivery workers, and the reason is the same as the
Kafka one: the ack must happen *after* the partner confirmed receipt, not when the method returned.

**Code:**

```java
/**
 * REQ-MQ-04 — manual ack. The ordering of the last two lines is the whole delivery
 * guarantee: deliver first, ack second. Swap them and you have at-most-once (silent loss).
 */
@RabbitListener(queues = "partner.acme", ackMode = "MANUAL")
public void deliver(AlertEvent alert,
                    Channel channel,
                    @Header(AmqpHeaders.DELIVERY_TAG) long tag,
                    @Header(name = AmqpHeaders.REDELIVERED, required = false) Boolean redelivered) {
    try {
        // BS-04: this call is wrapped in the per-tenant bulkhead + rate limiter (REQ-MQ-06).
        partnerClient.deliver(alert, alert.alertId());   // idempotency key → partner can dedup
        channel.basicAck(tag, false);

    } catch (PermanentDeliveryException e) {
        // 4xx from the partner: a bad payload or an unknown endpoint. Retrying is pointless.
        // requeue=false → straight to the DLX. No retry chain, no wasted attempts.
        channel.basicNack(tag, false, false);

    } catch (TransientDeliveryException e) {
        // 5xx / timeout. requeue=false ALSO, because our DLX-based retry chain (Q6) applies
        // the DELAY. requeue=true would redeliver instantly and spin the CPU.
        channel.basicNack(tag, false, false);

    } catch (Exception e) {
        // Unknown: treat as permanent to avoid an infinite loop, and alert on the DLQ rate.
        log.error("Unclassified failure delivering {}", alert.alertId(), e);
        channel.basicNack(tag, false, false);
    }
}
```

```java
// ❌ THE CLASSIC INCIDENT — never write this.
// A permanently-failing message is requeued with no delay, redelivered immediately,
// fails again... thousands of times per second. CPU at 100%, the queue never drains,
// and every OTHER message behind it is starved. This is RabbitMQ's poison-pill loop.
// catch (Exception e) { channel.basicNack(tag, false, true); }
```

**Follow-ups they'll ask:**
- *"Where does a requeued message go in the queue?"* — RabbitMQ attempts to return it to its
  **original position** (approximately the head), which is what makes the busy-loop so tight: it's
  redelivered essentially immediately, ahead of newer messages.
- *"Spring AMQP equivalents without touching the Channel?"* — throw `AmqpRejectAndDontRequeueException`
  to force `requeue=false` (→ DLX), or `ImmediateRequeueAmqpException` to force `requeue=true`. And
  `spring.rabbitmq.listener.simple.default-requeue-rejected` sets the default for ordinary exceptions
  — **verify the default value** for your version; it has historically been `true`, which is the
  dangerous direction, so set it explicitly.
- *"How does this compare to Kafka?"* — Kafka has no per-message ack at all; it has a per-partition
  cursor. So Kafka cannot "ack message 7 but not message 5" — a failure at 5 means either you skip it
  (DLT) or you stop. RabbitMQ's per-message ack is exactly what makes per-partner retry and DLQ
  natural, and it's the single strongest argument for RabbitMQ in BS-07.

**Pitfalls / red flags:** `requeue=true` in a generic catch block, or using `redelivered` as a retry
counter.

**Where in StreamPulse:** BS-07 / REQ-MQ-04; `DlqRetryIT` (failing WireMock partner → 3 retries → DLQ
→ replay).

**Verify:** the `default-requeue-rejected` default, and `ackMode` attribute support on
`@RabbitListener` in your Spring AMQP version.

---

## Q5. What does prefetch (`basic.qos`) do, and how does it affect fairness?

**30-second answer:** Prefetch caps how many **unacknowledged** messages the broker will push to a
consumer at once. Without it (prefetch = unlimited, the AMQP default) the broker shoves everything it
has at the first consumer that connects — so one consumer buys the whole queue into its local buffer
while the others sit idle, and if it dies, all of that has to be requeued. That's the "unfair
dispatch" problem. Setting prefetch low (1–10) gives you round-robin-ish fairness and fast failover;
setting it high gives you throughput, because the consumer never waits for a network round-trip
between messages. The trade is fairness and memory versus throughput, and the right value depends on
message processing time: fast handlers want a high prefetch, slow handlers want a low one. REQ-MQ-05
makes it per-partner and defaults it to 10.

**Deeper:**

```
prefetch = unlimited (the AMQP default)
  queue: [1..10000]
  consumer A connects first ──▶ receives ALL 10000 into its local buffer
  consumer B connects       ──▶ receives NOTHING until A acks
  A crashes                 ──▶ ~10000 messages requeued, long recovery, duplicates

prefetch = 1
  A: [1]   B: [2]   C: [3]    strict fairness; a slow consumer holds up only its own message
  ❌ every message costs a full round-trip: ack → broker → next deliver. Throughput-bound.

prefetch = 10  (REQ-MQ-05 default)
  A: [1..10]  B: [11..20]     pipelined delivery, bounded unfairness, bounded crash loss
```

| Prefetch | Fairness | Throughput | Memory per consumer | Crash cost |
|---|---|---|---|---|
| 1 | best | worst (round-trip bound) | minimal | 1 message requeued |
| 10–50 | good | good | bounded | up to N requeued |
| unlimited | worst | best in a benchmark | **unbounded** — an OOM risk | the whole queue requeued |

**Sizing heuristic:** prefetch ≈ `(round-trip latency + processing time) / processing time`, rounded
up — the number of messages needed to keep the consumer busy while the ack round-trip completes. For a
3-second partner HTTP call, that's ≈ 1, and you'd pick a small number. For a sub-millisecond handler,
you want tens or hundreds. Like most such formulas it's a starting point for measurement, not a fact.

**Prefetch is per channel (or per consumer)**, which is the detail that surprises people: with
`global=false` (the normal case) the limit applies to each consumer on the channel, so a channel with
5 consumers at prefetch 10 can hold 50 unacked messages. Spring AMQP's `SimpleMessageListenerContainer`
with `concurrency=5` and `prefetch=10` therefore holds up to 50 in flight per container.

**The memory story matters at scale:** unacked messages are held in the consumer's local buffer *and*
tracked by the broker. A high prefetch across many consumers is how you turn a queue backlog into
broker memory pressure, which triggers a **memory alarm**, which **blocks publishers** (Q10). So
prefetch is not only a consumer-side setting; at scale it's a broker stability setting.

**Code:**

```yaml
spring:
  rabbitmq:
    listener:
      simple:
        prefetch: 10            # REQ-MQ-05 default; overridden per partner below
        concurrency: 2          # initial consumer threads per listener
        max-concurrency: 8      # the container scales consumers between these
        acknowledge-mode: manual
        default-requeue-rejected: false   # ← set EXPLICITLY (see Q4). The default has
                                          #   historically been true, which busy-loops.
```

```java
/**
 * REQ-MQ-05 — prefetch per partner, because partners differ wildly in latency.
 * A fast partner (20ms) can pipeline; a slow one (3s) should hold ~1 message so that
 * its backlog stays in the QUEUE (visible, measurable) rather than in a consumer's
 * local buffer (invisible, and lost-then-requeued on a crash).
 */
@Bean
SimpleRabbitListenerContainerFactory partnerFactory(
        SimpleRabbitListenerContainerFactoryConfigurer configurer,
        ConnectionFactory connectionFactory,
        PartnerProperties props) {

    var factory = new SimpleRabbitListenerContainerFactory();
    configurer.configure(factory, connectionFactory);
    factory.setPrefetchCount(props.prefetch());          // default 10
    factory.setAcknowledgeMode(AcknowledgeMode.MANUAL);
    factory.setConcurrentConsumers(2);
    factory.setMaxConcurrentConsumers(8);
    // Bound how long the container waits on shutdown — pairs with BS-01's ordered drain.
    factory.setShutdownTimeout(20_000);
    return factory;
}
```

**Follow-ups they'll ask:**
- *"`SimpleMessageListenerContainer` vs `DirectMessageListenerContainer`?"* — `Simple` has its own
  thread pool and can scale consumers dynamically between `concurrency` and `maxConcurrency`;
  `Direct` dispatches directly on the connection factory's threads with a fixed consumer count and
  lower latency/overhead. `Simple` is the safer default when handler time varies; `Direct` when you
  want predictable low latency and control the consumer count yourself. **Verify** the current
  trade-off notes in the Spring AMQP reference for your version.
- *"How is this different from Kafka's `max.poll.records`?"* — superficially similar (both bound
  in-flight work) but structurally different: Kafka's is a *pull* limit the consumer chooses, and
  unprocessed records simply aren't fetched. RabbitMQ's is a *push* limit the broker enforces, and
  prefetched messages are already out of the queue and in the consumer's buffer. Kafka's model makes
  backlog visible as lag; RabbitMQ's can hide it in consumer memory.
- *"What's the right prefetch for a 3-second partner call?"* — 1 or 2. Anything more just moves the
  backlog into the consumer, where you can't see it and a crash requeues it.

**Pitfalls / red flags:** leaving prefetch unlimited and then being surprised that one consumer takes
everything, or that a broker memory alarm blocked publishers.

**Where in StreamPulse:** BS-07 / REQ-MQ-05, and the ADR on fairness vs memory.

**Verify:** Spring AMQP's default prefetch value for your version (it has been a fairly large number —
in the hundreds — for some time; do not assume 1).

---

## Q6. Explain DLX, and how do you build a delayed-retry chain with TTL?

**30-second answer:** A **dead-letter exchange** is a per-queue argument: when a message is
dead-lettered, RabbitMQ republishes it to that exchange instead of discarding it. Three things
dead-letter a message: it's rejected with `requeue=false`, its TTL expires, or the queue hits a
length/size limit. That third trigger is the trick that builds delayed retries **without a plugin**:
create a queue whose *only* purpose is to hold messages for N milliseconds (`x-message-ttl`) with no
consumer and a DLX pointing back at the work exchange. A message rejected by the worker goes to
`retry.1s`, sits there for one second, expires, is dead-lettered back to the work queue, and is
retried. Chain three of those with increasing TTLs and you have exponential backoff, entirely
broker-side, surviving consumer restarts. That's REQ-MQ-04: three retries then the DLQ.

**Deeper:**

```
                       nack(requeue=false)
  [ partner.acme ] ──────────────────────────▶ [ partner.dlx ]
        ▲                                             │  routing key decides the tier
        │                                             ├─▶ [ retry.1s   ] x-message-ttl=1000
        │                                             ├─▶ [ retry.5s   ] x-message-ttl=5000
        │                                             └─▶ [ retry.25s  ] x-message-ttl=25000
        │                                                      │ NO CONSUMER on these queues.
        │                                                      │ They exist only to hold + expire.
        │      ┌── DLX of retry.N = retry.return.x ────────────┘
        └──────┘   (routes back to partner.acme)

  after N attempts (counted from the x-death header) ──▶ [ partner.acme.dlq ]  ← parked, no TTL
```

**The `x-death` header** is what makes the attempt count work, and it's the detail that shows you've
built this: every time a message is dead-lettered, RabbitMQ *prepends/updates* an entry in an
`x-death` header array recording the queue, reason (`rejected`/`expired`/`maxlen`), and a **count**.
So the consumer can read `x-death[0].count` to decide "this is attempt 3, give up and park it". You
don't need to maintain your own counter header — though many teams do anyway, because it's explicit.

**The single biggest gotcha with TTL queues:** message TTL in a queue is evaluated **only at the head
of the queue**. A message with a 1-second TTL sitting behind a message with a 25-second TTL will not
expire until the one in front does. This means **you must not mix TTLs in one retry queue** — you need
one queue per delay tier, exactly as drawn above. Teams that use a per-message `expiration` property
in a single retry queue get wildly wrong backoff behaviour and can't work out why. Naming this
unprompted is a strong signal.

The alternative is the `rabbitmq_delayed_message_exchange` **plugin** (an `x-delayed-message` exchange
type with an `x-delay` header). It is cleaner — arbitrary per-message delays, no queue-per-tier — but
it's a community plugin, it holds pending messages in the broker's own store with its own scalability
characteristics, and it's another thing to install and operate. **Verify** its current status and
limits before depending on it. I'd choose TTL+DLX for a fixed, small set of tiers (StreamPulse's
case) and the plugin when delays are genuinely arbitrary.

**Code:**

```java
/**
 * REQ-MQ-04 — the retry ladder, declared as Declarables so RabbitAdmin creates
 * the whole topology at startup (or at partner-registration time).
 */
@Bean
Declarables retryTopology() {
    var workExchange  = new TopicExchange("alerts.x", true, false);
    var dlx           = new DirectExchange("partner.dlx", true, false);
    var returnEx      = new DirectExchange("partner.return.x", true, false);

    // Retry tiers: NO consumer, one TTL each. Their DLX sends the message BACK to the work queue.
    var retry1  = QueueBuilder.durable("partner.acme.retry.1s")
            .ttl(1_000).deadLetterExchange("partner.return.x")
            .deadLetterRoutingKey("partner.acme").build();
    var retry5  = QueueBuilder.durable("partner.acme.retry.5s")
            .ttl(5_000).deadLetterExchange("partner.return.x")
            .deadLetterRoutingKey("partner.acme").build();
    var retry25 = QueueBuilder.durable("partner.acme.retry.25s")
            .ttl(25_000).deadLetterExchange("partner.return.x")
            .deadLetterRoutingKey("partner.acme").build();

    // Terminal parking queue. NO TTL and NO DLX — messages stay until a human replays them.
    var dlq = QueueBuilder.durable("partner.acme.dlq").build();

    return new Declarables(
            workExchange, dlx, returnEx, retry1, retry5, retry25, dlq,
            BindingBuilder.bind(retry1).to(dlx).with("partner.acme.retry.1s"),
            BindingBuilder.bind(retry5).to(dlx).with("partner.acme.retry.5s"),
            BindingBuilder.bind(retry25).to(dlx).with("partner.acme.retry.25s"),
            BindingBuilder.bind(new Queue("partner.acme")).to(returnEx).with("partner.acme"),
            BindingBuilder.bind(dlq).to(dlx).with("partner.acme.dlq"));
}
```

```java
/**
 * The router: read the attempt count from x-death and choose the next tier.
 * Note this runs in the CONSUMER, not the broker — the broker gives us the count,
 * we decide the policy.
 */
private String nextTier(Message message) {
    @SuppressWarnings("unchecked")
    var deaths = (List<Map<String, Object>>) message.getMessageProperties()
            .getHeaders().get("x-death");

    long attempts = (deaths == null || deaths.isEmpty())
            ? 0
            : ((Number) deaths.get(0).get("count")).longValue();

    return switch ((int) attempts) {
        case 0 -> "partner.acme.retry.1s";
        case 1 -> "partner.acme.retry.5s";
        case 2 -> "partner.acme.retry.25s";
        default -> "partner.acme.dlq";       // REQ-MQ-04: 3 retries, then park
    };
}
```

```java
/**
 * REQ-MQ-04 — the admin replay endpoint. A DLQ without a replay path is a data graveyard.
 * Deliberately manual and audited: an automatic DLQ→work loop is an infinite retry.
 */
@PostMapping("/v1/admin/partners/{id}/dlq/replay")
ReplayResult replay(@PathVariable String id, @RequestParam(defaultValue = "100") int max) {
    int replayed = 0;
    for (int i = 0; i < max; i++) {
        Message m = rabbitTemplate.receive("partner." + id + ".dlq", 100);
        if (m == null) break;
        m.getMessageProperties().getHeaders().remove("x-death");   // reset the attempt counter
        rabbitTemplate.send("alerts.x", routingKeyOf(m), m);
        replayed++;
    }
    auditLog.record("dlq-replay", id, replayed);
    return new ReplayResult(replayed);
}
```

**Follow-ups they'll ask:**
- *"Why not just requeue with a `Thread.sleep`?"* — because you'd hold a consumer thread and a
  prefetch slot for the whole delay, and the delay would be lost on a restart. The broker-side TTL
  survives consumer restarts and costs no threads.
- *"How do you monitor this?"* — depth of each retry tier and of the DLQ, plus the **rate** into the
  DLQ. Depth tells you the backlog; rate tells you whether it's actively getting worse. Alert on the
  rate (same reasoning as the Kafka DLT, file 04 Q9).
- *"What if the DLQ itself fills up?"* — set `x-max-length` with `x-overflow=reject-publish` so the
  failure is loud, rather than `drop-head` which silently discards your oldest evidence. That choice
  is exactly the "make failure visible" principle again.
- *"Kafka's equivalent?"* — `@RetryableTopic` (file 04, Q9): retry topics with increasing delays and a
  DLT. Structurally the same design; RabbitMQ does it with TTL+DLX, Kafka with separate topics and
  delayed consumption. RabbitMQ's version is more natural because the broker already tracks per-message
  state.

**Pitfalls / red flags:** a single retry queue with per-message `expiration` (head-of-line TTL bug), or
a DLQ with no replay tooling and no alert.

**Where in StreamPulse:** BS-07 / REQ-MQ-04, `DlqRetryIT`.

**Verify:** `x-death` header structure and field names, and the current status/limits of the delayed
message exchange plugin.

---

## Q7. Quorum vs classic queues, and what is a lazy queue?

**30-second answer:** **Classic** queues are the original single-node queue; HA used to be provided by
*mirrored* classic queues, which had well-known failure modes around split-brain and synchronisation
and have been **deprecated and then removed** in the 4.x line. **Quorum** queues are the replacement:
Raft-based replication across an odd number of nodes, with a leader and followers, so durability and
failover semantics are explicit and predictable rather than best-effort. They're durable-only, and
they add the thing you actually want operationally: a broker-enforced **`x-delivery-limit`** that
dead-letters a message after N redeliveries — the poison-message protection Q4 said you can't build
from the `redelivered` flag. The cost is more disk and network (every message is replicated through
Raft) and some feature differences. **Lazy** mode was a classic-queue setting (`x-queue-mode=lazy`)
that kept messages on disk rather than in memory, for very deep queues; in newer versions classic
queues behave lazily by default, so the explicit setting is largely historical.

**Deeper:**

| | Classic | Classic mirrored | Quorum | Stream |
|---|---|---|---|---|
| Replication | none | mirror policy (legacy) | **Raft** | replicated log |
| Status | supported | **removed in 4.x** | the recommended durable type | since 3.9 |
| Durability | per-message | per-message | replicated, majority-ack | append-only log on disk |
| Poison protection | none built in | none | **`x-delivery-limit`** | n/a (offset-based) |
| Consumption model | destructive (ack deletes) | destructive | destructive | **non-destructive, offset-based — replayable** |
| Cost | cheapest | high, fragile | higher disk/network | disk-heavy, cheap CPU |
| Good for | transient, low-value, very high churn | — | **anything that matters** | fan-out + replay, "Kafka-ish" workloads |

**Quorum queue trade-offs worth naming:**
- Every message is written to a Raft log on a **majority** of nodes before it's confirmed, so
  durability is genuinely stronger and the confirm means more.
- More disk and network per message; noticeably higher cost for very high-throughput, low-value
  streams. That's a real reason classic queues still exist.
- Feature differences vs classic (historically: no per-message TTL, no priorities — **priorities were
  added in a later version**). This matters for the Q6 retry chain: my TTL retry queues use *queue*
  TTL (`x-message-ttl`), not per-message TTL, which is the compatible form. **Verify** the current
  feature matrix for your version; it has moved.
- `x-delivery-limit` is the headline operational win: the broker counts redeliveries and dead-letters
  automatically, so a requeue loop is bounded even if the consumer is buggy.

**Streams** deserve a mention because they're RabbitMQ's answer to "we want Kafka-like replay": an
append-only, replicated log with offset-based, non-destructive consumption and a retention policy. If
someone asks "why not just use RabbitMQ streams instead of Kafka?", the honest answer is: for moderate
throughput with replay needs inside an existing RabbitMQ estate, they're a genuinely good option; for
StreamPulse's telemetry volumes and its Kafka Streams processing, partition-based parallelism and the
Kafka ecosystem (Connect/Debezium, Schema Registry, Kafka Streams) are the reason to use Kafka.
That's an ecosystem argument, and it's a legitimate one.

**Code:**

```java
/**
 * REQ-MQ-02 + REQ-MQ-04. Quorum queue with a broker-enforced delivery limit.
 * x-delivery-limit is the safety net UNDER our TTL retry chain: even if the retry
 * routing has a bug and loops, the broker caps redeliveries and dead-letters.
 */
@Bean
Queue partnerQueue() {
    return QueueBuilder.durable("partner.acme")
            .quorum()                                   // Raft-replicated
            .deliveryLimit(5)                           // → x-delivery-limit; broker-enforced
            .deadLetterExchange("partner.dlx")
            .deadLetterRoutingKey("partner.acme.dlq")
            .maxLength(100_000)                         // bound the queue...
            .overflow(QueueBuilder.Overflow.rejectPublish)  // ...and fail LOUDLY, not by
                                                            // silently dropping the head
            .build();
}
```

```
# Operationally, the thing you must plan BEFORE going to production:
#   quorum queues want an ODD number of nodes (3 or 5) so a majority is well-defined.
#   With 2 nodes a single failure loses the majority and the queue becomes unavailable.
#   x-quorum-initial-group-size controls the replica count per queue.
```

**Follow-ups they'll ask:**
- *"What happens when a quorum queue's leader fails?"* — Raft elects a new leader from the followers;
  unacked messages are redelivered by the new leader. There is a brief unavailability window during
  election. That's a much more predictable story than mirrored classic queues, where synchronisation
  state and the `ha-promote-on-failure` policy decided whether you lost messages.
- *"When would you still use a classic queue?"* — transient, high-churn, low-value work where losing
  the queue on a node failure is acceptable and the Raft overhead isn't worth it. Be explicit that
  it's a deliberate durability trade, not a default.
- *"Lazy queues today?"* — the explicit `x-queue-mode=lazy` is largely superseded; newer classic
  queues page to disk by default. **Verify** for your version rather than quoting the old advice. The
  underlying concern is still real: a very deep queue held in memory is how you hit a memory alarm and
  block publishers (Q10).

**Pitfalls / red flags:** recommending mirrored classic queues for HA in 2026, or not knowing quorum
queues exist.

**Where in StreamPulse:** BS-07 / REQ-MQ-02; the compose `core` profile runs a single-node RabbitMQ
locally, and the ADR documents what changes with a real 3-node quorum cluster.

**Verify:** the quorum-queue feature matrix (per-message TTL, priorities), `x-delivery-limit`
behaviour, and classic-queue lazy defaults in your RabbitMQ version.

---

## Q8. What does Spring AMQP give you, and what do you have to get right yourself?

**30-second answer:** `RabbitTemplate` for publishing (with confirm/return callbacks and a
`MessageConverter`), `@RabbitListener` plus a container factory for consuming, and
`RabbitAdmin`/`Declarables` for declaring topology — declarations are idempotent and happen on
connection, so the topology is code, not a runbook. What Spring does **not** do for you: choose your
acknowledge mode correctly, decide `requeue` semantics, set prefetch, handle the confirm/return
correlation, or make your consumer idempotent. Those are the four places where a working demo becomes
a production incident. The one Spring-specific trap worth naming is the **`MessageConverter`**: the
default is Java serialization, which you should never use across services — set
`Jackson2JsonMessageConverter` (or an Avro/Protobuf converter) explicitly on both the template and the
listener factory, and pin the type mapping so a class rename doesn't break consumers.

**Deeper:**

| Component | Does | Watch out for |
|---|---|---|
| `CachingConnectionFactory` | pools connections/channels | `channelCacheSize` too small serialises publishers under concurrency |
| `RabbitTemplate` | publish, `convertAndSend`, `sendAndReceive` (RPC) | confirms are async — callbacks, not return values (Q3) |
| `@RabbitListener` | consume, with argument binding | `ackMode`, exception→requeue semantics (Q4) |
| `SimpleRabbitListenerContainerFactory` | the container | prefetch, concurrency, shutdown timeout |
| `RabbitAdmin` / `Declarables` | declare exchanges/queues/bindings on connect | re-declaring with *different* args → `PRECONDITION_FAILED` |
| `MessageConverter` | payload ↔ object | **default is Java serialization** — replace it |
| `RetryInterceptorBuilder` | client-side retry around the listener | blocking; holds the prefetch slot (Q9) |

**The type-mapping detail** that causes cross-service breakage: `Jackson2JsonMessageConverter` writes
a `__TypeId__` header with the fully-qualified class name, and the consumer uses it to deserialize.
If the consumer's class lives in a different package — which it will, in a different service — you
need a `DefaultJackson2JavaTypeMapper` with an explicit id-to-class map, or `setTrustedPackages`. The
better design is to declare the mapping explicitly so the wire format is a **contract** rather than an
accident of your package structure. (This is the same argument as using Avro with a registry for
Kafka, file 04 Q13 — the wire format should be a versioned artifact.)

**Code:**

```java
@Configuration
class AmqpConfig {

    /**
     * REQ-MQ-01/03. One place where the wire format, confirms and returns are decided.
     * Do NOT let the default Java-serialization converter survive into any real service.
     */
    @Bean
    RabbitTemplate rabbitTemplate(ConnectionFactory cf, MessageConverter converter) {
        var template = new RabbitTemplate(cf);
        template.setMessageConverter(converter);
        template.setMandatory(true);                      // REQ-MQ-03 — enable returns
        template.setChannelTransacted(false);             // confirms, not tx (Q3)
        // Publisher-side retry for CONNECTION failures (not for business failures).
        template.setRetryTemplate(RetryTemplate.builder()
                .maxAttempts(3)
                .exponentialBackoff(200, 2.0, 2_000)
                .retryOn(AmqpConnectException.class)
                .build());
        return template;
    }

    @Bean
    MessageConverter jsonConverter() {
        var converter = new Jackson2JsonMessageConverter(objectMapper());
        var mapper = new DefaultJackson2JavaTypeMapper();
        // Explicit contract: the wire carries "AlertEvent", not a package path that can change.
        mapper.setIdClassMapping(Map.of("AlertEvent", AlertEvent.class));
        mapper.setTrustedPackages("com.streampulse.contracts");
        converter.setJavaTypeMapper(mapper);
        return converter;
    }
}
```

```java
/**
 * The listener side. Note the SmartLifecycle phase: the RabbitMQ containers must stop
 * in the same ordered drain as the Kafka containers (BS-01 / REQ-LC-01), or a deploy
 * leaves partner deliveries half-done with unacked messages.
 */
@Bean
SimpleRabbitListenerContainerFactory partnerListenerFactory(
        ConnectionFactory cf, MessageConverter converter) {

    var factory = new SimpleRabbitListenerContainerFactory();
    factory.setConnectionFactory(cf);
    factory.setMessageConverter(converter);
    factory.setAcknowledgeMode(AcknowledgeMode.MANUAL);      // REQ-MQ-04
    factory.setPrefetchCount(10);                            // REQ-MQ-05
    factory.setDefaultRequeueRejected(false);                // explicit (Q4)
    factory.setPhase(250);                                   // BS-01 ordered shutdown
    factory.setShutdownTimeout(20_000);
    return factory;
}
```

**Follow-ups they'll ask:**
- *"How do you test this without a real broker?"* — Testcontainers RabbitMQ for integration
  (`PartnerRoutingIT`, `DlqRetryIT`), and plain unit tests for the routing-key builder and the
  `x-death` tier logic. The routing itself genuinely needs a broker — RabbitMQ's topic matcher is not
  something you should reimplement in a test double.
- *"What about `sendAndReceive` / RPC over AMQP?"* — it works (a reply-to queue plus a correlation id,
  with `DirectReplyToMessageListenerContainer` as the efficient form), but a synchronous RPC over a
  message broker usually means you wanted HTTP or gRPC. I'd push back on the design before tuning it.
- *"Connection recovery?"* — the Java client and Spring AMQP recover connections and re-declare
  topology automatically, but **in-flight unacked messages are requeued** on a connection drop, so
  duplicates after a network blip are expected. Idempotent consumers, again.

**Pitfalls / red flags:** leaving the default Java-serialization `MessageConverter`, or declaring
topology by hand in the RabbitMQ management UI so it isn't reproducible.

**Where in StreamPulse:** BS-07 / REQ-MQ-01…05, Phase 6.

**Verify:** `DefaultJackson2JavaTypeMapper` API names and `setTrustedPackages` behaviour in your
Spring AMQP version.

---

## Q9. Spring AMQP retry: interceptors vs the broker-side chain. Which do you use?

**30-second answer:** Spring AMQP can retry **in the consumer** with a `RetryOperationsInterceptor` on
the listener container — configured declaratively with
`spring.rabbitmq.listener.simple.retry.*`. That's blocking retry: the message stays unacked, the
consumer thread sleeps between attempts, and the prefetch slot stays occupied. It's fine for
sub-second transients and it preserves the message's position. For anything slower — the BS-04 partner
outage that lasts minutes — you want the **broker-side TTL/DLX chain from Q6**, because it costs no
threads, survives a consumer restart, and makes the backlog visible as queue depth rather than hiding
it in a sleeping consumer. It's the same blocking-vs-non-blocking trade as Kafka's `DefaultErrorHandler`
versus `@RetryableTopic` (file 04, Q9). StreamPulse uses a short blocking retry for instant transients
and the DLX chain for everything else.

**Deeper:**

| | Interceptor (blocking) | TTL + DLX chain (broker-side) |
|---|---|---|
| Where the delay happens | consumer thread `sleep` | broker, in a TTL queue |
| Consumer resources held | thread + prefetch slot | none |
| Survives consumer restart | ❌ attempts lost, message requeued from scratch | ✅ the message is in the retry queue |
| Backlog visibility | hidden (unacked count) | ✅ queue depth per tier |
| Ordering | preserved-ish | lost (the message goes to the back) |
| Max sensible delay | sub-second to a few seconds | minutes to hours |
| Attempt counting | in-memory (stateless) or keyed (stateful) | `x-death` count, durable |

**Stateless vs stateful interceptors** is the Spring-specific distinction worth knowing:
`RetryInterceptorBuilder.stateless()` keeps the retry state in memory for the duration of the call
(the message is never requeued between attempts) — simple, and the usual choice.
`RetryInterceptorBuilder.stateful()` rejects and requeues between attempts, tracking state by a
message key, which is what you need if the retry must survive a redelivery. Stateful requires a
`MessageKeyGenerator` and a cache, and it's fiddly; the broker-side chain usually solves the same
problem more cleanly.

**The `MessageRecoverer`** decides what happens when the interceptor's attempts are exhausted:
- `RejectAndDontRequeueRecoverer` (the default) — nack with `requeue=false`, so the DLX takes it.
- `RepublishMessageRecoverer` — republishes to an exchange/routing key you choose, **adding headers
  with the exception message and stack trace**. That header enrichment is genuinely valuable for a
  parking queue: the DLQ message carries *why* it failed, so triage doesn't require log archaeology.

**Code:**

```yaml
spring:
  rabbitmq:
    listener:
      simple:
        retry:
          enabled: true
          max-attempts: 3         # ← deliberately SMALL and FAST. This layer is only for
          initial-interval: 200ms #   instant transients (a connection blip, a fast 503).
          multiplier: 2.0         #   200ms, 400ms → total < 1s of held prefetch slot.
          max-interval: 1s
          stateless: true
```

```java
/**
 * REQ-MQ-04. The recoverer runs when the fast blocking retries are exhausted.
 * RepublishMessageRecoverer adds x-exception-message / x-exception-stacktrace headers,
 * so the DLQ entry explains itself. Then the TTL chain (Q6) handles the SLOW retries.
 */
@Bean
MessageRecoverer messageRecoverer(RabbitTemplate template) {
    return new RepublishMessageRecoverer(template, "partner.dlx", "partner.acme.retry.1s")
            .errorRoutingKeyPrefix("")          // we set the routing key explicitly above
            .addHeaders(true);                  // exception message + stacktrace headers
}
```

```java
/**
 * Two layers, explicitly. This is the answer to "how many retries do you do?" —
 * "three fast ones in-process for blips, then three slow ones broker-side for outages,
 *  then park. Nine attempts over ~31 seconds, and none of them holds a thread for long."
 *
 *   layer 1 (interceptor):  200ms, 400ms          → ~0.6s,  holds the prefetch slot
 *   layer 2 (TTL chain):    1s, 5s, 25s           → ~31s,   holds nothing
 *   layer 3 (DLQ):          parked for a human
 *
 *  ⚠ Check the arithmetic against BS-04's partner TimeLimiter (3s per attempt):
 *     worst case layer 1 = 3 × (3s call + backoff) ≈ 9.6s of held slot. With prefetch 10
 *     that's fine; with prefetch 1000 it would be a stall. The layers interact — size them
 *     together, not independently.
 */
```

**Follow-ups they'll ask:**
- *"Why not just set max-attempts to 50 in the interceptor?"* — because each attempt holds a consumer
  thread and a prefetch slot, so a broad partner outage converts into total consumer starvation for
  every *other* message in that queue. Head-of-line blocking, same as Kafka's blocking retry.
- *"How does this interact with Resilience4j?"* — carefully, and this is the retry-amplification point
  from file 05, Q5. If the partner call is already wrapped in `@Retry` (BS-04), and the listener
  interceptor also retries, and the TTL chain retries, you have three multiplied layers. I own the
  retry policy in **one** place — for the partner path, that's the TTL chain, with Resilience4j
  providing the *circuit breaker and bulkhead* rather than retries.
- *"How do you stop retrying entirely when the partner is down?"* — the circuit breaker
  (BS-04 / REQ-RS-02): once it's OPEN, delivery fails instantly with `CallNotPermittedException`, the
  message is dead-lettered to the retry tier without burning a 3-second timeout, and the backlog
  builds in a queue where you can see it.

**Pitfalls / red flags:** stacking three retry layers without doing the multiplication, or a long
blocking retry that starves a shared queue.

**Where in StreamPulse:** BS-07 / REQ-MQ-04 + BS-04 / REQ-RS-02, REQ-RS-05.

**Verify:** the `spring.rabbitmq.listener.simple.retry.*` property names and
`RepublishMessageRecoverer` method names for your Spring Boot version.

---

## Q10. What about ordering, flow control and broker-side backpressure in RabbitMQ?

**30-second answer:** RabbitMQ gives you **FIFO order within a single queue to a single consumer**,
and that's it — the moment you have multiple consumers on a queue, or a requeue, or a retry chain,
ordering is gone. There is no equivalent of Kafka's per-partition total order with a stable key
mapping, so if ordering matters you need one queue per ordering key with a single consumer, which
doesn't scale the way partitions do. On backpressure: RabbitMQ has **credit-based flow control**
internally and, more visibly, **resource alarms** — when memory or free disk crosses a watermark, the
broker **blocks publishing connections**. That's genuinely different from Kafka, where a backlog is
just disk and producers keep going: in RabbitMQ, a slow consumer eventually stops your publishers.
Your producers must handle `connection.blocked` and a blocked publish, or they hang.

**Deeper:**

**Ordering, precisely:**

| Situation | Order preserved? |
|---|---|
| One queue, one consumer, prefetch 1, no failures | ✅ FIFO |
| One queue, one consumer, prefetch 10, concurrent handling | ❌ your own concurrency reorders it |
| One queue, N consumers | ❌ |
| Any requeue or TTL retry | ❌ the message goes to the back (or head, unpredictably relative to others) |
| Priority queue | ❌ by design |
| Kafka, same key, same partition | ✅ **total order per key**, and it scales via partitions |

That table is the strongest single argument for Kafka in the StreamPulse telemetry path: per-session
ordering (file 04, Q2) is a hard requirement there and RabbitMQ has no way to give it to you at
scale. Conversely, the **alert** path has no ordering requirement at all — each alert is independent
and idempotent by `alertId` — which is exactly why BS-07 can use RabbitMQ without compromise. Matching
the broker to the ordering requirement is the decision, and being able to say "these two paths have
different requirements so they use different brokers" is a much better answer than picking one.

**Flow control and alarms:**

```
memory watermark (vm_memory_high_watermark) crossed
   └─▶ broker BLOCKS publishing connections (connection.blocked)
        └─▶ your publisher's next publish HANGS (or your client fires a blocked listener)
            └─▶ if that publisher is an HTTP request thread → thread exhaustion → outage

free disk watermark (disk_free_limit) crossed
   └─▶ same: publishers blocked
```

Two operational consequences to state:
1. **A slow consumer becomes a publisher outage.** In Kafka, a slow consumer produces lag and nothing
   else. In RabbitMQ, an unbounded growing queue consumes broker memory, trips the alarm, and blocks
   publishers. That's coupling that surprises people migrating from Kafka.
2. Therefore you **bound your queues**: `x-max-length` / `x-max-length-bytes` with an explicit
   `x-overflow` policy (`drop-head`, `reject-publish`, or `reject-publish-dlx`). Choosing
   `reject-publish` makes the failure explicit at the publisher instead of silently losing the oldest
   messages — the "make failure visible" principle again.

**Code:**

```java
/**
 * Handle connection.blocked explicitly. Without this, a broker memory alarm shows up
 * as "publishes randomly hang forever" — one of the harder RabbitMQ incidents to diagnose.
 */
@Bean
CachingConnectionFactory connectionFactory(RabbitProperties props) {
    var cf = new CachingConnectionFactory(props.getHost(), props.getPort());
    cf.setChannelCacheSize(50);                       // sized to publisher concurrency
    cf.addConnectionListener(new ConnectionListener() {
        @Override public void onCreate(Connection c) { }
        @Override public void onShutDown(ShutdownSignalException s) {
            log.error("RabbitMQ connection shut down: {}", s.getReason());
        }
    });
    // The client exposes blocked/unblocked notifications; surface them as a metric and a
    // health-indicator so "the broker is blocking us" is a dashboard line, not a mystery.
    cf.getRabbitConnectionFactory().addBlockedListener(
            reason -> meters.counter("rabbit.connection.blocked").increment(),
            ()     -> meters.counter("rabbit.connection.unblocked").increment());
    return cf;
}
```

```java
// Bound every queue. An unbounded queue is a broker-wide outage waiting for a slow consumer.
QueueBuilder.durable("partner.acme")
        .maxLength(100_000)
        .overflow(QueueBuilder.Overflow.rejectPublishDlx)   // overflow → DLX, not silently dropped
        .build();
```

**Follow-ups they'll ask:**
- *"How do you get ordering if you really need it in RabbitMQ?"* — one queue per ordering key with a
  single consumer, or a consistent-hash exchange plugin that maps keys to a fixed set of queues
  (which is essentially reinventing partitions, less well). At that point, ask whether Kafka is the
  right tool.
- *"What's the RabbitMQ equivalent of consumer lag?"* — queue depth (`messages_ready`) plus
  `messages_unacknowledged` and the publish/deliver rates. Depth is the analogue of lag; unacked is the
  analogue of in-flight. The key difference is that depth also directly threatens broker memory, so it
  is both a throughput signal *and* a stability signal.
- *"Priority queues?"* — `x-max-priority` on a queue plus a per-message priority. Useful (CRITICAL
  alerts before WARN), but priorities interact badly with prefetch: already-prefetched low-priority
  messages are processed before a newly arrived high-priority one. Low prefetch is the price of
  meaningful priorities.

**Pitfalls / red flags:** claiming RabbitMQ preserves ordering, or having no bound on queue length.

**Where in StreamPulse:** BS-07; Phase 8 dashboards include RabbitMQ queue depth alongside Kafka lag.

**Verify:** the default memory/disk watermark values and the exact `x-overflow` policy names for your
version.

---

## Q11. Kafka vs RabbitMQ, head to head. When does each win?

**30-second answer:** One sentence: **RabbitMQ is a smart broker with dumb consumers; Kafka is a dumb
broker with smart consumers.** RabbitMQ owns per-message state — routing, acks, TTL, priority,
redelivery counts — so it excels at *work distribution with complex routing and per-message
lifecycle*. Kafka owns nothing but an ordered log; the consumer owns its offset, so Kafka excels at
*high-throughput streams that multiple independent consumers read, replay and reprocess*. The three
questions that decide it for me: **Do I need to replay history?** (Kafka — RabbitMQ's classic queues
delete on ack). **Do I need per-key ordering at scale?** (Kafka — partitions). **Do I need per-message
routing, per-message retry/delay, or per-message ack?** (RabbitMQ — Kafka has none of these natively).
StreamPulse answers "yes" to different questions on different paths, which is why it uses both.

**Deeper:**

| Axis | Kafka | RabbitMQ |
|---|---|---|
| **Model** | distributed, partitioned, append-only log | queues fed by exchanges |
| **Retention** | time/size based; data survives consumption | destructive on ack (streams are the exception) |
| **Replay** | ✅ first-class — reset offsets, any consumer, any time | ❌ classic/quorum queues; ✅ streams |
| **Ordering** | ✅ total per partition, stable via key hashing | FIFO per queue per consumer only; lost on retry/multi-consumer |
| **Routing granularity** | topic + partition; consumers filter | ✅ exchange types, wildcard patterns, headers |
| **Per-message ack** | ❌ — a per-partition offset | ✅ ack/nack/reject per delivery |
| **Per-message TTL / delay / priority** | ❌ natively | ✅ |
| **Consumer scaling** | group members ≤ partitions; assignment by rebalance | competing consumers on a queue; add consumers freely |
| **Throughput ceiling** | very high; sequential disk I/O, zero-copy, batching | high, but per-message bookkeeping costs more |
| **Backpressure** | backlog = disk; producers unaffected | **alarms block publishers** |
| **DLQ story** | DIY: retry topics + DLT (`@RetryableTopic`) | ✅ native DLX + TTL retry chains |
| **Multiple independent readers** | ✅ each consumer group reads everything | requires a queue per consumer + fanout/topic bindings |
| **Operational weight** | heavier (KRaft/ZK, partitions, rebalances, Streams state) | lighter for small deployments; Raft to reason about for quorum |
| **Ecosystem** | Connect/Debezium, Schema Registry, Kafka Streams, ksqlDB | management UI, plugins, Shovel/Federation |
| **Message size** | designed for small records; large payloads hurt | similar; both want a claim-check pattern for blobs |

**Pick Kafka when:** you have a high-volume *event stream*; more than one independent consumer needs
the same data; you need replay (reprocessing after a bug, bootstrapping a new service, backfilling a
new read model); you need per-key ordering at scale; you want stream processing (windowing,
aggregation, joins) close to the data.

**Pick RabbitMQ when:** you have *tasks* rather than events; routing rules are per-subscriber and
change at runtime; you need per-message retry/delay/priority; you need per-message acknowledgement so
a single failure doesn't block a partition; the volume is moderate and the topology is complex.

**The StreamPulse split, as a defence of using both:**

| Path | Broker | Deciding property |
|---|---|---|
| Telemetry ingest → processing → sinks | **Kafka** | 50k ev/s, per-session ordering, replay, Kafka Streams windowing, several independent consumer groups |
| CDC config propagation | **Kafka** (compacted) | a new service must rebuild its whole cache from history — needs replay |
| Partner alert routing (BS-07) | **RabbitMQ** | per-partner wildcard routing declared at runtime, per-message retry/DLQ, no ordering requirement |
| Alert fan-out to AWS consumers | **SQS** (via moto) | AWS parity, managed, nothing to operate |

That last column is the actual answer to the interview question. Not "Kafka is better", but "these
paths have different properties, and here's the property that decided each one."

**Follow-ups they'll ask:**
- *"Isn't running two brokers operational overhead?"* — yes, and it's a real cost I'd weigh. For a
  small team I'd pick one and live with the compromise (Kafka with consumer-side filtering, or
  RabbitMQ streams for the replay). StreamPulse runs both partly *because* it's a portfolio project
  that needs to demonstrate both — I'd say that plainly rather than pretend it's a pure engineering
  optimum.
- *"Could RabbitMQ streams replace Kafka here?"* — for moderate throughput with replay, genuinely
  maybe. What I'd lose is the Kafka ecosystem: Debezium/Connect for the CDC path, Schema Registry for
  the Avro contracts, and Kafka Streams for the windowed aggregation. Those three are the reason, not
  raw throughput.
- *"What about NATS / Pulsar / Redpanda?"* — Pulsar separates serving from storage (BookKeeper) and
  offers both queue and stream semantics in one system; Redpanda is Kafka-API-compatible with no JVM;
  NATS JetStream is lightweight with good pub/sub ergonomics. I'd evaluate them on operational
  familiarity and ecosystem more than on benchmarks — **and I'd be honest that I have not run them in
  production**, rather than reciting comparisons I can't defend.

**Pitfalls / red flags:** "Kafka is faster" as the whole answer. Throughput is rarely the deciding
axis; ordering, replay and routing granularity are.

**Where in StreamPulse:** BS-07 / REQ-MQ-07 (the ADR), Phase 6.

---

## Q12. Now add SQS/SNS and Kinesis. Give me the full decision table.

**30-second answer:** **SQS** is a managed distributed queue: no ordering in Standard mode, at-least-once
delivery, visibility timeout instead of acks, a redrive policy for DLQs, and effectively unlimited
throughput — its FIFO variant adds ordering per `MessageGroupId` and dedup, at a throughput cost.
**SNS** is pub/sub fan-out that pushes to subscribers (SQS queues, Lambda, HTTP) with **filter
policies** on message attributes — that's the closest AWS analogue to RabbitMQ's topic exchange, and
the standard AWS pattern is SNS→SQS fan-out. **Kinesis Data Streams** is the closest AWS analogue to
Kafka: sharded, ordered per partition key within a shard, with a retention window you can replay —
but with a fixed per-shard throughput model and the KCL's DynamoDB lease table for coordination. The
axis that separates them all is the same one from Q11: who owns the message state, and can you replay.

**Deeper — the decision table (REQ-MQ-07's deliverable):**

| Axis | **Kafka** | **RabbitMQ** | **SQS (Standard)** | **SQS FIFO** | **SNS** | **Kinesis** |
|---|---|---|---|---|---|---|
| Shape | partitioned log | queues + exchanges | queue | queue | pub/sub topic | sharded log |
| Ordering | per partition (by key) | FIFO per queue/consumer only | **none** (best effort) | per `MessageGroupId` | none | per partition key in a shard |
| Retention / replay | ✅ time/size, replay by offset | ❌ (streams: ✅) | ❌ deleted on delete-message | ❌ | ❌ (transient) | ✅ within the retention window |
| Delivery guarantee | at-least-once (+ EOS in-Kafka) | at-least-once | at-least-once | exactly-once *processing* within a dedup window | at-least-once | at-least-once |
| Ack model | offset commit | per-message ack/nack | **visibility timeout + delete** | same | n/a (push) | checkpoint (KCL) |
| Routing granularity | topic only | ✅ exchange types + wildcards | none | none | ✅ **filter policies** on attributes | stream only |
| Per-message delay | ❌ | ✅ (TTL/DLX or plugin) | ✅ up to 15 min (documented limit — verify) | ✅ | ❌ | ❌ |
| DLQ | DIY (retry topics + DLT) | ✅ DLX | ✅ redrive policy + `maxReceiveCount` | ✅ | via the subscribed queue | DIY |
| Consumer scaling | ≤ partitions per group | competing consumers, unbounded | unbounded pollers | limited by group count | push | ≤ shards (or enhanced fan-out) |
| Throughput model | brokers + partitions you operate | broker capacity you operate | effectively unlimited, pay per request | lower than Standard (documented limits — verify) | high | **per-shard quotas** (verify) |
| Multi-consumer | ✅ consumer groups | queue per consumer + bindings | ❌ one queue = one logical consumer → use SNS fan-out | ❌ | ✅ that's its job | ✅ multiple apps, or enhanced fan-out |
| Ops cost | high (you run it) | medium | **none** (managed) | none | none | low (managed, but shard management) |
| Cost model | infrastructure | infrastructure | per request + data | per request | per publish/delivery | per shard-hour + PUT units |
| **Pick when** | high-volume streams, replay, per-key ordering, stream processing | complex routing, per-message retry/priority, task distribution | simple decoupling on AWS, spiky load, don't want to operate anything | you need ordering + dedup and can accept lower throughput | fan-out one message to many subscribers with attribute filters | Kafka-shaped needs inside AWS with less to operate |

**SQS mechanics worth knowing precisely**, because they're genuinely different from an ack model:

- **Visibility timeout**: receiving a message doesn't delete it — it makes it *invisible* for N
  seconds. You must call `DeleteMessage` to actually remove it. If your handler takes longer than the
  visibility timeout, the message becomes visible again and **another consumer processes it
  concurrently**. That's the classic SQS duplicate: not a broker bug, a timeout mis-sizing. Fix it by
  sizing the timeout above your worst-case processing time, or by calling
  `ChangeMessageVisibility` to extend it (a heartbeat).
- **Redrive policy**: `maxReceiveCount` on the queue moves a message to a DLQ after N receives. That
  *is* the poison-message protection, broker-side, and it's one of SQS's nicer properties.
- **Long polling** (`WaitTimeSeconds`) avoids the cost and latency of empty short polls.
- **FIFO** gives ordering per `MessageGroupId` and content-based or explicit deduplication within a
  dedup window, at a documented throughput limit lower than Standard. **Verify** the current numbers;
  high-throughput FIFO mode raised them.
- **Message size** is capped (256 KB as I recall); larger payloads use the extended client with S3 —
  the **claim-check pattern**. **Verify** the current limit.

**SNS + SQS fan-out** is the AWS idiom that maps onto BS-07's RabbitMQ topology:

```
RabbitMQ (BS-07)                          AWS equivalent
────────────────────────────────────────────────────────────────────────
topic exchange "alerts.x"        ≈        SNS topic
binding "alert.sportsnow.CRITICAL.*"  ≈   SNS subscription FILTER POLICY on message attributes
queue partner.acme               ≈        SQS queue subscribed to the topic
DLX + TTL retry chain            ≈        redrive policy (maxReceiveCount) + DLQ
per-message ack                  ≈        DeleteMessage within the visibility timeout
prefetch                         ≈        MaxNumberOfMessages per receive + visibility timeout
```

The mapping is close enough that REQ-MQ-01's `NotificationPublisher` port has **both** adapters, which
is the hexagonal payoff: the alert engine doesn't know which one is active. Where the analogy breaks:
SNS filter policies match on **attributes**, not on a hierarchical routing key, so complex routing
rules are expressed differently (and SNS filter policies do support numeric/prefix/anything-but
operators, which RabbitMQ topic patterns do not — it's a trade, not a strict subset).

**Kinesis vs Kafka**, since the Event Hub JD mentions both (and Phase 10 builds the adapter):

| | Kafka | Kinesis |
|---|---|---|
| Unit of parallelism | partition | **shard** |
| Throughput | what your brokers do | documented **per-shard** write/read quotas — verify |
| Consumer fan-out | each group reads independently, no extra cost | shared shard read throughput, or **enhanced fan-out** per consumer (extra cost) |
| Ordering | per partition by key | per partition key within a shard |
| Retention | configurable, effectively unbounded with tiered storage | configurable up to a documented maximum — verify |
| Resharding | partition increase (breaks key mapping) | shard split/merge, or on-demand mode |
| Coordination | broker-side group coordinator | **KCL + a DynamoDB lease table** you also pay for and must monitor |
| Ops | you run it (or MSK) | fully managed |

The honest summary: Kinesis is Kafka-shaped with less to operate and a harder throughput ceiling per
shard; the KCL's DynamoDB lease table is a real operational surface (hot partitions on the lease
table, lease-stealing storms) that people forget when they say "managed means no ops".

**Code:** the port that makes the choice swappable, which is the architectural point:

```java
/** REQ-MQ-01 — one port, three adapters, selected by Spring profile (BS-01 / REQ-LC-06). */
public interface NotificationPublisher {
    void publish(AlertEvent alert);
}

@Component @Profile("!sqs")
class RabbitNotificationPublisher implements NotificationPublisher {
    public void publish(AlertEvent a) {
        rabbit.convertAndSend("alerts.x", routingKey(a), a, corr(a));   // pattern routing
    }
}

@Component @Profile("sqs")
class SnsNotificationPublisher implements NotificationPublisher {
    public void publish(AlertEvent a) {
        sns.publish(PublishRequest.builder()
                .topicArn(topicArn)
                // SNS filter policies match on ATTRIBUTES, not a dotted routing key — so the
                // routing dimensions become attributes. Same routing decisions, different shape.
                .messageAttributes(Map.of(
                        "tenant",   attr(a.tenantId()),
                        "severity", attr(a.severity().name()),
                        "category", attr(a.category().name())))
                .message(json(a))
                .build());
    }
}
```

**Follow-ups they'll ask:**
- *"SQS or Kafka for a new service on AWS?"* — if it's decoupling two services with no replay or
  ordering requirement, SQS every time: zero operational cost and a DLQ for free. I'd only take on
  Kafka/MSK when I need replay, per-key ordering, multiple independent consumers, or stream
  processing. "Don't run a broker you don't need" is a legitimate senior position.
- *"Why does SQS have no ordering?"* — it's a distributed queue with no partition concept; messages
  are spread across servers for availability and throughput. FIFO adds ordering by constraining that,
  which is why it has lower throughput limits. The trade is explicit in the product design.
- *"How would you migrate BS-07 from RabbitMQ to SNS+SQS?"* — the port already exists, so the code
  change is an adapter. The *semantic* work is translating wildcard routing keys into filter policies
  and the TTL retry chain into a redrive policy plus a delay queue, and accepting that per-message
  delay tiers become harder. I'd write that translation table before committing to the migration.

**Pitfalls / red flags:** treating SQS's visibility timeout as an ack, or not knowing that SNS needs
SQS behind it for durable, retryable delivery.

**Where in StreamPulse:** BS-07 / REQ-MQ-07 (the ADR), Phase 6 (Rabbit) and Phase 10 (SQS/Kinesis on
moto).

**Verify:** every AWS number in this section — SQS visibility timeout defaults and maxima, message
size limit, retention limits, delay maximum, FIFO throughput limits (standard and high-throughput
mode), Kinesis per-shard quotas, enhanced fan-out limits, and Kinesis retention maxima. These are
documented service quotas and they change; I have quoted the *mechanisms* confidently and the
*numbers* only as "documented limits — check them".

---

## Q13. Walk me through BS-07 end to end: an alert reaches three partners, one of them is down.

*(The applied question.)*

**30-second answer:** The alert engine produces an `AlertEvent` with a deterministic
`alertId = hash(ruleId, windowStart)` and writes it to the **outbox** in the same transaction as any
state change (BS-05 / REQ-DL-06), so there's no dual-write. A publisher drains the outbox and
publishes to the topic exchange `alerts.x` with routing key
`alert.<tenant>.<severity>.<category>`, using **publisher confirms and the mandatory flag**: the
outbox row is only cleared on a confirm that wasn't also returned, and an unroutable message is a loud
error because it means a partner registered without a queue. RabbitMQ copies the message into every
queue whose binding matches — three partners here — and each is independent from that moment. Partner
workers use **manual ack** with **prefetch 10**, and each delivery goes through BS-04's per-tenant
`ThreadPoolBulkhead` and `RateLimiter` (REQ-MQ-06). The failing partner's messages are nacked with
`requeue=false` into the DLX and walk the **1s → 5s → 25s TTL ladder**, then park in
`partner.X.dlq`; the other two partners are completely unaffected, which is the whole point of
per-message acks and per-queue isolation. An admin endpoint replays the DLQ after the partner is
fixed.

**The flow, drawn:**

```
alert engine ──(same DB txn)──▶ [ alerts table + outbox row ]
                                        │  Debezium / outbox poller
                                        ▼
                          publish(alerts.x, "alert.sportsnow.CRITICAL.cdn",
                                  persistent, mandatory, CorrelationData)
                                        │
                     ┌──────────────────┼──────────────────┐
                     ▼                  ▼                  ▼
            [ partner.acme ]    [ partner.globo ]   [ audit.all ]
            binding:            binding:            binding:
            alert.sportsnow.    alert.*.*.cdn       alert.#
            CRITICAL.*
                     │                  │                  │
              deliver ✅          deliver ✅          archive ✅
                                        │
                              partner DOWN: nack(requeue=false)
                                        ▼
                          [ partner.dlx ] ─▶ retry.1s ─(TTL expires)─▶ back to partner.globo
                                            ─▶ retry.5s  ─▶ ...
                                            ─▶ retry.25s ─▶ ...
                                            ─▶ [ partner.globo.dlq ]  ← parked, alert fires
```

**The design decisions, each with its reason:**

| Decision | Why |
|---|---|
| Topic exchange, not a queue per severity | subscriptions are per *partner* and change at runtime (REQ-MQ-02) |
| Routing key general → specific | `*` is positional; the hierarchy determines what's expressible |
| Publisher confirms + mandatory | confirm ≠ routed; both are needed (REQ-MQ-03) |
| Outbox, not publish-in-transaction | no dual-write; survives a crash between DB commit and publish |
| Manual ack | ack after the partner confirmed, not when the method returned (REQ-MQ-04) |
| `requeue=false` always | `requeue=true` on a permanent failure is a CPU-burning hot loop |
| TTL/DLX ladder, not blocking retry | a minutes-long partner outage must not hold consumer threads |
| Quorum queue + `x-delivery-limit` | broker-enforced backstop under our own retry logic |
| `alertId` deterministic | the consumer can dedup; redelivery is safe (file 04, Q7) |
| Per-tenant bulkhead (REQ-MQ-06) | one slow partner must not starve the others (BS-04 / REQ-RS-05) |
| Bounded queues + `reject-publish` | an unbounded queue is a broker memory alarm waiting to happen |

**What I'd measure:** per-queue depth and unacked count, delivery success/failure rate per partner,
DLQ **rate** (not just depth), the retry-tier depths (a growing `retry.25s` means a partner is in a
sustained outage), and the `rabbit.publish.unroutable` counter — because that one means a
configuration bug, not a partner problem.

**Follow-ups they'll ask:**
- *"Why not just let each partner consume the Kafka `alerts.v1` topic?"* — three reasons, in order:
  they'd each need Kafka credentials and a consumer group in *our* cluster; there'd be no per-message
  retry or DLQ per partner; and routing would mean every partner reads everything and filters. The
  first one is also a security boundary argument — exposing a Kafka cluster to external parties is a
  much bigger surface than a per-partner queue.
- *"What if a partner is down for a week?"* — the DLQ is bounded, so at some point we stop accepting.
  Before that: alert on DLQ rate, and the replay endpoint exists so recovery is a runbook step, not an
  engineering project. The design-level answer is that alerts are time-sensitive — a week-old alert
  has no value — so beyond some age, *discarding is correct* and should be an explicit TTL rather than
  an accident.
- *"How do you onboard a new partner with zero downtime?"* — `POST /v1/partners/{id}/subscriptions`:
  declare a durable queue, bind it, start a listener. No publisher change and no deploy. That's the
  single clearest demonstration of what the exchange indirection buys you, and it's REQ-MQ-02.
- *"How does this compare to what you'd do on AWS?"* — SNS topic + per-partner SQS queue + filter
  policy + redrive policy. The `NotificationPublisher` port has both adapters (REQ-MQ-01), and the
  ADR (REQ-MQ-07) is the Q12 table.

> **Your story:** you ran a **RabbitMQ backbone at JustAnswer** — this is where that goes, and it's
> the most valuable paragraph in this file, because it's the one thing here I cannot write for you.
> Frame it in the same shape as the answers above: what the exchange topology was (direct or topic,
> and why), how many queues/consumers, what the ack and prefetch settings were and what made you
> choose them, what broke (a poison-message requeue loop? a memory alarm blocking publishers? an
> unrouted message? a mirrored-queue failover?), how you found it, and what you changed. Two or three
> concrete numbers — message rate, queue depth at peak, prefetch value — make it real. Keep "I built"
> for what you personally wrote and "the team built" otherwise. BS-07 is a **fictional design
> scenario** for a portfolio project: present it as *"I rebuilt the pattern in Java to show it end to
> end"*, never as a production incident.

**Where in StreamPulse:** BS-07 / REQ-MQ-01…07 end to end; `PartnerRoutingIT`, `PublisherConfirmIT`,
`DlqRetryIT`; Phase 6.

---

## Key terms

| Term | One-line meaning |
|---|---|
| **Exchange** | The routing function messages are published to. Publishers never address a queue directly. |
| **Binding** | Exchange → queue with a binding key. Adding one is how you subscribe without a deploy. |
| **Routing key** | The publisher-set string matched against bindings. `*` = one word, `#` = zero or more. |
| **Direct / topic / fanout / headers** | Exact match / wildcard pattern / broadcast / header-attribute match. |
| **Alternate exchange** | Where an exchange sends messages that matched no binding. |
| **Publisher confirm** | Async broker ack per message. Means "the broker has it", **not** "someone will get it". |
| **`mandatory` + basic.return** | Detects an unroutable message. Needed *in addition to* confirms. |
| **`delivery_mode=2`** | Persistent message. Needs a durable exchange *and* a durable queue to mean anything. |
| **Prefetch (`basic.qos`)** | Max unacked messages pushed to a consumer. Fairness and memory vs throughput. |
| **ack / nack / reject** | Per-message acknowledgement. `requeue=false` is what sends a message to the DLX. |
| **`redelivered`** | A boolean hint that a message was delivered before. **Not** a retry counter. |
| **DLX** | Dead-letter exchange: where rejected, expired or overflowed messages are republished. |
| **`x-death`** | Header RabbitMQ maintains with the dead-lettering history, including a usable count. |
| **TTL retry ladder** | Consumerless queues with `x-message-ttl` + DLX back to the work queue = broker-side backoff. |
| **Head-of-line TTL** | Queue TTL is evaluated only at the head — never mix delays in one retry queue. |
| **Quorum queue** | Raft-replicated durable queue; supports `x-delivery-limit`. The modern default. |
| **RabbitMQ stream** | Append-only replicated log with offset-based, replayable consumption. |
| **Resource alarm** | Memory/disk watermark crossed → the broker **blocks publishers**. Kafka has no equivalent. |
| **Visibility timeout (SQS)** | Receiving hides a message for N seconds; you must `DeleteMessage`. Not an ack. |
| **Redrive policy (SQS)** | `maxReceiveCount` → DLQ. Broker-side poison protection. |
| **Filter policy (SNS)** | Attribute-based subscription filtering — the AWS analogue of a topic binding. |
| **Shard (Kinesis)** | The unit of parallelism and of the throughput quota; the Kafka partition analogue. |
| **Claim check** | Store a large payload in S3 and send a reference — for both brokers' size limits. |

---

## Self-check (answer out loud, no notes)

1. A publisher gets a confirm. Name two reasons the message might still never be delivered to anyone.
2. Why is `basic.nack(tag, false, true)` in a generic catch block a production incident?
3. You need 1s/5s/25s retry delays. Why do you need three queues rather than one with per-message
   TTLs?
4. Give the one-sentence Kafka-vs-RabbitMQ distinction, then name the three questions that decide it.
5. In SQS, what happens if your handler takes longer than the visibility timeout — and what are the
   two fixes?
