# Env Vars and Setup

This document lists required configuration categories for any integration,
then provides exact variable names used in this repository.

All values in this document are placeholders. Do not copy example secrets into
source control.

## Publish-Safe Rules

- Commit only variable names to `.env.example`, never real values.
- Keep provider secret keys and webhook secrets server-side only.
- Only `NEXT_PUBLIC_*` values should be readable by client code.
- Redact secrets from logs, screenshots, and issue reports.

## Generic Configuration Categories

Server:

- Provider public key (if required by server library)
- Provider secret key
- Webhook signing secret/token
- Provider request timeout/retry tuning
- App frontend/base URL
- App API/base URL

Client:

- Provider public key for browser SDK
- API base URL
- Optional callback URL override (only if SDK requires a browser callback URL)

## Repository Variables

## Required Environment Variables

Server-side (API):

```env
# 100Pay
HUNDREDPAY_API_KEY=pk_xxx
HUNDREDPAY_SECRET_API_KEY=sk_xxx
HUNDREDPAY_WEBHOOK_SECRET=shared_verification_token
HUNDREDPAY_REQUEST_TIMEOUT_MS=15000

# Paystack
PAYSTACK_SECRET_KEY=sk_xxx
PAYSTACK_WEBHOOK_SECRET=whsec_xxx

# Flutterwave
FLUTTERWAVE_V3_PRIVATE_KEY=FLWSECK_xxx
FLUTTERWAVE_V3_WEBHOOK_SECRET=your_flw_webhook_secret

# App routing
FRONTEND_URL=https://app.example.com
API_URL=https://api.example.com
```

Client-side (web):

```env
NEXT_PUBLIC_HUNDREDPAY_API_KEY=pk_xxx
NEXT_PUBLIC_100PAY_CALLBACK_URL=https://app.example.com/api/webhooks/100pay
NEXT_PUBLIC_API_URL=https://api.example.com
```

Notes:

- `NEXT_PUBLIC_HUNDREDPAY_API_KEY` is used by `@100pay-hq/checkout` on the web.
- `HUNDREDPAY_SECRET_API_KEY` is used for server-side verification calls.
- `HUNDREDPAY_WEBHOOK_SECRET` must match provider webhook token.
- `FLUTTERWAVE_V3_PRIVATE_KEY` is used for both Bills API (VTU) and Hosted Checkout.
- `FLUTTERWAVE_V3_WEBHOOK_SECRET` is used to verify Flutterwave webhooks via HMAC-SHA256.
- `API_URL` and `FRONTEND_URL` should point to your deployment domains.
- Flutterwave does not require any public/client-side key — all calls are server-side.

## Route Wiring (Repository)

API payment routes are mounted under:

- `/v1/payments` (see `apps/api/src/routes/index.ts`)

Web app rewrites API calls:

- `/api/:path*` -> `${NEXT_PUBLIC_API_URL}/:path*` (see `apps/web/next.config.ts`)

This allows frontend code to call `/api/v1/payments/...` while backend lives on a separate origin.

## 100Pay Endpoints to Configure (Repository Example)

Webhook URL in 100Pay dashboard:

- `https://api.example.com/v1/payments/webhooks/100pay`

Provider callback relay URL (server-generated via `buildPaymentCallbackUrl`):

- `https://api.example.com/v1/payments/callback?returnPath=...`

Important note:

- In this repository, checkout calls currently set `call_back_url` in the web
 helper from `NEXT_PUBLIC_100PAY_CALLBACK_URL` (fallback
 `/api/webhooks/100pay`). For portability and correctness, prefer using a
 server-issued callback URL aligned with your callback relay endpoint.

## Packages in Use

API:

- `@100pay-hq/100pay.js`

Web:

- `@100pay-hq/checkout`

## Setup Checklist

- [ ] API env contains `HUNDREDPAY_API_KEY`, `HUNDREDPAY_SECRET_API_KEY`, `HUNDREDPAY_WEBHOOK_SECRET`
- [ ] Web env contains `NEXT_PUBLIC_HUNDREDPAY_API_KEY`
- [ ] Provider webhook points to `/v1/payments/webhooks/100pay`
- [ ] Payments router is mounted at `/v1/payments`
- [ ] Frontend rewrite `/api/:path*` is enabled
- [ ] Callback pages exist for wallet/marketplace/rentals flows
- [ ] Transactions persist provider metadata (`kind`, `provider`, `hostedUrl`, `checkoutIdempotencyKey`)
