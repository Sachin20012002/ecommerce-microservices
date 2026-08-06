# ADR-0001: Retain microservices as a learning constraint

- Status: Accepted
- Date: 2026-08-07
- Owners: Project maintainers

## Context

The repository contains independently runnable Spring services even though its current feature scope and team size would not commercially require this operational complexity. The project is being revived both as a working portfolio system and as preparation for architecture and distributed-systems interviews.

## Decision drivers

- Learn and demonstrate service ownership, asynchronous messaging, RPC, consistency, resiliency and observability.
- Preserve the useful work already represented by the repository.
- Complete a coherent business flow without pretending microservices are universally preferable.
- Be able to contrast the portfolio design with an appropriate production starting point.

## Considered options

### Convert immediately to a modular monolith

This would reduce deployment, testing, networking and consistency complexity. It would likely be the preferred starting point for a small commercial team, but it would remove much of the project's intended distributed-systems learning surface.

### Retain all existing boundaries permanently

This preserves the current layout but risks defending boundaries that have no independent ownership, scaling, security, availability, or data requirement.

### Retain boundaries provisionally and require justification

This preserves learning opportunities while allowing later consolidation. Each service and communication mechanism must gain an explicit responsibility, contract, failure model and test strategy.

## Decision

Retain the microservice architecture provisionally as an explicit learning constraint. Do not claim that it is required by current scale. Reassess individual boundaries during revival, and document consolidation or separation decisions in later ADRs.

Notification is considered a justified asynchronous boundary because provider latency and failure must not control the order transaction. Search remains a derived projection boundary. Other boundaries will be evaluated as their core flows are repaired.

## Consequences

### Positive

- The project supports practical discussion of distributed-system trade-offs.
- Existing Kafka, gRPC, discovery, gateway, and database-per-service work can be improved rather than discarded.
- Architecture choices become deliberate and reviewable.

### Negative

- Local development and CI require more infrastructure.
- Cross-service workflows require explicit consistency and failure handling.
- More security, observability, contract, and deployment work is required.

## Validation

- The reference order journey works across service boundaries from a clean local environment.
- Tests demonstrate both successful communication and dependency failures.
- Each retained service has a documented owner, source of truth, API/event contracts, and scaling rationale.
- Interview documentation can explain when a modular monolith would be preferable.

## Revisit when

- A service remains only a CRUD proxy without independent data or operational requirements.
- The cost of maintaining a boundary prevents completion of the reference journey.
- Deployment goals move from distributed-systems learning to minimizing operational cost.

