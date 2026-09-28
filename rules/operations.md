# Operations rules — build, deploy, hosting, backups, health

Security config: `security.md` §13–14. Performance: `performance.md`. DB backups: `database-*.md`.

## 1. Build and release

- Builds are reproducible from lockfiles, run in CI (or locally) and produce artifacts; production servers do not need the frontend toolchain. Artifacts contain no secrets.
- Production caches (config, routes, views, events, OPcache/bytecode) are built at the release switch; the app reads environment variables only in the config layer or cached config breaks.
- **Zero-downtime deploys:** maintenance-free migrations (expand → backfill → switch → contract), workers restarted at the switch, the previous release's hashed assets stay reachable for one release, clients reload once on chunk-load errors.
- **Prove each deploy step by its side effect, never its exit code:** after migrate, pending migrations = 0; health reads a release-id written by the new code; a black-box check verifies HTTPS, HSTS and CSP headers, `/.env` and `/.git` return 404, and a forced error returns the generic JSON. The verdict is GREEN or RED.
- Rollback plan for every release (previous artifact + backward-compatible schema). Feature flags/kill switches for risky features.
- Any environment change is followed by a cache clear + rebuild.

## 2. Production readiness (checked at deploy and in health)

Fails when: debug mode on, public URL not HTTPS, secure-cookie flag off, a required key/secret empty, trusted hosts unset, initial-admin bootstrap unset, backups stale, scheduler heartbeat stale. Implemented as a registry of invariants that **reports** and never throws at boot (a boot exception breaks every CLI command). A test boots production with the example env.

## 3. Web server

Serves only the public directory; denies dotfiles and `*.bak/.sql/.log/.env/.git`; restricts HTTP methods; removes version/`X-Powered-By` headers; security headers from one source; HTTP→HTTPS redirect + HSTS; compression; canonical host redirect (www ↔ apex) at the server. A test asserts the denied paths return 404.

## 4. Jobs and scheduler

- Every scheduled entry avoids overlap and finishes within its slot; daily jobs catch up; heavy jobs are staggered off-peak.
- Every job declares timeout, tries, backoff, is idempotent, chunks big sets and re-dispatches with a cursor when it cannot finish.
- A worker runs as a supervised process; on constrained hosts a cron-driven worker with a max-time bound below the cron interval.
- Web requests never wait on slow work (emails, PDFs, sitemaps, alerts, anything O(rows)).

## 5. Disk, files, capacity

- Every growing store has a cap and a purge: uploads (per-tenant quota), logs (rotation + retention), backups (rotation per destination), failed jobs, old releases. Disk usage is visible to operators.
- Uploads, backups, storage and `.env` live outside the web root. Capacity numbers (memory, disk, quotas) are read from the environment, never hardcoded.
- Email goes through configured SMTP/provider, queued, within the provider's sending limit (a setting).

## 6. Backups — local always, off-site optional-by-configuration

- **Local first:** every backup (database + private files) is written as an encrypted archive to a backup disk outside the web root on every run, whether or not an off-site copy is configured. Backup code never depends on a binary that may not exist on the host unless proven there.
- **Off-site (S3-compatible or other):** active only when its full configuration is present; the admin shows "configured / not configured", never a value. Not configured → local only, **silently** (no error, banner, failed health check, deploy block or nag).
- Configured → local first, then the same encrypted archive streamed off-site. An off-site failure never deletes, blocks or delays the local backup; it is recorded per destination, retried next run, and raises a grouped alert.
- Only encrypted archives leave the server; credentials are limited to one bucket/prefix; retention runs per destination and never deletes the newest archive.
- **Restore is tested on a schedule** (from off-site when configured, else local); health reports last-backup age and destination status.
- Tests: off-site unset → local succeeds, health green; set → both destinations hold the same archive; failing → local kept, one alert.

## 7. Observability and alerting

- Health endpoints check the **work**: scheduler heartbeat age, last backup age, oldest queued job, failed jobs, sync ages, DB reachability — not just "process is up".
- Metrics (RED, saturation), structured logs with request ids, error tracker without personal data, traces on outbound calls.
- Alerts to staff come only from scheduled, grouped jobs with escalating re-alerts; an optional external dead-man's-switch ping hits a configured URL every minute (unset → nothing).
- Admin "system" screen: queues, failed jobs, scheduler heartbeat, disk usage, cache clear, redacted log viewer.

## 8. Key rotation

Application encryption keys rotate with previous-keys support plus a chunked re-encrypt command; the old key is retired only when the sweep reaches 100 % (or encrypted secrets are lost). Provider keys and webhook secrets rotate with overlap.

## 9. Constrained-host profile (optional; use when the target is shared/cheap hosting)

When the hosting floor is "PHP/Python/etc. + SQL database + cron every minute + outbound HTTPS + SSH":
- No long-lived processes or connections (no WebSockets, SSE, long polling, resident workers required): live UI data uses polling with deliberate intervals; queues are cron-driven with bounded `--max-time`.
- Cache, sessions, queue, rate limiter and locks use the database driver; code never *requires* an in-memory store (a bigger host may switch by configuration).
- No shelling out or external binaries (PDF, image, backup tools): pure-language libraries; only extensions present on the production build.
- No triggers/stored procedures/events/second database; invariants live in constraints and transactions.
- No install/deploy/debug web routes; debug tooling never ships.
- Code needing more than the floor ships only as an optional, default-off capability.
