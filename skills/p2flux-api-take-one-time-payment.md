---
name: p2flux-take-one-time-payment
description: Take a single USDC payment on Base end to end with P2Flux — create the intent, present the hosted checkout, and verify settlement server-side before marking an order paid.
api: P2Flux API
operations:
  - createPayment
  - resolvePayment
  - verifyPayment
  - recoverPayment
---

# Take a one-time USDC payment

Grounded in the P2Flux OpenAPI (`openapi/p2flux-api-openapi.json`). The API needs no
credentials; the returned capability token is the bearer secret — keep it server-side.

## Steps

1. **Create the intent.** `POST /v1/payments` (`createPayment`) with the recipient
   wallet, amount (a decimal string like `"10.00"` — never a JSON number) and your
   internal order id kept on YOUR side. You get back a payment intent capability
   (`p2f1.…`) and the pay block the checkout needs. The `reference` is minted by P2Flux;
   store your `order -> reference` mapping.
2. **Present checkout.** Hand off to the hosted checkout (`https://pay.p2flux.com`, or
   `https://pay-test.p2flux.com` on Base Sepolia). The browser only opens it and reports
   back — that report is a claim, not proof.
3. **Verify server-side.** `POST /v1/payments/verify` (`verifyPayment`) with the intent
   and the transaction hash. This is the trust boundary: it returns HTTP 200 with a typed
   `{ valid, code }` verdict plus the settlement receipt. Mark the order paid ONLY on a
   `valid` verdict.
4. **Handle WAIT.** If the verdict/action is `WAIT` / `PAYMENT_CONFIRMING`, the money may
   already have moved — ask again about the SAME transaction; never start another payment.
5. **Recover a lost hash.** If you lost the transaction hash, `POST /v1/payments/recover`
   (`recoverPayment`) — it is a pure, idempotent read and works long after the intent
   expired.

## Rules
- No Idempotency-Key header exists; retries are safe because settlement is exactly-once.
- `INSUFFICIENT_BALANCE` / `INSUFFICIENT_ALLOWANCE` / `INVALID_SIGNATURE` →
  `CUSTOMER_ACTION_REQUIRED` (surface a wallet prompt, do not retry blindly).
- `RATE_LIMITED` (429) carries `retry_after` and a `Retry-After` header — back off.
