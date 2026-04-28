# 100Pay Webhook Settlement

This document describes webhook settlement principles and then maps to the
current repository implementation.

## Generic Principles

1. Authenticate webhook request (signature or shared token).
2. Normalize provider payload into internal shape.
3. Ignore irrelevant events safely.
4. Enforce idempotency with event IDs and transaction status gates.
5. Apply terminal transitions and domain side effects exactly once.

## Repository Mapping

Primary route:

- `POST /v1/payments/webhooks/100pay`

Implemented in:

- `apps/api/src/routes/payments.ts`

Auth helper:

- `apps/api/src/services/payment/hundredpay.service.ts` (`verifyWebhookToken`)

## Request Authentication (Repository)

The webhook expects header:

- `verification-token`

Validation uses timing-safe comparison against:

- `HUNDREDPAY_WEBHOOK_SECRET`

If token is missing/invalid, route returns 400.

## Event Filtering (Repository)

The route normalizes incoming payload and exits early unless:

- `type === "credit"`
- charge reference (`ref_id`/reference) is present

References are resolved from payload fields such as:

- `payload.data.charge.ref_id`
- `payload.reference`
- `payload.data.reference`

## Dedupe and Idempotency (Repository)

The handler protects against duplicate webhook deliveries by using:

- a persistent processed-event ID set (payment event id membership)
- transaction status checks (`pending` / `successful` only for relevant branches)

If an event was already processed, it returns 200 (`received: true`) without repeating settlement.

## Status Handling (Repository)

Computed provider status comes from `charge.status.value`.

Terminal success states:

- `paid`
- `overpaid`
- `underpaid` (wallet top-up only)

Terminal failure states:

- `failed`
- `cancelled`
- `canceled`
- `expired`

## Settlement Branches (Repository)

### A) Wallet top-up (`meta.kind = wallet_topup`)

- Non-success: updates payment status metadata and exits.
- Success: runs idempotent settlement logic.
- Settlement credits wallet once and updates resolved metadata fields.

### B) Number rental deposit (`meta.kind = number_rental_deposit`)

- On failure states while pending:
  - marks transaction failed
  - records cancel reason
  - releases reserved number lock
- On success:
  - marks transaction successful if pending
  - triggers rental creation
  - stores settlement markers for created rental

### C) Other 100Pay credit flows

For non-rental-deposit credit flows (including top-up and similar provider credits),
settlement resolves through the same idempotent credit settlement path.

## Relationship with Client Verification

Webhook is authoritative, but clients may verify/poll in parallel:

- Wallet UI uses `POST /v1/payments/wallet-topups/verify`
- Marketplace/rental UIs poll by-reference endpoints

This design intentionally tolerates race conditions between:

- callback signal arrival
- verify request
- webhook processing

## Expected Data Shape

The route is resilient to minor payload shape differences and extracts:

- reference
- charge status
- payment event id
- total paid amount

Unknown or irrelevant events still return 200 when safe to ignore.

Generic note:

- Different providers use different event names and nested payload keys.
- Keep extraction logic centralized so route handlers stay stable if payload
  shape changes.
