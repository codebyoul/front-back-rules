# Security rules — NON-NEGOTIABLE, every project, front and back

Every rule is MUST unless marked SHOULD. These floors apply to all code, all environments that hold real data, and all agents.
A project may make a rule stricter, never weaker. Section pointers: DB engine specifics in `database-*.md`; UI-side handling in `frontend.md`.

## 1. Principles

- **Deny by default.** A new route, field, role, file or setting is closed until explicitly opened.
- **Never trust the client**: browser, mobile app, headers, cookies, ids, flags, prices, roles, plan, file names, content types. The server derives identity, tenant, role, plan and price itself.
- **Fail closed.** A missing secret, failed verification, unreachable auth service or unknown state means refuse (503/deny), never skip the check.
- **Least privilege** for users, services, DB roles, API tokens, CI tokens, agents.
- **Defense in depth.** UI hiding + server authorization + database constraints; each layer assumes the others can fail.
- **Minimal data.** Do not collect, log, cache or send what you do not need.
- **Threat model every feature** against four attackers: anonymous; another tenant's user; a low-privilege member of the same tenant; a compromised admin.
- **Secure by default configuration**: production builds refuse to run insecurely (§14).

## 2. Authentication

- Passwords: ≥ 12 characters, breached-password check, no composition rules, no forced periodic expiry, no truncation, hashed with Argon2id (or bcrypt/scrypt with tuned cost). Never reversible encryption, never a fast hash.
- **MFA**: TOTP or WebAuthn/passkeys available to all users, mandatory for privileged roles. Recovery codes: high entropy, slow-hashed, single-use, consumed atomically. TOTP replay protection (store last accepted time-step). **SMS/email codes are never an authentication or recovery factor.**
- Sessions: server-side or signed+revocable; cookies `HttpOnly`, `Secure`, `SameSite=Lax` (or Strict), host-only (no broad `Domain`); id ≥ 128 bits from a CSPRNG; **regenerate the id** on login, MFA, privilege change, password/email change, impersonation start/stop, account switch; sliding idle timeout + absolute lifetime; logout destroys the session server-side; a credential change destroys the user's other sessions and notifies the user.
- Tokens (API, reset, invite, unsubscribe, pairing, magic links): ≥ 256-bit random, **stored hashed**, single-purpose, expiring, revocable, compared in constant time; a mismatch returns the same response as "not found".
- JWTs (if used): pin the algorithm, verify signature/issuer/audience/expiry, short lifetime, refresh-token rotation with reuse detection, never store in web storage, never put secrets/PII in the payload.
- OAuth/OIDC: authorization-code + PKCE, validate `state`/nonce and exact redirect URIs.
- **Machine tokens** (device, API, integration): scoped to an explicit ability, hashed at rest, revocable, per-token rate limits and idempotent events; the tenant comes from the token's owner. Mobile/IoT clients SHOULD add device-bound request signing (non-exportable key) and certificate pinning with at least two pinned roots; nothing is handed out when the feature is off, the account is suspended or a kill switch is set.
- Login/reset/registration are **anti-enumeration**: identical status, body shape, timing class and emails for existing and missing accounts ("If an account exists, we sent a link"). Registration with an existing email shows the same success screen and notifies the existing owner.
- Login throttling per account+IP and per IP; successful reset clears the limiter; limiter keys hash the canonical identifier.
- Email change: re-authenticate, verify the new address before switching, notify the old one. Reset links use the configured public URL, never the request `Host`.
- Step-up ("fresh") re-authentication (MFA within the last ≤ 5 min) for takeover-class actions: refunds, deleting/anonymizing users, price changes, role/permission grants, key rotation, impersonation, disabling MFA.
- Privileged/first admin comes from environment or a one-time secure bootstrap, never a hardcoded or seeded password; must change it at first login.

## 3. Authorization

- **Every request, every time, on the server**: object-level (can this user touch THIS record) and function-level (can this role call THIS operation). A hidden or disabled button is not authorization.
- One policy per resource and action; controllers/handlers call it before any work. Nested/child routes check the parent's access and that the parent is not deleted.
- **Tenant scoping:** every query for tenant-owned data goes through a scoped accessor (`currentTenant.things.findOrFail(id)`); never a global `find(id)` of a client-supplied id. Uniqueness is per tenant. Foreign or hidden resources → **404**, identical to "does not exist".
- **Mass assignment:** explicit allow-lists of writable fields; writes use validated data only; identity fields (`tenant_id`, `user_id`, `role`, `plan`, `price`, `is_admin`) are never client-writable.
- Roles cannot grant permissions they do not hold; no member changes/removes the owner; custom roles cannot contain admin permission keys; notify the owner of role/rights changes.
- **Async work re-authorizes at execution**: a queued job, scheduled task or webhook handler re-loads the record and re-checks tenant, account state, feature entitlement, credential revocation and kill switches. Enqueue-time authorization is never trusted.
- Impersonation ("view as"): read-only, with a stated reason, step-up authenticated, visibly bannered, fully audited.
- Every account-state gate (2FA pending, email unverified, terms not accepted, password change required, suspended) lives in ONE middleware stack with an explicit allow-list; a test asserts every other route carries it.

## 4. Input and output

- Validate every input at the boundary with allow-lists: type, length, range, format, enum; reject or ignore unknown fields (never persist them). Canonicalize before validating and comparing (Unicode NFKC, strip zero-width and bidi control characters for identifiers, lowercase emails).
- **SQL:** parameterized queries/ORM only; never string-concatenate input; identifiers (sort column, table) come from a server allow-list; escape `LIKE` wildcards; sanitize full-text operators.
- **No shell with user data.** If a process must run, pass an argument array, never a shell string; prefer libraries over binaries.
- **Output encoding by context** (HTML text, attribute, URL, JS, CSS). Templating auto-escapes; raw-HTML sinks (`innerHTML`, `dangerouslySetInnerHTML`, `v-html`, `{!! !!}`) are forbidden for user data.
- Rich HTML (admin content, emails, CMS) is sanitized with a maintained allow-list sanitizer **at write AND at render**; never compile database strings as templates; placeholders use whitelisted substitution that escapes values.
- Never `eval`, `new Function`, dynamic `require/import` of input, or native deserialization (`unserialize`, `pickle`, Java serialization, unsafe YAML) of untrusted data. Disable XML external entities. Prevent path traversal (normalize, allow-list base directory).
- Parameter pollution: a scalar field received as array/object → 422. Strict `Content-Type` (JSON API rejects others). **Body size limits** at web server and app; decompression/zip-bomb limits; recursion/depth limits on JSON.
- Regexes on user input are linear-time or bounded (ReDoS). Header/CRLF injection and log injection: strip newlines from user values in headers and logs.
- CSV/XLSX export: neutralize cells starting with `=`, `+`, `-`, `@`, tab, CR.
- Redirects: same-origin or allow-listed only (no open redirect); the return path after login is validated.
- Client-side validation is UX only; the server is authoritative.

## 5. Web platform hardening (front + back)

- **CSRF:** cookie-authenticated state-changing requests require a token or double-submit/same-site protection; safe methods never change state.
- **CORS:** none by default; if needed, an explicit origin allow-list, never `*` with credentials, never reflect `Origin`.
- **Headers on every response:** `Strict-Transport-Security` (with preload consideration), `Content-Security-Policy` (nonces/hashes, no `unsafe-inline`/`unsafe-eval` for scripts, `frame-ancestors 'none'`, `object-src 'none'`, `base-uri 'none'`), `X-Content-Type-Options: nosniff`, `Referrer-Policy`, `Permissions-Policy`, cross-origin isolation headers where feasible. `Cache-Control: no-store` on authenticated/sensitive responses. One source of truth for headers.
- Trusted hosts = the configured host(s) only; trusted proxies explicit (never "all"); forwarded headers cannot change the client IP unless from a trusted proxy (test it).
- Third-party scripts/styles: avoid; if unavoidable, allow-listed in CSP, Subresource Integrity, loaded only where needed. No CDN for core assets: self-host.
- Sensitive data never in URLs, query strings, `localStorage`/`sessionStorage`/IndexedDB (tokens, credentials, personal data). UI preferences only.
- External links `rel="noopener noreferrer"`; `postMessage` handlers check `origin`; iframes sandboxed; clickjacking blocked.
- Source maps and debug builds are not served publicly in production unless deliberately protected.

## 6. SSRF and outbound requests

- Any URL the user can influence: HTTPS only, no credentials in the URL, scheme/port allow-list, **resolve DNS and block private, loopback, link-local, CGNAT, multicast, metadata (169.254.169.254, IPv6 equivalents) ranges for IPv4 and IPv6**, re-check on every request and after every redirect, **connect to the validated IP** (defeat DNS rebinding), no redirects into blocked ranges.
- Timeouts (connect ≈ 5 s / total ≈ 15 s), response size cap, no reading local files via `file://`, `gopher://` etc.
- Prefer server-side allow-lists of vendor hosts over user-supplied URLs.
- Every outbound call goes through one client per service with timeouts, error mapping and redacted logging.

## 7. Secrets and cryptography

- Secrets (keys, tokens, passwords, webhook secrets, DB credentials) live only in the environment or a secret manager: **never in the repository, container images, client bundles, logs, error messages, tickets, chat, prompts or analytics**. Ship an example env file with dummy values.
- Secrets are **write-only in APIs** (return `has_secret: true`, never the value) and never shown in admin UIs.
- Secret scanning in pre-commit and CI. **A leaked secret is compromised forever**: rotate immediately, revoke sessions/tokens, review access logs, consider history contaminated even after deletion.
- Rotation without downtime: support a current key plus previous keys; re-encrypt data in chunks; retire the old key only when the sweep reaches 100 %.
- Encrypt sensitive data at rest at field level where it is a secret (tokens, third-party credentials, TOTP secrets); keep keys separate from the data and backups.
- Use vetted libraries and current primitives: AES-GCM or ChaCha20-Poly1305, HMAC-SHA-256+, Ed25519/ECDSA/RSA ≥ 2048, Argon2id. **No custom crypto, no MD5/SHA-1 for security, no ECB, no static IV/nonce reuse, no `Math.random`/non-CSPRNG for tokens.**
- TLS ≥ 1.2 for all traffic including internal; certificate validation is never disabled; HSTS in production.
- Compare secrets/signatures/tokens in constant time.

## 8. File uploads and downloads

- Validate size (before buffering), extension allow-list **and** magic bytes, MIME; refuse double extensions, null bytes, RTLO; random storage names; store **outside the web root** (or private bucket); never execute uploads.
- Process (image re-encode, metadata strip, antivirus if available, thumbnails) in a queued job with time and memory limits.
- Serve only through authorization plus a short-lived signed URL; `Content-Disposition: attachment`, server-generated ASCII filename, `nosniff`; user-uploaded HTML/SVG is never served inline from the app origin.
- Per-tenant quotas; deleting an upload or closing an account removes the file and all derived data (thumbnails, search rows, caches).

## 9. Rate limiting and abuse

- Rate-limit: login, registration, password reset, MFA verify, contact/report forms, search, exports, expensive endpoints, messaging/sending, public APIs, webhooks receivers. Key per user + IP (not IP alone: shared NATs); return 429 with `Retry-After`.
- **Bot protection on every public form and unauthenticated write is mandatory and layered — BOTH layers, always, one never replaces the other:**
  1. **ALTCHA** — self-hosted proof-of-work (open-source widget + server-side verification with an HMAC key from the environment; no third-party captcha service).
  2. **Image captcha, Amazon-style** — a server-generated image of distorted alphanumeric characters with noise, with a refresh button.
  Plus rate limit, honeypot and minimum fill time. Applies to register, contact, abuse report, password-reset request, newsletter, invitations and every other public form. It runs **first, before any side effect**, in this order: rate limit → honeypot + minimum fill time → ALTCHA → image captcha → business validation.
- **ALTCHA rules:** challenges are single-use (used signatures stored until they expire, so a replay is refused), short-lived, verified with the HMAC key; an expired/replayed challenge returns a distinct retry signal (the client silently fetches a new one and retries once); a bot signal (honeypot, too fast, bad solution) returns a **generic** message, never the reason. The challenge endpoint is public, rate-limited and `no-store`.
- **Image-captcha rules:** answer kept **server-side only** (never in a cookie, HTML, URL, alt text or filename); single-use, short expiry (~5 min), case-insensitive; **a new image after every failed attempt and on refresh**; the image endpoint is `no-store` and rate-limited (it is CPU-heavy); challenge ids are unguessable. Provide an **accessible alternative** (audio or equivalent non-visual challenge) so assistive-technology users are not locked out; it still requires ALTCHA + honeypot + limiters.
- **LOGIN is adaptive: the image captcha is NOT shown on a normal login.** ALTCHA + honeypot + rate limits always apply; the image captcha becomes **required only after N failed attempts** (N is a dynamic setting within code bounds, default 3, never 0). The failed-attempt counter follows these non-negotiable rules:
  - **Server-side and persistent**, in the shared store (database/cache), keyed by the hashed canonical identifier (+ client IP; plus an IP-only counter). **Never** held in the session, a cookie, `localStorage` or any client state.
  - **A page reload (F5), new tab, private window, cleared cookies, a new or regenerated session, or a different user agent MUST NOT reset it.** The client never computes it: it learns `captcha_required` from the server (in each login response and, on page load, from a status check for the IP key) and re-asks the server after every reload.
  - It is reset **only** by a successful login for that identifier (or a completed password reset), or by expiry of a sliding window after the last failure (dynamic, default 1 h, bounds 15 min–24 h). A wrong captcha answer counts as a failed attempt; passing the captcha alone does not reset it.
  - **Enforced by the server:** once the counter ≥ N, a login without a valid captcha answer is refused with the generic message even if the client omits the field or hides the widget. The order is unchanged (limits → honeypot → ALTCHA → captcha → credentials).
  - **Anti-enumeration:** the counter increments for every submitted identifier, whether or not the account exists; `captcha_required` depends only on the counter, never on account existence. Because an attacker could raise a victim's counter, the captcha only adds friction — it never locks the real user out.
  - Sensitive follow-up steps (MFA verify after the password step) inherit protection from the pending-login state and have their own per-user limits (§2).
- Per-user limits also cover MFA verify, recovery-code use, confirm-password, password change, MFA disable and logout-others.
- Signup abuse: validate emails (syntax, role mailboxes, disposable-domain lists, MX), canonicalize (case, `+tag`, provider dot rules) for limits, remember hashed identities of abusive/deleted accounts to cap re-signup, uniform non-revealing refusals. Treat upstream disposable-domain lists as untrusted: byte cap, canary checks (a known disposable domain present, a major provider absent), never shrink the list on a failed fetch.
- Bot protection (both layers, and the login counter) is never switchable off by a production flag; tests disable it only inside the test.
- Business-logic abuse: quotas, coupon/referral/trial farming, negative or huge quantities, price manipulation, race-condition double spend — tested (`testing.md`).
- A kill switch lets staff block a tenant/user's outbound actions immediately.

## 10. Webhooks and callbacks (inbound)

- Verify the provider signature over the raw body (HMAC or provider verify API) with constant-time compare; enforce a timestamp window; refuse unverifiable requests with a generic 404; **empty secret ⇒ 503, never skip verification**.
- Idempotent by unique event id (or content hash); process in a queue; store unknown event types and alert (grouped).
- **Cross-check payload against own records** (amount, currency, account, object ids); never trust amounts or states from the payload alone. Rate-limit the endpoint.

## 11. Data protection and privacy

- Classify data; minimize collection; retention limits with automated purge; anonymize instead of hard-delete when records must survive (financial, audit).
- No personal data or secrets in logs, URLs, analytics, error trackers, test fixtures or third-party AI services. Personal data minimized in emails/notifications.
- Data-subject rights (export, deletion) are supported by design; consent recorded with version, timestamp, IP, user agent.
- Keep a privacy register (what personal data, why, where, how long); update it in the same commit as any change to processing.
- Backups are encrypted, access-controlled and restore-tested; production data is never copied to dev/test.

## 12. Logging, audit and monitoring

- Structured logs with request id, tenant id (not names), user id; **no secrets, tokens, passwords, full message bodies, query bindings or request bodies**; user values escaped against newline forging; configurable retention.
- A security event log records: login failed, lockout, MFA failed, 401/403/404-on-foreign-id, 419/CSRF, 429, bot rejected, webhook signature rejected, privilege changes. Ids, hashed identifiers and IP only.
- **Immutable audit log** for privileged actions (actor, action, target, before/after, IP, user agent, time); append-only, admin-only, covering users, roles, plans/prices, settings, permissions, legal documents, keys.
- Alerts to operators come from scheduled, grouped jobs (one message per recipient per run, escalating re-alerts), never inline from request handlers (email-flood barrier).
- Unexpected errors show the user a generic message + support reference (request id); details go only to server logs.

## 13. Supply chain and CI/CD

- Lockfiles committed; dependency audit clean before every release; pin versions; automated update PRs reviewed.
- **Before adding any package**: confirm it exists, is the intended one (typosquatting/hallucinated names), maintained, permissively licensed, small, with no open advisories; review install/postinstall scripts. Never `curl | sh`. Prefer the platform/framework over a dependency (`dependencies.md`).
- Private package names are reserved on public registries (dependency-confusion).
- CI: least-privilege tokens, secrets scoped to the jobs that need them, third-party actions/plugins pinned to a commit SHA, no secrets exposed to untrusted (fork) pull requests, artifact provenance/signing where available, protected release branch, required reviews/checks.
- Build artifacts contain no secrets; production images run as non-root, minimal base, no dev tools.

## 14. Production configuration and infrastructure

- Debug mode off, no stack traces or framework banners, HTTPS only, secure cookies, correct trusted hosts — enforced by a **production-readiness check** that runs in deploy and health, reports failures, and **never crashes the app at boot** over configuration. A test boots production with the example env.
- Web server serves only the public directory; denies dotfiles, VCS folders, backups, dumps, logs, env files; restricts HTTP methods; removes version headers; a test asserts those paths return 404.
- Database: dedicated least-privilege application role (no DDL, no superuser/file privileges at runtime), separate migration credentials, TLS, no public exposure, no default accounts.
- No install/migrate/cache-clear/debug endpoints on the web. Admin surfaces require MFA and are not needlessly public (IP allow-list/VPN where possible).
- Egress restricted where feasible; internal services authenticated (no "trusted network" assumption).
- Patch runtimes, OS and dependencies on a schedule; unsupported versions are not deployed.

## 15. Errors

- One error envelope for the API: `{ message, code, errors?, request_id }`. Messages are short, translated, generic and never help an attacker (no "user not found", "wrong password", "email exists", SQL text, constraint names, paths).
- 404 identical for missing, foreign and hidden resources; 403 only where revealing existence is harmless.
- A unique-constraint violation maps to a deterministic 409/422; a lost DB connection to 503.

## 16. Apps that use LLMs / AI features

- Treat all model input **and output** as untrusted: prompt injection can come from documents, web pages, emails, user text and tool results.
- Never put secrets or other tenants' data in prompts/context; scope retrieval by tenant and authorization *before* it reaches the model.
- Give tools the least privilege; require confirmation for destructive or outward-facing actions; validate/escape model output before rendering, executing, or using it in queries.
- No personal data to a model provider without a legal basis/DPA and owner approval; rate-limit and budget calls; log metadata, not content.

## 17. Security testing and review

- Tests per `testing.md`: authorization, tenant isolation, injection payloads, IDOR, rate limits, headers, uploads, webhooks, SSRF, anti-enumeration.
- SAST, dependency and secret scanning in CI; container/IaC scanning where used.
- Independent security review or penetration test before launch and after major auth/payment changes.
- Incident readiness: runbook for leaked secret, compromised account, data exposure (rotate, revoke, contain, audit, notify as the law requires); kill switches for outbound actions; restore procedure tested.
