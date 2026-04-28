---
name: payments-integration
description: 'Implement deposits, checkout initialization, callback verification, and webhook settlement using Paystack and 100Pay in Next.js + Express apps. Use when building wallet top-ups, number rental deposits, rental extension deposits, subscription provider checkout, or 100Pay webhook processing. Covers server-authoritative checkout payloads, idempotent pending transaction lifecycle, callback-as-signal patterns, and settlement race handling.'
---

# Payments Integration (Paystack + 100Pay)

This skill documents production payment patterns that are reusable across
projects. It also includes a concrete mapping to this repository so examples
stay grounded in real code.

## Open-Source Safety

Use these docs as public guidance, but keep operational details private:

1. Never publish real API keys, webhook secrets, or signed payload samples.
2. Keep only placeholder domains and values in examples (`example.com`, `pk_xxx`, `sk_xxx`).
3. Do not include real user identifiers, phone numbers, or email addresses in docs.
4. Do not publish production incident IDs, internal ticket links, or private dashboards.
5. Treat callback and webhook payload logs as potentially sensitive; redact before sharing.

Repository example files:

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

Payment success is not a single event. It is a staged lifecycle:

Generic flow:

```
Client -> POST /payments/.../initiate -> pending tx + reference
       -> provider checkout SDK or hosted URL
       -> callback signal (not final settlement)
       -> verify/poll endpoint by reference
Webhook -> POST /payments/webhooks/provider
    -> auth signature/token
    -> dedupe event
    -> settle tx + apply domain side effects
```

Repository example:

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

## Current Route Surface (Repository Example)

Example 100Pay route shape used in this repository:

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
3. Client launches provider checkout from server-authoritative payload.
4. Client callback triggers verification/poll by reference.
5. Webhook verifies `verification-token` and settles pending transaction.
6. By-reference endpoints expose eventual business state (`rental_created`, `extension_applied`, or pending/failed/reversed).
7. UI redirects only after by-reference confirms terminal success.
8. All documentation examples use sanitized values and non-production domains.

Generic note:

- Path names, event names, and status names vary by provider and platform.
- Keep the flow shape, not the literal route names.

## Debugging Procedure

1. Query payment records by `reference` and inspect lifecycle state transitions.
2. Confirm webhook auth is valid and webhook events are being received.
3. Confirm callback path uses the correct reference and verify/poll endpoint.
4. Verify failure paths release reservations/locks and success paths apply domain side effects exactly once.
5. Verify idempotency markers prevent duplicate settlement and duplicate side effects.

Public note:

- Keep low-level schema names and internal persistence field names in private runbooks, not in public docs.
