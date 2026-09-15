---
name: p2flux-refund-payment
description: Refund a settled P2Flux USDC payment — lock the refund terms from the original settlement, send the transfer from the merchant wallet, and verify it on-chain. Enforce refund idempotency yourself.
api: P2Flux API
operations:
  - prepareRefund
  - resolveRefund
  - verifyRefund
---

# Refund a settled payment

Grounded in the P2Flux OpenAPI (`openapi/p2flux-api-openapi.json`).

## Steps
1. **Lock terms.** `POST /v1/refunds/prepare` (`prepareRefund`) with the original
   settlement and the amount units to refund. Returns a refund token (`p2refund1.…`,
   15-minute prepare window). The refund can be full or partial but never exceeds the
   original amount.
2. **Optional: read back.** `POST /v1/refunds/resolve` (`resolveRefund`) reports what a
   refund token authorizes, for the page/flow that holds it.
3. **Send the transfer** from the merchant wallet (the refund is an on-chain USDC transfer
   from you back to the payer, not a P2Flux-held balance clawback).
4. **Verify.** `POST /v1/refunds/verify` (`verifyRefund`) with the original settlement,
   amount units and the refund transaction hash. `REFUND_CONFIRMING` (409) → `WAIT`
   (money may have moved; re-check the same transaction).

## Rules
- **You must enforce refund idempotency.** Neither SDK nor the API tracks whether a
  payment was already refunded — dedupe on your side before calling `prepareRefund`.
- Refund errors: `REFUND_AMOUNT_INVALID`, `REFUND_WRONG_MERCHANT`,
  `REFUND_TRANSACTION_MISMATCH`, `REFUND_ORIGINAL_PAYMENT_INVALID` — all mean the refund
  as described will not settle; fix the inputs rather than retrying identically.
- No published time limit on refundability, but the refund TOKEN expires 15 minutes after
  prepare — re-prepare if it lapses.
