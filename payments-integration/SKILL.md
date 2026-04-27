---
name: payments-integration
description: 'Implement deposits, checkout initialization, callback verification, and webhook settlement using Paystack and 100Pay in Next.js + Express apps. Use when building wallet top-ups, number rental deposits, rental extension deposits, subscription provider checkout, or 100Pay webhook processing. Covers server-authoritative checkout payloads, idempotent pending transaction lifecycle, callback-as-signal patterns, and settlement race handling.'
---

# Payments Integration (Paystack + 100Pay)

This skill documents production patterns used in this repository for handling
payments end-to-end, with focus on 100Pay flows implemented across:

- `apps/api/src/routes/payments.ts`
- `apps/api/src/services/payment/hundredpay.service.ts`
- `apps/web/utils/payWith100Pay.ts`
- `apps/web/app/(app)/wallet/page-client.tsx`
- `apps/web/components/marketplace/number-purchase-flow.tsx`
- `apps/web/components/marketplace/extend-rental-dialog-button.tsx`

## When to Use

Use this skill when you need to:

- Add or debug wallet top-up checkout (100Pay or Paystack)
- Add or debug number rental deposit checkout
- Add or debug rental extension deposit checkout
- Handle callback redirect pages and client-side polling confirmation
- Build resilient 100Pay webhook settlement (idempotency, retries, races)
- Reuse this architecture in another project

## Mental Model

In this implementation, payment success is not a single event. It is a staged
lifecycle:

1. Server init route creates (or resumes) a pending transaction.
2. Client opens provider checkout.
3. Client callback is treated as a signal, not final settlement.
4. Server webhook performs authoritative settlement.
5. Client polls by reference to observe terminal state and redirect UX.

```
Client -> POST /v1/payments/.../initiate -> pending Transaction + reference
       -> 100Pay modal (shop100Pay.setup)
       -> callback/onPayment signal
       -> GET /v1/payments/.../by-reference (poll)
Webhook -> POST /v1/payments/webhooks/100pay
        -> verify token, dedupe event, settle transaction
        -> create rental/apply extension/credit wallet
```

## Golden Rules

1. Treat 100Pay callback/onPayment as a signal only. Final state is server settlement.
2. Always return server-authoritative customer identity in checkout payloads.
3. Reuse pending checkouts for short windows (30 min in this codebase) instead of creating duplicates.
4. Keep one durable provider reference per transaction and use it for lookup/polling.
5. Dedupe webhook events (`meta.processedPaymentEventIds`) and callback triggers.
6. Release side effects on failure states (for rental deposits, release number reservation).

## Current Route Surface (Repository)

100Pay-relevant server routes:

- `POST /v1/payments/wallet-topups/initiate`
- `POST /v1/payments/wallet-topups/verify`
- `GET /v1/payments/initiated`
- `POST /v1/payments/rental-deposits/initiate`
- `GET /v1/payments/rental-deposits/pending`
- `GET /v1/payments/rental-deposits/by-reference`
- `POST /v1/payments/rental-extension-deposits/initiate`
- `GET /v1/payments/rental-extension-deposits/by-reference`
- `POST /v1/payments/webhooks/100pay`
- `GET /v1/payments/callback` (redirect relay)

## File Map (Reference Modules)

| Topic | File |
|------|------|
| 100Pay client launcher and callback behavior | [hundredpay-client.md](./references/hundredpay-client.md) |
| 100Pay initiate/verify server flows (wallet + rentals) | [hundredpay-deposit-route.md](./references/hundredpay-deposit-route.md) |
| 100Pay webhook settlement and idempotency | [hundredpay-webhook.md](./references/hundredpay-webhook.md) |
| Env vars and provider setup | [env-and-setup.md](./references/env-and-setup.md) |
| Pitfalls and race conditions | [pitfalls.md](./references/pitfalls.md) |
| Paystack wrappers/routes/webhook | [paystack-server.md](./references/paystack-server.md), [paystack-deposit-route.md](./references/paystack-deposit-route.md), [paystack-webhook.md](./references/paystack-webhook.md) |
| Payout flow with 100Pay | [hundredpay-payout.md](./references/hundredpay-payout.md) |
| Pay-link flow | [pay-link-flow.md](./references/pay-link-flow.md) |
| Data model guidance | [data-model.md](./references/data-model.md) |

## 100Pay End-to-End Checklist

1. Validate env and keys (`HUNDREDPAY_API_KEY`, `HUNDREDPAY_SECRET_API_KEY`, `HUNDREDPAY_WEBHOOK_SECRET`, `NEXT_PUBLIC_HUNDREDPAY_API_KEY`).
2. Initiate route returns:
   - `reference`
   - `hostedUrl`
   - `resumed`
   - `checkout` payload (customer, billing, metadata)
3. Client launches `payWith100Pay` with checkout payload from server.
4. Client callback triggers verification/poll by reference.
5. Webhook verifies `verification-token` and settles pending transaction.
6. By-reference endpoints expose eventual business state (`rental_created`, `extension_applied`, or pending/failed/reversed).
7. UI redirects only after by-reference confirms terminal success.

## Debugging Procedure

1. Query `Transaction` by `reference` and inspect `status`, `meta.kind`, `meta.provider`, `meta.paymentStatus`, `meta.processedPaymentEventIds`.
2. Confirm webhook token is valid and `POST /v1/payments/webhooks/100pay` is receiving events.
3. Confirm callback path uses the correct reference and poll endpoint.
4. For rental deposits, verify reservation release/creation side effects:
   - `PhoneNumber.status` transitions (`available` -> `reserved` -> `rented` or rollback)
   - `meta.rentalCreated` and `meta.rentalId`
5. For extension deposits, verify `meta.extensionApplied` and rental idempotency keys.
