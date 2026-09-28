# Technical Architecture — Commerce Ops Platform

## Architecture style

Start as a **modular monolith**.

One deployable Spring Boot application, one PostgreSQL database, clear internal module boundaries.

This keeps the project understandable while still allowing good domain separation.

## Proposed package structure

```text
com.jamesmakarov.commerceops
├── auth
├── customer
├── catalog
├── inventory
├── order
├── payment
├── delivery
├── support
├── agent
└── shared
```

Each business module should generally contain its own:

```text
controller/
application/
domain/
repository/
dto/
```

Do not create interfaces and abstractions automatically. Introduce them when they isolate a real responsibility.

## Initial technology choices

- Java 21
- Spring Boot
- Spring Web
- Spring Validation
- Spring Data JPA
- Spring Security
- PostgreSQL
- Flyway
- Spring Boot Actuator
- OpenAPI documentation
- JUnit 5
- Mockito where useful
- Testcontainers
- Docker Compose
- GitHub Actions

Later:

- Spring AI
- Ollama + Llama
- pgvector
- Prometheus/Grafana
- messaging only when justified

## Database strategy

Use Flyway from the first migration.

Example:

```text
src/main/resources/db/migration/
├── V1__create_customers.sql
├── V2__create_catalog_and_inventory.sql
├── V3__create_orders.sql
└── ...
```

Avoid relying only on Hibernate automatic schema generation.

Development:

```text
spring.jpa.hibernate.ddl-auto=validate
```

The migrations are the source of truth.

## Transactions

Important operations must be transactional.

Examples:

### Create order

One transaction should:

1. validate products;
2. reserve stock;
3. create order;
4. create order items.

If any operation fails, none of the changes should remain.

### Cancel order

One transaction should:

1. validate allowed state transition;
2. release inventory when applicable;
3. create refund state when necessary;
4. update order status.

## Inventory concurrency

Inventory is the first deliberate concurrency problem in the project.

Use optimistic locking:

```java
@Version
private Long version;
```

Two concurrent order attempts must not be able to reserve the same final unit.

Create integration tests that actually execute concurrent reservations.

## Idempotency

Payment processing should accept an idempotency key.

Example header:

```text
Idempotency-Key: d9b57...
```

The same key for the same payment operation should return the original result instead of creating a duplicate payment.

This should be enforced with a unique database constraint, not only application memory.

## Error handling

Use a global `@RestControllerAdvice`.

Return a consistent problem format such as:

```json
{
  "code": "INSUFFICIENT_STOCK",
  "message": "There is not enough stock for SKU ABC-123",
  "timestamp": "...",
  "path": "/api/orders"
}
```

Never return raw stack traces to clients.

## Observability

Actuator endpoints should include at least:

- health;
- info;
- metrics.

Add structured logs for important lifecycle events:

- order created;
- payment approved/declined;
- inventory reservation failed;
- order cancelled;
- delivery failure;
- agent tool execution.

Do not log secrets, passwords, access tokens or complete payment data.

## Tests

### Unit tests

Use for isolated domain rules.

Examples:

- valid/invalid order status transitions;
- cancellation rules;
- price calculation;
- score/amount calculations;
- permission decisions where isolated.

### Integration tests

Use Spring Boot + Testcontainers for:

- repository behavior;
- migrations;
- unique constraints;
- transactions;
- order creation;
- inventory reservation;
- payment idempotency.

Prefer real PostgreSQL in integration tests instead of replacing database behavior with H2.

### API tests

Use MockMvc or WebTestClient for:

- request validation;
- authorization;
- status codes;
- serialized responses.

## CI

Every push and pull request should eventually run:

```text
compile
unit tests
integration tests
format/static checks
container build
```

Do not merge intentionally failing CI into main.

## AI architecture

The agent belongs in the `agent` module but depends on application services, not repositories.

```text
User
  |
AgentController
  |
AgentService
  |
Spring AI / Ollama
  |
Tool functions
  |
Application services
  |
Domain rules
  |
Repositories
```

This is a critical boundary.

Bad:

```text
LLM -> repository -> database
```

Good:

```text
LLM -> controlled tool -> application service -> domain rule -> repository
```

## Human approval

Read-only tools execute directly.

Write tools should return a proposed operation first for important actions.

Example:

```text
User: Cancel order 184.

Agent:
Order 184 is PAID but not shipped.
Cancellation would create a refund of R$ 129.90 and release 2 inventory units.
Confirm cancellation?
```

Only after confirmation should the application execute the command.

## Agent security

The model must never determine authorization.

The authenticated user context is enforced by Spring Security and tool services.

If a SUPPORT user is not allowed to perform a refund, the tool must reject it regardless of what the LLM asks.

## Internal events

Use Spring application events first for decoupled reactions where useful.

Possible events:

- `OrderCreated`
- `PaymentApproved`
- `OrderCancelled`
- `OrderShipped`
- `DeliveryFailed`

If the project later has a genuine need for independent consumers, delivery guarantees or asynchronous scaling, then evaluate Kafka/RabbitMQ.

Do not introduce distributed infrastructure before that need exists.
