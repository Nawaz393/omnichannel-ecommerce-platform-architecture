# Diagram Sources

[← Main README](../README.md) · [← Documentation Index](../docs/README.md)

- [Cloud System Context](cloud-system-context.mmd) — Next.js 16 App Router, Core POS PostgreSQL DB via Prisma Accelerate, In-House POS REST API, and Authorize.Net architecture topology.
- [Prisma ERD](prisma-erd.mmd) — Entity-relationship diagram of all core Prisma models (Products, Variants, Stock, Carts, Orders, Quotes).
- [Payment Saga Sequence](payment-saga-sequence.mmd) — Two-phase payment authorization, In-House POS integration, and automated compensating void workflow.
- [B2B Checkout State Machine](b2b-checkout-state-machine.mmd) — Finite state machine modeling retail checkout and B2B wholesale quotation approvals.
- [Direct POS Integration Pipeline](pos-direct-integration.mmd) — Dual-tier architecture combining direct POS database queries via Prisma with direct In-House POS REST API orchestration.

GitHub renders Mermaid natively in Markdown. These source files keep architecture models modular and version-controlled.
