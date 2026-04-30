---
name: payments-integration
description: 'Implement deposits, checkout initialization, callback verification, and webhook settlement using 100Pay, Paystack, Flutterwave, and multiwallet in Next.js + Express apps. Use when building wallet top-ups, number rental deposits, rental extension deposits, VTU purchases, subscription provider checkout, or webhook processing. Covers server-authoritative checkout payloads, idempotent pending transaction lifecycle, callback-as-signal patterns, provider-agnostic dispatch, and settlement race handling.'
---

# Payments Integration (100Pay + Paystack + Flutterwave + Multiwallet)

This skill documents production payment patterns used in a monorepo (Express API + Next.js web app). All four providers share one dispatch helper and one pending-transaction lifecycle. Examples are grounded in real code but use sanitized values.

## Open-Source Safety

1. Never publish real API keys, webhook secrets, or signed payload samples.
2. Use placeholder domains and values in examples (`example.com`, `pk_xxx`, `sk_xxx`).
3. Do not include real user identifiers, phone numbers, or email addresses.
4. Treat callback and webhook payload logs as sensitive — redact before sharing.

## Repository Example Files

- `apps/api/src/routes/payments.ts` — all payment routes
- `apps/api/src/services/payment/hundredpay.service.ts`
- `apps/api/src/services/payment/paystack.service.ts`
- `apps/api/src/services/payment/flutterwave.service.ts`
- `apps/web/utils/payWith100Pay.ts`
- `apps/web/app/(app)/wallet/page-client.tsx`
- `apps/web/app/(app)/top-up/page-client.tsx`
- `apps/web/components/shared/payment-flow/payment-method-selector.tsx`
- `apps/web/components/marketplace/number-purchase-flow.tsx`
- `apps/web/components/marketplace/extend-rental-dialog-button.tsx`

## When to Use

- Add or debug wallet top-up checkout (any provider)
- Add or debug number rental deposit or extension checkout
- Add or debug VTU airtime/data purchases via provider checkout
- Handle callback redirect pages and client-side polling confirmation
- Build resilient webhook settlement (idempotency, retries, race handling)
- Add a new payment provider to an existing multi-provider system
- Reuse this architecture in a different project

---

## Provider Comparison

| Capability | 100Pay | Paystack | Flutterwave | Wallet |
|---|---|---|---|---|
| Hosted URL | Yes | Yes | Yes | No |
| Frontend SDK modal | Yes (`@100pay-hq/checkout`) | No | No | No |
| Redirect flow | No | Yes (new tab) | Yes (new tab) | No |
| Callback params | `transactionId`, `reference` | `reference`, `trxref` | `tx_ref`, `transaction_id`, `status` | N/A |
| Webhook event | `type: "credit"` | `event: "charge.success"` | `event: "charge.completed"` | N/A |
| Webhook auth | `verification-token` header (shared secret) | `x-paystack-signature` HMAC-SHA512 | `verif-hash` HMAC-SHA256 | N/A |
| Supported currencies | Crypto + NGN | NGN | NGN (extendable) | Any wallet symbol |
| Verify endpoint | `GET /charges/verify/:id` | `GET /transaction/verify/:ref` | `GET /transactions/:id/verify` | N/A |

---

## Mental Model: Staged Payment Lifecycle

Payment success is not a single event. Every provider flow follows the same four-stage shape:

```
1. INITIATE  — server creates pending Transaction + reference, calls provider API
2. CHECKOUT  — client opens hosted URL (new tab) or SDK modal
3. CALLBACK  — provider redirects user back; treat as signal only, not settlement
4. SETTLE    — server verifies with provider, applies domain side effects exactly once
               (via webhook OR client-triggered verify call)
```

Wallet payments skip stages 2–3 entirely (immediate debit).

---

## Provider-Agnostic Dispatch (Key Pattern)

This codebase uses a single `initiateProviderCheckout()` function that all initiate routes call regardless of provider. This keeps route logic clean and provider differences isolated:

```ts
type CheckoutProvider = "100pay" | "paystack" | "flutterwave";

async function initiateProviderCheckout(params: {
  provider: CheckoutProvider;
  reference: string;
  amount: number;          // in primary currency units (e.g. NGN, not kobo)
  currency: string;        // ISO 3-letter
  country: string;         // ISO 2-letter
  description: string;
  callbackUrl: string;     // where provider redirects user after payment
  customer: { userId: string; email: string; name: string; phone: string; };
  metadata: Record<string, unknown>;  // stored in Transaction.meta, echoed to provider
}): Promise<{
  hostedUrl: string;       // URL to open (in new tab or SDK modal)
  checkout: CheckoutPayload | null;  // non-null only for 100Pay (SDK needs it)
  providerChargeId?: string;
}>
```

- **100Pay**: calls `hundredPayService.createCharge()`, returns `hostedUrl` + full `checkout` payload for the client SDK.
- **Paystack**: calls `paystackService.initializePayment()` with amount in kobo (`amount * 100`), returns `authorizationUrl` as `hostedUrl`, `checkout: null`.
- **Flutterwave**: calls `flutterwaveService.createHostedCheckout()`, returns `link` as `hostedUrl`, `checkout: null`.

Reference helper:

```ts
function createPaymentReference(prefix: string) {
  const compact = Date.now().toString(36).toUpperCase();
  const random = crypto.randomUUID().replace(/-/g, "").slice(0, 12).toUpperCase();
  return `${prefix}-${compact}-${random}`;
  // e.g. WALLET-M0KIH441-B6B0A46992AA
}
```

Callback URL helper:

```ts
function buildPaymentCallbackUrl(returnPath: string): string {
  return new URL(returnPath, process.env.FRONTEND_URL).toString();
  // e.g. https://app.example.com/wallet
}
```

---

## Golden Rules

1. **Callback is a signal, not settlement.** Never credit wallet or activate rental on callback alone.
2. **Always use server-authoritative customer identity.** Fetch user from DB in the initiate route; do not trust client-supplied email/name.
3. **Reuse pending checkouts** for 30-minute windows before creating a new one (idempotency key indexed).
4. **One durable reference per transaction.** All polling and webhook lookup uses this reference.
5. **Deduplicate webhook events** using `meta.processedPaymentEventIds` array + transaction status gate.
6. **Release side effects on failure.** Rental deposit failure releases number reservation lock.
7. **Verify before settling.** Always call the provider's verify API before applying domain effects.
8. **Flutterwave needs `transactionId` (numeric).** Paystack needs `reference`. 100Pay needs `transactionId`. Capture the correct param from the redirect URL.

---

## Route Surface (This Repository)

### Wallet Top-Up

- `POST /v1/payments/wallet-topups/initiate` — providers: `100pay | paystack | flutterwave`
- `POST /v1/payments/wallet-topups/verify` — accepts `transactionId` + `reference`; provider-aware

### Number Rental Deposit

- `POST /v1/payments/rental-deposits/initiate` — providers: `100pay | paystack | flutterwave`
- `GET /v1/payments/rental-deposits/pending`
- `GET /v1/payments/rental-deposits/by-reference`

### Rental Extension Deposit

- `POST /v1/payments/rental-extension-deposits/initiate` — providers: `100pay | paystack | flutterwave`
- `GET /v1/payments/rental-extension-deposits/by-reference`

### VTU (Airtime / Data)

- `POST /v1/payments/vtu/airtime/initiate` — rails: `wallet | 100pay | paystack | flutterwave`
- `POST /v1/payments/vtu/data/initiate` — same rails
- `POST /v1/payments/vtu/verify` — Flutterwave verify + bill settlement trigger
- `GET /v1/payments/vtu/by-reference` — polls transaction + triggers bill settlement if needed
- `GET /v1/payments/vtu/pending` — returns resumable pending provider checkout

### Subscriptions

- `POST /v1/payments/subscriptions/initiate` — provider: `100pay` only (currently)

### Webhooks

- `POST /v1/payments/webhooks/100pay`
- `POST /v1/payments/webhooks/paystack`
- `POST /v1/payments/webhooks/flutterwave` — handles both `charge.completed` (hosted checkout) and bill payment events

### Misc

- `GET /v1/payments/initiated` — list pending initiated payments for current user
- `GET /v1/payments/callback` — redirect relay page

---

## Initiate Route Pattern

Every initiate route follows this structure:

```ts
// 1. Auth + validate body
// 2. Idempotency: check for existing pending tx with same idempotencyKey
//    - If found and age < 30 min: return resumed: true + existing hostedUrl
//    - If stale: mark failed, release any locks, continue
// 3. Resolve pricing, currency conversion, wallet
// 4. Build server-authoritative customer object from DB user
// 5. Create reference: createPaymentReference("PREFIX")
// 6. Build callbackUrl: buildPaymentCallbackUrl("/return-path")
// 7. Call initiateProviderCheckout({ provider, reference, amount, ... })
// 8. Create pending Transaction with all metadata
// 9. Return { reference, hostedUrl, resumed: false, checkout }
//    - checkout is non-null only for 100Pay (client SDK needs it)
//    - checkout is null for Paystack and Flutterwave (redirect only)
```

---

## Client-Side Checkout Dispatch

The frontend checks the provider and opens checkout accordingly:

```ts
// 100Pay: open SDK modal
if (provider === "100pay") {
  await payWith100Pay({ ref_id, apiKey, billing, customer, metadata },
    onClose, onError, (reference) => verifyTopUp({ transactionId, reference })
  );
}

// Paystack or Flutterwave: open hosted URL in new tab
if (provider === "paystack" || provider === "flutterwave") {
  window.open(result.hostedUrl, "_blank", "noopener,noreferrer");
  // Show toast; settlement happens via webhook or callback page polling
}

// Wallet: immediate — no redirect, show result inline
```

---

## Callback URL Handling

Each provider redirects to the `callbackUrl` passed at initiate time with different query params:

| Provider | Redirect params |
|---|---|
| 100Pay | `transactionId`, `reference` (via postMessage from iframe) |
| Paystack | `?reference=REF&trxref=REF` |
| Flutterwave | `?tx_ref=REF&transaction_id=NUM&status=completed` |

**Reference extraction** (frontend):

```ts
const reference = params.reference || params.trxref || params.tx_ref || "";
const transactionId = params.transaction_id || params.transactionId || params.id || "";
```

**Flutterwave-specific**: `transaction_id` is a numeric ID required for the verify API call. Must be captured alongside `tx_ref`.

**Callback page flow**:

1. Extract `reference` (+ `transactionId` for Flutterwave).
2. If Flutterwave: call `POST /vtu/verify` or the relevant verify endpoint with `{ reference, transactionId }` to trigger settlement.
3. Poll `by-reference` or `verify` until status is terminal (`successful`, `failed`, `reversed`).
4. Show success/failure UX.
5. Clean up callback params from URL (`router.replace`).

---

## Verify Route Pattern

Verify routes exist for flows where the client can actively trigger settlement (instead of waiting for a webhook):

```ts
// POST /v1/payments/wallet-topups/verify
// Accepts: { transactionId, reference }
// Provider-aware:
//   - 100Pay: calls hundredPayService.verifyTransaction(transactionId)
//   - Paystack: calls paystackService.verifyPayment(reference)
//   - Flutterwave: calls flutterwaveService.verifyTransaction(transactionId)
// On success: runs settlement function, credits wallet
// Returns: { amountCredited, symbol, status }

// POST /v1/payments/vtu/verify  (Flutterwave VTU only)
// Accepts: { reference, transactionId }
// Marks pending TX as successful, runs settleVtuBillFromTransaction()
// Returns: { reference, status, billCreated }
```

---

## Webhook Settlement Pattern

All webhook handlers follow this shape:

```ts
// 1. Verify auth signature/token — return 400 on failure
// 2. Extract event type and data
// 3. Route by event type:
//    - charge.completed / charge.success / type: "credit"
// 4. Find pending Transaction by reference/tx_ref
// 5. Deduplicate: check meta.processedPaymentEventIds — return 200 if already processed
// 6. Check status: if not "pending", return 200
// 7. Verify payment with provider API (authoritative amount)
// 8. Apply terminal transition + domain side effects atomically
// 9. Push event ID to processedPaymentEventIds
// 10. Return 200 always (after auth check) — 4xx causes retries
```

### Domain Side Effects by `meta.kind`

| `meta.kind` | Success effect | Failure effect |
|---|---|---|
| `wallet_topup` | Credit wallet, resolve currency conversion | Mark failed |
| `number_rental_deposit` | Create rental record, activate number | Mark failed, release number reservation lock |
| `rental_extension_deposit` | Apply extension to existing rental | Mark failed |
| `vtu_purchase` | Mark successful (bill created by separate settle fn) | Mark failed, refund wallet debit if wallet rail |

---

## Wallet (Multiwallet) Payment Rail

For flows that support direct wallet debit (VTU, subscriptions):

```ts
// rail === "wallet"
// 1. Resolve wallet by symbol (create if missing)
// 2. Convert amount if wallet currency !== NGN
// 3. computeWalletBalance() — snapshot approach, not stored balance field
// 4. Check sufficient balance
// 5. postTransaction() — debit wallet via ledger entry
// 6. Execute domain action (create bill, etc.)
// 7. On failure: postTransaction() with type: "credit" to refund
```

Key pattern — wallet balance is computed from ledger (sum of credits minus debits), not read from a `balance` field. This prevents race conditions.

Multiple wallet symbols are supported (`NGN`, `USD`, `CRDTS`, etc.). The checkout currency is resolved via `currencyService.convert()` for cross-currency flows.

---

## VTU-Specific Pattern

VTU (airtime/data) has two distinct payment rails:

**Provider rail** (`100pay | paystack | flutterwave`):

1. Initiate creates a pending Transaction (not a bill yet).
2. Client redirects to provider checkout.
3. On callback: call `POST /vtu/verify` with `{ reference, transactionId }`.
4. Verify route: confirms payment with Flutterwave, marks TX successful, calls `settleVtuBillFromTransaction()`.
5. `settleVtuBillFromTransaction()` calls Flutterwave Bills API to execute the actual airtime/data purchase.
6. Poll `GET /vtu/by-reference` until `billCreated: true`.

**Wallet rail**:

1. Debit wallet immediately.
2. Call Flutterwave Bills API synchronously.
3. On bill failure: refund wallet via ledger entry.
4. Return result immediately.

---

## File Map (Reference Modules)

| Topic | File |
|---|---|
| 100Pay client launcher and callback behavior | [hundredpay-client.md](./references/hundredpay-client.md) |
| 100Pay initiate/verify server flows | [hundredpay-deposit-route.md](./references/hundredpay-deposit-route.md) |
| 100Pay webhook settlement | [hundredpay-webhook.md](./references/hundredpay-webhook.md) |
| Paystack server wrapper | [paystack-server.md](./references/paystack-server.md) |
| Paystack deposit route | [paystack-deposit-route.md](./references/paystack-deposit-route.md) |
| Paystack webhook | [paystack-webhook.md](./references/paystack-webhook.md) |
| Flutterwave hosted checkout + verify + webhook | [flutterwave.md](./references/flutterwave.md) |
| Multiwallet and wallet rail | [multiwallet.md](./references/multiwallet.md) |
| Environment variables and setup | [env-and-setup.md](./references/env-and-setup.md) |
| Data model | [data-model.md](./references/data-model.md) |
| Pitfalls and race conditions | [pitfalls.md](./references/pitfalls.md) |
| Payout flow | [hundredpay-payout.md](./references/hundredpay-payout.md) |
| Pay-link flow | [pay-link-flow.md](./references/pay-link-flow.md) |

---

## End-to-End Checklist (Any Provider)

1. Env vars configured for all providers you intend to use.
2. Initiate route: create reference, call `initiateProviderCheckout()`, persist pending TX with full metadata, return `{ reference, hostedUrl, resumed, checkout }`.
3. Frontend: dispatch to SDK modal (100Pay) or open new tab (Paystack/Flutterwave).
4. Callback page: extract correct params per provider, call verify endpoint if needed, poll until terminal state.
5. Verify route: call provider verify API, apply settlement, return result.
6. Webhook handler: verify signature, deduplicate, find TX by reference, settle idempotently.
7. By-reference endpoints: reflect DB state + trigger lazy settlement if TX is successful but effects not yet applied.
8. Failure paths: release locks, refund wallet debits, mark TX failed.
9. All labels in UI match provider name from `meta.provider` / `meta.paymentRail`.

## Debugging Procedure

1. Find Transaction by `reference` in DB. Check `status`, `meta.kind`, `meta.provider`, `meta.billCreated`, `meta.processedPaymentEventIds`.
2. Check webhook logs — confirm auth passed and event was received.
3. Verify callback page extracted the correct params (right field per provider).
4. Confirm verify endpoint was called with both `reference` and `transactionId` for Flutterwave.
5. Check `settleVtuBillFromTransaction` was called and `billCreated` was set.
6. For rental flows: confirm number reservation lock was released on failure.
