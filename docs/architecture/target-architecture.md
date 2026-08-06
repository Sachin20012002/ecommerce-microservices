# Target Architecture

## Goal

Create a locally reproducible, secure and testable reference implementation of one complete e-commerce journey. The target is not to simulate every production capability; it is to demonstrate sound boundaries and explicitly reasoned trade-offs.

## Target business flow

```text
Authenticate customer
        |
Browse product catalog
        |
Modify owned cart
        |
Validate price and inventory
        |
Create idempotent order
        |
Confirm payment outcome
        |
Publish versioned order event
        |
Notification service delivers through configured channels
```

## Direction

- Retain the gateway and service boundaries initially for distributed-systems learning.
- Make product/inventory and order data authoritative in relational storage.
- Treat Elasticsearch as a rebuildable, eventually consistent projection.
- Use synchronous communication only when the caller needs an immediate authoritative answer.
- Use Kafka for decoupled propagation to search and notifications.
- Introduce reliable event publication before claiming dependable event-driven behavior.
- Enforce authentication at the edge and authorization/resource ownership in each service.
- Use asymmetric JWT signing or a standards-based identity provider after a dedicated decision.
- Run local dependencies through containers and test integrations with disposable infrastructure.

## Notification boundary

The notification service will consume versioned domain events such as `OrderConfirmed` and `OrderCancelled`. It will own templates, channel selection, delivery attempts, provider adapters, retry policy, dead-letter handling, and idempotency records. Order processing will not wait for email or SMS delivery.

Local development will use fake or local providers. Real-provider credentials will never be required for automated tests.

## Explicit non-goals for the first revival milestone

- Multi-region deployment
- Exactly-once delivery claims
- A custom workflow engine
- A separate service for every entity
- Kubernetes before the containerized local flow is reliable
- Premature performance optimization without measurements

## Success criteria

- A new contributor can start the system from a clean checkout using documented commands.
- The reference customer journey completes end to end.
- Invalid ownership, duplicate requests, provider failures, and unavailable dependencies have defined behavior.
- Unit and integration tests verify important business and integration paths.
- No credentials or personal data are committed or logged.
- Architecture decisions and limitations can be explained during an interview.

