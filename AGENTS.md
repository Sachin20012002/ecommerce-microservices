# Repository Working Agreement

This file defines the default rules for humans and coding agents working in this repository.

## Project objective

Revive this e-commerce backend as a reliable, explainable distributed-systems portfolio project. Prefer a small number of complete, tested business flows over adding disconnected technologies or unfinished services.

The primary reference journey is:

1. Register or sign in.
2. Browse products.
3. Add products to an authenticated customer's cart.
4. Validate the cart and create an order.
5. Process or confirm payment.
6. publish order-domain events.
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
- Kafka events must have a clear owner, purpose, stable name, event ID, version, and timestamp.
- Consumers must tolerate duplicate delivery where side effects are possible.
- Elasticsearch is a derived search index, never the product source of truth.
- Notification delivery must not be part of the synchronous order transaction.

## Security

- Never commit API keys, signing keys, passwords, tokens, phone numbers, or other secrets.
- Use environment variables or an approved secret provider. Commit only safe examples.
- Treat any secret previously committed to Git as compromised and rotate it.
- Never return provider secrets through an API.
- Authentication does not replace authorization: services must check resource ownership and roles.
- Avoid logging tokens, OTPs, payment data, or personal information.

## Implementation conventions

- The target runtime is Java 17 or newer; changes to it require an ADR.
- Use `BigDecimal` for money and specify currency and rounding behavior.
- Validate API input at the boundary and return a consistent error format.
- Avoid business logic in controllers.
- Prefer constructor injection.
- Do not use `System.out` for application logging.
- Database schema changes must use migrations once migration tooling is introduced.
- Public contracts require backward-compatibility consideration.

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

