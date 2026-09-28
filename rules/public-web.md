# Public web rules — i18n, SEO, legal, abuse reporting [optional module: public site / multi-language / regulated data]

## 1. Languages and localization [i18n]

- Supported languages are decided per project; every user-facing string exists in **all** of them, a CI check fails on missing keys, and no text is hardcoded in components, templates, emails or messages. Code identifiers stay English.
- **Locale resolution order:** the signed-in user's saved locale → `locale` cookie → `Accept-Language` → default. The cookie is set on first resolution and updated whenever the user changes language (via the API, which also saves the user's locale); 1 year, `Secure`, `SameSite=Lax`.
- **RTL languages:** `<html lang dir="rtl">`; logical CSS properties everywhere (`margin-inline-start`, `ps-/pe-/ms-/me-`, `text-start`), never left/right-specific; icons that imply direction are mirrored via a registry; test every screen in RTL. No uppercase or letter-spacing in scripts where it is wrong.
- Layouts tolerate +30 % text length; no text in images; pluralization/gender/ICU messages via the i18n library, never string concatenation.
- Dates, numbers, currencies and units are formatted per locale (`Intl` on the client, the language's formatter on the server); **store UTC, display in the user's/tenant's time zone**.
- Emails, notifications and error messages are translated too (error codes stay stable English snake_case).
- Public URLs: default language at `/`, others under a language prefix (`/fr/...`); **slugs stay English**; `hreflang` alternates + `x-default` on every page; **no automatic language redirects** (they harm SEO and users): show a dismissible suggestion banner.

## 2. SEO — public pages (target Lighthouse SEO = 100)

- Each indexable page: unique `<title>` (≤ 60) and meta description (≤ 160) per locale; canonical URL; `hreflang`; Open Graph + Twitter card; structured data as appropriate (Organization, WebSite, SoftwareApplication/Product on pricing, FAQPage, BreadcrumbList); exactly one `<h1>` different from the title; semantic landmarks; descriptive crawlable links (`<a href>`, no JS-only navigation); images in WebP/AVIF with `width`/`height`/`alt`; real 404/410 status codes.
- `sitemap.xml` (all locales, `lastmod`) and `robots.txt`; app, admin, auth and API pages are `noindex` and disallowed (they still meet every other point).
- Public pages are server-rendered (or statically generated), ship ≈ 0 JavaScript until an interactive island is used, have no render-blocking JS, self-hosted fonts with `font-display`, critical CSS only, HTTP 304 + compression + a page cache that never stores personal data.
- Targets: Lighthouse Performance/Accessibility/Best-Practices ≥ 95 on mobile in every locale (`performance.md`).
- **A public route ships only with an automated test asserting these tags in the rendered HTML.** A page below the SEO target blocks the commit and the deploy.
- Probe public pages with a cache-bypassing request or a test, never a bare request after an edit; purge the page cache after every asset rebuild.

## 3. Legal pages and consent

- Publish, versioned in the database and available in every language: Terms of Service, Acceptable Use Policy, Privacy Policy (GDPR/CCPA-style rights as applicable to your users), Cookie Policy + consent banner (**only essential cookies until consent; "Reject" as visible as "Accept"**), Refund/Cancellation Policy (if charging), Data Processing Addendum (if processing data for customers), Disclaimer (third parties, no warranty where applicable), Legal notice/company information.
- State clearly: user responsibilities and compliance with applicable law; forbidden uses (spam, fraud, phishing, harassment, illegal content, abuse of verification flows, evading limits); indemnification; liability cap; immediate suspension for abuse; cooperation with lawful requests; sub-processors; data retention periods.
- **Record acceptance** (document version, timestamp, IP, user agent) at signup and on every new version; publishing a new version forces re-acceptance where the law requires it and blocks the app until accepted.
- Legal texts are **drafts until reviewed by a qualified lawyer** before launch and are marked as such in the admin until approved.
- **Company identity fields: filled → displayed, empty → hidden.** They never block a deploy, appear in health checks, banners or reminder emails, or use "required/non-compliant" wording in owner-facing screens. (Acceptance by *users* is different: that does gate app use.)
- The privacy register (what personal data, why, where, how long) is updated in the same commit as any processing change; retention is limited and configurable.

## 4. Abuse reporting

- A public abuse-report form (with the full bot protection of `security.md` §9) feeds an admin inbox; staff can act (suspend, kill switch) with an evidence log; every action is audited.
- Reports never reveal reporter identity to the reported party; alerts to staff are grouped (`security.md` §12).

## 5. Accessibility statement (SHOULD)

Publish the accessibility target (WCAG 2.2 AA), known gaps and a contact channel; test key flows with keyboard and a screen reader before launch.
