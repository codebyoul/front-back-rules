# Performance rules — NON-NEGOTIABLE, every project, front and back

Performance is part of "done". Measure before and after any hot-path change and put the numbers in the commit body.
Defaults below are floors a project may tighten. DB engine specifics: `database-mysql.md`, `database-postgres.md`. Perceived speed and states: `ux.md`.

## 1. Principles

- **Measure, don't guess.** No optimization without a measurement; no shipping a known anti-pattern (N+1, unbounded query, blocking call in a hot path) "to optimize later".
- Design for the worst realistic case: 100 000 rows in a tenant, 1 000 concurrent users, a slow 3G phone, a slow third-party.
- **Query cost is bounded by the page or tenant slice, never by the platform** (no O(all rows) on a request path; dashboards read pre-computed rollups).
- Performance never trades against security or correctness: do not cache authorization decisions without invalidation, do not skip validation, do not weaken checks for speed.
- When performance and storage genuinely conflict, performance wins: ship the fast shape and report its storage cost.
- Prefer removing work (less data, fewer requests, less JS) over speeding it up.

## 2. Budgets (measured, enforced in CI where possible)

| Area | Default budget |
|---|---|
| API read p95 | ≤ 300 ms (excluding third-party calls) |
| API write p95 | ≤ 500 ms |
| A request that needs slow work | returns within ~1 s and moves the work to a queue |
| DB query in a hot path | ≤ 50 ms typical; > 200 ms needs a justification and a plan |
| Queries per request | bounded and asserted in tests (N+1 fails the build) |
| LCP / INP / CLS (p75, mobile) | ≤ 2.5 s / ≤ 200 ms / ≤ 0.1 |
| TTFB (p75) | ≤ 800 ms |
| Initial JS of the app shell | ≤ 200 KB gzip (route/feature chunks lazy-loaded) |
| Content/marketing pages | ≈ 0 JS until an interactive island is used |
| Main-thread task | ≤ 50 ms; no long tasks during interaction |

## 3. Backend

- **N+1 is forbidden.** Eager-load relations; disable lazy loading outside production (fail loudly in dev/test); assert query counts in tests for list endpoints.
- Select only needed columns; no `SELECT *` on wide tables; no fetching a collection to count or filter it in application code — filter, sort, aggregate and count in SQL or in pre-computed rollups.
- **Everything list-shaped is paginated** (`backend.md` §4); large/infinite lists use cursor (keyset) pagination; `per_page` is capped; never `?all=1`.
- **Stream, never slurp:** exports, imports, backups and large queries use cursors/chunks and streamed responses; never build a whole CSV/PDF/ZIP in memory.
- **Index every foreign key and every filtered/sorted column;** composite indexes match the real query shape (equality columns first, then range/sort); an index must earn its place (proven by EXPLAIN) and unused/duplicate ones are dropped.
- **Queues for slow or external work** (emails, PDFs, webhooks, bulk sends, thumbnails, exports). Jobs declare timeout, retries and backoff (exponential + jitter), are idempotent, process big sets in bounded chunks, and dead-letter after final failure. Bounded concurrency; backpressure instead of unbounded queues.
- **External calls:** always time-bounded (connect + total), retried only when idempotent, protected by circuit breaker/bulkhead where a slow dependency could exhaust workers; degrade gracefully when they fail.
- **Transactions are short:** no external calls, no user waits inside a transaction; lock rows in a consistent order; keep lock scope minimal.
- **Caching** is deliberate: cache only what is measured slow and read-heavy; keys include every variable (tenant, user, locale, version); **explicit invalidation on every write** (versioned keys / generation counters); TTL is a safety net, not the mechanism; protect against stampedes (locking or jittered TTL); store only serializable primitives (rebuild objects after the read); never share per-user data across users; cache tests render twice (cold + warm).
- HTTP: compression (brotli/gzip), `ETag`/`Last-Modified` + 304, correct `Cache-Control` (immutable for hashed assets, `no-store` for private data), keep-alive/HTTP2+, request/response size caps, no chatty APIs (offer batch/expand endpoints instead of many round trips).
- Connection pooling for DB and HTTP clients; pool size from measured concurrency; no connection per request.
- No blocking I/O or sleeps on request paths; no unbounded in-memory collections; no O(n²) over user-controlled n; bounded regexes.
- Heavy scheduled work is staggered off-peak (never all at the same minute), chunked to fit its slot, protected against overlap, and **catches up** (a daily job is due when its last success is > 24 h old).
- Resource sizes (memory, disk, quotas, batch sizes) come from configuration/environment, never absolute literals.
- Rate limits and quotas protect capacity; readiness checks fail before overload.
- **Observability:** RED metrics (rate, errors, duration) with p50/p95/p99, slow-query log, tracing on outbound calls, queue depth/age, failed jobs, cache hit rate; alerts on budgets.
- **Load test** before launch and after any hot-path change (realistic data volume); record before/after.

## 4. Frontend

- **Ship less JavaScript:** route-level and feature-level code splitting, lazy-load below-the-fold and rarely used UI (dialogs, editors, charts), tree-shakeable imports, no barrel imports that defeat tree-shaking. Check a dependency's size before adding it (`dependencies.md`). A bundle-size budget fails CI.
- **No render-blocking JS** on public pages; stylesheets are in `<head>`; critical CSS small; third-party scripts deferred/async and justified.
- **Images:** modern formats (WebP/AVIF), responsive `srcset/sizes`, explicit `width`/`height`, lazy-load below the fold, prioritize the LCP image (no lazy on it), no oversized originals.
- **Fonts:** self-hosted, subset, `font-display: swap|optional`, preload only the critical face(s).
- **Layout stability:** reserve space (skeletons at final size, image dimensions); nothing shifts after load (CLS).
- **Data fetching:** no waterfalls (start independent requests together); dedupe identical requests; abort superseded requests; deliberate cache freshness per data type; prefetch on intent (hover/touch-start); load lazily only what visible UI needs; no refetch storms; pause polling when the tab is hidden and back off on errors.
- **Rendering:** virtualize lists over ~100 rows; debounce/throttle search, resize, scroll handlers; memoize only measured hot spots; keep state as local as possible; avoid re-rendering large trees on every keystroke; move heavy CPU work to workers; passive event listeners; animations on `transform`/`opacity` only.
- **Perceived performance:** acknowledge every action within 100 ms; skeletons instead of blank areas; keep old content while refreshing; optimistic updates only where safe (`ux.md` §4); honest progress for long operations, never fake speed.
- Build output is hashed and long-cached; the previous release's assets stay reachable for one release; the app reloads once on a chunk-load error.
- Test on a low-end phone profile and throttled network; Lighthouse/Web Vitals in CI on key pages.

## 5. Review checklist (any change touching a hot path)

- [ ] Query count and slowest query measured (before/after).
- [ ] New/changed filters and sorts have matching indexes (EXPLAIN attached).
- [ ] Lists paginated; no unbounded loops or in-memory accumulation.
- [ ] Slow/external work is queued, bounded, idempotent.
- [ ] Cache keys and invalidation defined and tested.
- [ ] Bundle size delta and Web Vitals checked.
- [ ] Load behaviour at 10× current data volume considered.
