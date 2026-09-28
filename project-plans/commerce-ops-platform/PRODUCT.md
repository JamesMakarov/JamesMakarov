# Product Specification — Commerce Ops Platform

## Product idea

Commerce Ops Platform is an internal operations backend for an e-commerce company.

It centralizes the lifecycle of:

- customers;
- products;
- inventory;
- orders;
- payments;
- deliveries;
- support tickets.

The first version is intentionally backend-first. A frontend is optional and should not delay the API, tests or domain model.

The final platform will include an AI operations assistant powered locally through Ollama/Llama. The assistant will answer operational questions using real data from the platform and, later, request controlled actions through application tools.

## Primary users

### Operations analyst

Needs to understand what happened to an order, inspect stock, delivery and payment state, and coordinate exceptions.

### Support analyst

Needs a consolidated view of the customer and order and must be able to open and follow support tickets.

### Administrator

Manages internal users and has broader operational permissions.

The first iteration does not require a customer-facing application.

## Core business scenarios

### 1. Product and inventory

An operator can:

- create a product;
- update its commercial data;
- activate/deactivate it;
- add stock;
- inspect available and reserved stock.

Each product has a unique SKU.

Inventory must distinguish:

- available quantity;
- reserved quantity.

Order creation must never allow available stock to become negative.

## 2. Order creation

An order contains one or more products.

When an order is created:

1. customer existence is validated;
2. products are validated;
3. current prices are copied into the order items;
4. inventory is reserved;
5. the total is calculated by the backend;
6. the order enters `PENDING_PAYMENT`.

The client must never be trusted to send the final order total.

## 3. Payment

For the portfolio version, payment is simulated internally.

Possible statuses:

- `PENDING`
- `APPROVED`
- `DECLINED`
- `REFUNDED`

An approved payment moves the order to `PAID`.

Payment operations must support an idempotency key so the same request cannot accidentally charge/process the same operation twice.

## 4. Fulfillment and delivery

After payment approval, an order can progress through:

- `PAID`
- `PREPARING`
- `SHIPPED`
- `DELIVERED`

Delivery maintains a timeline of events such as:

- package created;
- carrier pickup;
- in transit;
- delivery attempt;
- delivered;
- delivery failure.

## 5. Cancellation

Cancellation rules must live in the domain service rather than controllers.

Initial rules:

- `PENDING_PAYMENT`: can be cancelled and reserved stock is released;
- `PAID`: can be cancelled before shipment; refund is recorded and stock is released;
- `SHIPPED`: direct cancellation is rejected;
- `DELIVERED`: direct cancellation is rejected.

Rules can evolve later through documented policies.

## 6. Support

A support ticket can be linked to:

- customer;
- order;
- delivery.

Example categories:

- payment;
- cancellation;
- delivery;
- damaged item;
- inventory;
- other.

Statuses:

- `OPEN`
- `IN_PROGRESS`
- `RESOLVED`
- `CLOSED`

## Domain entities

### Customer

Suggested fields:

- `id: UUID`
- `name`
- `email`
- `status`
- `createdAt`
- `updatedAt`

### Product

- `id: UUID`
- `sku`
- `name`
- `description`
- `price`
- `active`
- timestamps

Money must use `BigDecimal`, never `double`.

### Inventory

- `productId`
- `availableQuantity`
- `reservedQuantity`
- `version`

Use optimistic locking to protect concurrent stock modifications.

### Order

- `id: UUID`
- `customerId`
- `status`
- `total`
- timestamps

### OrderItem

- `orderId`
- `productId`
- `skuSnapshot`
- `productNameSnapshot`
- `unitPrice`
- `quantity`
- `subtotal`

Snapshots preserve the commercial data used when the order was created.

### Payment

- `id: UUID`
- `orderId`
- `amount`
- `method`
- `status`
- `idempotencyKey`
- timestamps

### Delivery

- `id: UUID`
- `orderId`
- `trackingCode`
- `status`
- timestamps

### DeliveryEvent

- `id: UUID`
- `deliveryId`
- `type`
- `description`
- `occurredAt`

### SupportTicket

- `id: UUID`
- `customerId`
- `orderId`
- `category`
- `status`
- `description`
- timestamps

## Initial REST API

The exact routes can evolve, but the first contract should resemble:

```text
POST   /api/customers
GET    /api/customers/{id}

POST   /api/products
GET    /api/products
GET    /api/products/{id}
PATCH  /api/products/{id}

POST   /api/inventory/{productId}/receipts
GET    /api/inventory/{productId}

POST   /api/orders
GET    /api/orders/{id}
GET    /api/orders
POST   /api/orders/{id}/cancel

POST   /api/orders/{id}/payments
GET    /api/orders/{id}/payments

POST   /api/orders/{id}/ship
POST   /api/deliveries/{id}/events
GET    /api/orders/{id}/delivery

POST   /api/support/tickets
GET    /api/support/tickets/{id}
PATCH  /api/support/tickets/{id}
```

## Authentication and authorization

Use Spring Security.

Initial roles:

- `ADMIN`
- `OPS`
- `SUPPORT`

JWT-based authentication can be added once the domain endpoints exist.

Authorization must be enforced in the backend rather than hidden only in UI behavior.

## AI operations assistant

The assistant is introduced only after the platform has reliable domain services and tests.

### Read-only tools first

- `getOrder(orderId)`
- `getCustomer(customerId)`
- `getInventory(productId)`
- `getPaymentStatus(orderId)`
- `getDeliveryTimeline(orderId)`
- `listSupportTickets(orderId)`

Example:

> Why is order 184 delayed?

The model must inspect platform data through tools before responding.

### Controlled write tools later

- `createSupportTicket(...)`
- `requestOrderCancellation(...)`
- `addInternalNote(...)`

The model does not write directly to the database.

All write operations pass through the same application services and authorization rules used by the REST API.

Destructive or financially relevant actions must require explicit human confirmation.

## AI audit

Store enough information to explain what the agent did.

### AgentExecution

- `id`
- `userId`
- `model`
- `request`
- `status`
- `startedAt`
- `finishedAt`

### AgentToolCall

- `id`
- `executionId`
- `toolName`
- `arguments`
- `resultSummary`
- `durationMs`
- `status`

Do not store secrets or unnecessary sensitive content in logs.

## RAG — later milestone

Operational policies can eventually be indexed for retrieval:

- cancellation policy;
- refund policy;
- delivery policy;
- support playbooks.

The agent can then combine:

1. live order/customer data;
2. retrieved policy passages;
3. model reasoning.

PostgreSQL + pgvector is a reasonable evolution so the project can keep one main database initially.

## What this project should demonstrate

By completion, the repository should provide evidence of:

- Java and Spring Boot;
- REST API design;
- relational modeling;
- PostgreSQL;
- migrations;
- transactions;
- concurrency control;
- idempotency;
- authentication and authorization;
- automated unit tests;
- integration tests;
- Testcontainers;
- Docker;
- CI;
- health checks;
- logs/metrics;
- architecture documentation;
- controlled LLM tool calling;
- auditability of AI actions.

## What not to add initially

Do not begin with:

- microservices;
- Kubernetes;
- Kafka;
- Redis;
- vector search;
- Llama;
- frontend.

Those are optional evolutions. Add them only after a concrete requirement makes them useful.
