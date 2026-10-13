# Engel E-Commerce Platform — Engineering Case Study

A documentation-first architectural case study of **Engel E-Commerce** (`engel-web-app`), an enterprise full-stack retail and B2B wholesale platform bridging web storefront interactions with brick-and-mortar back-office ERP systems (ForeverPOS / SAK Retail Solutions).

![Next.js 16](https://img.shields.io/badge/Next.js-16_App_Router-000000?logo=next.js&logoColor=white)
![React 19](https://img.shields.io/badge/React-19_RSC-61DAFB?logo=react&logoColor=black)
![TypeScript](https://img.shields.io/badge/TypeScript-5.x-3178C6?logo=typescript&logoColor=white)
![Prisma ORM](https://img.shields.io/badge/Prisma-6.13_%7C_Accelerate-2D3748?logo=prisma&logoColor=white)
![AWS RDS](https://img.shields.io/badge/AWS_RDS-PostgreSQL_Stockholm-232F3E?logo=amazon-aws&logoColor=white)
![AWS Lambda](https://img.shields.io/badge/AWS_Lambda-Python_ETL-FF9900?logo=aws-lambda&logoColor=white)
![Authorize.Net](https://img.shields.io/badge/Payments-Authorize.Net_Accept.js-004B87)

---

## Executive Overview

**Engel E-Commerce** provides a hybrid digital commerce solution supporting both direct-to-consumer retail shopping and B2B wholesale quotation management. The platform unites high-performance React Server Components with resilient serverless cloud infrastructure in AWS Region `eu-north-1` (Stockholm).

### Architectural Highlights
1. **Next.js 16 App Router & React 19**: Server Components render product catalog pages with zero client bundle overhead; Server Actions handle cart mutations and checkout orchestration.
2. **Prisma ORM 6.13 with Accelerate**: Overcomes serverless PostgreSQL connection exhaustion through global edge connection pooling and edge-level SWR caching.
3. **Two-Phase Payment Saga with Compensating Voids**: Card tokens generated via Authorize.Net Accept.js are authorized (`AUTH_ONLY`), and if downstream ERP synchronization fails, an automated compensating void (`voidTransactionRequest`) executes to guarantee zero phantom charges.
4. **Serverless Catalog & Inventory ETL (`engels_lambda`)**: An AWS Lambda event microservice continuously ingests vendor catalog files, normalizes tax categories, and performs atomic delta stock syncs using PostgreSQL advisory locks.

---

## Cloud Architecture Topology

```mermaid
flowchart TB
    subgraph ClientLayer ["1. Client Runtime (Browser)"]
        Browser["React 19 UI (Tailwind CSS v4)"]
        Zustand["Zustand Store (cart-v2 in LocalStorage)"]
        AcceptJS["Authorize.Net Accept.js (Card Nonce Generator)"]
    end

    subgraph VercelEdge ["2. Vercel Serverless Runtime"]
        RSC["React Server Components (Catalog & Layouts)"]
        Actions["Server Actions ('use server')"]
        API["API Route: /api/checkout"]
    end

    subgraph AWSStockholm ["3. AWS Cloud (eu-north-1 Stockholm)"]
        Accelerate["Prisma Accelerate Connection Pooler"]
        RDS[("AWS RDS PostgreSQL: foreverpos")]
        S3[("Amazon S3: forever-pos-bucket")]
        LambdaWorker["AWS Lambda: engels_lambda (Catalog Sync)"]
        EventBridge["EventBridge Scheduled Cron"]
    end

    subgraph ExternalServices ["4. External Enterprise Rails"]
        AuthNet["Authorize.Net Payment Processing Gateway"]
        ForeverPOS["ForeverPOS API (Back-Office ERP)"]
    end

    Browser --> AcceptJS
    AcceptJS --> Browser
    Browser --> RSC
    Browser --> Actions
    Browser --> API
    Actions --> Accelerate
    API --> Accelerate
    Accelerate --> RDS
    API --> AuthNet
    API --> ForeverPOS
    Actions --> S3
    EventBridge --> LambdaWorker
    LambdaWorker --> RDS
    LambdaWorker --> ForeverPOS
```

---

## Documentation Index

| Topic | Guide |
|---|---|
| Next.js App Router lifecycle & runtime boundaries | [Architecture](docs/architecture.md) |
| Domain boundaries & bounded context specifications | [Domain Map](docs/domain-map.md) |
| Production TypeScript, Prisma & Lambda implementations | [Code Examples](docs/code-examples.md) |
| Two-phase payment saga & compensating void recovery | [Payment Saga](docs/payment-saga-and-fault-tolerance.md) |
| Prisma ORM 6.13, Accelerate pooling & schema design | [Prisma Database](docs/prisma-database-architecture.md) |
| AWS Lambda microservices & serverless ETL pipeline | [Serverless & Lambdas](docs/serverless-infrastructure-and-lambdas.md) |
| Architecture Decision Records | [ADRs](docs/decisions/README.md) |
| Reusable Mermaid source files | [Diagrams](diagrams/README.md) |

---

## Technology Map

| Layer | Technologies / Services |
|---|---|
| **Web Framework** | Next.js 16 (App Router), React 19 (Server Components), Tailwind CSS v4 |
| **State & Forms** | Zustand (persisted `cart-v2`), React Hook Form, Zod |
| **ORM & Database** | Prisma ORM 6.13, `@prisma/extension-accelerate`, AWS RDS PostgreSQL |
| **Serverless Compute** | AWS Lambda (Python 3.11, containerized `engels_lambda`), EventBridge |
| **Object Storage** | Amazon S3 (`forever-pos-bucket`), AWS CloudFront |
| **Payment Gateway** | Authorize.Net Accept.js, 2-phase authorization saga |
| **Back-Office Integration**| ForeverPOS API / SAK Retail Solutions REST bridge |

---

## Contact

**M Nawaz Khan — Full Stack Developer & Cloud Systems Engineer**
- GitHub: [Nawaz393](https://github.com/Nawaz393)
- LinkedIn: [Muhammad Nawaz Khan](https://www.linkedin.com/in/nawaz393)

[Documentation Index](docs/README.md)
