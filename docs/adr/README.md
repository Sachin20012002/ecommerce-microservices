# Architecture Decision Records

ADRs capture decisions that materially affect service boundaries, contracts, persistence, security, consistency, deployment, or operations.

## Lifecycle

1. Copy `template.md` to the next numbered file.
2. Set the status to `Proposed` while alternatives are being debated.
3. Change it to `Accepted` when implementation is authorized.
4. Do not rewrite an accepted decision to hide changed thinking. Add a new ADR that supersedes it.

## Index

| ADR | Status | Decision |
| --- | --- | --- |
| [0001](0001-retain-microservices-for-learning.md) | Superseded | Retain service boundaries as an explicit learning constraint |
| [0002](0002-production-runtime-and-eks-platform.md) | Accepted | Standardize Java 21, supported Spring Boot 3.x and Amazon EKS |
| [0003](0003-bounded-contexts-and-data-ownership.md) | Accepted | Define target service boundaries and owned data stores |
| [0004](0004-checkout-saga-and-message-reliability.md) | Accepted | Use an Order-owned Saga, SQS commands, MSK facts and Outbox/Inbox |
| [0005](0005-aws-delivery-configuration-and-observability.md) | Accepted | Standardize Terraform, CI/GitOps, AWS configuration and telemetry |

