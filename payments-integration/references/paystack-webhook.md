# Paystack Webhook (Idempotent Credit)

Route: `POST /api/webhooks/paystack`

Responsibilities (in order):
1. **Verify HMAC SHA-512 signature** with `PAYSTACK_SECRET_KEY` against raw
   request body and the `x-paystack-signature` header.
2. Parse JSON. Persist a `WebhookEvent` row immediately for observability.
3. For `event === "charge.success"`:
   - Atomically claim the pending transaction by `providerReference`.
   - Re-verify with `verifyPayment(reference)` (authoritative amount + fees).
   - Compute `netAmount = (amount - fees) / 100` (kobo → NGN).
   - Credit the destination wallet, mark transaction `confirmed`.
4. Always respond 200 once we've safely persisted intent — return 4xx only on
   genuine auth/parse failures so Paystack retries appropriately.

## Signature Verification

```ts
import crypto from "crypto";

const secret = process.env.PAYSTACK_SECRET_KEY!;
const signature = request.headers.get("x-paystack-signature");
const body = await request.text(); // RAW body — do not JSON.parse first

const hash = crypto.createHmac("sha512", secret).update(body).digest("hex");
if (hash !== signature) {
  return NextResponse.json({ error: "Invalid signature" }, { status: 401 });
}
```

> Read the body as text *once*, then `JSON.parse(body)`. Re-reading or using
> `request.json()` first will break the HMAC (whitespace differences).

## Atomic Claim (prevents duplicate processing)

```ts
const transaction = await WalletTransaction.findOneAndUpdate(
  { providerReference: reference, status: "pending" },
  { $set: { status: "processing" } },
  { new: true }
);

if (!transaction) {
  // Already processed (or being processed by another instance)
  const existing = await WalletTransaction.findOne({
    providerReference: reference,
    status: { $in: ["confirmed", "processing"] },
  });
  if (existing) return { success: true, message: "Already processed" };
  return { success: false, message: "Transaction not found" };
}
```

## Authoritative Amount via verifyPayment

```ts
const verification = await verifyPayment(reference);
if (!verification.status || verification.data.status !== "success") {
  transaction.status = "pending"; // revert so it can retry
  await transaction.save();
  return { success: false, message: "Payment verification failed" };
}

const amount    = verification.data.amount / 100;       // gross NGN
const fee       = (verification.data.fees || 0) / 100;
const netAmount = amount - fee;
```

## Credit Wallet + Confirm Transaction

```ts
const wallet = await Wallet.findById(transaction.walletId || transaction.to?.walletId);
if (!wallet) {
  transaction.status = "failed";
  await transaction.save();
  return { success: false, message: "Wallet not found" };
}

wallet.balance += netAmount;
await wallet.save();

transaction.fee = fee;
transaction.amount = netAmount;
transaction.originalAmount = amount;
transaction.status = "confirmed";
transaction.confirmedAt = new Date();
await transaction.save();
```

## Pay-link Branching

If `transaction.metadata?.source === "pay_link"`, route to the
deposit→transfer flow. See [pay-link-flow.md](./pay-link-flow.md).

## Full skeleton

```ts
export async function POST(request: Request) {
  const secret = process.env.PAYSTACK_SECRET_KEY;
  const signature = request.headers.get("x-paystack-signature");
  const body = await request.text();

  if (!secret || !signature) return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  const hash = crypto.createHmac("sha512", secret).update(body).digest("hex");
  if (hash !== signature) return NextResponse.json({ error: "Invalid signature" }, { status: 401 });

  let payload: any;
  try { payload = JSON.parse(body); }
  catch { return NextResponse.json({ error: "Invalid JSON" }, { status: 400 }); }

  if (payload.event === "charge.success") {
    const ref = payload.data?.reference;
    if (!ref) return NextResponse.json({ error: "Missing reference" }, { status: 400 });
    const result = await processWebhook(ref);
    if (!result.success) return NextResponse.json({ error: result.message }, { status: 400 });
  }

  return NextResponse.json({ received: true });
}
```

## Observability

Persist every webhook in a `WebhookEvent` collection with provider, type,
payload, headers (fingerprinted, never raw secrets), status transitions,
linked transaction id. This is how you debug "why didn't this credit?".
