# ADR 002: Prisma ORM 6 & Accelerate Connection Pooling on AWS RDS

[← ADR Index](README.md) · [← Main README](../../README.md) · [← Documentation Index](../README.md)

- **Status:** Accepted
- **Date:** 2025-11-20
- **Deciders:** Lead Full-Stack Engineer (M Nawaz Khan)

---

## Context

The omnichannel e-commerce backend runs on serverless infrastructure, connecting to an enterprise PostgreSQL database instance. 

In a traditional stateful application, a single Node.js process maintains a long-lived database connection pool (e.g., 10 to 20 connections). However, serverless functions scale horizontally by instantiating independent runtime containers per concurrent request. If 250 shoppers simultaneously hit checkout routes, search catalog pages, and add items to carts, 250 distinct serverless instances attempt to establish direct TCP connections to PostgreSQL.

PostgreSQL enforces a hard ceiling on concurrent connections (`max_connections = 150` on standard RDS instance classes). Exceeding this limit causes immediate database failure (`FATAL: sorry, too many clients already`), halting checkout flows and cascading into 500 Internal Server Errors.

## Decision

We adopted **Prisma ORM 6.13.0** augmented with the **`@prisma/extension-accelerate` (v2.0.2)** client extension:

```typescript
import { PrismaClient } from '@prisma/client';
import { withAccelerate } from '@prisma/extension-accelerate';

const prisma = globalForPrisma.prisma || new PrismaClient().$extends(withAccelerate());
```

1. **Serverless Connection Multiplexing**: All database queries from Vercel edge/serverless functions route through Prisma Accelerate's globally distributed pooling proxy. Accelerate holds a steady, managed pool of long-lived connections to AWS RDS, multiplexing hundreds of incoming ephemeral client queries without opening new database sockets.
2. **Edge Read Caching**: Static catalog queries (e.g., category listings, brand hierarchies, general settings) leverage Accelerate's global caching layer, reducing read load on AWS RDS and returning catalog data in single-digit milliseconds.
3. **Transaction Safety**: Accelerate natively supports interactive transactions (`prisma.$transaction(async (tx) => { ... })`), ensuring ACID guarantees for atomic cart merges and inventory reservations.

## Consequences

### Positive
- **Immunity from Connection Exhaustion**: Peak shopping spikes on Vercel do not overwhelm AWS RDS connection limits.
- **Low Query Latency**: Pre-warmed TCP pools eliminate the 50–100ms connection handshake overhead on every serverless cold start.
- **Unified TypeScript ORM Model**: Prisma Client generates type-safe schemas directly from `schema.prisma`, maintaining type safety across 40+ relational tables.
- **Zero Infrastructure Maintenance**: Avoids provisioning, patching, and maintaining self-hosted PgBouncer instances.

### Negative / Operational Challenges
- **Managed Service Dependency**: Relies on Prisma Accelerate infrastructure availability.
- **Transaction Timeouts**: Interactive transactions over Accelerate enforce a default maximum execution timeout (15 seconds), requiring fast, non-blocking queries inside transactions.
- **Network Overhead on Edge Misses**: Uncached queries incur an additional proxy hop through the Accelerate routing gateway before hitting RDS Stockholm.

## Trade-offs

- Direct database connections (`DATABASE_URL=postgresql://...`) are still required during Prisma CLI migrations (`prisma migrate deploy` or `prisma db push`) because DDL schema alters cannot execute through Accelerate's query proxy. We maintain dual environment variable definitions: `DIRECT_URL` for migrations and `DATABASE_URL` (Accelerate protocol `prisma://...`) for runtime.

## Alternatives Considered

1. **AWS RDS Proxy**:
   - *Rejected*: Requires Vercel functions to route through AWS VPC peering or dedicated NAT Gateways, adding significant configuration complexity, latency, and AWS VPC egress costs without edge caching capabilities.
2. **Self-Hosted PgBouncer on EC2 / ECS**:
   - *Rejected*: High operational overhead, patching burden, and single point of failure unless paired with high-availability load balancers.
3. **Unpooled Direct Database Connections**:
   - *Rejected*: Proved catastrophic during load testing, exhausting `max_connections` within seconds under simulated concurrency of 120 users.
