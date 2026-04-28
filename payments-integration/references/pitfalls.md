# Common Pitfalls and Fixes (100Pay-Focused)

## 1) Treating callback as final settlement

Problem:

- UI shows success immediately on `callback`/`onPayment` without server confirmation.

Fix pattern:

- Callback is a signal only.
- Use a verify endpoint or by-reference polling before success UX.

Repository example:

- Wallet flow calls `/v1/payments/wallet-topups/verify`.
- Rental flows poll `/v1/payments/.../by-reference` until `rental_created` or `extension_applied`.

## 2) Duplicate callback/onPayment handling

Problem:

- 100Pay SDK may trigger both callback paths, causing duplicate verification requests.

Fix pattern:

- Add one-shot guard in checkout helper.
- Only first callback path proceeds.

Repository example:

- `apps/web/utils/payWith100Pay.ts` uses `callbackSettled`.

## 3) Checkout modal left open after payment signal

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

- Webhook auth reads `HUNDREDPAY_WEBHOOK_SECRET`.

## 5) Creating duplicate pending transactions

Problem:

- Every initiate request creates a fresh pending transaction.

Fix pattern:

- Resume pending transactions younger than 30 minutes.
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

## 10) Building customer identity from unstable client state

Problem:

- Empty `customer.user_id` or `customer.email` at checkout launch.

Fix pattern:

- Build customer identity on server in initiate response.
- Client consumes server checkout payload directly.
