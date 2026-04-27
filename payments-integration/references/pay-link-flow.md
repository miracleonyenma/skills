# Pay-link Flow (Third-party Pays a Tagged Recipient)

Pattern for public payment links like `/pay/@username`. The payer can be
authenticated or anonymous. The flow is uniform across Paystack and 100Pay:

1. Create a **pending deposit** on the *sender* wallet.
2. Webhook: deposit-then-transfer (sender wallet pass-through → recipient wallet) inside a single Mongo session/transaction.

## Init Route

`POST /api/payment/initialize` (Paystack) / `POST /api/payment/initialize/100pay`

Inputs: `{ recipientTag, amount, email?, paymentCurrency?, originalAmount?, ngnRate? }`.

Logic:
- If unauthenticated and `email` belongs to an existing user → `409 { error: "email_registered" }` (client redirects to login).
- If unauthenticated → auto-create a `User` with a unique generated tag and a personal wallet (`isAutoCreatedAccount: true`).
- If authenticated → resolve sender's default wallet.
- Resolve recipient's default wallet (lazy-create if missing).
- Reject self-payment.
- Create a `pending` deposit transaction on the sender wallet with metadata:

```ts
metadata: {
  source: "pay_link",
  payerEmail: senderEmail,
  recipientTag,
  recipientId: recipient._id.toString(),
  senderWalletId:    senderWallet._id.toString(),
  recipientWalletId: recipientWallet._id.toString(),
  isAutoCreatedAccount: !currentUser,
  // cross-currency:
  ...(paymentCurrency && paymentCurrency !== "NGN" && {
    paymentCurrency, originalAmount, ngnRate,
  }),
}
```

- Call provider (Paystack `initializePayment` / 100Pay returns `customer` + `ref_id`).

## Webhook Branch

In `processWebhook(...)`, detect `transaction.metadata?.source === "pay_link"` and route to `processPayLinkDeposit` instead of the simple credit path.

## processPayLinkDeposit (atomic, multi-step)

```ts
const session = await mongoose.startSession();
try {
  await session.withTransaction(async () => {
    // 1. Pass-through deposit into sender wallet
    await Wallet.findByIdAndUpdate(senderWalletId, { $inc: { balance: netAmount } }, { session });

    // 2. Confirm the deposit txn
    txn.fee = fee; txn.amount = netAmount; txn.originalAmount = grossAmount;
    txn.status = "confirmed"; txn.confirmedAt = new Date();
    await txn.save({ session });

    // 3. Debit sender wallet (transfer out)
    await Wallet.findByIdAndUpdate(senderWalletId, { $inc: { balance: -netAmount } }, { session });

    // 4. Convert if recipient currency differs
    let creditAmount = netAmount, creditCurrency = senderCurrency;
    if (recipientCurrency !== senderCurrency) {
      try {
        const { amountConverted } = await currencyService.convert(netAmount, senderCurrency, recipientCurrency);
        creditAmount = amountConverted; creditCurrency = recipientCurrency;
      } catch { /* fall back to sender currency */ }
    }
    await Wallet.findByIdAndUpdate(recipientWalletId, { $inc: { balance: creditAmount } }, { session });

    // 5. transfer_out on sender wallet
    await WalletTransaction.create([{
      walletId: senderWalletId, actorId: txn.actorId, beneficiaryId: recipientId,
      type: "transfer_out", amount: netAmount, currency: senderCurrency, status: "confirmed",
      from: { type: "wallet", userId: txn.actorId, walletId: senderWalletId, details: { walletType: "personal", currency: senderCurrency } },
      to:   { type: "wallet", userId: recipientId, walletId: recipientWalletId, details: { walletType: "personal", currency: senderCurrency } },
      metadata: { source: "pay_link", direction: "sent", payerEmail, recipientTag, linkedDepositReference: reference },
      description: `Payment to @${recipientTag}`,
      createdAt: new Date(), confirmedAt: new Date(),
    }], { session });

    // 6. transfer_in on recipient wallet (with conversion fields if applicable)
    const [creditTx] = await WalletTransaction.create([{
      walletId: recipientWalletId, actorId: txn.actorId, beneficiaryId: recipientId,
      type: "transfer_in", amount: creditAmount, currency: creditCurrency, status: "confirmed",
      from: { type: "wallet", userId: txn.actorId, walletId: senderWalletId, details: { walletType: "personal", currency: senderCurrency, payerEmail } },
      to:   { type: "wallet", userId: recipientId, walletId: recipientWalletId, details: { walletType: "personal", currency: creditCurrency } },
      metadata: {
        source: "pay_link", direction: "received", payerEmail, linkedDepositReference: reference,
        ...(creditCurrency !== senderCurrency && { convertedFrom: senderCurrency, originalAmount: netAmount }),
      },
      description: `Payment from ${payerEmail}`,
      createdAt: new Date(), confirmedAt: new Date(),
    }], { session });
  });
} finally {
  await session.endSession();
}
```

## Why deposit-then-transfer (instead of crediting recipient directly)?

- Keeps a clean audit trail: every wallet movement is a row.
- Lets pay-link payers (including auto-created accounts) see the transaction history.
- Makes refunds straightforward — reverse the `transfer_out` and `deposit`.
- Currency conversion happens at one well-defined boundary.
