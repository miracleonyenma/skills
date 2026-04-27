# Env Vars and Setup (Current 100Pay Implementation)

## Required Environment Variables

Server-side (API):

```env
# 100Pay
HUNDREDPAY_API_KEY=pk_xxx
HUNDREDPAY_SECRET_API_KEY=sk_xxx
HUNDREDPAY_WEBHOOK_SECRET=shared_verification_token
HUNDREDPAY_REQUEST_TIMEOUT_MS=15000

# Paystack (used in same payments router)
PAYSTACK_SECRET_KEY=sk_xxx
PAYSTACK_WEBHOOK_SECRET=whsec_xxx

# App routing
FRONTEND_URL=https://app.example.com
NEXT_PUBLIC_API_URL=https://api.example.com
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

## Route Wiring

API payment routes are mounted under:

- `/v1/payments` (see `apps/api/src/routes/index.ts`)

Web app rewrites API calls:

- `/api/:path*` -> `${NEXT_PUBLIC_API_URL}/:path*` (see `apps/web/next.config.ts`)

This allows frontend code to call `/api/v1/payments/...` while backend lives on a separate origin.

## 100Pay Endpoints to Configure

Webhook URL in 100Pay dashboard:

- `https://api.example.com/v1/payments/webhooks/100pay`

Provider callback relay URL (server-generated via `buildPaymentCallbackUrl`):

- `https://api.example.com/v1/payments/callback?returnPath=...`

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
