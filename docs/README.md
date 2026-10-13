# Documentation Index

[← Main README](../README.md)

This documentation provides an architectural breakdown, domain boundaries, production code examples, and technical deep-dives for the **Engel E-Commerce Platform**.

---

## Architectural Guides & Deep Dives

- [System Architecture](architecture.md) — Next.js 16 App Router server/client runtime boundaries, Server Actions, and edge routing.
- [Domain Map & System Decomposition](domain-map.md) — Catalog, Cart, Order, Payment Saga, B2B Quotations, and Serverless ETL bounded contexts.
- [Representative Code Examples](code-examples.md) — Production TypeScript, Prisma ORM, Authorize.Net Accept.js saga, and Python Lambda implementations.
- [Two-Phase Payment Saga & Compensating Voids](payment-saga-and-fault-tolerance.md) — Authorize.Net two-phase commit, network timeout recovery, and compensating voids.
- [Prisma Database Architecture](prisma-database-architecture.md) — Prisma ORM 6.13, connection pooling with Accelerate, schema models, and AWS RDS PostgreSQL optimization.
- [Serverless Infrastructure & AWS Lambdas](serverless-infrastructure-and-lambdas.md) — AWS Lambda microservices (`engels_lambda`), EventBridge triggers, and catalog ETL pipelines.

---

## Architectural Decision Records (ADRs)

- [ADR Index](decisions/README.md) — Log of design choices and trade-offs.
- [ADR 001: Next.js 16 App Router + React 19](decisions/001-nextjs-app-router-react-19.md) — Server Components vs. traditional Client SPA.
- [ADR 002: Prisma Accelerate Connection Pooling](decisions/002-prisma-accelerate-connection-pooling.md) — Overcoming serverless PostgreSQL connection exhaustion.
- [ADR 003: Two-Phase Payment Saga with Compensating Void](decisions/003-two-phase-payment-saga.md) — Guaranteeing zero phantom card authorizations across ERP sync boundaries.

---

## Visual Diagrams

- [Diagram Sources](../diagrams/README.md) — Reusable Mermaid diagrams for cloud context, Prisma ERD, payment saga sequence, B2B state machine, and serverless pipelines.
