# 100Pay Webhook Settlement (Current Implementation)

Primary route:

- `POST /v1/payments/webhooks/100pay`

Implemented in:

- `apps/api/src/routes/payments.ts`

Auth helper:

- `apps/api/src/services/payment/hundredpay.service.ts` (`verifyWebhookToken`)

## Request Authentication

The webhook expects header:

- `verification-token`

Validation uses timing-safe comparison against:

- `HUNDREDPAY_WEBHOOK_SECRET`

If token is missing/invalid, route returns 400.

## Event Filtering

The route normalizes incoming payload and exits early unless:

- `type === "credit"`
- charge reference (`ref_id`/reference) is present

References are resolved from payload fields such as:

- `payload.data.charge.ref_id`
- `payload.reference`
- `payload.data.reference`

## Dedupe and Idempotency

The handler protects against duplicate webhook deliveries by using:

- `meta.processedPaymentEventIds` (payment event id set membership)
- transaction status checks (`pending` / `successful` only for relevant branches)

If an event was already processed, it returns 200 (`received: true`) without repeating settlement.

## Status Handling

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

## Settlement Branches

### A) Wallet top-up (`meta.kind = wallet_topup`)

- Non-success: updates payment status metadata and exits.
- Success: calls `settleSuccessfulHundredPayCreditTransaction(...)`.
- This function credits wallet once and updates resolved metadata fields.

### B) Number rental deposit (`meta.kind = number_rental_deposit`)

- On failure states while pending:
  - marks transaction failed
  - records cancel reason
  - releases reserved number lock (`releaseRentalDepositReservation`)
- On success:
  - marks transaction successful if pending
  - triggers rental creation (`createRentalFromDepositPayment`)
  - stores `meta.rentalCreated` and `meta.rentalId`

### C) Other 100Pay credit flows

For non-rental-deposit credit flows (including top-up and similar provider credits),
settlement resolves through `settleSuccessfulHundredPayCreditTransaction(...)`.

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
