# 100Pay Payouts (Withdrawals)

Use the server-side SDK (`@100pay-hq/100pay.js`) to push funds to a user's
100Pay PayID. Supports a multi-wallet "waterfall" so a single requested
amount can be filled across several wallet currencies.

## Prerequisites

User must have linked their 100Pay account; store at minimum:

```ts
user.hundredPay = {
  payId: "...",      // recipient identifier on 100Pay
  appName: "...",    // optional, for UI
};
```

## Route shape

`POST /api/wallet/withdraw`

Request body:
```ts
{ amount: number; pin?: string; passkeyAssertion?: AuthenticationResponseJSON }
```

Steps:
1. Auth (`getCurrentUser`).
2. Rate limit (e.g. 5 / 15 min).
3. Validate amount + connection (`user.hundredPay?.payId`).
4. Verify PIN or Passkey assertion (transactional auth).
5. Select user's default wallet, ensure `balance >= amount`.
6. Create `payout` transaction with status `processing`.
7. Call `hundredPayClient.transfer.executeTransfer(...)`.
8. Atomic conditional debit: `findOneAndUpdate({ _id, balance:{$gte:amount} }, { $inc:{ balance:-amount } })`.
9. Mark transaction `confirmed` + send notification.

## Transfer call

```ts
const transferResult = await hundredPayClient.transfer.executeTransfer({
  amount,
  symbol: wallet.currency || DEFAULT_CURRENCY, // requested currency
  to: currentUser.hundredPay.payId,
  transferType: "internal",
  note: `Wallet withdrawal (ref: ${reference})`,
  multiWallet: true,
  walletOrder: ["NGN", "USDC", "PAY"],
  wallets:     ["NGN", "USDC", "PAY"],
});
```

Response includes `withdrawalAnalysis` when `multiWallet`:
```ts
{
  sufficientFunds: boolean;
  totalAvailable:  number;
  requestedAmount: number;
  requestedCurrency: string;
  withdrawalPlan: Array<{
    symbol: string;
    walletId: string;
    amountInWalletCurrency: number;
    amountInRequestedCurrency: number;
    conversionRate: number;
  }>;
}
```

Log each `withdrawalPlan` entry — invaluable when reconciling.

## Race-safe debit

```ts
const updated = await Wallet.findOneAndUpdate(
  { _id: wallet._id, balance: { $gte: amount } },
  { $inc: { balance: -amount } },
  { new: true }
);
if (!updated) {
  transaction.status = "failed";
  transaction.description += " - Insufficient funds at debit time";
  await transaction.save();
  return NextResponse.json({ error: "Insufficient funds" }, { status: 400 });
}
```

## Failure modes to handle explicitly

- Provider returns "insufficient platform balance" → notify ops + mark txn failed; do NOT debit.
- Provider HTTP timeout → mark txn `pending` and add a reconciliation job that lists pending payouts, queries 100Pay status, and resolves them.
- PayID typo → fail before debiting.
