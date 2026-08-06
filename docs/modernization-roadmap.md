# Modernization Roadmap

Each phase is implemented on one or more dedicated branches. Architecture decisions are discussed and recorded before material implementation.

## Phase 0: Governance and baseline

- Add repository working agreement and architecture documentation.
- Record current limitations without overstating production readiness.
- Establish ADR and branch conventions.
- Capture a clean baseline build when the required toolchain is available.

Exit criterion: project rules, current architecture, target direction, and priorities are reviewable.

## Phase 1: Credential and security containment

- Revoke and rotate exposed Razorpay and Twilio credentials.
- Remove secrets and personal phone numbers from source, configuration, tests, and Git history.
- Introduce safe environment-variable configuration and `.env.example` files.
- Prevent secret-bearing files from being committed.
- Correct README security claims.

Exit criterion: secret scanning finds no committed credentials, and applications fail safely when required secrets are absent.

## Phase 2: Reproducible development baseline

- Add Maven Wrapper and align on a supported Java runtime.
- Add containerized MySQL, MongoDB, Kafka and Elasticsearch dependencies.
- Define database creation and health checks.
- Fix gRPC port/client configuration.
- Document deterministic startup, shutdown, and smoke-test commands.

Exit criterion: a clean checkout can build and start locally using documented commands.

## Phase 3: Identity and authorization

- Decide between asymmetric self-issued JWT and an OIDC provider.
- Authenticate at the gateway without relying on gateway checks alone.
- Enforce customer ownership in cart, checkout, and order services.
- Restrict administrative catalog operations.
- Add safe CORS, actuator, rate-limit, and OTP policies.

Exit criterion: automated tests prove anonymous, customer, cross-customer, and administrator behavior.

## Phase 4: Complete checkout and order lifecycle

- Replace floating-point currency with `BigDecimal`.
- Calculate authoritative checkout summaries.
- Validate current products, prices, availability and ownership.
- Define inventory reservation and release behavior.
- Create idempotent orders with explicit state transitions.
- Integrate payment through a provider interface and verify callbacks/signatures.
- Complete or replace the cart following order confirmation.

Exit criterion: the reference purchase flow works and its important failures are tested.

## Phase 5: Reliable domain events

- Define versioned event envelopes and topic naming.
- Publish create, update and delete product lifecycle events.
- Add consumer idempotency, retry and dead-letter handling.
- Introduce an outbox or another documented atomic-publication strategy.
- Make the search projection rebuildable.

Exit criterion: database commits and event delivery have documented, tested recovery behavior.

## Phase 6: Notification service

- Record notification boundary and delivery-semantics ADRs.
- Consume order events asynchronously.
- Add template, email, SMS and local fake-provider abstractions.
- Persist delivery attempts and enforce idempotency.
- Add bounded retry and dead-letter processing.
- Expose operational status without exposing message content or personal data.

Exit criterion: duplicate events and transient/permanent provider failures are tested without contacting real providers.

## Phase 7: Quality and operations

- Expand unit, integration, contract, and end-to-end coverage.
- Add schema migrations.
- Add structured logs, correlation IDs, metrics, traces, dashboards and alerts.
- Define timeouts, circuit breaking and graceful degradation.
- Upgrade supported framework and dependency versions through dedicated ADRs.

Exit criterion: CI verifies the clean build and reference journey, and operational failure modes are observable.

## Phase 8: Portfolio and interview readiness

- Rewrite the README around demonstrated capabilities and limitations.
- Add architecture and sequence diagrams.
- Provide sanitized API examples and demo data.
- Record scale assumptions, bottlenecks and alternative designs.
- Prepare concise explanations of consistency, idempotency, partitioning, caching and failure recovery.

Exit criterion: every public claim is supported by running code, tests, or explicitly labeled future work.
