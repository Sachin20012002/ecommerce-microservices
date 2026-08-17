# Current Architecture

## Purpose

This document records the architecture found in the repository before modernization. It describes the implementation rather than claiming production readiness.

> **Historical baseline:** this document is not the architecture direction. The
> production source of truth is [Target Architecture](target-architecture.md),
> supported by the accepted ADRs under `docs/adr/`.

## System shape

The project is a Maven reactor containing ten Spring modules. Six are business services and four provide platform or shared-contract capabilities.

```text
Client
  |
  v
API Gateway (Spring Cloud Gateway)
  |
  +-- User Service -------- MongoDB, Twilio, Kafka
  +-- Product Service ----- MySQL, Kafka producer, gRPC server
  +-- Category Service ---- MySQL, gRPC client
  +-- Cart Service -------- MongoDB, gRPC client, Kafka
  +-- Order Service ------- MySQL, Razorpay
  +-- Filter Service ------ Elasticsearch, Kafka consumer

Services ---------- Eureka service registry
Configuration ----- Spring Cloud Config backed by a Git repository
Contracts --------- Shared Protocol Buffers module
```

## Module responsibilities

| Module | Current responsibility | Persistence/integration |
| --- | --- | --- |
| `user-service` | OTP login, JWT creation, users, customers and addresses | MongoDB, Twilio, Kafka |
| `product-service` | Products, brands, types, sizes, tax, discounts and images | MySQL, Kafka, gRPC |
| `category-service` | Category, subcategory and child-category hierarchy | MySQL, gRPC |
| `cart-service` | Carts, cart items and partial checkout state | MongoDB, gRPC, Kafka |
| `order-service` | Basic order persistence and partial Razorpay order creation | MySQL, Razorpay |
| `filter-service` | Product filtering using a denormalized search model | Elasticsearch, Kafka |
| `api-gateway` | Path-based routing and Eureka-backed load balancing | Spring Cloud Gateway |
| `service-registry` | Runtime service registration and discovery | Eureka |
| `cloud-config-server` | Centralized configuration retrieval | Git-backed Config Server |
| `proto` | Generated product-query gRPC types | Protocol Buffers |

## Communication

- Clients call JSON/HTTP endpoints through the gateway.
- Category and cart synchronously query product data using gRPC.
- Product creation publishes a Kafka message consumed by the filter service.
- Services register with Eureka and the gateway routes through logical service names.
- JWTs are signed and verified using one shared symmetric key.

## Data and consistency model

The project follows database-per-service in structure: product, category, and order use separate MySQL schemas; user and cart use separate MongoDB databases; filter maintains an Elasticsearch projection.

Product-to-search synchronization is eventually consistent. The implementation currently publishes creation events but does not provide a complete update/delete lifecycle, an outbox, idempotency, or dead-letter handling.

Checkout and order processing do not yet form a complete consistency boundary. Inventory reservation, authoritative price validation, payment verification, idempotency, and failure compensation remain undefined.

## Known architectural gaps

- Runtime secrets and local credentials are committed in configuration and source code.
- Cart, order, filter, gateway, registry, and config endpoints lack consistent security controls.
- The gateway routes requests but does not currently enforce authentication as documented in the README.
- Checked-in gRPC server/client ports are inconsistent, and cart lacks an explicit product client address.
- Checkout summary is unfinished, and order placement is primarily direct persistence of client input.
- Payment capture, signature verification, webhook processing, and state transitions are incomplete.
- Tests depend on live infrastructure or providers and contain few business assertions.
- There is no containerized local environment. A Maven Wrapper is present, but a
  deterministic clean-checkout runtime has not been completed.
- There are no database migrations, distributed tracing, resiliency policies, or reliable event-publication mechanism.

## Superseded direction

Earlier governance treated the microservice shape primarily as a learning
constraint and deferred Kubernetes. That direction is superseded. The accepted
production target consolidates Product and Category into Catalog, separates
Inventory and Payment, makes Order the checkout process manager, and deploys on
Amazon EKS using AWS-managed data and messaging services.

