# Payment and billing rules [optional module: any product that charges money]

Security: `security.md` §10. Data: `backend.md` §6 (money, immutability). Providers: `integrations.md`.

## 1. Architecture

- **Recommended default: your own billing engine owns subscriptions and invoices** (plans, prices, currencies, trials, coupons, periods, renewals, proration, cancellation, dunning, invoices, receipts, credit notes, tax fields) in your database; **payment providers are payment rails only** behind one `PaymentProvider` interface: save a payment method, charge your invoice (on-session or merchant-initiated renewal), refund, verify webhooks. No plan, price, coupon, trial or invoice is created at a provider; you avoid lock-in and state divergence. (A project that deliberately chooses provider-managed subscriptions records that decision and applies §2–§6 anyway.)
- Providers are enabled, ordered and limited by currency/country from the admin; adding one is a decision + adapter.

## 2. Card data and browser trust

- **Card data never touches your servers (PCI scope minimized):** use the provider's embedded fields/hosted elements inside your own page; only provider tokens/ids, brand, last 4, expiry, country and status are stored. Provider scripts are the only third-party scripts allowed, loaded only on the payment-method screen from the provider's domain with a CSP allow-list. Unavoidable provider-controlled steps (3-D Secure, wallet approval) are accepted.
- **Never trust the browser.** An invoice is paid only after server-side verification: a provider lookup plus a webhook verified with the provider's signature scheme, cross-checked against your stored payment attempt, invoice and account.

## 3. Correctness

- **Charged amount and currency always come from your invoice**, never from the browser or a callback. A webhook whose amount/currency/account differs is rejected and flagged.
- Money is fixed-point decimal + currency code; rounding rules are defined once; tax and proration computations are unit-tested with edge cases.
- **Idempotency:** every charge attempt has a key derived from invoice + attempt number; retries never double-charge; "record payment" and webhooks are idempotent by unique external id.
- **Refunds call the provider's refund API first** (the provider that took the payment); status changes only after the provider confirms. Refunds, price changes and manual payment confirmation need step-up authentication, an external reference and an audit row.
- Webhooks: verify signature, store the event id (unique), process in a queue, unknown types stored + grouped alert, dead-letters alert. Provider error text is mapped to your translated messages.
- Financial records (payments, invoices, subscriptions, credit notes, ledgers) are **immutable**: corrected by new rows, never deleted or edited; personal data is anonymized after retention.
- Every paid invoice gets a generated PDF invoice/receipt in the account's billing language.

## 4. Lifecycle and dunning

- Free tier (if any) has no card requirement. Optional trials are dynamic per plan. Monthly/yearly/lifetime billing; renewals charge the saved method automatically or the user pays manually.
- **Failed payment:** grace period (dynamic) with scheduled retries and emails, then downgrade — **no data loss**.
- **Proportional consequences:** a user never loses **read** access to their own data over an unpaid invoice; an unpaid renewal downgrades after grace; an unpaid upgrade reverts to the previously paid plan, never suspends. The banner and the dunning job use the **same function**, so a message never promises a consequence the job will not apply.
- Over-limit after a downgrade: data kept, read-only above the limit.
- Upgrade prompts appear only when a limit is actually hit and state the limit and the plans that allow it (`ux.md`).

## 5. Operations

- Sandbox/test credentials in development and QA; live keys only in the production environment; provider secrets never in the database or shown in admin.
- Public pricing shows real data from the database (prices, trials, discounts); never a hardcoded or invented figure (`design-conversion.md` §1).
- Reconciliation job compares provider records with local invoices and alerts on drift.
- Tests never call a real provider or move real money; webhook tests cover valid/invalid signatures, replays, amount mismatch and out-of-order events.
