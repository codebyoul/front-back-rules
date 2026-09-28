# Configuration rules — dynamic by default

Rule: when "a constant in code" and "a value in the database" are both possible for a business value, choose the database, edited from an admin area with bounds, audit and cache invalidation.

## 1. What must be dynamic

- **Business values:** prices, currencies, plan limits and features, trials, coupons, discounts, thresholds (late/overdue days, low-battery %, offline minutes), intervals (polling, digests, reminders), retention periods, rate limits, quotas, caps, TTLs, timeouts within safe bounds.
- **Content:** message/email templates, announcement banners, help articles, CMS pages, legal documents, labels of admin-defined options.
- **Option lists:** payment methods, disposable-email lists, role mailboxes, stop keywords, price regions, supported countries/currencies, device/product presets.
- **Switches:** features (global on/off, visibility, plan access, assignment), payment providers (enabled, order, currencies), registration open/closed, maintenance mode, public forms, notification channels, default theme.
- **Roles:** custom roles are data built from code-defined permission keys.

## 2. What stays in code

- **Secrets:** always environment/secret manager; the admin shows "configured / missing", never a value; never stored in a setting.
- **Security floors:** minimum password length, hashing algorithm, breached-password check, HMAC algorithms, webhook timestamp-window upper bound, TLS-only user URLs, SSRF blocked ranges, CSP shape, mandatory staff MFA, step-up window upper bound. A setting may make a floor *stricter*, never weaker; bounds are enforced in code.
- **Protocol facts:** encoding limits (e.g. message segment sizes), E.164, provider API shapes, message formats, event and failure codes.
- **Keys:** permission, event, limit and setting keys map to code paths: declared in code (manifests), synced to the DB; their *values* are dynamic.
- **Infrastructure:** queue names, cache store, DB connection.

## 3. How a dynamic value is built

1. Declare it in the owning module's registry: key, type, default, bounds, allowed scopes (`platform → plan → account → user`), who may edit, step-up flag, i18n label and help keys.
2. Read only through the settings/entitlements service; never a config file for a business value and never a literal.
3. Validate every write against declared bounds on the server; the admin form is generated from the declaration (type → control, bounds → validation).
4. Cache resolved values (per tenant where scoped) and **invalidate on every write** (versioned keys/generation counters when the cache has no tags); TTL only as a safety net.
5. Audit every change (actor, before/after, IP, time). Takeover-class settings (prices, provider enable, hidden-feature assignment) need step-up authentication.
6. A default always exists, so a fresh install works with an empty settings table. Seeders upsert declarations and never overwrite values an admin changed.

## 4. Tests and review

- Each dynamic value: a test that changing the value changes the behaviour, and a test that out-of-bounds writes are rejected (422).
- A registry test fails if code reads an undeclared key or a key lacks an i18n label.
- Review rejects business literals: a magic number for a limit, price, threshold, interval or retention is a defect.
- Environment variables are read only in the config layer; production caches configuration; every environment change is followed by clearing and rebuilding caches.
