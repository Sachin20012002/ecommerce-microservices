# Production Modernization Roadmap

The destination is [Production Target Architecture](architecture/target-architecture.md).
Each phase requires a dedicated work item, relevant ADR updates, tests, operational
evidence, and reviewable migration/rollback steps. A cheaper development platform
may reduce managed-service capacity; it does not change the application design.

Known security defects in the legacy implementation are addressed while each
affected boundary is redesigned and verified again during final hardening. This
sequence does not authorize postponing an actively exploitable production risk;
the legacy system is not a production deployment.

## 1. Architecture/contracts and engineering baseline

- Define bounded-context ownership, API/event schemas, money rules, identity
  propagation, idempotency keys, error contracts and data-classification rules.
- Establish reproducible builds, local dependencies, migrations, test strategy,
  contract validation and clean-checkout verification.

Exit: target contracts and baseline quality gates are executable and reviewable.

## 2. Java/Spring modernization

- Upgrade to Java 21 and select a current compatible supported Spring Boot 3.x and
  Spring Cloud baseline.
- Resolve Jakarta migration, dependency compatibility, security defaults,
  actuator behavior and container runtime requirements.

Exit: the supported runtime builds and tests reproducibly without changing target
business ownership.

## 3. Catalog consolidation and ownership cleanup

- Consolidate Product and Category into Catalog on Aurora PostgreSQL.
- Isolate Pricing behind its own interfaces and persistence ownership within the
  Catalog deployment.
- Remove sellable quantity and media bytes from Catalog; define SKU and media
  metadata contracts.

Exit: Catalog, Pricing and future Inventory ownership no longer overlap.

## 4. Customer/Cart cleanup

- Move authentication toward Cognito/OIDC and place profiles/addresses in Customer
  Profile on Aurora PostgreSQL.
- Restrict Cart to authenticated customer intent and migrate it to DynamoDB with
  conditional updates, versioning and an abandoned-cart policy.
- Remove cross-service Cart provisioning and Cart-owned checkout/payment state.

Exit: identity, profile and Cart have independent ownership and authorization.

## 5. Inventory and Payment boundaries

- Create Inventory with SKU/location balances, reservations, releases, expiry and
  an auditable adjustment model on dedicated Aurora PostgreSQL.
- Create Payment with provider abstraction, intent, authorization, capture,
  void/refund, signed webhook inbox, idempotency and reconciliation on dedicated
  Aurora PostgreSQL.

Exit: Catalog and Order no longer own inventory or provider payment state.

## 6. Order/Saga implementation

- Make Order the durable checkout process manager with immutable price/item
  snapshots, explicit states and request idempotency.
- Implement reserve-inventory, authorize-payment, confirmation/rejection,
  ambiguous-outcome recovery and compensation paths.
- Keep payment authorization and capture as separate operations governed by
  fulfillment/business policy.

Exit: success, rejection, timeout, duplicate and compensation paths are tested.

## 7. Outbox/Inbox and SQS/MSK event architecture

- Add transactional outboxes, consumer inboxes/deduplication and idempotent side
  effects.
- Use SQS/DLQs for point-to-point commands and work queues; use MSK/Kafka for
  durable domain facts, replay and fan-out.
- Define schema compatibility, partition keys, retry classes, DLQ operations,
  replay and backlog observability. Do not claim exactly-once processing.

Exit: database state and message delivery have tested recovery behavior.

## 8. Search, Notification and media

- Replace Filter with a rebuildable OpenSearch projection using snapshot plus
  event catch-up.
- Add asynchronous Notification delivery with provider fakes, deduplication,
  bounded retries and DLQs.
- Move media objects to S3/CloudFront while Catalog retains metadata.

Exit: Search can be rebuilt and downstream latency cannot block Order confirmation.

## 9. AWS/EKS, Terraform and GitOps platform

- Provision multi-AZ networking, EKS, ECR, IAM, data services, messaging, edge,
  secrets and base observability through Terraform.
- Remove Eureka and Spring Cloud Config in favor of Kubernetes discovery and
  AWS/Kubernetes configuration.
- Use GitHub Actions with OIDC for CI/artifacts and Argo CD GitOps for reviewed
  production deployment, promotion and rollback.

Exit: immutable artifacts can be promoted and the platform can be recreated from
reviewed infrastructure and deployment definitions.

## 10. Final hardening

- Complete authorization, secret rotation, network policy, data protection,
  dependency/container/IaC scanning and payment-security verification.
- Add CloudWatch/OpenTelemetry/X-Ray dashboards, SLOs, alerts and runbooks.
- Load-test access patterns before introducing Redis or changing database choices.
- Exercise AZ/dependency failure, Saga recovery, DLQ replay, payment
  reconciliation, backup restoration, scaling and rollback.

Exit: production acceptance criteria in the target architecture are demonstrated
by automated tests and operational evidence.
