# Testing rules

## 1. Principles

- Tests prove behaviour and protect against regressions; coverage is judged by risk (money, auth, tenancy, data loss), not by a percentage.
- Deterministic: no real network, no real payment/SMS/email/push, no sleeps, no dependence on wall-clock or test order. Time is controlled (freeze/travel or backdated fixtures); rows created in one test transaction share `now()`.
- The full suite runs to the end (no stop-on-first-failure) before "done"; report failed and skipped counts honestly.
- A flaky test is a bug: fix it or quarantine it with a ticket; never retry-until-green.
- Tests import the same enums/constants as production (no magic strings). No test-only branches in production code.
- Every bug fix ships a regression test that fails before and passes after.

## 2. Backend: every endpoint has tests for

- **401** unauthenticated.
- **Tenant isolation:** another tenant's id → 404; lists never leak foreign rows.
- **Authorization:** each role/permission → allowed / 403 / 404 as specified; a low-privilege member of the same tenant cannot escalate.
- **Validation:** 422 for missing/invalid/oversized/wrong-type input; unknown fields ignored; injected ids (tenant, user, role, plan, price) ignored.
- **Data exposure:** no secrets, hashes, tokens, stack traces, internal ids in responses.
- **CSRF** on state-changing routes (cookie auth); **module/feature gating** (hidden → 404); **plan limits** (N+1th attempt fails).
- **Idempotency:** the same request twice creates one effect.
- Privileged/admin endpoints: non-admin denied, step-up auth enforced, audit row written.
- **Every 404 test first asserts a 200 on the right URL** (otherwise it proves nothing).
- Each cap/limit test first proves the N+1th attempt is refused.

### Shared route dataset
One dataset of all routes feeds shared tests, so a new route is covered automatically:
- a test enumerates the router and fails when a non-public route lacks authentication (explicit allow-list for public ones), and when a route lacks the required gate middleware;
- actors in each gate state (pending 2FA, unverified email, terms not accepted, suspended) are blocked;
- injection payloads (SQL, XSS, template `{{7*7}}`, `../`, CRLF, very long strings) and malformed ids never cause a 500;
- responses carry no server/framework version headers.

### Architecture tests
Enforced by tests, not by review: strict typing declared, no debug helpers (`dd`, `print`, `console.log`) committed, no raw environment reads outside config, no raw SQL with variables, module boundaries respected, controllers use the response helpers, only approved classes may email admin addresses.

### Concurrency tests
Where state is shared (rate limits, single-use tokens, webhook event ids, payment recording, quota counters, stock/balance changes): fire two parallel requests; exactly one wins; the loser gets a deterministic error, never a 500.

### Security tests
- **Bot protection (every public form):** missing/invalid ALTCHA → 422; replayed ALTCHA → 422; missing/wrong image captcha → 422; replayed captcha → 422; filled honeypot → 422; a bot signal returns the generic message.
- **Login adaptive captcha:** below N failures no image is required; at N the image is required and a request without it is refused even if the field is omitted; a wrong captcha counts as a failure; **the requirement survives a reload, a fresh client with no cookies/session, and a new session for the same identifier+IP**; a nonexistent identifier increments the counter and yields identical responses; success resets it; the window expiry resets it; ALTCHA stays required throughout.
Rate limits, lockout, session fixation/regeneration, password-reset and login same-response (anti-enumeration: status, body shape, timing class, emails sent), security headers, upload validation, webhook signature rejection, SSRF blocklist.

## 3. Frontend tests

- Unit tests for pure logic: money/date math, formatters, validation schemas, permission helpers, design-token contrast (both themes).
- Component/integration tests for states: loading, empty, error, permission-denied, long text.
- End-to-end at three viewports (360×740, 768×1024, 1440×900), both themes, RTL if supported; keyboard-only path for key flows; accessibility checks (no serious violations).
- Security e2e: another user's ids show the not-found view; injected ids ignored; XSS payloads render inert; no secrets in responses; caches cleared on logout/login (account B never sees account A's data).
- Async hook tests: start the action in a synchronous act, then wait for the outcome outside it.
- Artifacts (screenshots, traces) go to an ignored folder, never the repo root.

## 4. Test traps

- Authentication state and request-scoped services persist across requests in one test: reset them when switching actors.
- A controller/handler instance may be cached by the router: rebinding its dependencies between two requests has no effect; split the test.
- HTTP fakes accumulate and the first match wins: use a fresh fake per scenario and forbid stray requests.
- JSON test requests may send no cookies; queued cookies persist across requests: flush and prove the test fails without the fix.
- After-commit hooks/events do not fire inside a transaction-wrapped test: set state before anything resolves it.
- Helper functions declared at file level are global in some runners: prefix them by subject.
- A cached-read test renders twice (cold, then warm) to catch serialization bugs.
- Tests against the database run on every supported engine (`database-mysql.md`, `database-postgres.md`).
