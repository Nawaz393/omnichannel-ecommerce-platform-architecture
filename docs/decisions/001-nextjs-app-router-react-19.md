# ADR 001: Next.js 16 App Router & React 19 Server Components

[← ADR Index](README.md) · [← Main README](../../README.md) · [← Documentation Index](../README.md)

- **Status:** Accepted
- **Date:** 2025-11-18
- **Deciders:** Lead Full-Stack Engineer (M Nawaz Khan)

---

## Context

The omnichannel e-commerce platform requires a digital storefront capable of sub-second product discovery, high search engine visibility (SEO) across deep catalog hierarchies, and seamless omnichannel cart interactions. Traditional single-page applications (SPAs) ship extensive client-side JavaScript bundles, causing slow First Contentful Paint (FCP) and Largest Contentful Paint (LCP) times on mobile devices. 

Furthermore, data mutations such as cart item adjustments and guest-to-customer account migrations require direct database access, session cookies, and inventory validation. In a decoupled SPA architecture, every user action requires round-trip network hops through dedicated REST endpoints, exposing API surfaces and duplicating schema types across client and server.

## Decision

We chose **Next.js 16.1.1 (App Router)** combined with **React 19.1.0** and **TypeScript 5** as the core foundation for both web application and server-side execution.

1. **React Server Components (RSC) as Default**: Product catalogs, category listings, and marketing landing pages execute on the server. Data is fetched directly via Prisma ORM at the edge/server layer, shipping zero JavaScript bundle weight for static markup and styles.
2. **Strict Server/Client Boundaries**:
   - `'use client'` is reserved strictly for interactive user-interface primitives: the Zustand cart drawer, dynamic variant selectors, TanStack Query customer dashboards, and Authorize.Net Accept.js credit card inputs.
   - Server Actions (`'use server'`) handle write operations (such as `attachActiveCartToUser`, `setCartItemQtyServer`, and popularity aggregation) directly with database transactions.
3. **Route Handlers with Node.js Runtime**: For complex external integrations requiring legacy Node.js modules or native cryptography (such as the Authorize.Net SDK in `/api/checkout`), we explicitly designate `export const runtime = 'nodejs'`.

## Consequences

### Positive
- **Drastically Reduced Client Bundle**: Catalog pages stream pre-rendered HTML directly to the browser. Only the interactive islands (e.g., cart buttons, carousels) hydrate.
- **Type-Safe Full-Stack Mutations**: Server Actions allow client components to invoke server mutations with end-to-end TypeScript parameter verification without manual REST boilerplate.
- **Seamless Cookie & Session Coordination**: App Router native `cookies()` utility allows secure, synchronous access to 30-day anonymous UUID guest cookies and user auth tokens without additional middleware overhead.
- **Turbopack Dev Performance**: High-speed incremental compilation during local development.

### Negative / Operational Challenges
- **Serialization Boundaries**: Props passed from Server Components to Client Components must be serializable JSON. Decimal objects from Prisma must be explicitly converted to numbers or strings before transmission.
- **Serverless Cold Starts**: Next.js App Router serverless instances require optimized dependency trees to maintain p95 response latencies below 300ms.

## Trade-offs

- Choosing React 19 Server Components requires rigorous discipline around where client state (`useState`, `useStore`) lives versus server-executed data loaders.
- Server Actions do not replace all REST routes: external webhook endpoints and payment controllers requiring raw request stream handling or specific HTTP header semantics remain implemented as dedicated App Router Route Handlers.

## Alternatives Considered

1. **Next.js Pages Router**:
   - *Rejected*: Deprecated mental model, heavier client-side bundle sizes, and lack of component-level streaming and async Server Actions.
2. **Decoupled Single Page App (React SPA + Express/NestJS API)**:
   - *Rejected*: Required maintaining two separate deployment pipelines, custom CORS configurations, dual TypeScript models, and suffered from inferior SEO for retail catalog discovery.
3. **Remix / React Router v7**:
   - *Rejected*: While providing robust data loaders and form actions, Vercel native hosting, Turbopack optimizations, and ecosystem maturity with Prisma Accelerate strongly favored Next.js 16.
