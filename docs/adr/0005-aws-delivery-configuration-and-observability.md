# ADR-0005: Standardize AWS delivery, configuration and observability

- Status: Accepted
- Date: 2026-08-17
- Owners: Project maintainers

## Context

The repository has no infrastructure-as-code, container delivery pipeline,
production secret/configuration mechanism, deployment reconciler, or distributed
observability direction. Direct CI mutation of production would combine artifact
creation and privileged deployment in one trust boundary.

## Decision

- Terraform manages AWS infrastructure and managed EKS add-ons.
- GitHub Actions uses AWS OIDC federation for CI, tests, checks, SBOM/provenance,
  image creation and ECR publication.
- GitOps, preferably Argo CD, owns reviewed production Kubernetes deployment,
  promotion, reconciliation and rollback.
- AWS Secrets Manager owns secrets. Parameter Store and Kubernetes ConfigMaps own
  non-secret configuration.
- CloudWatch is the primary operational backend; OpenTelemetry provides logs,
  metrics and traces with X-Ray-compatible tracing.
- S3 plus CloudFront serves product media. OpenSearch remains a rebuildable Search
  projection. Redis is optional and evidence-driven.

## Alternatives considered

- **Direct deployment from GitHub Actions:** fewer components, but grants CI broad
  production mutation rights and weakens drift detection and promotion review.
- **Spring Cloud Config and repository secrets:** preserve legacy conventions but
  duplicate platform capabilities and create unsafe secret distribution.
- **Provider-specific application instrumentation only:** reduces setup but couples
  services to one telemetry API and weakens trace portability.

## Consequences

- Infrastructure, artifact creation and deployment reconciliation have separate
  ownership and credentials.
- Production promotion is declarative and auditable; emergency procedures still
  require documented, time-bound access.
- Telemetry cost, retention, sampling, PII redaction and cardinality require active
  governance.

## Validation

- Terraform can recreate an environment from reviewed state and configuration.
- CI publishes immutable artifacts without long-lived AWS credentials.
- Argo CD detects drift and promotes/rolls back by reviewed desired-state changes.
- A checkout trace correlates HTTP, SQS, Kafka and database work, with actionable
  alerts for Saga and messaging failure modes.
