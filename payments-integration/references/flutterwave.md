# Flutterwave Integration (Hosted Checkout + Bills + Webhook)

This document covers the Flutterwave integration as implemented in this repository. Flutterwave is used for two distinct purposes:

1. **Hosted Checkout (Standard)** — wallet top-ups, rental deposits, extension deposits, and provider-rail VTU purchases.
2. **Bills API** — the actual airtime/data purchase execution for VTU flows.

Both use the same `FLUTTERWAVE_V3_PRIVATE_KEY`. The `FlutterwaveService` class in `apps/api/src/services/payment/flutterwave.service.ts` handles all calls.

---

## Service Methods

### `createHostedCheckout(input)` → `{ link: string }`

Calls `POST https://api.flutterwave.com/v3/payments`. Returns a hosted payment link to redirect the user to.

```ts
const result = await flutterwaveService.createHostedCheckout({
  txRef: "WALLET-M0KIH441-B6B0A46992AA",  // your unique reference
  amount: 5000,                             // in primary units (NGN), not kobo
  currency: "NGN",
  redirectUrl: "https://app.example.com/wallet",  // user lands here after payment
  customer: {
    email: "user@example.com",
    name: "John Doe",
    phonenumber: "+2348012345678",
  },
  description: "Wallet top-up",
  title: "10Digit Payment",
  meta: { kind: "wallet_topup", userId: "abc123" },
});
// result.link = "https://checkout.flutterwave.com/v3/hosted/pay/..."
```

The `meta` object is passed through to Flutterwave and echoed back in webhook payloads. Store enough context to look up the Transaction by reference on webhook receipt.

### `verifyTransaction(transactionId)` → `FlutterwaveVerifyTransactionResult`

Calls `GET /transactions/{id}/verify`. `transactionId` is the **numeric** ID Flutterwave appends to the redirect URL as `?transaction_id=`.

```ts
const verified = await flutterwaveService.verifyTransaction(transactionId);
// verified.status    — "successful" | "failed" | "pending"
// verified.txRef     — matches your tx_ref / reference
// verified.amount    — actual charged amount
// verified.currency  — "NGN"
```

Always verify with this endpoint before crediting wallets or executing bills. Do not trust the redirect URL's `?status=` param.

### `createBillPayment(billerCode, itemCode, input)` → `FlutterwaveCreateBillResult`

Calls `POST /billers/{billerCode}/items/{itemCode}/payment`. Used for VTU airtime/data execution.

```ts
const bill = await flutterwaveService.createBillPayment("BIL099", "AT099", {
  country: "NG",
  customer: "+2348012345678",
  amount: 100,
  recurrence: "ONCE",
  type: "AIRTIME",
  reference: "VTU-M0KIH441-B6B0A46992AA",
});
```

### `verifyWebhookHash(body, incomingHash)` → `boolean`

Verifies the HMAC-SHA256 signature from the `verif-hash` header. `body` must be the parsed JSON object (not re-stringified from raw text).

```ts
const valid = flutterwaveService.verifyWebhookHash(req.body, req.headers["verif-hash"]);
if (!valid) return res.status(400).json({ message: "Invalid signature" });
```

---

## Redirect URL and Callback Params

When the user finishes payment, Flutterwave redirects to your `redirectUrl` with these query params:

| Param | Type | Description |
|---|---|---|
| `tx_ref` | string | Your reference (matches `txRef` you passed at init) |
| `transaction_id` | number | Flutterwave's numeric ID — required for `verifyTransaction()` |
| `status` | string | `"completed"` or `"cancelled"` — **treat as signal only** |

Frontend extraction:

```ts
// In callback page (e.g. /wallet or /top-up)
const txRef = searchParams.get("tx_ref") || "";
const transactionId = Number(searchParams.get("transaction_id") || 0);

// Clean up URL params after reading
router.replace(pathname, { scroll: false }); // remove tx_ref, transaction_id, status
```

**Critical**: `transaction_id` must be captured for the verify call. Without it, `verifyTransaction()` cannot be called and settlement will not happen.

---

## Initiate Route Pattern (Hosted Checkout)

Flutterwave is dispatched by the shared `initiateProviderCheckout()` function. For any provider checkout initiate route:

```ts
const reference = createPaymentReference("WALLET");
const callbackUrl = buildPaymentCallbackUrl("/wallet");

const checkoutInit = await initiateProviderCheckout({
  provider: "flutterwave",   // or "100pay" | "paystack"
  reference,
  amount: 5000,              // NGN units, not kobo
  currency: "NGN",
  country: "NG",
  description: "Wallet top-up",
  callbackUrl,
  customer: {
    userId: user._id.toString(),
    email: user.email,
    name: getCustomerName(user),
    phone: user.phone || "",
  },
  metadata: { kind: "wallet_topup", userId: user._id.toString() },
});

// checkoutInit.hostedUrl = Flutterwave payment link
// checkoutInit.checkout = null  (only non-null for 100Pay)
```

The pending Transaction stores `meta.provider = "flutterwave"` for all webhook and verify lookups.

**Important**: Flutterwave only supports NGN for hosted checkout in this integration. The initiate route should validate that the requested currency is NGN and return a 400 otherwise.

---

## Verify Route Pattern

### Wallet top-up verify

`POST /v1/payments/wallet-topups/verify` accepts `{ reference, transactionId }` and is provider-aware:

```ts
// When meta.provider === "flutterwave":
const verified = await flutterwaveService.verifyTransaction(transactionId);
// if verified.status === "successful": run settleSuccessfulFlutterwaveWalletTopupTransaction()
```

`settleSuccessfulFlutterwaveWalletTopupTransaction()` handles currency conversion (via `currencyService.getUnifiedRate()`), credits the wallet ledger, and marks the Transaction as successful with all settlement metadata stored in `meta.*`.

### VTU verify

`POST /v1/payments/vtu/verify` accepts `{ reference, transactionId }`:

1. Find Transaction by `reference` + `userId` + `meta.kind === "vtu_purchase"` + `meta.provider === "flutterwave"`.
2. Call `flutterwaveService.verifyTransaction(transactionId)`.
3. If `verified.status !== "successful"` → return 402.
4. If Transaction is pending: update to `status: "successful"`, then call `settleVtuBillFromTransaction(tx)`.
5. `settleVtuBillFromTransaction` calls the Flutterwave Bills API to execute the actual airtime/data purchase and sets `meta.billCreated = true`.
6. Return `{ reference, status, billCreated }`.

---

## Webhook Handler

Route: `POST /v1/payments/webhooks/flutterwave`

The handler uses `req.body` as parsed JSON (not raw text) because Flutterwave's HMAC is computed over the stringified JSON body.

```ts
// 1. Verify hash
const incomingHash = req.headers["verif-hash"];
if (!incomingHash || !flutterwaveService.verifyWebhookHash(req.body, incomingHash)) {
  return res.status(400).json({ message: "Invalid webhook signature" });
}

const event = String(req.body?.event || "").toLowerCase();
const data  = req.body?.data;
```

### Event: `charge.completed` (hosted checkout settlement)

Triggered when a hosted checkout payment is completed.

```ts
if (event === "charge.completed") {
  const txRef = String(data.tx_ref || "").trim();

  // 1. Find Transaction by reference + provider
  const chargeTx = await Transaction.findOne({
    reference: txRef,
    "meta.provider": "flutterwave",
  });

  // 2. Guard: only pending transactions
  if (!chargeTx || chargeTx.status !== "pending") {
    return res.status(200).json({ received: true });
  }

  // 3. Deduplicate
  const eventId = String(data.id || txRef);
  if (chargeTx.meta.processedPaymentEventIds?.includes(eventId)) {
    return res.status(200).json({ received: true });
  }

  // 4. Non-successful charge
  const chargeStatus = String(data.status || "").toLowerCase();
  if (chargeStatus !== "successful") {
    await Transaction.updateOne({ _id: chargeTx._id, status: "pending" }, {
      $set: { status: "failed", "meta.flutterwaveWebhookStatus": chargeStatus },
      $push: { "meta.processedPaymentEventIds": eventId },
    });
    return res.status(200).json({ received: true });
  }

  // 5. Settle by meta.kind
  const kind = chargeTx.meta.kind;
  const flwTransactionId = Number(data.id || 0);

  if (kind === "wallet_topup") {
    const verified = await flutterwaveService.verifyTransaction(flwTransactionId);
    await settleSuccessfulFlutterwaveWalletTopupTransaction({
      transactionDoc: chargeTx, reference: txRef, transactionId: flwTransactionId,
      providerStatus: verified.status, providerCurrency: verified.currency,
      chargedAmount: verified.chargedAmount,
    });
  } else {
    // rental_deposit or rental_extension_deposit
    // Mark successful; rental activation handled by polling
    await Transaction.updateOne({ _id: chargeTx._id, status: "pending" }, {
      $set: {
        status: "successful",
        "meta.flutterwaveTransactionId": flwTransactionId,
        "meta.paymentStatus": "successful",
      },
      $push: { "meta.processedPaymentEventIds": eventId },
    });
  }
}
```

### Event: bill payment events (VTU via Bills API)

For `event` strings containing `"bill"` (e.g. `"bill.payment.completed"`):

```ts
const txRef = String(data.tx_ref || data.customer_reference || "").trim();
const tx = await Transaction.findOne({ reference: txRef, "meta.kind": "vtu_purchase" });

// Deduplicate, check status
// On success: set status: "successful", meta.billCreated: true
// On failure: set status: "failed", then refund wallet if meta.paymentRail === "wallet"
```

---

## Transaction Metadata Fields

All Flutterwave Transactions store these fields in `meta`:

| Field | Description |
|---|---|
| `meta.provider` | `"flutterwave"` |
| `meta.kind` | `"wallet_topup"` \| `"number_rental_deposit"` \| `"rental_extension_deposit"` \| `"vtu_purchase"` |
| `meta.paymentRail` | `"flutterwave"` (or `"wallet"` for wallet-rail VTU) |
| `meta.flutterwaveTransactionId` | Numeric Flutterwave transaction ID (set after verify) |
| `meta.flutterwave_tx_ref` | The `tx_ref` echoed from Flutterwave |
| `meta.flutterwaveStatus` | `"successful"` \| `"failed"` (from verify) |
| `meta.flutterwaveWebhookStatus` | Raw status string from webhook payload |
| `meta.billCreated` | `true` once VTU bill was executed (VTU flows only) |
| `meta.processedPaymentEventIds` | Array of deduplicated event IDs |
| `meta.resolvedAmountCredited` | Final NGN amount credited (wallet topup) |
| `meta.providerCurrency` | Currency from Flutterwave verify result |

---

## Key Differences from Paystack and 100Pay

| Concern | Flutterwave | Paystack | 100Pay |
|---|---|---|---|
| Amount units | NGN units (not kobo) | Kobo (`amount * 100`) | NGN units |
| Redirect param | `tx_ref` | `reference` / `trxref` | postMessage callback |
| Verify ID | numeric `transaction_id` | string `reference` | `transactionId` |
| Webhook header | `verif-hash` (HMAC-SHA256) | `x-paystack-signature` (HMAC-SHA512) | `verification-token` (shared secret) |
| Webhook hash input | parsed JSON body (stringified) | raw request body (text) | plain token comparison |
| VTU execution | Bills API (`/billers/.../payment`) | N/A | N/A |
| Client-side SDK | None (redirect only) | None (redirect only) | `@100pay-hq/checkout` modal |

---

## Pitfalls

1. **Missing `transaction_id` on callback** — `tx_ref` alone is not enough to verify. Always capture `transaction_id` from the redirect URL.
2. **Wrong webhook hash input** — Paystack uses the raw body as text; Flutterwave uses `JSON.stringify(parsedBody)`. Do not swap them.
3. **`status=completed` on redirect is not authoritative** — always call `verifyTransaction()` before settlement.
4. **VTU needs two steps** — Flutterwave payment confirms the user paid, but `settleVtuBillFromTransaction()` must also run to actually purchase airtime/data via the Bills API.
5. **Currency** — only NGN is supported for hosted checkout in this implementation. Add a guard in the initiate route.
