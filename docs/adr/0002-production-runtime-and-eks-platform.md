# ADR-0002: Standardize the production runtime and platform

- Status: Accepted
- Date: 2026-08-17
- Owners: Project maintainers
- Supersedes: ADR-0001 where it treats local learning as the target

## Context

The legacy system uses Java 17, Spring Boot 2.7, Eureka and Spring Cloud Config,
with no production container platform. The project objective is now a production
AWS architecture rather than provisional retention of the existing platform.

## Decision

- Target Java 21 and a current compatible supported Spring Boot 3.x baseline.
  Select exact Spring Boot and Spring Cloud versions during the upgrade task after
  compatibility and migration testing.
- Run production workloads on Amazon EKS and store images in ECR.
- Use Kubernetes Services/DNS for discovery and AWS/Kubernetes-native
  configuration. Remove Eureka and Spring Cloud Config during migration.
- Keep production multi-AZ. Non-production may reduce capacity and redundancy but
  uses the same application architecture and artifacts.

## Alternatives considered

- **Retain Java 17/Spring Boot 2.7:** minimizes immediate change but carries the
  legacy framework and security baseline into all later work.
- **Use ECS:** operationally simpler, but does not meet the selected Kubernetes
  production platform direction.
- **Keep Eureka and Config Server on EKS:** duplicates capabilities, adds startup
  dependencies and preserves an unnecessary platform stack.

## Consequences

- Jakarta and dependency migrations must be completed before business redesign is
  layered onto the new runtime.
- EKS introduces cluster, networking, policy and upgrade responsibilities that
  must be managed as product infrastructure rather than hidden complexity.
- Exact framework versions are not frozen prematurely in an architectural ADR.

## Validation

- Java 21 clean build and automated tests on the selected supported Spring line.
- Services discover each other through Kubernetes without Eureka and receive
  configuration without Config Server.
- The same image digest is promoted from test to production.
