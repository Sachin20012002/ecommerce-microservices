# Production Target Architecture

## Authority and scope

This document is the source of truth for the target architecture. The current
codebase remains a legacy starting point and does not yet implement this design.
Accepted ADRs record the reasons behind the decisions.

Production runs on AWS with Amazon EKS. Cheaper non-production environments may
use smaller or single-instance infrastructure, shorter retention, and reduced
redundancy. They must preserve the same application boundaries, contracts,
security controls, consistency semantics, and deployable artifacts.

The initial production topology is one region across multiple Availability Zones.
Multi-region business-state replication is not implied by this target.

## Runtime and engineering baseline

- Java 21.
- A current compatible supported Spring Boot 3.x baseline. Exact Spring Boot and
  Spring Cloud versions are selected and verified during the upgrade task.
- Explicit API and event contracts, database migrations, deterministic builds,
  automated tests, and container images promoted by immutable digest.
- `BigDecimal` plus explicit currency and rounding rules for money.

## Bounded contexts

| Boundary | Owns | Does not own |
| --- | --- | --- |
| Identity | Cognito/OIDC authentication, token issuance, federation and MFA policy | Customer addresses, carts or authorization decisions inside services |
| Customer Profile | Customer profile, identity mapping, saved addresses and preferences | Authentication credentials or Cart state |
| Catalog | Products, category taxonomy, brands, descriptive attributes, SKU definitions and media metadata | Price offers, sellable inventory or media objects |
| Pricing | Authoritative offers, discounts, tax inputs, currency and effective periods | Product description or stock |
| Inventory | On-hand and available-to-promise quantities, reservations, releases and adjustments by SKU/location | Catalog description, orders or payment state |
| Cart | Customer-owned cart intent and cart lines | Checkout, inventory reservation, payment or order state |
| Order | Immutable order snapshots, order lifecycle, idempotency and the checkout Saga/process manager | Inventory balance, payment provider state or notification delivery |
| Payment | Payment intents, provider references, authorization, capture, void/refund, webhooks and reconciliation | Order or inventory state |
| Search | OpenSearch query model built from domain facts | Authoritative product, price or inventory data |
| Notification | Templates, channel policy, delivery attempts, provider adapters and delivery idempotency | Synchronous order completion |

Product and Category consolidate into Catalog. Pricing is an independent bounded
context because its lifecycle and authority differ from descriptive product data.
It is initially implemented as a strongly isolated module within the Catalog
deployment, with its own interfaces and ownership rules, so it can be extracted
without redefining contracts.

Do not add Supplier, Fulfillment, Shipment, Returns, Recommendation, or other
services until implemented business requirements justify their ownership and
failure boundaries.

## Storage ownership

| Boundary | Target source of truth | Decision |
| --- | --- | --- |
| Customer Profile | Aurora PostgreSQL | Relational integrity, auditability and controlled PII access |
| Catalog | Aurora PostgreSQL | Current taxonomy, relationship and administration requirements are relational; DynamoDB requires a demonstrated access-pattern decision |
| Pricing | Aurora PostgreSQL, initially isolated within Catalog | Effective periods, currencies, audit history and constraints |
| Inventory | Dedicated Aurora PostgreSQL | Correct reservation and adjustment transactions first; revisit partitioning or DynamoDB only from measured contention and access patterns |
| Cart | DynamoDB | Customer/cart-key access, conditional mutation, elastic throughput and abandoned-cart TTL |
| Order | Dedicated Aurora PostgreSQL | Order snapshots, state transitions, Saga state, idempotency and outbox |
| Payment | Dedicated Aurora PostgreSQL | Isolation, audit, provider-event uniqueness and reconciliation |
| Search | Amazon OpenSearch Service | Rebuildable read projection only |
| Product media | S3 through CloudFront | Catalog stores metadata and object references, not media bytes |

Production transactional boundaries do not share schemas or database credentials.
Non-production may consolidate physical database capacity while retaining logical
ownership and independent migrations.

ElastiCache Redis is optional. Introduce it only for a measured hot-read,
rate-limiting, or short-lived computation use case. Inventory, Order, Payment,
Saga, and idempotency state remain authoritative in their owned stores.

## Communication model

- External APIs use REST/JSON unless an ADR replaces the contract.
- Immediate authoritative reads may use REST or gRPC with deadlines, bounded retry
  budgets, cancellation, identity propagation and defined failure behavior.
- Amazon MSK/Kafka carries durable domain facts that need replay, ordering by an
  aggregate key, multiple consumers, or projection building.
- SQS with DLQs is preferred for point-to-point Saga commands and work queues.
- Service APIs and messages expose stable contracts, not persistence entities.
- Kafka and SQS delivery are treated as at least once. The system does not claim
  exactly-once business processing.

### Event contract and reliability

Every event has a stable domain name, event ID, aggregate type and ID, partition
key, schema version, occurrence timestamp, producer, correlation ID and causation
ID. Payloads contain the minimum non-sensitive data required by consumers.

Relational producers write domain state and an outbox record in the same database
transaction. A retriable publisher or change-data-capture connector delivers the
outbox to MSK or SQS. Side-effecting consumers persist an inbox/deduplication
record atomically with their local state change. DynamoDB producers use DynamoDB
Streams when an event is required.

Retries are bounded, use backoff and jitter, and distinguish transient, permanent
and poison failures. DLQs are alarmed, inspectable and replayable through an
operator-owned procedure. Event schema compatibility is checked in CI.

## Checkout Saga

Order owns the durable checkout process manager. Inventory and Payment remain
autonomous and accept commands only through their published contracts.

```text
Authenticated checkout request + idempotency key
  -> read customer-owned Cart snapshot
  -> obtain authoritative Pricing for the referenced SKUs
  -> atomically create PENDING Order + Saga state + outbox
  -> ReserveInventory command
  -> InventoryReserved | InventoryRejected
  -> on reservation success: AuthorizePayment command
  -> PaymentAuthorized | PaymentRejected | PaymentOutcomeUnknown
  -> confirm or reject Order
  -> compensate Payment and/or Inventory when required
  -> atomically publish OrderConfirmed through the outbox
  -> Search/Notification/Cart and other downstream consumers react asynchronously
```

The Order stores immutable SKU, description, unit price, discount, tax, quantity,
currency and total snapshots. It never trusts cart totals or client-supplied order
state.

Required compensation includes:

- inventory rejection: reject the order without starting payment;
- payment rejection or expiry: reject the order and release the reservation;
- failure after payment authorization but before confirmation: void the
  authorization, or refund if capture already occurred, then release inventory;
- ambiguous provider timeout: keep a recoverable state and reconcile through
  provider lookup and signed webhook events rather than assuming failure;
- duplicate requests, commands, events and callbacks: return or preserve the
  previously recorded business outcome.

Payment authorization and capture are distinct lifecycle operations. Checkout may
authorize funds, but capture timing follows fulfillment and business policy; it is
not performed blindly during checkout. Payment owns capture, void, refund,
webhook verification and reconciliation state.

Checkout may return a completed result within a bounded latency budget. Otherwise
it returns an accepted Order/status resource while the persisted Saga continues.

## Search, notification and media

- Search consumes Catalog and Pricing facts and may include availability hints.
  It is eventually consistent and can be rebuilt from an authoritative snapshot
  followed by event catch-up. Search results cannot authorize price or stock.
- Notification consumes confirmed Order facts asynchronously. Provider latency or
  failure never participates in the Order transaction.
- Media uploads use controlled direct-to-S3 flows. S3 Block Public Access remains
  enabled; CloudFront serves approved objects. Catalog stores media metadata and
  versioned object references.

## AWS/EKS platform

- Amazon EKS spans multiple Availability Zones; workloads use Kubernetes Services
  and DNS for discovery. Eureka is removed.
- AWS/Kubernetes-native configuration replaces Spring Cloud Config. Secrets
  Manager holds secrets; Parameter Store and ConfigMaps hold non-secret settings.
- ECR stores scanned, immutable container artifacts.
- Route 53, ACM, CloudFront, WAF and an AWS-managed load balancer provide the
  public edge. A custom gateway service requires a distinct policy or composition
  responsibility; routing alone is insufficient.
- MSK, SQS/DLQs, Aurora PostgreSQL, DynamoDB, OpenSearch, S3/CloudFront and optional
  ElastiCache provide managed data and integration capabilities.
- Workloads use least-privilege pod identities, private networking, restricted
  ingress/egress, encryption in transit and at rest, non-root containers,
  readiness/startup probes, graceful shutdown, disruption budgets and topology
  spreading.
- Terraform manages AWS infrastructure and managed Kubernetes add-ons.
- GitHub Actions performs CI, tests, security/schema checks, SBOM/provenance and
  artifact creation using OIDC federation rather than long-lived AWS keys.
- GitOps, preferably Argo CD, reconciles reviewed production manifests and owns
  deployment promotion and rollback. CI does not directly administer production.

## Observability and operations

CloudWatch is the primary operational backend. Applications and platform agents
emit structured logs, metrics and distributed traces through OpenTelemetry, with
X-Ray-compatible tracing and correlation across HTTP, SQS and Kafka.

Each boundary defines service-level indicators and alerts for latency, errors,
saturation and availability. The checkout flow additionally measures Saga age and
terminal outcomes, reservation expiry, payment ambiguity/reconciliation, outbox
backlog, Kafka consumer lag, retries and DLQ depth. Backups, point-in-time recovery,
restore tests and failure runbooks are part of production readiness.

## Security direction

- Authentication moves from shared symmetric JWTs toward Cognito/OIDC.
- Edge authentication does not replace service-level authorization, customer
  ownership checks or administrative roles.
- Payment is isolated and stores provider tokens/references only; no raw card data
  or provider secrets are returned or logged.
- Secrets are rotated and scoped per workload. Logs and events exclude tokens,
  OTPs, payment data and unnecessary personal data.
- Known legacy security defects are addressed while each affected boundary is
  redesigned and verified again during final hardening; they do not change the
  target boundaries.

## Explicit non-goals

- Exactly-once business-processing claims.
- A custom workflow engine before the persisted Order process manager proves
  insufficient.
- DynamoDB for Catalog without documented access patterns and trade-offs.
- Redis as an authoritative store or an unmeasured default layer.
- A separate service for every entity.
- Multi-region active-active transactions before routing, consistency, conflict
  and reconciliation policies are designed.

## Production acceptance criteria

- The documented checkout path succeeds and recovers from duplicates, dependency
  timeouts, rejected inventory, ambiguous payment outcomes and compensation.
- Every boundary has an owned source of truth, migrations, contracts, access
  controls, telemetry, health behavior and recovery procedure.
- Search is demonstrably rebuildable and Notification is outside the synchronous
  order path.
- CI creates immutable artifacts; GitOps promotes them; Terraform can recreate the
  platform; restore and failure tests validate recovery objectives.
- Reduced-cost environments change infrastructure capacity, not application
  correctness or architecture.
