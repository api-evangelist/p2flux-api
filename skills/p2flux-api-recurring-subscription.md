---
name: p2flux-recurring-subscription
description: Set up a P2Flux recurring USDC subscription (EIP-712 authorization), charge each billing period from your backend, and let the customer cancel on-chain.
api: P2Flux API
operations:
  - createSubscription
  - resolveSubscription
  - finalizeSubscription
  - charge
  - recoverCharge
  - subscriptionStatus
  - prepareSubscriptionCancellation
---

# Set up and run a recurring subscription

Grounded in the P2Flux OpenAPI (`openapi/p2flux-api-openapi.json`).

## Setup (customer signs once)
1. `POST /v1/subscriptions` (`createSubscription`) with recipient, per-period amount and
   period. Returns subscription terms and a setup token (`p2setup2.…`, 24-hour window).
2. `POST /v1/subscriptions/resolve` (`resolveSubscription`) to get the exact EIP-712
   payload the customer signs in their wallet.
3. `POST /v1/subscriptions/finalize` (`finalizeSubscription`) with the customer's
   signature. Returns the `p2s2.` charge capability — the bearer secret that authorizes
   charges. Keep it server-side, encrypted at rest.

## Each billing period (merchant-triggered, from your server)
4. `POST /v1/charges` (`charge`) with your stored reference. Safe to retry — a charge is
   allowed at most once per period.
   - `ALREADY_CHARGED` → `SUCCESS` (this period is paid).
   - `NOT_DUE` → `WAIT` (period not open yet).
   - `INSUFFICIENT_BALANCE` / `INSUFFICIENT_ALLOWANCE` → `CUSTOMER_ACTION_REQUIRED`.
   - `PERMISSION_REVOKED` / `SUBSCRIPTION_EXPIRED` → `STOP_SUBSCRIPTION` (permanent).
5. If a charge returned `ALREADY_CHARGED` but you lost the hash, `POST /v1/charges/recover`
   (`recoverCharge`) with the period index.
6. `POST /v1/subscriptions/status` (`subscriptionStatus`) reads current state from chain.

## Cancellation (customer-owned)
7. `POST /v1/subscriptions/revoke/prepare` (`prepareSubscriptionCancellation`) returns the
   calldata the customer's wallet sends to cancel. The capability never has to leave your
   server; for a browser flow, mint a cancel token with `createCancellationSession`.

## Rules
- The p2s2 capability can only collect the signed amount, once per period, to the signed
  recipient — it cannot over-charge.
- No Idempotency-Key header; the once-per-period guarantee makes `charge` retry-safe.
