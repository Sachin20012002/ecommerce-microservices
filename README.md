# E-commerce Microservices

This repository is a learning-focused e-commerce backend built with Java 17, Spring Boot, and Spring Cloud. It is being modernized into a reliable and explainable distributed-systems portfolio project.

The goal is not to collect technologies or claim production readiness. The goal is to complete, test, and document a coherent customer journey while learning the design decisions and failure modes behind each service.

## Project status

The repository is under active modernization. It contains a working microservice foundation, but the complete purchase flow, security model, payment lifecycle, reliable event delivery, local infrastructure, and automated test coverage are not finished.

The modernization sequence starts with credential and security containment. After that, each microservice will be reviewed and improved individually: clarify its purpose and ownership, repair its implementation, document its contracts and failure behavior, and add meaningful unit and integration tests before moving to the next service.

See [TODO](TODO) for the active work queue and [the modernization roadmap](docs/modernization-roadmap.md) for the broader direction.

## Reference customer journey

1. Register or sign in.
2. Browse the product catalog.
3. Add products to an authenticated customer's cart.
4. Validate product data, prices, inventory, and cart ownership.
5. Create an idempotent order.
6. Process and confirm the payment outcome.
7. Publish versioned order-domain events reliably.
8. Deliver notifications asynchronously.

## Current architecture

![Current e-commerce microservices architecture](images/current-architecture.svg)

The Maven reactor currently contains ten modules: six business services, three platform services, and one shared gRPC contract module.

| Module | Current purpose | Data and integrations |
| --- | --- | --- |
| `user-service` | OTP login, JWT creation, users, customers, and addresses | MongoDB, Twilio, Kafka |
| `product-service` | Products, brands, types, sizes, tax, discounts, and images | MySQL, Kafka producer, gRPC server |
| `category-service` | Category hierarchy | MySQL, gRPC client |
| `cart-service` | Carts, cart items, and partial checkout state | MongoDB, gRPC client, Kafka |
| `order-service` | Basic order persistence and partial payment-order creation | MySQL, Razorpay |
| `filter-service` | Product filtering over a denormalized search model | Elasticsearch, Kafka consumer |
| `api-gateway` | Path-based routing to registered services | Spring Cloud Gateway, Eureka |
| `service-registry` | Service registration and discovery | Eureka |
| `cloud-config-server` | Centralized configuration retrieval | Git-backed Spring Cloud Config |
| `proto` | Shared product-query contracts and generated types | Protocol Buffers, gRPC |

Clients use JSON/HTTP through the API Gateway. Category and cart synchronously query product data over gRPC. Product creation publishes a Kafka message that updates the filter service's Elasticsearch projection. Services register with Eureka.

For implementation details and an honest assessment of current gaps, read [Current Architecture](docs/architecture/current-architecture.md). The intended direction is described in [Target Architecture](docs/architecture/target-architecture.md).

## Technology stack

- Java 17
- Spring Boot 2.7
- Spring Cloud Gateway, Config, and Netflix Eureka
- Maven multi-module build
- MySQL and MongoDB
- Elasticsearch
- Kafka
- gRPC and Protocol Buffers
- Hibernate / Spring Data

These versions describe the current repository, not necessarily the final modernization target. Framework or runtime upgrades will be evaluated separately and recorded when they materially affect the architecture.

## Known limitations

- Secrets and personal configuration require containment and rotation before the repository can be treated as safe.
- Authentication and resource-level authorization are inconsistent across services.
- Local setup is not reproducible from a clean checkout yet; the Maven Wrapper is available, but there is no containerized dependency stack.
- Checkout, inventory validation, order state transitions, payment verification, and compensation are incomplete.
- Kafka publication does not yet provide an outbox, complete event lifecycle, duplicate handling, retries, or dead-letter processing.
- Tests provide limited business assertions and may depend on live infrastructure or providers.
- Database migrations, tracing, resiliency policies, and complete operational documentation are not present.

These are modernization tasks, not hidden production-readiness claims.

## Build and local development

### Prerequisites

- JDK 17
- The service-specific MySQL, MongoDB, Kafka, Elasticsearch, Eureka, and configuration dependencies

The current reactor can be compiled with:

```powershell
.\mvnw.cmd clean install
```

On Linux or macOS, use `./mvnw clean install`. The wrapper pins Maven 3.9.16 and downloads it on first use; a global Maven installation is not required.

A deterministic clean-checkout startup procedure has not been completed. Do not expect all services to start from this command alone. Containerized dependencies, safe example configuration, health checks, startup order, and smoke-test commands are planned in the reproducible-development phase.

## API documentation

When the gateway and relevant services are running with the current local routing configuration, Swagger UI is expected at:

- Product: `http://localhost:9191/meesho/product-microservice/swagger-ui/index.html`
- Category: `http://localhost:9191/meesho/category-microservice/swagger-ui/index.html`
- Filter: `http://localhost:9191/meesho/filter-microservice/swagger-ui/index.html`
- User: `http://localhost:9191/meesho/user-microservice/swagger-ui/index.html`
- Cart: `http://localhost:9191/meesho/cart-microservice/swagger-ui/index.html`
- Order: `http://localhost:9191/meesho/order-microservice/swagger-ui/index.html`

These endpoints will be verified and updated during each service review.

## How the project will be improved

Work is intentionally incremental:

1. Contain credentials and establish a safe security baseline.
2. Make the repository reproducible from a clean checkout.
3. Select one microservice and document its purpose, data ownership, contracts, dependencies, and non-goals.
4. Repair that service's implementation and security boundaries.
5. Add meaningful unit and integration tests, including failure cases.
6. Update its operational and API documentation.
7. Complete the review before moving to the next service.
8. Integrate the completed services into the reference customer journey.

Significant choices are recorded as [Architecture Decision Records](docs/adr/README.md). The first accepted decision explains why microservices are retained as a learning constraint while acknowledging when a modular monolith would be the more practical choice.

## Documentation

- [Current architecture](docs/architecture/current-architecture.md)
- [Target architecture](docs/architecture/target-architecture.md)
- [Modernization roadmap](docs/modernization-roadmap.md)
- [Architecture Decision Records](docs/adr/README.md)
- [Repository working agreement](AGENTS.md)

## License

This project is licensed under the [MIT License](LICENSE).
