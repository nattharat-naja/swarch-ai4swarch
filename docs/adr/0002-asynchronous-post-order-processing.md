# ADR-0002: Asynchronous Post-Order Processing with RabbitMQ

- **Status:** Proposed
- **Date:** 2026-10-27
- **Deciders:** To be confirmed

## Context and problem

Requirement R-12 says that after an order is placed, the system must send a confirmation email through MailFast, award loyalty points through `loyaltyservice`, and record an `OrderPlaced` event for marketing. Additional post-order work may be introduced later, such as SMS, fraud scoring, and recommendation updates.

The scenario states that checkout has a p95 target of no more than 500 ms, while MailFast has p95 latency of 1.5 s and p99 latency of 8 s. Peak order traffic is 200 orders/s, while MailFast is limited to 50 emails/s. A synchronous MailFast call would therefore risk exceeding the checkout latency target, and the provider's stated rate limit is below peak order traffic if each order generates one confirmation email.

The current code and persistence behavior must be verified before implementation. In particular, this ADR does not assume that the current checkout flow already persists orders or has an outbox.

## Decision drivers

1. **Availability:** Temporary failures in email, loyalty, or analytics should not unnecessarily fail checkout.
2. **Performance:** Keep `PlaceOrder` p95 at or below 500 ms.
3. **Correctness:** Award loyalty points within one minute and prevent duplicate point awards for an order.
4. **Scalability and modifiability:** Allow new post-order actions without adding synchronous calls to checkout.
5. **Operational fit:** Prefer an open-source component that can be self-hosted on Kubernetes and fits the team's experience. The scenario's stated team experience with RabbitMQ should be confirmed against the source scenario.

## Considered options

1. Call all downstream services synchronously from `checkoutservice`.
2. Publish an `OrderPlaced` event through RabbitMQ, with a durable outbox and independent consumers.
3. Run goroutines or an in-process background worker in `checkoutservice`, without a message broker.

### Pros and cons

| Option | Pros | Cons |
|---|---|---|
| 1. Synchronous calls | Simple flow; downstream results and errors are immediately visible. | MailFast p95 of 1.5 s exceeds the 500 ms checkout p95 target; checkout is coupled to downstream availability and latency; 50 emails/s is below the 200 orders/s peak. |
| 2. RabbitMQ + durable outbox | Decouples checkout response from consumer work; queues work during temporary downstream failures; allows independent retries and additional consumers. | Adds RabbitMQ, outbox storage, monitoring, retries, and dead-letter operations; consumers must safely handle duplicate delivery; asynchronous work may complete later. |
| 3. In-process background worker | Avoids adding a broker and may be simple to prototype. | Work held only in process memory can be lost on process failure; durable retry and coordination across replicas need additional design; independent consumers are harder to operate. |

## Decision

Choose option 2: publish an `OrderPlaced` event asynchronously through RabbitMQ and use separate consumers for MailFast email delivery, loyalty points, and marketing analytics.

The checkout path must not wait for consumers to finish their work. Use a durable outbox so that the order state and the event to be published can be committed atomically in the same durable transaction, where the application's persistence model permits it. A publisher then relays outbox records to RabbitMQ and marks them as published only after the broker has confirmed receipt. The exact database, schema, and transaction boundary must be established after inspecting the repository; do not implement this as a claim about existing persistence.

Assume at-least-once message delivery. Consumers must be idempotent. For loyalty, use a stable order identifier (such as `orderId`, if the domain confirms it is unique and available) as the idempotency key. The deduplication record and points award must be committed atomically, so redelivery cannot award points twice. The service must acknowledge a message only after its effect is durably committed.

Limit MailFast requests to no more than 50 emails per second, including retry traffic, using a rate limiter coordinated across all consumer instances. Retry transient errors with bounded backoff. Route messages that cannot be processed after the configured retry policy to a dead-letter queue for inspection and controlled replay. Define alerting and recovery procedures before production use.

At 200 orders/s, if every order produces one email and MailFast processes 50 emails/s, the potential backlog grows by about 150 emails/s while that peak persists. This is a simple rate difference, not a complete capacity forecast. Measure backlog and age, and agree on acceptable email delay, retention, and recovery capacity. RabbitMQ buffers the mismatch but does not remove it.

## Consequences

### Positive

- MailFast, loyalty, and analytics latency does not need to be part of the checkout response path.
- Consumers can be retried and scaled independently.
- New post-order actions can be added as consumers rather than synchronous checkout calls.
- Durable outbox publication and idempotent consumers improve recovery from failures and duplicate delivery.

### Negative

- RabbitMQ, outbox storage, retries, dead-letter handling, and operational monitoring add complexity.
- Post-order effects are eventually consistent; email and analytics may be delayed.
- MailFast's rate limit can cause backlog growth during sustained traffic above 50 emails/s.

### Risks and mitigations

- **Order/event dual-write risk:** Define a durable transaction that commits the order and outbox event together. Verify the actual order persistence model first.
- **Duplicate loyalty processing:** Enforce durable idempotency at `loyaltyservice`; the deduplication record and points update must be atomic.
- **MailFast throttling:** Apply a distributed rate limit across all consumers, count retries toward the limit, and monitor queue depth and oldest-message age.
- **Poison messages or repeated failure:** Use bounded retries and a dead-letter queue, with alerts and a documented replay procedure.
- **Broker or publisher outage:** Keep unpublished events durably in the outbox and resume publishing after recovery; monitor outbox age and volume.
- **Payment/shipping succeeded but order/outbox persistence failed:** Define order state transitions and compensation/reconciliation behavior before rollout. The current repository must be inspected to determine the actual failure boundary.

## Validation

- Load-test `PlaceOrder` at the scenario's peak of 200 orders/s and verify p95 remains at or below 500 ms while consumers are slow or unavailable.
- Simulate MailFast latency, transient failures, throttling, and outage. Verify the total request rate, including retries, never exceeds 50 emails/s across all consumer instances.
- Deliver the same order event multiple times and concurrently; verify `loyaltyservice` awards points only once per order and meets the one-minute requirement.
- Stop and restart consumers and RabbitMQ while events are pending; verify durable events are processed after recovery without duplicate loyalty effects.
- Test publisher crashes before and after broker confirmation to verify that events are not silently lost and duplicate publication is safe.
- Verify dead-letter alerts, replay procedures, and monitoring for outbox age, queue depth, oldest-message age, consumer errors, and MailFast throttling.
- Confirm the order/outbox transaction and failure behavior using the actual repository and persistence implementation.

## Open questions

- Who approves the decision and owns RabbitMQ, outbox storage, and production operations?
- Which database and transaction boundary will make the order and outbox event durable together?
- What should happen if payment or shipping succeeds but order/outbox persistence fails?
- What is the acceptable confirmation-email delay, queue retention, and maximum backlog?
- What exact MailFast retry guidance, authentication, and rate-limit semantics apply?
- Does `loyaltyservice` support a durable idempotency key and atomic points update? What is the authoritative order identifier?
- What event schema, retention, and delivery expectations does marketing analytics require?
- What retry limits, dead-letter alert thresholds, and replay permissions should be used?
- Are RabbitMQ and the team's claimed operational experience confirmed by the scenario and repository?
