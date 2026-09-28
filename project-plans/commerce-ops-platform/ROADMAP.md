# Implementation Roadmap — Commerce Ops Platform

## Development principle

Build the project vertically.

A milestone should leave the application executable and tested.

Do not create every entity first and business behavior months later.

---

# Milestone 0 — Bootstrap

## Goal

A production-shaped Spring Boot skeleton that starts locally, talks to PostgreSQL and is validated by CI.

## Tasks

- [ ] Create repository `commerce-ops-platform`
- [ ] Generate Spring Boot project with Java 21
- [ ] Add Spring Web
- [ ] Add Validation
- [ ] Add Spring Data JPA
- [ ] Add PostgreSQL driver
- [ ] Add Flyway
- [ ] Add Actuator
- [ ] Add test dependencies
- [ ] Add Testcontainers PostgreSQL
- [ ] Create Docker Compose with PostgreSQL
- [ ] Configure application profiles
- [ ] Create first Flyway migration
- [ ] Expose health endpoint through Actuator
- [ ] Add GitHub Actions
- [ ] Add initial README
- [ ] Add `ARCHITECTURE.md`

## Definition of done

```text
docker compose up -d
./mvnw test
./mvnw spring-boot:run
```

all work, and CI is green.

---

# Milestone 1 — Customers and catalog

## Goal

Create the first useful REST resources.

## Tasks

- [ ] Customer entity + migration
- [ ] Product entity + migration
- [ ] Unique customer email
- [ ] Unique product SKU
- [ ] Use BigDecimal for money
- [ ] Create customer endpoint
- [ ] Fetch customer endpoint
- [ ] Create product endpoint
- [ ] List products with pagination
- [ ] Fetch product endpoint
- [ ] Update/activate/deactivate product
- [ ] Request validation
- [ ] Global exception handler
- [ ] Integration tests with PostgreSQL
- [ ] OpenAPI documentation

## Definition of done

Invalid requests return controlled errors and all persistence tests use Testcontainers.

---

# Milestone 2 — Inventory and concurrency

## Goal

Model stock correctly and prove that concurrent operations cannot oversell.

## Tasks

- [ ] Inventory table
- [ ] Stock receipt operation
- [ ] Available/reserved quantities
- [ ] Optimistic locking with `@Version`
- [ ] Reserve stock use case
- [ ] Release stock use case
- [ ] Insufficient stock exception
- [ ] Transaction boundaries
- [ ] Concurrent reservation integration test

## Critical test

Given one available unit, two concurrent reservations must not both succeed.

---

# Milestone 3 — Orders

## Goal

Create transactional orders that reserve inventory.

## Tasks

- [ ] Order table
- [ ] Order item table
- [ ] Order status enum
- [ ] Snapshot SKU/name/price in order item
- [ ] Create order command
- [ ] Calculate totals on server
- [ ] Reserve inventory atomically
- [ ] Order detail endpoint
- [ ] Paginated order search
- [ ] Filter by status/customer
- [ ] Order lifecycle tests
- [ ] Transaction rollback tests

## Definition of done

A failed order never leaves partial reservations or partial rows behind.

---

# Milestone 4 — Payment and cancellation

## Goal

Introduce state transitions, idempotency and financial consistency.

## Tasks

- [ ] Payment entity
- [ ] Payment status
- [ ] Simulated approval/decline
- [ ] Idempotency key
- [ ] Unique DB constraint for idempotency
- [ ] Approved payment transitions order to `PAID`
- [ ] Cancellation service
- [ ] Release stock on allowed cancellation
- [ ] Record refund for paid cancellation
- [ ] Reject invalid transitions
- [ ] Tests for repeated payment requests
- [ ] Tests for cancellation rules

---

# Milestone 5 — Delivery and support

## Goal

Complete the operational story of an order.

## Tasks

- [ ] Delivery entity
- [ ] Delivery event timeline
- [ ] Tracking code
- [ ] Ship order
- [ ] Delivery status transitions
- [ ] Support ticket entity
- [ ] Create/update support tickets
- [ ] Link ticket to customer/order
- [ ] Delivery failure workflow
- [ ] Operational query endpoint that returns order + payment + delivery summary

That final summary endpoint will later become useful to the AI agent.

---

# Milestone 6 — Security and operational quality

## Goal

Turn the backend into a service that looks deployable.

## Tasks

- [ ] Spring Security
- [ ] Internal user model
- [ ] Roles: ADMIN, OPS, SUPPORT
- [ ] JWT authentication
- [ ] Endpoint authorization
- [ ] Method-level authorization where useful
- [ ] Actuator configuration
- [ ] Structured logging
- [ ] Correlation/request ID
- [ ] Metrics
- [ ] Docker application image
- [ ] Container health check
- [ ] CI container build
- [ ] Architecture documentation update

---

# Milestone 7 — AI agent, read-only

## Goal

Add Ollama/Llama without allowing the model to mutate business state.

## Tasks

- [ ] Add Spring AI
- [ ] Add Ollama configuration
- [ ] Create `agent` module
- [ ] Agent chat endpoint
- [ ] Tool: `getOrder`
- [ ] Tool: `getCustomer`
- [ ] Tool: `getInventory`
- [ ] Tool: `getPaymentStatus`
- [ ] Tool: `getDeliveryTimeline`
- [ ] Tool: `listSupportTickets`
- [ ] Require tools for factual operational answers
- [ ] Agent execution audit
- [ ] Tool-call audit
- [ ] Tests with model boundary mocked
- [ ] Document limitations

## Example acceptance scenario

Question:

```text
Why is order <id> delayed?
```

Expected behavior:

1. agent retrieves the order;
2. retrieves delivery timeline;
3. optionally checks support tickets;
4. answers only from retrieved platform data;
5. records which tools were used.

---

# Milestone 8 — Controlled agent actions

## Goal

Allow useful actions without giving uncontrolled authority to the model.

## Tasks

- [ ] Tool: create support ticket
- [ ] Tool: add internal note
- [ ] Tool: request cancellation
- [ ] Human confirmation workflow
- [ ] Authorization enforced inside tool service
- [ ] Idempotency for agent-initiated commands
- [ ] Audit success/failure
- [ ] Test unauthorized tool invocation
- [ ] Test confirmation requirement

Direct database mutation by the LLM is forbidden.

---

# Milestone 9 — Policies and RAG

## Goal

Allow the assistant to answer questions that require company policy, not only live database state.

## Tasks

- [ ] Introduce pgvector
- [ ] Policy document ingestion
- [ ] Chunking strategy
- [ ] Embeddings
- [ ] Similarity search
- [ ] Cancellation/refund policy corpus
- [ ] Retrieval citations in agent answers
- [ ] Tests for retrieval boundaries

Only add this after tool calling is stable.

---

# Optional Milestone 10 — Messaging

Only implement if a concrete asynchronous workflow makes it useful.

Candidate:

```text
PaymentApproved -> fulfillment process
DeliveryFailed -> support automation
```

Start with Spring application events.

Move to Kafka/RabbitMQ only when you want to demonstrate broker semantics such as retries, independent consumers and delivery guarantees.

---

# First coding session

Do only this:

1. Create the repository.
2. Generate the Spring Boot skeleton.
3. Start PostgreSQL with Docker Compose.
4. Configure Flyway.
5. Create a tiny `V1__baseline.sql`.
6. Make the application boot.
7. Make one Testcontainers integration test connect to PostgreSQL.
8. Add CI.

Do **not** create Product, Order or AI code in the first session.

The first objective is proving that the engineering foundation is reliable.
