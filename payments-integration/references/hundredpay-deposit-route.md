# 100Pay Initiate + Verify Server Flows

This repository implements 100Pay in the Express payments router:

- `apps/api/src/routes/payments.ts`
- `apps/api/src/services/payment/hundredpay.service.ts`

Base path:

- `/v1/payments`

## 1) Wallet top-up initiate

Route:

- `POST /v1/payments/wallet-topups/initiate`

What it does:

1. Requires auth and validates payload.
2. Verifies transaction PIN (`verifyTransactionPin`).
3. Validates provider (`100pay` or `paystack`).
4. Resolves wallet by symbol.
5. Builds server-authoritative customer identity from DB user data.
6. Resumes recent pending 100Pay checkout (same amount/currency, age < 30 min).
7. Otherwise creates a new reference and creates 100Pay charge.
8. Persists pending `Transaction` with provider metadata and hosted URL.
9. Returns:
   - `reference`
   - `hostedUrl`
   - `resumed`
   - `checkout` payload used by client SDK launcher

Core metadata fields used later:

- `meta.kind = "wallet_topup"`
- `meta.provider = "100pay"`
- `meta.providerChargeId`
- `meta.hostedUrl`
- `meta.currency`, `meta.country`, `meta.callbackUrl`

## 2) Wallet top-up verify

Route:

- `POST /v1/payments/wallet-topups/verify`

What it does:

1. Accepts `transactionId` and/or `reference`.
2. Tries `hundredPayService.verifyTransaction(transactionId)` first.
3. Falls back to `verifyChargeByReference(reference)` if needed.
4. Extracts normalized verification payload (`extractHundredPayVerificationPayload`).
5. Verifies resolved reference belongs to current user and pending top-up transaction.
6. Rejects mismatched or missing references.
7. If provider status is still pending/not settled, updates `meta.paymentStatus` and returns 409.
8. If settled, calls `settleSuccessfulHundredPayCreditTransaction(...)`.
9. Returns credited amount/currency from resolved metadata.

Why this route exists:

- It gives client flows an explicit verification endpoint while webhook settlement races are still possible.

## 3) Number rental deposit initiate

Route:

- `POST /v1/payments/rental-deposits/initiate`

What it does:

1. Validates number, platform, user rental limits, pricing.
2. Converts rent currency to requested checkout currency.
3. Creates reservation lock on number (`rental-deposit-lock:<idempotencyKey>`).
4. Builds 100Pay charge payload with metadata:
   - `kind = "number_rental_deposit"`
   - `numberId`, `platformId`, `idempotencyKey`, `userId`, `provider`
5. Creates pending transaction with `meta.kind = number_rental_deposit`.
6. Returns `checkout` payload for client SDK and reference for polling.

Also supports:

- Pending transaction resume (< 30 min)
- Stale pending expiry + reservation release

## 4) Rental extension deposit initiate

Route:

- `POST /v1/payments/rental-extension-deposits/initiate`

What it does:

1. Validates active rental and pricing.
2. Computes extension price and fiat conversion.
3. Creates 100Pay charge with metadata:
   - `kind = "rental_extension_deposit"`
   - `rentalId`, `extensionMinutes`, `idempotencyKey`, `userId`, `provider`
4. Stores pending transaction and returns checkout payload.
5. Supports resume for recent pending extension checkouts.

## 5) By-reference confirmation endpoints (used by client pollers)

Routes:

- `GET /v1/payments/rental-deposits/by-reference`
- `GET /v1/payments/rental-extension-deposits/by-reference`

These routes bridge callback -> settlement races by exposing eventual business state:

- Rental deposit success state: `rental_created`
- Extension deposit success state: `extension_applied`
- Otherwise returns transaction state (`pending`, `successful`, `failed`, `reversed`)

Client checkout callback pages poll these endpoints until terminal state.

## 6) 100Pay service wrapper

File:

- `apps/api/src/services/payment/hundredpay.service.ts`

Methods used in these flows:

- `createCharge(...)`
- `verifyTransaction(transactionId)`
- `verifyChargeByReference(reference)`
- `verifyWebhookToken(token)`

Integration details:

- Uses retry/backoff wrapper (`withProviderRetries`) and timeout (`fetchWithTimeout`).
- Uses timing-safe token comparison for webhook auth.
- Uses `@100pay-hq/100pay.js` SDK for transaction verify by transaction id.
