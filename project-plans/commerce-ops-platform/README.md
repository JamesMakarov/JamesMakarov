# Commerce Ops Platform

Project plan for a backend-focused portfolio project built with Java and Spring Boot.

The goal is to build a realistic commerce operations backend first and add an AI operations agent only after the core domain is reliable.

## Documents

- [PRODUCT.md](PRODUCT.md) — product scope, users, workflows and domain model.
- [ARCHITECTURE.md](ARCHITECTURE.md) — technical architecture, modules, reliability and AI design.
- [ROADMAP.md](ROADMAP.md) — implementation phases and definition of done.

## Guiding rule

The platform must remain useful if the AI layer is removed.

The AI agent is a client of the domain services through controlled tools. It does not bypass authorization, business rules or persistence boundaries.
