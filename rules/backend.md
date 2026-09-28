# Backend rules — any language, any framework

Vocabulary is generic; map it to your stack (README). Security: `security.md`. Performance: `performance.md`. DB: `database-*.md`. Tests: `testing.md`.
Tags: **[multi-tenant]** applies only to multi-tenant products (drop otherwise).

## 1. Architecture and layering

- Layers: **route → request schema (validation) → authorization (policy) → use-case object (action/service) → data access → response serializer.** Handlers/controllers stay thin: no business logic, no queries beyond scoped lookup.
- Business logic lives in single-purpose use-case objects (`StartJob`, `RecordPayment`) that can be called from HTTP, jobs, CLI and tests alike; one path for every source of the same operation (user, schedule, API, automation) so guards and audit apply everywhere.
- **Modular monolith by default:** modules own their models, use-cases, policies, jobs, routes, migrations, tests. A module never queries another module's tables or internals; it calls that module's public use-cases/contracts or listens to its events. Shared code lives in one kernel.
- **Request-scoped state is never held by long-lived objects.** Controllers, services and singletons may be cached across requests: never inject/store the current user, tenant, permissions or request in them; resolve per call. Test a role/user switch between two requests. (Same for SSR servers: no module-level per-user state.)
- Dependency injection over globals; interfaces for external services (payment, messaging, storage) so they can be faked.
- Types everywhere (parameters, returns, properties), strict mode, enums for every state/type, `final`/sealed by default, no `mixed`/`any` escape hatches.
- Business values (limits, prices, thresholds, intervals, retention) are configuration (`configuration.md`), never literals.
- The application name/brand comes from configuration, never hardcoded in code, seeders, emails or translations.

## 2. Identity and tenancy [multi-tenant]

- The current user, tenant (account), role and plan come **from the session/token only**. Never accept tenant/user/owner ids, roles, plan ids or prices from the client; a body carrying one is ignored and a test proves it. Switching tenant is one endpoint that checks membership.
- Every tenant-owned row has `tenant_id` (FK, first column of composite indexes). Uniqueness is per tenant (`unique(tenant_id, …)`), except true global identities (user email).
- All access through **scoped accessors** plus a policy; never an unscoped `find(clientId)`. A model-layer guard sets `tenant_id` on create and throws if it changes on update (a safety net under scoped access, not a replacement). Consider database row-level security as defense in depth (`database-postgres.md`).
- **Every id in a request is scoped-validated** (exists AND belongs to the tenant). Another tenant's id → 404/422, never a silent link.
- Token callers (device/API tokens, no session) take the tenant from the token's owner; same scoping.
- Jobs, listeners and scheduled tasks have no session: they receive the tenant id explicitly, re-load it, scope every query, and re-check authorization (`security.md` §3). Never read the "current user" inside a job.
- Module/feature gating: no access ⇒ **404** (never 403), so hidden features are never revealed. Entitlements are cached per tenant and invalidated on plan/assignment change.
- Notification recipients are resolved by permission, never by role name; the owner is told about member/role changes.

## 3. Validation and input

- A request schema for every write: strict types, lengths, ranges, enums, formats (E.164 phones, IANA time zones, ISO currency). Use `validated()` output only; never spread the raw request into a model.
- Only parameterized queries/ORM; identifiers from allow-lists (`security.md` §4).
- Every id/enum/option in a request is validated against the tenant's data (§2).
- Search input: escape LIKE wildcards, neutralize full-text operators, cap length; a search never returns a 500.
- Names used for uniqueness, slugs or display are NFKC-normalized with zero-width/bidi characters stripped.
- Admin-editable text is never code: no compiling stored strings as templates; whitelisted placeholder substitution with escaping.
- Cache stores arrays/scalars only; value objects are rebuilt after the read.

## 4. Lists, pagination, queries

- **Every list endpoint is server-side paginated from day one, whatever today's size** — user area, admin, dropdown search, public islands, nested lists, export previews. The API contract never changes later.
  - Offset (`page` + `per_page`) for small/admin tables; **cursor/keyset** for large, infinite or history lists. Default 25, hard max 100; dropdown search default 20.
  - No unbounded `limit`, no `?all=1`, no whole-table read, no unpaginated array. A small code-bounded set (enum options, ≤ 50) is a single resource, not a list.
- Default order `created_at desc, id desc` (stable, no duplicates/gaps between pages). Rows include `created_at`. Sorting/filtering columns are allow-listed and indexed.
- Filter, sort, count and aggregate in SQL or rollups, never in application loops over fetched rows.
- Query cost is bounded by page/tenant slice, never O(platform).
- Exports/imports/backups stream (cursors, chunks); CSV/XLSX cells are injection-neutralized (`security.md` §4).
- A route dataset test calls every list endpoint and asserts the pagination meta.

## 5. API contract and errors

- One response contract for the whole API, built only by shared helpers (response helper, base resource/serializer, global exception renderer). A handler never returns a raw array/model or a bespoke envelope; an architecture test enforces it.
  - Single: `{ "data": {…} }` (201 on create with the resource).
  - List: `{ "data": […], "meta": { "pagination": {…} } }` — page: `{type:"page", page, per_page, total, last_page}`; cursor: `{type:"cursor", per_page, next_cursor, prev_cursor}`. Extra context (facets, totals) under `meta`.
  - Action without body: 204; action changing a resource returns the resource.
- Fields: `snake_case` (or one consistent convention); opaque UUID/ULID `id`; ISO-8601 UTC timestamps; money `{amount:"123.00", currency:"USD"}` as string decimal; enums as stable English snake_case; booleans `is_/has_/can_`; server-computed capabilities under `can:{}`; declared fields are always present (no null-vs-missing ambiguity).
- Error envelope for all non-2xx: `{ "message", "code", "errors"?, "request_id" }`; translated, generic, never enumerating.

| Case | Status | `code` |
|---|---|---|
| Not authenticated | 401 | `unauthenticated` |
| Forbidden | 403 | `forbidden` |
| Missing / other tenant / hidden feature | 404 | `not_found` |
| CSRF mismatch | 419 | `csrf_mismatch` |
| Validation | 422 | `validation_failed` |
| Locked | 423 | `locked` |
| Rate limited (+`Retry-After`) | 429 | `too_many_requests` |
| Plan limit reached | 403 | `plan_limit_reached` |
| Edit conflict / duplicate | 409 | `conflict` |
| Server error | 500 | `server_error` (generic + request id) |

- One global exception renderer builds these; no try/catch in handlers reshaping errors; never swallow exceptions (log with context, no secrets, then rethrow/render). Unique violations map to 409 (or a field 422), lost DB connection to 503 — never SQL text or constraint names.
- **Idempotency:** creates and side-effecting actions (payments, refunds, commands, sends, invitations, imports) accept an `Idempotency-Key`; duplicates return the first result; an in-flight duplicate gets a retryable 409. Webhooks and inbound events are idempotent by unique external id.
- Versioned under `/api/v1`; breaking changes get a new version with a documented deprecation window; a renamed public URL gets a permanent 301. Route names/paths in one language, kebab-case plural paths, dotted route names.
- Machine-readable API description (OpenAPI) generated from code and committed; the client is typed from it.
- Everything a signed-in user can read in the UI is also readable through the API (read parity), same policies, same pagination.

## 6. Data integrity

- **Invariants are enforced by the database** (unique, FK, CHECK, NOT NULL), not only by pre-checks; multi-step writes run in one transaction; jobs/mails/events dispatched inside a transaction run **after commit**.
- **Deletion:** soft-delete parents only; children are hidden through the parent and every child route/query checks the parent is not trashed (tested); child FKs `RESTRICT`, never cascade. **Financial and audit records are immutable** (payments, invoices, subscriptions, ledgers are never deleted with a parent; corrections are new rows). After the retention period, **anonymize** personal data instead of hard-deleting.
- **Optimistic concurrency** on full-form edits of shared records: a `version` column echoed by the form; `UPDATE … WHERE id=? AND version=?` → 409 on mismatch. Not for toggles, status transitions or bulk actions.
- **Shared mutable state** (balances, payments, quotas, counters, inventory, unique resources): design the concurrency explicitly — constraints, transactions, row locks, unique indexes, idempotency keys or versions; never check-then-act without one of them. Never rely on UI disabling to prevent double submission.
- Money: fixed-point decimal + currency code, never floats. Ids: opaque, non-sequential when exposed (UUIDv7/ULID or a public id column).
- Conditional uniqueness (one active X per Y) is enforced by the database (partial unique index or the engine's equivalent, see `database-*.md`).
- Every table is either immutable-business (payments, invoices, audit) or has a retention purge job; logs are never kept forever by accident.
- Idempotent retries never double-charge, double-bill or double-send.

## 7. Time — UTC only (NON-NEGOTIABLE)

- **Every time in every database is UTC, never another time zone** (all tables, logs, queues, sessions, cache, backups, exports). App zone, runtime zone, scheduler zone and DB session zone are UTC and never changed at runtime; API timestamps are ISO-8601 with `Z`; no per-value time-zone or offset column is stored to interpret a timestamp (the tenant/user display zone is a setting applied only at display). No exception, no "local time for this one table". The DB never converts zones; a user's local date/time is converted to UTC **before** storing. Calendar-only values (billing day, payment date) are `date`, never zone-converted.
- Convert to the tenant/user zone (IANA, validated) only at display or when computing a calendar day (server) / `Intl` (client).
- **The server owns "now":** decisions derived from time ("late", "expired", "billed today", "grace ends") are computed server-side and returned as fields; the client never decides them from its clock. Calendar-day caps ("per day") use the tenant zone.
- A test asserts app zone, runtime zone and DB session zone are UTC and that no SQL converts zones.
- **Daily jobs catch up, never skip** (due when last success > 24 h old).

## 8. Jobs, queues, scheduling

- Slow, external or O(rows) work goes to jobs. Every job: idempotent, declares timeout/tries/backoff, chunks big sets, re-dispatches with a cursor when it cannot finish, and re-authorizes on execution (`security.md` §3).
- Payloads stay compatible across one release; restart workers at deploy. Dispatch after commit.
- Scheduler entries never overlap and finish within their slot; heavy jobs are staggered.
- Each automatic email/notification inserts a unique (recipient, kind, subject, period) ledger row with the enqueue, so it is sent once and catch-up is safe. User-transactional mails are queued; security mails are exempt from frequency caps.

## 9. Privileged back-office (admin)

- Server-side role flag **and** policy on every admin route; MFA mandatory; short session; step-up for takeover-class actions (`security.md` §2).
- First admin from environment/bootstrap; production refuses to start without it (readiness check); never a seeded password.
- **Immutable audit** of every admin action and every platform-model change. Manual overrides (payment confirmation, trial/grace extension) need an external reference and a cumulative cap.
- Health checks measure the **work**, not liveness: scheduler heartbeat age, last backup age, oldest queued job, failed jobs, feed-sync age. Failures feed the grouped alert.
- Staff can: suspend/unsuspend, force logout, reset MFA, set plan, view-as (read-only, reason, audited), kill switch, maintenance mode, announcements, feature switches.

## 9a. Feature and entitlement gating

- A feature can be globally off (disappears from menus, routes, API, jobs, schedules, emails; data kept), hidden (visible only to assigned tenants), or plan-gated. Dependencies between features are declared; a feature whose dependency is off is off.
- Limits (seats, items, messages, retention) live in the database, are edited by admin, never hardcoded. Over limit after a downgrade: data is kept, read-only above the limit, never deleted.
- Enable/toggle endpoints re-check entitlement server-side (test: a free user cannot enable a paid/hidden feature).

## 10. External services, uploads, email

- One client per external service: timeouts, SSRF guard for user URLs, retries with backoff only for idempotent reads, error mapping to your own translated messages, redacted logging (`integrations.md`).
- Uploads: `security.md` §8; processed in queued jobs with limits; served via policy + short-lived signed URL.
- PDF/report engines run with remote fetching off and a fixed local assets directory; every value escaped.
- No personal data or private content goes to third-party AI providers from servers without owner approval (`security.md` §16).

## 11. Conventions and migrations

- Formatter + linter + static analysis at the strictest achievable level; no baselines to hide new issues.
- Config from files + environment; environment variables are read only in the config layer; DB-resolved settings are cached and invalidated on write.
- Migrations are forward-only once pushed; **expand → backfill → switch → contract** across releases for live tables; each migration runs on every supported engine; migrations never depend on application code that may change.
- Seeders are idempotent (upsert by unique key), transactional, split into production seeders (plans, features, settings, bootstrap admin from env) and dev-only ones; no hardcoded brand or shared plain passwords in production seeders.
- Public URL/route/table/column/enum names are English, stable and never reused for a different meaning.
- Logs: `security.md` §12; privacy register updated with any change to personal-data processing.
