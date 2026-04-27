# Paystack Server SDK Wrapper

Tiny, dependency-free wrapper around two Paystack endpoints. Keep the secret
key server-only. Paystack expects amounts in **kobo** (1 NGN = 100 kobo).

```ts
// lib/paystack.ts
const PAYSTACK_SECRET = process.env.PAYSTACK_SECRET_KEY;

export async function initializePayment(
  email: string,
  amount: number,
  reference: string,
  callbackUrl?: string
) {
  const response = await fetch(
    "https://api.paystack.co/transaction/initialize",
    {
      method: "POST",
      headers: {
        Authorization: `Bearer ${PAYSTACK_SECRET}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        email,
        amount: Math.ceil(amount * 100), // kobo
        reference,
        callback_url:
          callbackUrl || `${process.env.NEXT_PUBLIC_APP_URL}/wallet`,
      }),
    }
  );

  if (!response.ok) {
    const errorBody = await response.text();
    throw new Error(
      `Paystack initialization failed: ${response.status} ${response.statusText} - ${errorBody}`
    );
  }
  return await response.json(); // { status, data: { authorization_url, reference, access_code } }
}

export async function verifyPayment(reference: string) {
  const response = await fetch(
    `https://api.paystack.co/transaction/verify/${reference}`,
    {
      method: "GET",
      headers: { Authorization: `Bearer ${PAYSTACK_SECRET}` },
    }
  );

  if (!response.ok) {
    const errorBody = await response.text();
    throw new Error(
      `Paystack verification failed: ${response.status} ${response.statusText} - ${errorBody}`
    );
  }
  return await response.json();
  // verification.data.status === "success" indicates payment captured
  // verification.data.amount  is in kobo (gross)
  // verification.data.fees    is in kobo
}
```

## Notes
- Always re-`verifyPayment` in the webhook before crediting. Don't trust the webhook body's amount.
- Reference must be unique per attempt (use `crypto.randomBytes(16).toString("hex")`).
- `callback_url` is where the user lands after paying. The server-side credit happens via the webhook regardless.
