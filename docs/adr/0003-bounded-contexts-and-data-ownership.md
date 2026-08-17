# ADR-0003: Define target bounded contexts and data ownership

- Status: Accepted
- Date: 2026-08-17
- Owners: Project maintainers

## Context

Legacy boundaries follow repository modules rather than business ownership:
Product and Category are coupled by RPC; Product owns quantity, prices and media;
Cart owns checkout/payment state; Payment is duplicated; and User owns both
authentication and customer profile data.

## Decision

- Consolidate Product and Category into Catalog.
- Treat Pricing as an independent bounded context, initially a strongly isolated
  module within Catalog and extractable behind stable interfaces.
- Create autonomous Inventory and Payment services.
- Limit Cart to customer cart intent. Order owns the checkout Saga/process manager
  but not Inventory or Payment state.
- Separate Customer Profile from authentication and move authentication toward
  Cognito/OIDC.
- Keep Search and Notification as asynchronous projection/delivery boundaries.

Target sources of truth are:

- Aurora PostgreSQL: Customer Profile, Catalog/Pricing initially, Inventory,
  Order and Payment, with production isolation by transactional boundary;
- DynamoDB: Cart;
- OpenSearch: rebuildable Search projection only;
- S3/CloudFront: product media objects and delivery.

Catalog remains on Aurora PostgreSQL until documented access patterns justify a
different store. Redis is added only for a measured cache or throttling need and
never owns transactional state.

## Alternatives considered

- **Retain Product and Category services:** preserves code layout but retains a
  synchronous dependency with no independent ownership or scale rationale.
- **Place Catalog and Cart in DynamoDB by default:** optimizes for an assumed scale
  before keys, queries, constraints and item growth are demonstrated.
- **Keep inventory in Catalog and payment in Order/Cart:** avoids new deployments
  but makes independent correctness, isolation, authorization and recovery
  impossible.
- **Extract Pricing immediately:** creates deployment overhead before its contract
  and business behavior are mature; isolation inside Catalog preserves the later
  extraction path.

## Consequences

- Existing modules and schemas are migration inputs, not target services.
- Cross-boundary references use stable IDs and snapshots, never database joins.
- Order snapshots price and item facts; Search cannot authorize price or stock.
- Production database isolation costs more than a shared cluster but reduces blast
  radius and enables independent backup, migration and access control.

## Validation

Each boundary must publish its owned data, non-goals, API/events, authorization,
migrations, recovery behavior and independent tests before cutover.
