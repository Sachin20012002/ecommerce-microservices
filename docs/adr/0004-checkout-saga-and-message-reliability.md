# ADR-0004: Orchestrate checkout with durable commands and domain events

- Status: Accepted
- Date: 2026-08-17
- Owners: Project maintainers

## Context

The legacy checkout does not validate authoritative price or inventory and has no
durable order workflow, payment verification, compensation, or reliable database
to-message publication. Existing Kafka topics mix commands and facts without
schemas or idempotency.

## Decision

Order owns a persisted checkout Saga/process manager:

```text
Cart snapshot -> authoritative Pricing -> PENDING Order + Saga/outbox
-> ReserveInventory -> InventoryReserved/Rejected
-> AuthorizePayment -> confirm/reject Order
-> compensate Payment/Inventory as required
-> OrderConfirmed -> asynchronous consumers
```

- Use SQS with DLQs for point-to-point Saga commands and work queues.
- Use MSK/Kafka for durable domain facts that require replay, ordering by
  aggregate, fan-out, or projection building.
- Relational producers use a transactional outbox. Side-effecting consumers use a
  transactional inbox/deduplication record and idempotent handlers.
- Treat delivery as at least once. Do not claim exactly-once business processing.
- Separate payment authorization from capture. Capture timing follows fulfillment
  and business policy; Payment owns capture, void, refund and reconciliation.
- Persist ambiguous provider outcomes and reconcile them rather than guessing.

## Alternatives considered

- **Synchronous distributed transaction:** unavailable across these autonomous
  stores/providers and creates unacceptable coupling.
- **Choreographed checkout:** reduces central coordination but makes this ordered,
  compensating business process harder to understand and recover.
- **Kafka for every command:** provides one transport but is a poor default for
  single-consumer work queues, per-command DLQs and operational redrive.
- **Immediate payment capture during checkout:** conflates authorization with
  settlement and creates avoidable refund exposure.

## Consequences

- Checkout is normally asynchronous beyond a bounded request latency and exposes
  an Order status resource.
- Inventory and Payment remain autonomous and can reject commands independently.
- Duplicate commands, events, webhooks and requests are normal operating cases.
- Operators need visibility and replay procedures for Saga age, outbox backlog,
  consumer lag, retries, ambiguous payments and DLQs.

## Validation

Automated tests cover success, inventory rejection, payment rejection, ambiguous
timeouts, duplicate delivery, expired reservations, compensation, restart/replay
and terminal-state invariants.
