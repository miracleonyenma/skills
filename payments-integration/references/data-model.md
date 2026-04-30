# Data Model (Transaction + Wallet + WebhookEvent)

Mongoose, but the shape applies to any datastore. This repository uses a unified `Transaction` collection instead of a `WalletTransaction` collection.

## Transaction (unified record for all payment flows)

```ts
{
  userId: ObjectId,        // owner
  walletId: ObjectId,      // target wallet

  reference: string,       // unique per transaction (e.g. "WALLET-M0KIH441-B6B0A46992AA")
  status: "pending" | "successful" | "failed" | "reversed",
  type: "credit" | "debit",
  symbol: string,          // wallet symbol ("NGN", "CRDTS", "USD")
  amount: number,          // in symbol units

  from: { name: string, user?: string, wallet?: string },
  to:   { name: string, user?: string, wallet?: string },

  idempotencyKey: string,  // unique sparse index for preventing duplicate ledger entries
  balanceStamp: number,    // balance after this transaction (snapshot)

  meta: {
    // Common
    kind: "wallet_topup" | "number_rental_deposit" | "rental_extension_deposit"
        | "vtu_purchase" | "vtu_purchase_refund" | "subscription",
    provider: "100pay" | "paystack" | "flutterwave" | null,
    paymentRail: "wallet" | "100pay" | "paystack" | "flutterwave",
    idempotencyKey: string,
    description: string,

    // Pending checkout state
    hostedUrl: string,           // URL to open in checkout
    checkout: object | null,     // 100Pay SDK payload; null for Paystack/Flutterwave
    callbackUrl: string,

    // Provider identifiers
    providerChargeId: string,        // 100Pay charge ID
    flutterwaveTransactionId: number, // Flutterwave numeric tx ID
    flutterwaveStatus: string,
    flutterwaveWebhookStatus: string,
    flutterwave_tx_ref: string,

    // Settlement state
    paymentStatus: "pending" | "paid" | "successful" | "failed",
    paymentConfirmedAt: string,
    processedPaymentEventIds: string[], // deduplicate webhook events
    resolvedAmountCredited: number,
    resolvedCurrency: string,
    providerCurrency: string,
    providerRate: number,

    // VTU-specific
    billCreated: boolean,
    billError: string | undefined,
    billerCode: string,
    itemCode: string,
    itemName: string,
    recipientPhone: string,
    network: string,
    serviceType: "airtime" | "data",
    walletDebitTxId: string,   // debit TX ID for provider-rail bill settlement
    walletRefundTxId: string,  // refund TX ID if bill failed

    // Rental-specific
    numberId: string,
    platformId: string,
    rentalId: string,
    extensionMinutes: number,
    rental_created: boolean,
    extension_applied: boolean,
    cancelReason: string,
  }
}
```

Indexes:

- `{ reference: 1 }` unique — primary lookup key.
- `{ idempotencyKey: 1 }` unique sparse — prevents duplicate ledger entries.
- `{ userId: 1, "meta.kind": 1, "meta.idempotencyKey": 1 }` — resume pending checkouts.
- `{ "meta.provider": 1, status: 1 }` — webhook lookup.

## Wallet

```ts
{
  userId: ObjectId,
  teamId: ObjectId | null,
  symbol: string,          // "NGN", "CRDTS", "USD", etc.
  type: "personal" | "team",
  isDefault: boolean,
}
```

**Important**: There is no stored `balance` field. Balance is always computed from ledger entries using `computeWalletBalance(walletId)`.

## WebhookEvent (observability)

Persist every inbound webhook for audit/debugging.

```ts
{
  provider: "paystack" | "100pay" | "flutterwave",
  type: string,            // event type
  payload: Mixed,          // raw event body
  status: "received" | "processing" | "completed" | "failed",
  failureReason?: string,
  linkedTransactionId?: ObjectId,
  createdAt, updatedAt,
}
```

## State Machine

```
              initiate route
  pending  ──────────────────► (created with reference + hostedUrl)
     │
     │ webhook OR verify endpoint
     ▼
  successful  ── domain side effects applied (rental created, bill executed, wallet credited)
     │
  failed      ── locks released, wallet refunded if needed
```

For VTU provider-rail flows, there is an additional sub-state tracked in `meta`:

```
  status: "successful" + meta.billCreated: false  →  bill settlement in progress
  status: "successful" + meta.billCreated: true   →  fully settled
```

## Idempotency invariants

- `reference` unique — one reference per payment attempt.
- `idempotencyKey` unique sparse — one ledger entry per logical debit/credit.
- Webhook claim uses `findOneAndUpdate({_id, status:"pending"}, {$set:{status:"..."}})`.
- Pending TX resume: `findOne({ userId, "meta.idempotencyKey": key, createdAt: { $gte: 30-min-ago } })`.
- Processed event dedup: `meta.processedPaymentEventIds` array — push event ID at settlement.
- Balance is always computed from ledger (`computeWalletBalance`), never from a stored `balance` field.

## Legacy Shape (WalletTransaction)

Some earlier reference files in this folder document a `WalletTransaction` shape (used by Paystack patterns from earlier versions). In this repository the unified `Transaction` model above is the authoritative shape.
