# 100Pay Client Integration (Current Pattern)

Client integration in this repository is centralized in:

- `apps/web/utils/payWith100Pay.ts`

The helper wraps `shop100Pay.setup` from `@100pay-hq/checkout` and standardizes:

- Checkout payload shape
- Callback/onPayment deduping
- Modal close on successful callback signal
- Error and close callbacks for feature flows

## Shared Helper

```ts
// apps/web/utils/payWith100Pay.ts
"use client";

import { COUNTRIES, CURRENCIES, shop100Pay } from "@100pay-hq/checkout";

const payWith100Pay = async (data, onPayClose?, onPayError?, onPayCallback?) => {
  if (typeof window === "undefined") return;

  let callbackSettled = false;

  const closeCheckoutModal = () => {
    const closeButton = document.getElementById("close_100pay_btn");
    if (closeButton instanceof HTMLButtonElement) {
      closeButton.click();
    }
  };

  const handlePayCallback = (reference: string) => {
    if (callbackSettled) return;
    callbackSettled = true;
    closeCheckoutModal();
    onPayCallback?.(reference);
  };

  shop100Pay.setup(
    {
      ref_id: data.ref_id,
      api_key: data.apiKey,
      billing: {
        amount: data.billing.amount,
        currency: data.billing.currency as CURRENCIES,
        description: data.billing.description,
        country: data.billing.country as COUNTRIES,
        pricing_type: data.billing.pricing_type,
      },
      customer: {
        user_id: data.customer.user_id,
        name: data.customer.name,
        email: data.customer.email,
        phone: data.customer.phone,
      },
      metadata: { ...data.metadata },
      call_back_url:
        process.env.NEXT_PUBLIC_100PAY_CALLBACK_URL ||
        `${window.location.origin}/api/webhooks/100pay`,
      onClose: () => onPayClose?.(),
      callback: (reference) => handlePayCallback(reference),
      onError: (error: string) => onPayError?.(error),
      onPayment(reference: string) {
        handlePayCallback(reference);
      },
    },
    { maxWidth: "500px" },
  );
};
```

## How It Is Used

### Wallet top-up UI

File:

- `apps/web/app/(app)/wallet/page-client.tsx`

Pattern:

1. Call `walletsService.initiateTopUp(...)`
2. Use `result.checkout` payload from server
3. Launch `payWith100Pay(...)`
4. On callback, call `walletsService.verifyTopUp({ transactionId, reference })`
5. Refresh wallets/transactions after successful verification

### Number rental deposit UI

File:

- `apps/web/components/marketplace/number-purchase-flow.tsx`

Pattern:

1. Call `numbersService.initiateRentalDeposit(...)`
2. Launch `payWith100Pay(...)` with server checkout payload
3. On callback, poll `numbersService.getRentalDepositByReference(reference)`
4. Redirect only when status is `rental_created`

### Rental extension deposit UI

File:

- `apps/web/components/marketplace/extend-rental-dialog-button.tsx`

Pattern:

1. Call `numbersService.initiateRentalExtensionDeposit(...)`
2. Launch `payWith100Pay(...)`
3. On callback, poll `numbersService.getRentalExtensionDepositByReference(reference)`
4. Confirm only when status is `extension_applied`

## Important Client Rules

1. Do not build `customer.user_id` from local storage state; use server response.
2. Treat callback/onPayment as a signal, not final settlement.
3. Use server-issued `result.reference` as primary reference; callback reference is fallback.
4. Keep callback UX resilient to webhook races using polling or verify endpoint.
5. Close the modal on callback to avoid stuck checkout state in UI.

## Callback Relay Page (Wallet)

Wallet flow includes callback relay UI at:

- `apps/web/app/(app)/wallet/checkout-callback/_components/checkout-callback-content.tsx`

It posts callback params to parent/opener window so wallet page can continue verification asynchronously.
