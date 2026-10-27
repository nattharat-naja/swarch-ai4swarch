# Compare Run A and Run B

| Criterion | Run A | Run B |
|---|---|---|
| Uses scenario numbers (latency, peak, MailFast limits)? | No. Says performance targets and limits are unspecified. | Yes: checkout p95 ≤ 500 ms, MailFast p95 1.5 s / p99 8 s, peak 200 orders/s, MailFast limit 50 emails/s. |
| Options considered | 1. Synchronous calls. 2. Event-driven messaging. 3. Hybrid approach. | 1. Synchronous calls. 2. RabbitMQ + durable outbox. 3. In-process goroutines/background worker. |
| Exactly-once loyalty points and MailFast rate limit | Mentions deduplication/idempotency generally but doesn't specify how. Doesn't address the rate limit. | Uses `orderId` as an idempotency key for loyalty and limits MailFast to 50 calls/s. However, exactly-once effects still depend on the loyalty service implementing atomic deduplication. |
| Anything invented? | No obvious invented numbers or technologies, but leaves important scenario requirements unaddressed. | Potentially. RabbitMQ, durable outbox, dead-letter queues, publisher confirms, and `orderId` as the idempotency key are proposed design choices, not necessarily scenario requirements. Verify them against the scenario and repository. |

## What I changed from the AI's draft

| # | What you changed from the AI's draft | Why |
|---:|---|---|
| 1 | Added the concrete latency targets, peak order volume, and MailFast rate limit. | The architecture must be justified using actual scenario constraints rather than generic assumptions. |
| 2 | Specified an idempotency key for loyalty points and required atomic duplicate protection. | At-least-once delivery can cause duplicate processing; the business effect must remain exactly once per order. |
| 3 | Added a durable outbox, independent consumer retries, dead-letter handling, and backlog monitoring, while marking persistence details for verification. | These measures improve recovery from failures, but must not be presented as existing capabilities without evidence. |
