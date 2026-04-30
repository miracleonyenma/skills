# Multiwallet / Wallet Rail Pattern

This document describes the wallet payment rail — where users pay directly from their in-app wallet balance instead of going through an external checkout provider.

The wallet rail is used for VTU (airtime/data) purchases and subscriptions. It coexists with provider rails (`100pay`, `paystack`, `flutterwave`) in the same initiate route, selected via the `paymentRail` field.

---

## Core Functions

All wallet operations use three shared helpers from `apps/api/src/services/wallet/`:

### `getOrCreateWallet(userId, symbol)` → `WalletDoc`

Finds or creates a wallet for the user with the given currency symbol (`"NGN"`, `"CRDTS"`, `"USD"`, etc.). Never fails silently — always returns a wallet.

```ts
const wallet = await getOrCreateWallet(userId, "CRDTS");
// wallet._id — used as walletId for postTransaction calls
```

### `computeWalletBalance(walletId)` → `number`

Computes the current spendable balance from the ledger (sum of credits minus debits). **Do not use a stored `balance` field** — always compute from ledger to avoid race conditions.

```ts
const balance = await computeWalletBalance(wallet._id.toString());
if (balance < requiredAmount) {
  return res.status(400).json({ message: "Insufficient balance" });
}
```

### `postTransaction({ userId, walletId, symbol, amount, type, idempotencyKey, ... })` → `TransactionDoc`

Creates a ledger entry (debit or credit) against a wallet. The `idempotencyKey` prevents duplicate debits/credits.

```ts
const debitTx = await postTransaction({
  userId,
  walletId: wallet._id,
  symbol: paymentSymbol,
  amount: walletDebitAmount,
  type: "debit",
  idempotencyKey: `vtu:${idempotencyKey}:debit`,
  reference,
  from: { name: "User Wallet", user: userId, wallet: wallet._id },
  to: { name: "SYSTEM_VTU" },
  meta: { kind: "vtu_purchase", ... },
});
```

---

## VTU Wallet Rail Flow

When `paymentRail === "wallet"`:

1. **Resolve wallet** — `getOrCreateWallet(userId, walletSymbol)`
2. **Convert amount if needed** — if `walletSymbol !== "NGN"`, use `currencyService.convert(amountNGN, "NGN", walletSymbol)` to get `walletDebitAmount`
3. **Check balance** — `computeWalletBalance(wallet._id)` must be ≥ `walletDebitAmount`
4. **Debit wallet** — `postTransaction({ type: "debit", ... })` with idempotency key
5. **Execute bill** — call `flutterwaveService.createBillPayment()` immediately (synchronous)
6. **On success** — create a `vtu_purchase` Transaction record marked `status: "successful"`, `meta.billCreated: true`
7. **On bill failure** — refund via `postTransaction({ type: "credit", ... })` with refund idempotency key, then return error

```ts
// Step 4: Debit
const debitTx = await postTransaction({
  userId, walletId: wallet._id, symbol: paymentSymbol, amount: walletDebitAmount,
  type: "debit",
  idempotencyKey: `vtu:${idempotencyKey}:debit`,
  ...
});
debitTransactionId = debitTx._id.toString();

// Step 5: Execute bill
try {
  const billResult = await flutterwaveService.createBillPayment(billerCode, itemCode, {
    country: "NG", customer_id: normalizedPhone, amount: amountNGN, reference,
  });

  // Step 6: Record success
  const tx = await Transaction.create({
    userId, reference, status: "successful",
    meta: { kind: "vtu_purchase", billCreated: true, paymentRail: "wallet", ... },
  });

  return res.json({ success: true, data: { reference, billCreated: true, transaction: tx } });
} catch (billErr) {
  // Step 7: Refund
  await postTransaction({
    userId, walletId: wallet._id, symbol: paymentSymbol, amount: walletDebitAmount,
    type: "credit",
    idempotencyKey: `vtu:${idempotencyKey}:refund`,
    ...
  });
  return res.status(502).json({ message: "Bill purchase failed — wallet refunded" });
}
```

---

## VTU Provider Rail vs Wallet Rail Comparison

| Step | Wallet Rail | Provider Rail (100Pay/Paystack/Flutterwave) |
|---|---|---|
| PIN required | Yes | No (PIN skipped for external checkouts) |
| Balance check | Yes, before debit | No |
| Pending TX created | No (settled inline) | Yes — pending until payment confirmed |
| Hosted URL | None | Yes — `hostedUrl` returned to client |
| Bill executed | Synchronously in initiate route | After payment confirmed via `settleVtuBillFromTransaction()` |
| Wallet debit | At initiate time | At settle time (inside `settleVtuBillFromTransaction`) |
| Refund on failure | Immediate via ledger credit | Refund via ledger credit in `settleVtuBillFromTransaction` catch |
| Response shape | `{ reference, billCreated: true, transaction }` | `{ reference, hostedUrl, checkout, billCreated: false }` |

---

## Multi-Symbol Wallet Support

The wallet system supports multiple wallet symbols (`NGN`, `CRDTS`, `USD`, etc.). The conversion flow:

```ts
const walletSymbol = normalizeSymbol(req.body.walletSymbol || "CRDTS");

if (walletSymbol !== "NGN") {
  const converted = await currencyService.convert(amountNGN, "NGN", walletSymbol);
  walletDebitAmount = Math.round((converted.amountConverted + Number.EPSILON) * 1000) / 1000;
  // e.g. 1000 NGN → 50 CRDTS at 20 NGN/CRDTS rate
}
```

The VTU quote endpoint (`GET /v1/payments/vtu/quote`) shows the user how much of their chosen wallet symbol a given `amountNGN` costs:

```ts
// GET /v1/payments/vtu/quote?paymentRail=wallet&walletSymbol=CRDTS&amountNGN=1000
// Returns: { payableAmount: 50, payableCurrency: "CRDTS", conversionRate: 20 }
```

---

## Settlement via `settleVtuBillFromTransaction(txDoc)`

For provider-rail VTU flows, the bill is not created at initiate time. Once the provider payment is confirmed (via webhook or verify route), `settleVtuBillFromTransaction()` is called with the now-successful Transaction:

```ts
async function settleVtuBillFromTransaction(txDoc) {
  if (txDoc.status !== "successful" || txDoc.meta.billCreated === true) return txDoc;

  // For provider rails: first debit the NGN wallet (internal bookkeeping)
  if (isProviderRail) {
    const debitTx = await postTransaction({
      userId, walletId, symbol: "NGN", amount: amountNGN, type: "debit",
      idempotencyKey: `vtu:${idempotencyKey}:provider-wallet-debit`,
      ...
    });
  }

  // Execute bill
  const billResult = await flutterwaveService.createBillPayment(billerCode, itemCode, {
    country: "NG", customer_id: recipientPhone, amount: amountNGN, reference,
  });

  // Mark billCreated
  await Transaction.updateOne({ _id: txDoc._id }, {
    $set: { "meta.billCreated": true, "meta.flutterwaveRef": billResult.reference },
  });

  // On failure: refund NGN wallet debit + mark meta.billCreated: false
}
```

---

## `GET /vtu/by-reference` Status Logic

The polling endpoint checks both payment settlement and bill creation:

```ts
const isProviderRail = ["100pay", "paystack", "flutterwave"].includes(meta.paymentRail);

if (tx.status === "successful" && meta.billCreated === true) {
  return { status: "successful", billCreated: true };
}

if (tx.status === "successful" && isProviderRail && !meta.billCreated) {
  // Bill not yet created — trigger it lazily
  await settleVtuBillFromTransaction(tx);
  return { status: "processing", billCreated: false };
}

if (tx.status === "pending" && isProviderRail) {
  return { status: "pending", billCreated: false };
}
```

This lazy trigger in `by-reference` ensures bill settlement happens even if the webhook fires before `settleVtuBillFromTransaction` has run.

---

## Idempotency Keys Pattern

All wallet debits and credits use structured idempotency keys to prevent duplicates:

| Operation | Key pattern |
|---|---|
| Wallet rail VTU debit | `vtu:{idempotencyKey}:debit` |
| Wallet rail VTU refund | `vtu:{idempotencyKey}:refund` |
| Provider rail VTU wallet debit (settle) | `vtu:{idempotencyKey}:provider-wallet-debit` |
| Provider rail VTU wallet refund (bill fail) | `vtu:{idempotencyKey}:provider-wallet-refund` |
| Wallet top-up credit | `payment-topup:{transactionId}` |
| Subscription debit | similar prefix pattern |

---

## Pitfalls

1. **Never read a stored balance field** — always use `computeWalletBalance()`. Stored balances go stale under concurrent requests.
2. **Refund on any bill failure** — if the wallet was debited before the bill fails, always post a matching credit refund with its own idempotency key.
3. **Duplicate-reference bill errors** — Flutterwave Bills API may return a "duplicate reference" error on retry. Use `isFlutterwaveDuplicateReferenceError()` to detect this and reconcile via `getBillStatus()` instead of creating a new bill.
4. **Multi-symbol conversion** — if `walletSymbol !== "NGN"`, conversion rate is fetched at request time. Rate drift between quote and settle is expected and acceptable.
5. **Provider rail debit happens at settle time, not initiate time** — the wallet is NOT debited when the user clicks "Pay with Flutterwave". It is debited inside `settleVtuBillFromTransaction()` after payment is confirmed.
