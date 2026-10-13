# Prisma Database Architecture & PostgreSQL Optimization

[← Main README](../README.md) · [← Documentation Index](README.md)

This document details the database architecture of the **Enterprise Omnichannel E-Commerce Platform**, covering **Prisma ORM 6.13**, Core POS PostgreSQL persistence, connection pooling with `@prisma/extension-accelerate`, and multi-tier pricing relations.

---

## 1. The Serverless Connection Exhaustion Problem

Next.js deployed to serverless infrastructure scales horizontally across ephemeral function instances. Under traffic spikes, hundreds of concurrent function executions spin up. If each function instance establishes direct connections to the PostgreSQL database, the connection limit (`max_connections`) is rapidly exhausted, producing fatal `FATAL: remaining connection slots are reserved for non-replication superuser connections` errors.

To solve this, the platform routes all database traffic through **Prisma Accelerate**:

```typescript
import { PrismaClient } from '@prisma/client'
import { withAccelerate } from '@prisma/extension-accelerate'

const globalForPrisma = globalThis as unknown as { prisma: any }

export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({
    datasourceUrl: process.env.DATABASE_URL, // Prisma Accelerate connection string
    log: process.env.NODE_ENV === 'development' ? ['query', 'error', 'warn'] : ['error'],
  }).$extends(withAccelerate())

if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = prisma
```

---

## 2. Core Relational Schema Highlights

The database schema models high-concurrency retail products, B2B price tiers, warehouse stock, and cart sessions:

### 2.1 Variant Hierarchy & Tax
```prisma
model products {
  id               String            @id @default(uuid())
  name             String
  description      String?
  vat_id           String
  vat              vat               @relation(fields: [vat_id], references: [id])
  variants         productvariants[]
  group_id         String
  group            productgroup      @relation(fields: [group_id], references: [id])
  created_at       DateTime          @default(now())
}

model productvariants {
  id               String            @id @default(uuid())
  product_id       String
  product          products          @relation(fields: [product_id], references: [id], onDelete: Cascade)
  sku              String            @unique
  retail_price     Decimal           @db.Decimal(12, 2)
  wholesale_price  Decimal?          @db.Decimal(12, 2)
  stock            stock[]
  order_items      orderitems[]
}
```

### 2.2 Warehouse Inventory Tracking
```prisma
model stock {
  id               String            @id @default(uuid())
  variant_id       String
  variant          productvariants   @relation(fields: [variant_id], references: [id])
  warehouse_id     String
  warehouse        warehouse         @relation(fields: [warehouse_id], references: [id])
  quantity         Int               @default(0)

  @@unique([variant_id, warehouse_id])
}
```

---

## 3. Query Optimization & Caching Strategy

Using Prisma Accelerate enables declarative query-level caching directly from Server Actions:

```typescript
// Cache top-level catalog groups for 5 minutes at the edge
export async function getCachedCatalogCategories() {
  return await prisma.productgroup.findMany({
    where: { parent_id: null },
    include: { children: true },
    cacheStrategy: {
      ttl: 300,        // Cache edge TTL in seconds
      swr: 60,         // Stale-While-Revalidate window
    },
  })
}
```

Transactional mutations (e.g., checkout and order creation) bypass edge caches and execute with direct transactional guarantees against the primary POS database.

Related: [Payment Saga](payment-saga-and-fault-tolerance.md) · [POS Direct Integration](pos-api-and-direct-database-integration.md) · [Code Examples](code-examples.md)
