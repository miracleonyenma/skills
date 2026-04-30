# Common Pitfalls and Fixes

## 1) Treating callback as final settlement

Problem:

- UI shows success immediately on callback/redirect without server confirmation.

Fix pattern:

- Callback is a signal only — never credit wallet or activate rental on callback alone.
- Use a verify endpoint or by-reference polling before success UX.

Repository example:

- Wallet top-up calls `POST /v1/payments/wallet-topups/verify`.
- VTU Flutterwave calls `POST /v1/payments/vtu/verify` then polls `GET /v1/payments/vtu/by-reference`.
- Rental flows poll `GET /v1/payments/.../by-reference` until `rental_created` or `extension_applied`.

## 2) Duplicate callback/onPayment handling

Problem:

- 100Pay SDK may trigger both callback paths, causing duplicate verification requests.

Fix pattern:

- Add one-shot guard in checkout helper.
- Only first callback path proceeds.

Repository example:

- `apps/web/utils/payWith100Pay.ts` uses `callbackSettled`.

## 3) Checkout modal left open after payment signal (100Pay)

Problem:

- User remains in checkout modal after callback arrives.

Fix pattern:

- Explicitly close modal when callback is accepted.

Repository example:

- `payWith100Pay` clicks `#close_100pay_btn` on callback.

## 4) Wrong webhook secret variable

Problem:

- Config uses a token env name that does not match runtime code.

Fix pattern:

- 100Pay webhook reads `HUNDREDPAY_WEBHOOK_SECRET`.
- Paystack webhook reads `PAYSTACK_SECRET_KEY` (not a separate webhook secret variable).
- Flutterwave webhook reads `FLUTTERWAVE_V3_WEBHOOK_SECRET`.

## 5) Creating duplicate pending transactions

Problem:

- Every initiate request creates a fresh pending transaction.

Fix pattern:

- Resume pending transactions younger than 30 minutes (match by `meta.idempotencyKey`).
- Expire stale pending rows and clear related side effects.

## 6) Not releasing reserved number on failed deposit

Problem:

- Rental deposit fails but number remains reserved.

Fix pattern:

- On failed/cancelled/expired webhook states, mark transaction failed and call reservation release.

## 7) Missing reference consistency checks

Problem:

- Verification result reference differs from the pending transaction reference.

Fix pattern:

- Verify route compares provided reference and verified reference, rejects mismatch.

## 8) Missing idempotent webhook event tracking

Problem:

- Re-delivered webhook events replay settlement logic.

Fix pattern:

- Store processed event IDs in `meta.processedPaymentEventIds` and short-circuit repeats.

## 9) Assuming webhook arrives before client poll

Problem:

- Client checks status before webhook has settled transaction.

Fix pattern:

- Poll with retry/backoff windows and return in-progress statuses cleanly.
- `GET /vtu/by-reference` lazily triggers `settleVtuBillFromTransaction()` if TX is successful but bill not yet created.

## 10) Building customer identity from unstable client state

Problem:

- Empty `customer.user_id` or `customer.email` at checkout launch.

Fix pattern:

- Build customer identity on server in initiate response.
- Client consumes server checkout payload directly.

## 11) Flutterwave: missing `transaction_id` on callback (Flutterwave-specific)

Problem:

- Only capturing `tx_ref` from the Flutterwave redirect URL but not `transaction_id`.
- `verifyTransaction()` requires the numeric `transaction_id`, not the string reference.

Fix pattern:

```ts
const txRef = searchParams.get("tx_ref") || "";
const transactionId = Number(searchParams.get("transaction_id") || 0);
// Must pass BOTH to the verify endpoint:
await vtuService.verifyCheckout({ reference: txRef, transactionId });
```

## 12) Flutterwave webhook: wrong hash input format

Problem:

- Computing HMAC over the raw request body string (like Paystack) instead of the parsed JSON.
- Flutterwave computes `HMAC-SHA256(JSON.stringify(parsedBody), secret)`.

Fix pattern:

- Parse JSON body first, then call `verifyWebhookHash(req.body, incomingHash)` where `req.body` is already the parsed object.
- Do NOT re-stringify from raw text — JSON key ordering must match.

## 13) VTU: bill not executed after Flutterwave payment

Problem:

- Flutterwave payment confirmed but airtime/data never delivered.
- Root cause: `settleVtuBillFromTransaction()` was never called after payment confirmation.

Fix pattern:

- After Flutterwave redirect: call `POST /vtu/verify` with `{ reference, transactionId }` which marks TX successful and calls `settleVtuBillFromTransaction()`.
- Webhook for `charge.completed`: only marks TX successful for non-wallet_topup kinds — bill settlement still requires `settleVtuBillFromTransaction()` via by-reference or verify call.

## 14) VTU by-reference: Flutterwave rail not included in status check

Problem:

- `GET /vtu/by-reference` only checked `["100pay", "paystack"]` for the `isProviderRail` condition, so Flutterwave VTU transactions were stuck in `"pending"` even after payment.

Fix pattern:

- Include `"flutterwave"` in the `isProviderRail` array:

```ts
const isProviderRail = ["100pay", "paystack", "flutterwave"].includes(meta.paymentRail);
```

## 15) Wallet rail: reading stored balance instead of computing from ledger

Problem:

- Reading a `balance` field directly from the wallet document leads to race conditions under concurrent transactions.

Fix pattern:

- Always use `computeWalletBalance(walletId)` which sums ledger credits minus debits.

## 16) Provider rail VTU: wallet debit not refunded on bill failure

Problem:

- `settleVtuBillFromTransaction()` debits the NGN wallet for provider-rail flows before calling the Bills API.
- If the Bills API call fails, the debit must be reversed.

Fix pattern:

- Catch `createBillPayment()` errors and post a matching credit with a refund idempotency key.
- Check for `isFlutterwaveDuplicateReferenceError()` before refunding — it means the bill was actually created and needs reconciliation, not a refund.

Problem:

- Client checks status before webhook has settled transaction.

Fix pattern:

- Poll with retry/backoff windows and return in-progress statuses cleanly.

## 10) Building customer identity from unstable client state

Problem:

- Empty `customer.user_id` or `customer.email` at checkout launch.

Fix pattern:

- Build customer identity on server in initiate response.
- Client consumes server checkout payload directly.
