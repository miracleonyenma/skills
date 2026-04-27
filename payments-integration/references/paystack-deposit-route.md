# Paystack Wallet Deposit Init Route

Route: `POST /api/wallet/deposit`

Responsibilities:
1. Authenticate user.
2. Resolve user's default wallet (create one if missing).
3. Create a `pending` deposit `WalletTransaction` with a generated reference.
4. Call `initializePayment(email, amount, reference)` and return the
   `authorization_url` to the client for redirect.

```ts
// app/api/wallet/deposit/route.ts
import { NextResponse } from "next/server";
import crypto from "crypto";
import connectToDatabase from "@/lib/mongodb";
import { Wallet } from "@/models/Wallet";
import { WalletTransaction } from "@/models/WalletTransaction";
import { getCurrentUser } from "@/lib/auth";
import { initializePayment } from "@/lib/paystack";
import { DEFAULT_CURRENCY } from "@/lib/currency";

export async function POST(request: Request) {
  const user = await getCurrentUser();
  if (!user) return NextResponse.json({ error: "Unauthorized" }, { status: 401 });

  const { amount } = await request.json();
  if (!amount || amount <= 0)
    return NextResponse.json({ error: "Invalid amount" }, { status: 400 });

  await connectToDatabase();

  // 1. Resolve / lazy-create default wallet
  let wallet =
    (await Wallet.findOne({ userId: user.userId, type: "personal", isDefault: true })) ||
    (await Wallet.findOne({ userId: user.userId, type: "personal" }).sort({ createdAt: 1 }));

  if (!wallet) {
    wallet = await Wallet.create({
      userId: user.userId,
      type: "personal",
      currency: DEFAULT_CURRENCY,
      balance: 0,
      isDefault: true,
    });
  }

  const reference = crypto.randomBytes(16).toString("hex");

  // 2. Pending transaction (idempotency key = reference, unique sparse index)
  await WalletTransaction.create({
    walletId: wallet._id,
    actorId: user.userId,
    type: "deposit",
    amount,
    currency: wallet.currency,
    status: "pending",
    providerReference: reference,
    from: {
      type: "external_deposit",
      details: { provider: "paystack", reference },
    },
    to: {
      type: "wallet",
      userId: user.userId,
      walletId: wallet._id,
      details: {
        userId: user.userId,
        walletId: wallet._id,
        currency: wallet.currency,
        walletType: "personal",
      },
    },
  });

  // 3. Provider call
  const paystackData = await initializePayment(user.email, amount, reference);
  if (!paystackData.status) {
    return NextResponse.json({ error: "Payment initialization failed" }, { status: 500 });
  }

  return NextResponse.json({ authorizationUrl: paystackData.data.authorization_url });
}
```

## Client (redirect flow)

```tsx
const res = await client.post<{ authorizationUrl: string }>(
  "/api/wallet/deposit",
  { amount: grossAmount }
);
if ("authorizationUrl" in res) window.location.href = res.authorizationUrl;
```

## Variants
- **Pay-link (sender pays a tagged recipient)**: pending transaction is on
  the *sender* wallet with metadata `source: "pay_link"` and embedded
  `senderWalletId` / `recipientWalletId`. The webhook then performs
  deposit-then-transfer. See [pay-link-flow.md](./pay-link-flow.md).
