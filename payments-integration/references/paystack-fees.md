# Paystack Fee Math

Paystack (Nigeria) standard local card pricing:
- 1.5% of the transaction
- + NGN 100 flat (waived if amount < NGN 2,500)
- Total fee capped at NGN 2,000

## Fee from Gross

```ts
export function calculatePaystackFee(amount: number): number {
  if (amount <= 0) return 0;
  let fee = amount * 0.015;
  if (amount >= 2500) fee += 100;
  if (fee > 2000) fee = 2000;
  return fee;
}
```

## Gross from Net (so payee receives an exact net amount)

Three cases derived algebraically:
- **Waived (Gross < 2500):** `Net = 0.985 * Gross` → `Gross = Net / 0.985`
- **Standard:** `Net = 0.985 * Gross - 100` → `Gross = (Net + 100) / 0.985`
- **Capped (Fee = 2000):** `Gross = Net + 2000`

```ts
export function calculateGrossFromNet(netAmount: number): number {
  if (netAmount <= 0) return 0;

  // Case: waived
  let gross = netAmount / 0.985;
  if (gross < 2500) return gross;

  // Case: standard
  gross = (netAmount + 100) / 0.985;
  const fee = gross * 0.015 + 100;
  if (fee > 2000) return netAmount + 2000; // cap reached
  return gross;
}
```

## UI Pattern (toggle "include fees")

- `mode === "gross"` (default): user types what they pay; show resulting net.
- `mode === "net"`: user types what they want received; show total to pay.

```ts
let grossAmount = 0, netAmount = 0, fee = 0;
if (numAmount > 0) {
  if (mode === "gross") {
    grossAmount = numAmount;
    fee = calculatePaystackFee(grossAmount);
    netAmount = grossAmount - fee;
  } else {
    netAmount = numAmount;
    grossAmount = calculateGrossFromNet(netAmount);
    fee = grossAmount - netAmount;
  }
}
```

## Cross-currency note

Paystack only charges NGN. If your wallet/UI uses another currency, convert
to NGN at init time and store both `originalAmount` and `ngnRate` in
metadata so the webhook can credit the original currency.
