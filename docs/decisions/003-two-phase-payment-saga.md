# ADR 003: Distributed Two-Phase Payment Saga with Compensating Transactions

[← ADR Index](README.md) · [← Main README](../../README.md) · [← Documentation Index](../README.md)

- **Status:** Accepted
- **Date:** 2025-11-22
- **Deciders:** Software Architecture Team, Lead Full-Stack Engineer (M Nawaz Khan)

---

## Context

In an enterprise omnichannel retail environment, placing a digital order spans three independent, decoupled distributed systems:
1. **The Payment Gateway (Authorize.Net)**: Charges or reserves customer credit card funds.
2. **The Cloud E-Commerce Database (PostgreSQL / Prisma)**: Manages customer profiles, persistent shopping carts, and web order history.
3. **The Physical Store In-House POS ERP System**: Manages real-world physical warehouse inventory, fiscal vouchers, and store accounting ledgers.

### The Single-Step Failure Mode
A naive checkout flow executes a single-step immediate capture (`AUTH_CAPTURE`) at Authorize.Net, followed by calling the external POS API to create the store voucher. If the external POS API is temporarily unavailable, returns a 504 gateway timeout, or fails due to network partitions, the customer's card is billed immediately, but the warehouse never receives the fulfillment order. 

This results in **"ghost charges"**, customer support escalations, payment dispute chargeback fees, and painful manual reconciliation between Authorize.Net merchant deposits and POS store sales receipts.

## Decision

We engineered a **Distributed Two-Phase Commit Saga with Automated Compensating Voids** orchestrated inside the Next.js server route handler (`src/app/api/checkout/route.ts`):

```
[Customer Request]
        │
        ▼
1. Pre-Flight Validation ───> Cart & VAT Recalculation (Strict ±$0.01 Tamper Check)
        │
        ▼
2. Phase 1 (Auth Hold)   ───> Authorize.Net: AUTHONLYTRANSACTION (Lock funds, no charge)
        │
        ├────────────────────> [FAIL] ──> Abort checkout (No card funds held)
        ▼ [OK]
3. External POS Voucher  ───> In-House POS: POST /Voucher/order
        │
        ├────────────────────> [FAIL] ──> COMPENSATING VOID: Authorize.Net VOIDTRANSACTION
        ▼ [OK]                            (Instant fund release, Return 500)
4. Phase 2 (Capture)     ───> Authorize.Net: PRIORAUTHCAPTURETRANSACTION
        │
        ├────────────────────> [FAIL] ──> COMPENSATING ROLLBACK: POS DELETE /Voucher/order
        ▼ [OK]                            & Authorize.Net VOIDTRANSACTION (Return 502)
5. POS Ledger Settle     ───> In-House POS: PUT /Voucher/order/{id}/payment (Credit Ledger)
        │
        ├────────────────────> [FAIL] ──> COMPENSATING VOID: Authorize.Net VOIDTRANSACTION
        ▼ [OK]                            & POS DELETE /Voucher/order (Return 500)
[Order Confirmed]
```

1. **Authorization Hold (`AUTHONLYTRANSACTION`)**: Instead of charging immediately, the server instructs Authorize.Net to place a verified funds hold on the customer's card.
2. **Compensating Voids**: If the external POS voucher generation fails, the coordinator executes `gatewayVoidPayment(authResult.transactionId)`, instantly releasing the hold at the cardholder's issuing bank.
3. **Prior Auth Capture (`PRIORAUTHCAPTURETRANSACTION`)**: Once the physical store system successfully books the voucher, the held transaction is captured.
4. **Bidirectional Rollback on Settle Failure**: If the final POS ledger settlement fails, the capture is voided and the voucher is deleted via `cancelPosOrder`.

## Consequences

### Positive
- **Elimination of Ghost Charges**: A card is only definitively captured if and only if the external fulfillment system successfully registers the voucher and ledger.
- **Immediate Customer Remediation**: The compensating void releases customer credit card holds instantly rather than forcing customers to wait 5–7 business days for an uncaptured authorization to expire at their bank.
- **PCI-DSS SAQ-A Boundary**: Combined with client-side Accept.js tokenization, credit card numbers never touch the Next.js backend, limiting regulatory scope.
- **Mathematical Integrity**: The saga enforces strict server-side cart VAT recalculations before triggering Phase 1, preventing client-side parameter manipulation.

### Negative / Operational Challenges
- **Sequential Network Latency**: Executing sequential API calls (Authorize $\rightarrow$ POS Create $\rightarrow$ Capture $\rightarrow$ POS Settle) incurs combined latency of approximately 1.2 to 2.0 seconds during checkout.
- **Edge-Case Network Partitions**: If a network connection severs during the execution of a compensating void, the system logs the incident for automated reconciliation.

## Trade-offs

- We chose synchronous coordination within the Next.js Route Handler over asynchronous message broker sagas (e.g., Temporal or AWS Step Functions). For an interactive web checkout, customers expect an immediate synchronous confirmation receipt with an order ID. Asynchronous background capture introduces UI polling complexity and ambiguous intermediate order states.

## Alternatives Considered

1. **Single-Step Immediate Capture (`AUTH_CAPTURE`)**:
   - *Rejected*: Proved unacceptable due to orphaned charges whenever POS APIs timed out under peak store hours.
2. **Dual-Write Webhooks**:
   - *Rejected*: Capturing funds first and relying on POS webhooks caused out-of-order execution and delays in sending order confirmation emails.
3. **External Saga Orchestrators (AWS Step Functions)**:
   - *Rejected*: Added infrastructure complexity, cold-start latency, and external state machine overhead for a 4-step linear transaction flow.
