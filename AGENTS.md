# Repository Working Agreement

This file defines the default rules for humans and coding agents working in this repository.

## Project objective

Evolve this backend toward the production architecture defined in
`docs/architecture/target-architecture.md`. The target is a secure, observable,
failure-aware e-commerce system deployed to AWS on Amazon EKS. Prefer a small
number of complete, tested business flows over disconnected technologies or
unfinished services.

The production architecture is the source of truth. Non-production environments
may reduce capacity and redundancy to control cost, but must not weaken service
boundaries, contracts, consistency, security, or application design.

The primary reference journey is:

1. Register or sign in.
2. Browse products.
3. Add products to an authenticated customer's cart.
4. Validate the cart and create an order.
5. Reserve inventory and authorize payment through the Order-owned checkout Saga.
6. Confirm or reject the order and publish order-domain events.
7. Deliver notifications asynchronously.

## Branch and change discipline

- Create one branch for each logical work item.
- Use branch prefixes such as `chore/`, `fix/`, `feature/`, `test/`, and `docs/`.
- Do not mix formatting, dependency upgrades, refactoring, and product behavior in one branch unless they are inseparable.
- Preserve unrelated working-tree changes.
- Keep commits reviewable and explain why a change is needed.
- Do not commit generated build output, IDE state, credentials, or local runtime data.

## Architecture workflow

Before a material architecture or technology change:

1. Describe the problem and current implementation.
2. Identify functional and non-functional requirements.
3. Compare realistic alternatives and their trade-offs.
4. State the decision for both the project's current scale and a larger-scale deployment.
5. Record significant decisions as an ADR under `docs/adr/`.
6. Implement and verify the smallest coherent increment.

Do not introduce infrastructure solely to increase the number of technologies in the project.

## Service boundaries

- A service owns its data; other services must not query its database directly.
- REST is the external API unless a documented decision replaces it.
- Synchronous internal calls must define timeouts and failure behavior.
- Catalog owns products, categories, and descriptive SKU data. Pricing is an
  independent bounded context, initially isolated within Catalog.
- Cart owns customer cart intent only. Order owns the checkout process manager;
  Inventory and Payment remain autonomous services.
- Kafka events must have a clear owner, purpose, stable name, event ID, aggregate
  key, version, timestamp, correlation ID, and causation ID.
- SQS is preferred for point-to-point commands and work queues; Kafka/MSK is the
  durable backbone for domain facts, replay, and fan-out.
- Producers that update a database and publish an event use an outbox or an
  equivalent atomic change-capture mechanism. Side-effecting consumers use an
  inbox/deduplication record and must tolerate duplicate delivery.
- OpenSearch is a rebuildable projection, never the Catalog source of truth.
- Do not claim exactly-once business processing.
- Notification delivery must not be part of the synchronous order transaction.

## Security

- Never commit API keys, signing keys, passwords, tokens, phone numbers, or other secrets.
- Use environment variables or an approved secret provider. Commit only safe examples.
- Treat any secret previously committed to Git as compromised and rotate it.
- Never return provider secrets through an API.
- Authentication does not replace authorization: services must check resource ownership and roles.
- Avoid logging tokens, OTPs, payment data, or personal information.
- Production secrets belong in AWS Secrets Manager. Parameter Store and
  Kubernetes ConfigMaps are for non-secret configuration.
- Authentication moves toward Cognito/OIDC. Each service still enforces its own
  authorization and ownership rules.

## Implementation conventions

- The target runtime is Java 21 with a current compatible supported Spring Boot
  3.x baseline. Exact Spring Boot and Spring Cloud versions are selected and
  verified during the runtime-upgrade work item.
- Use `BigDecimal` for money and specify currency and rounding behavior.
- Validate API input at the boundary and return a consistent error format.
- Avoid business logic in controllers.
- Prefer constructor injection.
- Do not use `System.out` for application logging.
- Database schema changes must use migrations once migration tooling is introduced.
- Public contracts require backward-compatibility consideration.

## Platform conventions

- Amazon EKS/Kubernetes is the production container platform; ECR stores images.
- Kubernetes Services/DNS replace Eureka. AWS/Kubernetes configuration mechanisms
  replace Spring Cloud Config.
- Terraform owns AWS infrastructure. GitHub Actions owns CI and artifact creation.
  GitOps, preferably Argo CD, owns production Kubernetes deployment and promotion.
- CloudWatch with OpenTelemetry and X-Ray is the core telemetry direction.
- Redis is introduced only after a measured caching or throttling use case is
  documented; it must not become authoritative business state.

## Testing expectations

- Unit tests must isolate business rules and contain meaningful assertions.
- Integration tests should use disposable infrastructure such as Testcontainers.
- Tests must not contact real payment, email, SMS, or other paid providers.
- Provider integrations require fake or sandbox adapters.
- Cover success, validation, authorization, duplicate, timeout, and failure paths as appropriate.

## Definition of done

A work item is complete when:

- The intended behavior and non-goals are documented.
- Relevant architecture decisions are recorded.
- Code builds from a clean checkout using documented commands.
- Automated tests cover the important behavior.
- Secrets and private data are absent.
- Local setup documentation is updated.
- Operational behavior, including logs, health, retry, and failure handling, is considered.
- The implementation can be explained, including alternatives and trade-offs.

