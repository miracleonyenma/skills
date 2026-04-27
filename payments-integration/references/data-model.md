# Data Model (Wallet + WalletTransaction + WebhookEvent)

Mongoose, but the shape applies to any datastore.

## Wallet

```ts
{
  userId: ObjectId,        // owner
  type: "personal" | ...,
  currency: string,        // ISO code, e.g. "NGN", "USD", "USDC"
  balance: number,
  isDefault: boolean,
}
```

Index: `{ userId: 1, type: 1, isDefault: 1 }`.

## WalletTransaction (the single source of truth)

```ts
{
  walletId: ObjectId,      // primary wallet for this row
  purseId:  ObjectId|null, // optional grouping
  actorId:  ObjectId,      // who initiated
  beneficiaryId: ObjectId|null,

  type: "deposit" | "transfer_in" | "transfer_out" | "payout" | "refund",
  amount: number,          // settled amount (in `currency`)
  currency: string,
  status: "pending" | "processing" | "confirmed" | "failed",

  providerReference: string|undef, // unique sparse key for idempotency
  fee: number,             // optional, for deposits
  originalAmount: number,  // gross before fee/conversion

  from: {
    type: "user" | "wallet" | "external_deposit" | "external_withdrawal",
    userId, walletId, purseId,
    details: Mixed,        // e.g. { provider: "paystack", reference }
  },
  to: { /* same shape */ },

  metadata: Mixed,         // flexible per flow (pay_link, conversion, etc.)
  description: string,
  createdAt, confirmedAt,
}
```

Indexes:
- `{ providerReference: 1 }` **unique sparse** — idempotency key.
- `{ walletId: 1, createdAt: -1 }` — wallet history.
- `{ purseId: 1, createdAt: -1 }` sparse — purse history.

## WebhookEvent (observability)

Persist every inbound webhook. Schema sketch:

```ts
{
  provider: "paystack" | "100pay",
  type: string,                    // event.type
  payload: Mixed,                  // raw event body
  headers: Mixed,                  // fingerprinted, never raw secrets
  meta: { requestPath, requestMethod, responseStatusCode?, outcome? },
  status: "received" | "processing" | "completed" | "failed",
  failureReason?: string,
  linkedTransactionId?: ObjectId,
  logs: Array<{ at, kind, message, data? }>,
  createdAt, updatedAt,
}
```

A `webhookService` helper exposes:
```ts
createEvent(provider, type, payload, headers, meta) -> doc
linkTransaction(eventId, txnId)
addLog(eventId, kind, message, data?)
updateStatus(eventId, status, reason?, data?)
setResponseInfo(eventId, statusCode, outcome)
```

## State Machine

```
            init route
pending  ──────────────► (created with providerReference)
   │
   │ webhook claims atomically
   ▼
processing
   │
   ├─ success ─► confirmed (+ confirmedAt)
   └─ failure ─► failed   (or revert to `pending` for verification retries)
```

## Idempotency invariants

- `providerReference` unique (sparse) — DB enforces single row per attempt.
- Webhook claim uses `findOneAndUpdate({status:"pending"}, {status:"processing"})`.
- Wallet credit uses `$inc` (never read-modify-write the balance).
- Race-safe debit uses `findOneAndUpdate({balance:{$gte:amount}}, {$inc:{balance:-amount}})`.
