# Enterprise Omnichannel Retail E-Commerce Platform — Engineering Case Study

A documentation-first architectural case study of an enterprise full-stack retail and B2B wholesale commerce platform bridging modern web storefront interactions with brick-and-mortar back-office ERP systems via direct in-house POS REST APIs and direct POS PostgreSQL persistence.

![Next.js 16](https://img.shields.io/badge/Next.js-16_App_Router-000000?logo=next.js&logoColor=white)
![React 19](https://img.shields.io/badge/React-19_RSC-61DAFB?logo=react&logoColor=black)
![TypeScript](https://img.shields.io/badge/TypeScript-5.x-3178C6?logo=typescript&logoColor=white)
![Prisma ORM](https://img.shields.io/badge/Prisma-6.13_%7C_Accelerate-2D3748?logo=prisma&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-Core_POS_DB-336791?logo=postgresql&logoColor=white)
![POS Integration](https://img.shields.io/badge/POS_Integration-Direct_Dual--Tier-0052CC)
![Authorize.Net](https://img.shields.io/badge/Payments-Authorize.Net_Accept.js-004B87)

---

## Executive Overview

The **Enterprise Omnichannel Retail Platform** provides a hybrid digital commerce solution supporting direct-to-consumer retail shopping and B2B wholesale customer workflows. The architecture pairs Next.js 16 App Router and React 19 Server Components with a **Dual-Tier Direct In-House POS Integration** pattern, eliminating unnecessary middle-tier queues or serverless ETL functions.

### Architectural Highlights
1. **Next.js 16 App Router & React 19**: Server Components render product catalog pages with zero client bundle overhead; Server Actions orchestrate atomic cart mutations and checkout flows.
2. **Prisma ORM 6.13 with Accelerate**: Overcomes serverless PostgreSQL connection exhaustion through global edge connection pooling and edge-level SWR caching against the Core POS database.
3. **Dual-Tier Direct POS Integration**: Eliminates asynchronous ETL queues and serverless middle-tiers. Read workloads execute directly against the POS database via Prisma; transactional mutations (order voucher creation, payment ledger settlement, cancellations) execute synchronously via the in-house POS REST API.
4. **Two-Phase Distributed Payment Saga**: Card tokens generated via Authorize.Net Accept.js undergo authorization holds (`AUTH_ONLY`), downstream POS voucher generation, prior auth capture, and POS ledger posting, protected by automated compensating voids (`voidTransactionRequest`) and cancellations on failure.

---

## Cloud Architecture Topology

```mermaid
flowchart TB
    subgraph ClientLayer ["1. Client Runtime (Browser)"]
        Browser["React 19 UI (Tailwind CSS v4)"]
        Zustand["Zustand Store (cart-v2 in LocalStorage)"]
        AcceptJS["Authorize.Net Accept.js (Card Nonce Generator)"]
    end

    subgraph AppRuntime ["2. Next.js Application Runtime (Vercel)"]
        RSC["React Server Components (Catalog & Layouts)"]
        Actions["Server Actions ('use server')"]
        API["API Route: /api/checkout (Saga Orchestrator)"]
        AxiosClient["POS REST Client (src/lib/axiosInstance.ts)"]
    end

    subgraph DataStorage ["3. Database & Media Infrastructure"]
        Accelerate["Prisma Accelerate Connection Pooler"]
        CoreDB[("Core POS PostgreSQL Database")]
        MediaCDN[("Enterprise Catalog Media Archive (Amazon S3)")]
    end

    subgraph ExternalSystems ["4. In-House POS & Payment Rails"]
        InHousePOS["In-House POS REST API (/Voucher/order, /Customer)"]
        AuthNet["Authorize.Net Payment Processing Gateway"]
    end

    Browser --> AcceptJS
    AcceptJS --> Browser
    Browser --> RSC
    Browser --> Actions
    Browser --> API
    Browser --> Zustand

    RSC --> Accelerate
    Actions --> Accelerate
    API --> Accelerate
    Accelerate --> CoreDB

    API --> AxiosClient
    AxiosClient --> InHousePOS
    API --> AuthNet
    RSC --> MediaCDN
```

---

## Documentation Index

| Topic | Guide |
|---|---|
| Next.js App Router lifecycle & runtime boundaries | [Architecture](docs/architecture.md) |
| Domain boundaries & bounded context specifications | [Domain Map](docs/domain-map.md) |
| Production TypeScript, Prisma & POS integration implementations | [Code Examples](docs/code-examples.md) |
| Two-phase payment saga & compensating void recovery | [Payment Saga](docs/payment-saga-and-fault-tolerance.md) |
| Prisma ORM 6.13, Accelerate pooling & schema design | [Prisma Database](docs/prisma-database-architecture.md) |
| Dual-tier in-house POS REST API & direct database integration | [Direct POS Integration](docs/pos-api-and-direct-database-integration.md) |
| Architecture Decision Records | [ADRs](docs/decisions/README.md) |
| Reusable Mermaid source files | [Diagrams](diagrams/README.md) |

---

## Technology Map

| Layer | Technologies / Services |
|---|---|
| **Web Framework** | Next.js 16 (App Router), React 19 (Server Components), Tailwind CSS v4 |
| **State & Forms** | Zustand (persisted `cart-v2`), React Hook Form, Zod |
| **ORM & Database** | Prisma ORM 6.13, `@prisma/extension-accelerate`, Core POS PostgreSQL Database |
| **Direct POS Integration** | Direct in-house POS REST API (`/Voucher/order`, `/payment`, `/Customer`), Axios |
| **Object Storage** | Enterprise Catalog Media Archive (Amazon S3 / CDN) |
| **Payment Gateway** | Authorize.Net Accept.js, 2-phase authorization saga with compensating voids |
| **Back-Office ERP** | In-House Physical Store POS ERP System |

---

## Contact

**M Nawaz Khan  Full Stack Developer**
- GitHub: [Nawaz393](https://github.com/Nawaz393)
- LinkedIn: [Muhammad Nawaz Khan](https://www.linkedin.com/in/nawaz393)

[Documentation Index](docs/README.md)
