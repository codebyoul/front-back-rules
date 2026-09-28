# Integration rules — third-party APIs, channels, webhooks, devices [optional module]

Applies to any outbound integration (messaging, IoT/device clouds, payment rails, CRMs, storage). Security: `security.md` §6, §10. Payments: `payments.md`.

## 1. Structure

- One adapter per provider behind one interface; **one pipeline for every provider** (authorization, plan limits/quotas, kill switch, safeguards, idempotency key, TTL, audit). Adapters only translate a command into a provider call and a provider answer into a normalized result or state report — they never bypass the pipeline.
- A new integration requires a decision record, an adapter spec section and an allow-list entry in the same commit. Look up the vendor's current documentation before implementing; record the facts (endpoints, limits, auth, error codes) in a research note.
- Availability is decided by **configuration**, never by environment name. An unconfigured integration is off and hidden (its routes 404), silently: no error, banner or nag. Every integration also has a dynamic platform on/off switch; an unconfigured one cannot be switched on.
- Prefer a plain HTTP client behind your own interface over vendor SDKs; no heavyweight SDK for one call (`dependencies.md`).

## 2. Direction and network

- **Outbound calls only**, to vendor hosts on a code allow-list or to user-entered HTTPS URLs behind the SSRF guard (`security.md` §6). Providers/devices reach you only through your **signed webhooks**.
- Never inbound connections into users' networks: no port forwarding, VPN, LAN/private addresses, direct device IPs — and no help text suggesting them.
- No long-lived custom daemons for integrations when the host floor forbids them; use a bridge that exposes HTTP publish/webhook interfaces.

## 3. Credentials and secrets

- Connection credentials (API keys, tokens, OAuth refresh tokens, webhook secrets) use field-level encryption, are **write-only** in the API (`has_token: true`), never logged, never in exceptions. Platform OAuth client secrets live in the environment; the admin shows "configured / missing".
- Least-privilege scopes; refresh tokens rotated; revocation of a connection stops all queued work for it.

## 4. Sending, retries, limits

- Every call has connect+total timeouts, an idempotency key, and mapped error codes (provider text is mapped to your translated messages; raw text only in logs).
- **Retry only when nothing reached the vendor or the operation is an absolute set** (idempotent); never retry a non-idempotent action blindly. At most one safe reroute to a fallback path.
- Vendor rate limits are declared per adapter and enforced per connection; a 429 backs off with jitter.
- The call an end user waits for (e.g. a control command) runs in the request within a bounded timeout; retries go to the queue. Other sends (bulk, reminders, receipts) are queued and rate-limited per tenant/plan.

## 5. Inbound: webhooks, events, state reports

- Verify authenticity (HMAC or per-connection secret, constant-time compare, timestamp window), be idempotent by event id or content hash, process in a queue, rate-limit, and answer unverifiable requests with a generic 404 (`security.md` §10).
- **A provider's answer is evidence, not a promise:** distinguish `accepted/sent` (the provider took it) from `confirmed` (a state report shows the target state). Show the difference in the UI.
- **State reports and replies from third parties never start privileged or dangerous actions.** They may stop, alert, or update state; starting/activating anything requires an authorized command through the pipeline.
- Unknown event types are stored and raise a grouped alert; dead-lettered events alert.

## 6. Safety-critical commands (actuators, money movement, irreversible sends)

- Every command records who/when/why before it is sent; commands carry a **TTL (never sent late)** and an idempotency key (**never sent twice**); duplicates return the first result.
- Every "start" plans its "stop": prefer the device's own timer with a server-side stop as a backup; safety stops run even when late. Nothing restarts after a fault.
- Expected state and confirmed state are separate; the UI never shows "confirmed" on an expectation.
- A guard checks every start (manual or automatic) against safeguards/interlocks; **a stop is never blocked by our own switches, limits or call budgets** (the interaction with a global kill switch is an explicit, recorded owner decision).
- Kill switch: staff can block a tenant's outbound actions immediately; queued work re-checks it (`security.md` §3, §9).

## 7. Tests

External services are never hit: HTTP is faked, stray requests fail the test; webhook tests send valid and invalid signatures; each adapter has contract tests from recorded vendor responses; idempotency and retry paths are tested.
