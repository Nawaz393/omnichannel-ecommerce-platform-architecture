# Production Code Examples

[← Main README](../README.md) · [← Documentation Index](README.md)

This document contains representative, production-grade code implementations extracted directly from the **Enterprise Omnichannel E-Commerce Platform** codebase.

---

## 1. Two-Phase Payment Saga Controller with Compensating Voids

**Source File**: `src/app/api/checkout/route.ts`  
**Pattern**: Distributed Saga Execution Coordinator with Automated Compensating Transactions.

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { userService } from '@/services/user';
import { revalidateCartTotal as computeCart } from './lib/cart';
import { createOrderPayload as buildOrderPayload } from './lib/order';
import { insertOrderThroughApi, updatePaymentStatus as pushPaymentStatus, cancelPosOrder } from './lib/pos';
import {
  authorizePayment as gatewayAuthorizePayment,
  capturePayment as gatewayCapturePayment,
  voidPayment as gatewayVoidPayment,
} from './lib/payments';
import type { PaymentData } from './lib/types';
import { SHIPPING_COST } from '@/data/static-vars';

// Enforce standard Node.js runtime for Authorize.Net native SDK and crypto support
export const runtime = 'nodejs';

export async function POST(request: NextRequest) {
  try {
    const authHeader = request.headers.get('authorization');
    if (!authHeader) {
      return NextResponse.json({ success: false, error: 'Missing Authorization header' }, { status: 401 });
    }

    const paymentData: PaymentData = await request.json();

    if (!paymentData.amount || paymentData.amount <= 0) {
      return NextResponse.json({ success: false, error: 'Invalid payment amount' }, { status: 400 });
    }

    // Step 0A: Verify authenticated identity matches payload customer
    try {
      const me = await userService.getUserWithAuth(authHeader);
      if (!me?.customerId || me.customerId !== paymentData.customerInfo.customerId) {
        return NextResponse.json({ success: false, error: 'User verification failed' }, { status: 403 });
      }
    } catch (e) {
      console.error('User verification error:', e);
      return NextResponse.json({ success: false, error: 'Unable to verify user' }, { status: 401 });
    }

    // Step 0B: Server-Side Cart & VAT Mathematical Revalidation
    const cartRevalidationRes = await computeCart(paymentData.customerInfo.customerId);
    if (!cartRevalidationRes) {
      return NextResponse.json({ success: false, error: 'Cart is empty or not found' }, { status: 400 });
    }

    const totalAmount = Number(cartRevalidationRes.total.toFixed(2));
    const expectedAmount = totalAmount + SHIPPING_COST + cartRevalidationRes.totalVat;

    // Strict anti-tampering check (prevent client-side parameter manipulation)
    if (Math.abs(paymentData.amount - expectedAmount) > 0.01) {
      return NextResponse.json({ success: false, error: 'Payment amount does not match cart total' }, { status: 400 });
    }

    const orderPayload = buildOrderPayload(
      paymentData,
      totalAmount,
      SHIPPING_COST,
      cartRevalidationRes.totalVat,
      cartRevalidationRes.cart,
    );

    // Step 1: Phase 1 — Authorize payment hold on card (AUTHONLYTRANSACTION)
    const invoiceNumber = `${Date.now()}-${paymentData.customerInfo.customerId}`;
    const authResult = await gatewayAuthorizePayment(
      paymentData,
      orderPayload,
      expectedAmount,
      cartRevalidationRes.totalVat,
      SHIPPING_COST,
      invoiceNumber,
    );
    if (!authResult.success) {
      return NextResponse.json(authResult, { status: 400 });
    }

    // Step 2: Create physical store voucher order in In-House POS
    const orderApiRes = await insertOrderThroughApi(orderPayload, authHeader);
    if (!orderApiRes?.voucherId) {
      // Compensating Transaction: Void the card hold immediately
      await gatewayVoidPayment(authResult.transactionId);
      return NextResponse.json({ success: false, error: 'Failed to create order in external system' }, { status: 500 });
    }
    const orderId = orderApiRes.voucherId as string;

    // Step 3: Phase 2 — Prior Auth Capture (PRIORAUTHCAPTURETRANSACTION)
    const captureResult = await gatewayCapturePayment(authResult.transactionId, expectedAmount);
    if (!captureResult.success) {
      // Compensating Transactions: Cancel POS voucher and void card hold
      await cancelPosOrder(orderId, authHeader);
      await gatewayVoidPayment(authResult.transactionId);
      return NextResponse.json(captureResult, { status: 502 });
    }

    // Step 4: Settle external POS payment ledger
    const updatePaymentStatusRes = await pushPaymentStatus(orderId, Number(expectedAmount.toFixed(2)), authHeader);
    if (!updatePaymentStatusRes?.success) {
      // Compensating Transactions: Void captured payment and delete POS voucher
      const voidRes = await gatewayVoidPayment(captureResult.transactionId);
      await cancelPosOrder(orderId, authHeader);
      if (!voidRes.success) {
        console.error('CRITICAL: Automated void failed; manual reconciliation required for tx:', captureResult.transactionId);
      }
      return NextResponse.json(
        {
          success: false,
          error: 'Payment succeeded but failed to update external system; reversed payment attempt made',
        },
        { status: 500 },
      );
    }

    return NextResponse.json(
      {
        success: true,
        transactionId: captureResult.transactionId,
        orderData: { orderId, amount: Number(expectedAmount.toFixed(2)) },
      },
      { status: 200 },
    );
  } catch (error) {
    console.error('Payment processing error:', error);
    return NextResponse.json(
      { success: false, error: 'Internal server error during payment processing' },
      { status: 500 },
    );
  }
}
```

---

## 2. Client-Side PCI-DSS SAQ-A Card Tokenization

**Source File**: `src/services/payment-service.ts`  
**Pattern**: Browser-side credential isolation via Authorize.Net `Accept.js`.

```typescript
import { AcceptAPI, AcceptResponse, PaymentToken } from '@/types';
import { toast } from 'sonner';
import type { CardEntryData } from '@/types/payment-gateway';

class PaymentServiceError extends Error {
  constructor(
    message: string,
    public code?: string,
  ) {
    super(message);
    this.name = 'PaymentServiceError';
  }
}

export class PaymentService {
  private static readonly CARD_NUMBER_REGEX = /^\d{13,19}$/;
  private static readonly CVV_REGEX = /^\d{3,4}$/;

  private static getAuthData(): { clientKey: string; apiLoginID: string } {
    const clientKey = process.env.NEXT_PUBLIC_AUTHORIZE_NET_CLIENT_KEY || '';
    const apiLoginID = process.env.NEXT_PUBLIC_AUTHORIZE_NET_API_LOGIN_ID || '';

    if (!clientKey || !apiLoginID) {
      throw new PaymentServiceError('Payment configuration is missing. Please contact support.', 'CONFIG_MISSING');
    }

    return { clientKey, apiLoginID };
  }

  private static validateCardData(data: CardEntryData): void {
    const sanitizedCardNumber = data.cardNumber.replace(/\s/g, '');

    if (!this.CARD_NUMBER_REGEX.test(sanitizedCardNumber)) {
      throw new PaymentServiceError('Invalid card number format', 'INVALID_CARD');
    }

    if (!this.CVV_REGEX.test(data.cvv)) {
      throw new PaymentServiceError('Invalid CVV format', 'INVALID_CVV');
    }

    const [month, year] = data.expMonth.split('/');
    if (!month || !year || isNaN(Number(month)) || isNaN(Number(year))) {
      throw new PaymentServiceError('Invalid expiration date format', 'INVALID_EXPIRY');
    }
  }

  private static formatCardData(data: CardEntryData) {
    const [month, year] = data.expMonth.split('/');

    return {
      cardNumber: data.cardNumber.replace(/\s/g, ''),
      month: month.trim(),
      year: year.trim().padStart(4, '20'),
      cardCode: data.cvv,
    };
  }

  /**
   * Tokenizes card data in the browser. Raw PAN/CVV never leaves the client.
   */
  static async getPaymentToken(formData: CardEntryData): Promise<PaymentToken> {
    try {
      this.validateCardData(formData);

      const authData = this.getAuthData();
      const cardData = this.formatCardData(formData);

      return await this.dispatchPaymentData(authData, cardData);
    } catch (error) {
      if (error instanceof PaymentServiceError) {
        toast.error(error.message);
        throw error;
      }

      const message = 'An unexpected error occurred during payment processing';
      toast.error(message);
      throw new PaymentServiceError(message, 'UNKNOWN_ERROR');
    }
  }

  private static dispatchPaymentData(
    authData: { clientKey: string; apiLoginID: string },
    cardData: ReturnType<typeof PaymentService.formatCardData>,
  ): Promise<PaymentToken> {
    return new Promise((resolve, reject) => {
      try {
        const acceptAPI = (window as unknown as { Accept: AcceptAPI }).Accept;

        if (!acceptAPI) {
          reject(
            new PaymentServiceError(
              'Payment service not available. Please refresh and try again.',
              'SERVICE_UNAVAILABLE',
            ),
          );
          return;
        }

        acceptAPI.dispatchData({ authData, cardData }, (response: AcceptResponse) => {
          if (response.messages.resultCode === 'Error') {
            const errorMessages = response.messages.message.map((msg) => msg.text).join(' ');
            toast.error(errorMessages);
            reject(new PaymentServiceError(errorMessages, 'PAYMENT_ERROR'));
          } else {
            resolve({
              dataDescriptor: response.opaqueData.dataDescriptor,
              dataValue: response.opaqueData.dataValue,
            });
          }
        });
      } catch (error) {
        console.error(error);
        reject(new PaymentServiceError('Failed to initialize payment processing', 'INIT_FAILED'));
      }
    });
  }
}
```

---

## 3. Atomic Guest-to-Customer Cart Merge Server Action

**Source File**: `src/server/queries/cart-action.ts`  
**Pattern**: Next.js Server Action with Prisma ACID Transaction and Live Stock Clamping.

```typescript
'use server';

import prisma from '@/lib/prisma';
import { cookies } from 'next/headers';
import type { Prisma } from '@prisma/client';

export async function attachActiveCartToUser(input: {
  userId: number; // Logged-in customer ID
  companyId: number;
  anonymousId?: string | null;
}) {
  const { userId, companyId } = input;
  if (!userId) throw new Error('userId is required');

  const jar = await cookies();
  const anonCookieKey = 'anonId';
  const anonId = input.anonymousId ?? jar.get(anonCookieKey)?.value ?? null;

  const result = await prisma.$transaction(async (tx) => {
    // 1. Locate active anonymous guest cart
    const anonCart = anonId
      ? await tx.carts.findFirst({
          where: {
            status: 'active',
            anonymousid: anonId,
            ...(companyId ? { companyid: companyId } : {}),
          },
        })
      : null;

    // 2. Locate active customer cart
    const userCart = await tx.carts.findFirst({
      where: {
        status: 'active',
        userid: userId,
        ...(companyId ? { companyid: companyId } : {}),
      },
    });

    // Case A: Neither exists -> Create fresh user cart
    if (!anonCart && !userCart) {
      const fresh = await tx.carts.create({
        data: {
          userid: userId,
          anonymousid: null,
          status: 'active',
          companyid: companyId,
        },
      });
      return readSummary(tx, fresh.cartid);
    }

    // Case B: Only anonymous cart exists -> Re-assign ownership directly to user
    if (anonCart && !userCart) {
      const claimed = await tx.carts.update({
        where: { cartid: anonCart.cartid },
        data: { userid: userId, anonymousid: null },
      });
      return readSummary(tx, claimed.cartid);
    }

    // Case C: Only user cart exists -> Return existing cart
    if (!anonCart && userCart) {
      return readSummary(tx, userCart.cartid);
    }

    // Case D: Both exist -> Merge anonymous items into user cart with live stock clamping
    if (!userCart || !anonCart) throw new Error('Unexpected state');

    if (anonCart.cartid !== userCart.cartid) {
      const anonItems = await tx.cartitems.findMany({
        where: { cartid: anonCart.cartid },
      });

      for (const it of anonItems) {
        const existing = await tx.cartitems.findFirst({
          where: {
            cartid: userCart.cartid,
            productid: it.productid,
            variantid: it.variantid,
          },
        });

        // Query live warehouse inventory to clamp quantities
        const variant = it.variantid
          ? await tx.productvariants.findUnique({
              where: { variantid: it.variantid },
              select: { stockquantity: true },
            })
          : null;

        const availableRaw = variant?.stockquantity != null ? Number(variant.stockquantity) : null;
        const available = Number.isFinite(availableRaw) ? Number(availableRaw) : Number.POSITIVE_INFINITY;
        const currentQty = Number(existing?.quantity ?? 0);
        const nextQtyRaw = currentQty + Number(it.quantity);
        const nextQty = Number.isFinite(available) ? Math.min(nextQtyRaw, available) : nextQtyRaw;

        if (nextQty <= 0) {
          if (existing) await tx.cartitems.delete({ where: { cartitemid: existing.cartitemid } });
        } else if (!existing) {
          await tx.cartitems.create({
            data: {
              cartid: userCart.cartid,
              productid: it.productid,
              variantid: it.variantid,
              quantity: nextQty,
              unitprice: Number(it.unitprice),
              name: it.name,
              image: it.image ?? null,
              sku: it.sku ?? null,
              barcode: it.barcode ?? null,
            },
          });
        } else {
          await tx.cartitems.update({
            where: { cartitemid: existing.cartitemid },
            data: { quantity: nextQty, unitprice: existing.unitprice },
          });
        }
      }

      // Deactivate and transition anonymous cart state
      await tx.carts.update({
        where: { cartid: anonCart.cartid },
        data: { status: 'merged', anonymousid: null },
      });
    }

    return readSummary(tx, userCart.cartid);
  });

  // Clear 30-day anonymous guest cookie on successful merge
  if (jar.get(anonCookieKey)) {
    jar.set(anonCookieKey, '', { path: '/', maxAge: 0, httpOnly: false });
  }

  return result;
}
```

---

## 4. Prisma Accelerate Client Extension

**Source File**: `src/lib/prisma.ts`  
**Pattern**: Singleton client with Accelerate edge connection pooling and global caching.

```typescript
import { PrismaClient } from '@prisma/client';
import { withAccelerate } from '@prisma/extension-accelerate';

const globalForPrisma = global as unknown as {
  prisma: PrismaClient;
};

// Extend Prisma Client with Accelerate pooling and global edge cache
const prisma =
  globalForPrisma.prisma ||
  new PrismaClient({
    log: process.env.NODE_ENV === 'development' ? ['query', 'error', 'warn'] : ['error'],
  }).$extends(withAccelerate());

if (process.env.NODE_ENV !== 'production') {
  globalForPrisma.prisma = prisma;
}

export default prisma;
```

---

## 5. Sales-Ranked Dynamic Popular Products Query

**Source File**: `src/server/queries/product.ts`  
**Pattern**: Server-side Prisma aggregation across relational POS receipts.

```typescript
'use server';

import { COMPANY_ID } from '@/data/static-vars';
import prisma from '@/lib/prisma';
import { getMinVariantPrice, getProductImage } from '@/lib/product-fields';

type PopularProductsParams = {
  page?: number;
  pageSize?: number;
  categoryId?: number;
};

export const getPopularProducts = async ({ page = 1, pageSize = 10, categoryId = 0 }: PopularProductsParams) => {
  // 1. Fetch matching catalog entities
  const products = await prisma.products.findMany({
    where: {
      companyid: +COMPANY_ID,
      ...(categoryId !== 0 && { productgroupid: categoryId }),
    },
    select: {
      productid: true,
      productname: true,
      description: true,
      rate: true,
      images: true,
      productgroup: {
        select: { groupid: true, groupname: true },
      },
      productvariants: {
        select: { variantid: true, sku: true, price: true },
      },
    },
  });

  const productIds = products.map((product) => product.productid);

  // 2. Aggregate sales volume directly from saledetail transactions
  const salesTotals = productIds.length
    ? await prisma.saledetail.groupBy({
        by: ['productid'],
        where: {
          companyid: +COMPANY_ID,
          productid: { in: productIds },
        },
        _sum: { total: true },
      })
    : [];

  const totalsByProductId = new Map<number, number>();
  for (const row of salesTotals) {
    if (row.productid != null) {
      totalsByProductId.set(row.productid, Number(row._sum.total ?? 0));
    }
  }

  // 3. Sort catalog items descending by real sales volume
  const sorted = products
    .map((product) => ({
      product,
      totalSales: totalsByProductId.get(product.productid) ?? 0,
    }))
    .sort((a, b) => {
      if (b.totalSales !== a.totalSales) return b.totalSales - a.totalSales;
      return (a.product.productname ?? '').localeCompare(b.product.productname ?? '');
    });

  const offset = (page - 1) * pageSize;
  const data = sorted.slice(offset, offset + pageSize).map(({ product }) => {
    const variants =
      product.productvariants?.map((variant) => ({
        variantId: variant.variantid,
        sku: variant.sku ?? '',
        price: variant.price != null ? Number(variant.price) : null,
      })) ?? [];

    return {
      id: product.productid,
      name: product.productname ?? '',
      description: product.description ?? '',
      rate: product.rate != null ? Number(product.rate) : getMinVariantPrice(variants),
      categoryId: product.productgroup?.groupid ?? null,
      categoryName: product.productgroup?.groupname ?? null,
      image: getProductImage({ images: product.images }),
      variants,
    };
  });

  return { data, totalCount: sorted.length };
};
```

---

## 6. Direct In-House POS REST API Integration Service

**Source Files**: `src/app/api/checkout/lib/pos.ts` and `src/lib/axiosInstance.ts`  
**Pattern**: Direct HTTP API Client with Request Interceptors, Token Injection, and Compensating Actions.

```typescript
// src/lib/axiosInstance.ts - Dedicated In-House POS Axios Instance
import axios, { AxiosError } from 'axios';

export const posApi = axios.create({
  baseURL: process.env.NEXT_PUBLIC_POS_API_URL,
  headers: {
    'Content-Type': 'application/json',
  },
  timeout: 10000,
});

// Request Interceptor: Injects Authorization Bearer Token
posApi.interceptors.request.use(
  async (config) => {
    if (typeof window === 'undefined') return config;
    const token = localStorage.getItem('accessToken');
    if (token) {
      config.headers = config.headers || {};
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error),
);

// src/app/api/checkout/lib/pos.ts - Direct POS Integration Functions
import { posApi } from '@/lib';
import type { Order } from './types';

export const insertOrderThroughApi = async (order: Order, authHeader: string) => {
  const res = await posApi.post('/Voucher/order', order, {
    headers: { Authorization: authHeader },
  });
  return res.data;
};

export const updatePaymentStatus = async (id: string, total: number, authHeader: string) => {
  const updatePaymentPayload = {
    cash: 0,
    credit: total,
    zilla: 0,
    otherPayment: 0,
    shippingAmount: 0,
    sendEmail: true,
  };

  try {
    const res = await posApi.put(`/Voucher/order/${id}/payment`, updatePaymentPayload, {
      headers: { Authorization: authHeader },
    });
    return { success: [200, 201, 202, 204].includes(res.status) };
  } catch (error) {
    console.error('Error updating payment status:', error);
    return { success: false };
  }
};

export const cancelPosOrder = async (id: string, authHeader: string) => {
  try {
    await posApi.delete(`/Voucher/order/${id}`, {
      headers: { Authorization: authHeader },
    });
    return { success: true };
  } catch (error) {
    console.error('Failed to cancel POS order', error);
    return { success: false };
  }
};
```

Related Guides: [Payment Saga Deep Dive](payment-saga-and-fault-tolerance.md) · [Prisma Architecture](prisma-database-architecture.md) · [POS Direct Integration](pos-api-and-direct-database-integration.md)
