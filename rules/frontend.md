# Frontend rules — any UI framework (React, Vue, Svelte, Angular, Solid…), SPA or SSR

Generic vocabulary: *server-state cache library* (TanStack Query, SWR, Apollo, RTK Query…), *form library*, *schema validator* (zod, valibot, yup…), *component kit* (design-system components you own), *router*.
Security: `security.md` (§4, §5). Performance: `performance.md` §4. States/feedback/navigation/forms UX: `ux.md`. Tests: `testing.md` §3.

## 0. Premium UI standard (NON-NEGOTIABLE — before ANY frontend change, however small)

- Every component, page and element (tiny or large) is production-grade and premium, at the level of a top-tier SaaS. Never basic, generic, placeholder-looking or low-effort.
- Small components get the same design care as major pages.
- Strong hierarchy, spacing, typography, alignment, responsiveness, accessibility.
- Every applicable state is designed: hover, focus, active, loading, empty, error, disabled, success.
- One cohesive system via kit + tokens; nothing designed in isolation.
- Before done, visually review; fix anything unfinished, inconsistent or generic.

## 1. Stack discipline

- One UI framework, used as designed: never alias/shim/replace it (e.g. a compat layer) and never swap or drop a framework dependency to chase a metric; improve performance inside the framework (splitting, laziness, fewer deps).
- **Rendering model is a project decision** (client-only SPA, SSR, SSG, islands). Rules below apply to all; under SSR/SSG additionally: server-only secrets never reach the client bundle; **no module-level per-user/per-request state** (it leaks between users); hydration must be deterministic; auth checks run on the server render too.
- One component kit, owned by the project and themed by design tokens; never hand-roll what it provides (button, input, select, dialog, sheet, table, tabs, card, badge, alert, toast, field, combobox, calendar). Feature components compose kit components; a variant belongs in the kit, not in a one-off class list. Look up a kit component's current API in its docs before using it.
- Runtime dependencies are a short allow-list (`dependencies.md`): framework, router, server-state cache, form library, table/virtualization helpers, schema validator, i18n, the component kit's primitives. Everything else is the platform (`fetch`, `Intl`, CSS transitions).
- One typed API client (`fetch` + CSRF handling); no interceptor that swallows errors; types generated from the API description.
- Tailwind-style utility CSS or another styling system: pick one; no inline styles, CSS-in-JS mixtures or raw hex colours in components.

## 2. Data fetching and server state

- **Every** API read goes through the server-state cache library; every write is a mutation that invalidates the affected keys on success. No raw `fetch` in components.
- Route guards/loaders fetch through the cache's "ensure" API so prefetch-on-hover is deduplicated, never a raw call.
- Query keys are structured and centralized per feature (`['items']`, `['items', id]`, `['items', {page, search}]`).
- **Freshness is deliberate:** tenant data (state, balances, messages) `staleTime` 0; only static data (plans, translations, feature registry) gets a longer time, with a comment.
- **Lists are paginated from the first commit** (`page`/`cursor` in the key); never assume a collection is small.
- **Lazy data:** load only what visible UI needs; dialog queries run only when open; tab content mounts only when active; never prefetch "just in case".
- **Dialogs open when their data is ready** (fetch on click, mount once at final size; keep previous data when the key changes while open) — never an empty shell that jumps.
- **Big option lists** (> 50 possible items) are async searchable comboboxes on a paginated search endpoint, debounced; never download a whole list to filter locally.
- **Table/list state lives in the URL** (page, size, sort, search, filters, tab), validated by a schema, so back/refresh/share work.
- **Live data:** polling with a deliberate interval, paused when the tab is hidden, backed off on errors (or server push where the platform allows).
- Race/stale-response rules: `ux.md` §10.

## 3. Security in the browser

- **Cache isolation:** clear the entire client cache **before** setting the user or navigating on every auth transition — login, logout, MFA confirm, email verification, password change, impersonation start/stop, session-expiry (401). Prevents showing one account's data to another.
- **Error before data:** a detail/edit view checks `isError` before rendering `data` (caches keep stale data after a failed refetch).
- Never send user, tenant, role, plan or price ids in requests; the server resolves them.
- **UI gating uses server-computed capabilities** (`can('items.start')` from the "me" endpoint), never role names, plan names or settings read in the browser. Hiding is UX; the policy is the authority. When an entitlement ends, drop that feature's cached queries.
- **Routes and menus are registered only for the features the "me" endpoint returns** (entitlements); an inaccessible or hidden feature exists nowhere in the UI, and its cached data is dropped when access ends.
- No tokens, credentials or personal data in `localStorage`/`sessionStorage`/IndexedDB (UI preferences only).
- No raw-HTML sinks with user data, no `eval`/`new Function`, no inline handler strings; external links `rel="noopener noreferrer"`; same-origin redirects only.
- Idle timeout: warn before the session expires, then log out and clear the cache.
- No secrets in the bundle: anything shipped to the browser is public.

## 4. Side effects discipline (framework-agnostic)

Derived state is computed, not synchronized by effects. Do not use effect hooks/watchers to: derive state from state, fetch data (use the cache library), react to a flag by calling an action (use the event handler/mutation), reset state on id change (key the component by id). The only allowed effects are one-time mount syncs (focus, third-party widget, browser subscription) through one shared wrapper. If your framework has effect primitives (React `useEffect`, Vue `watch`, Angular `effect`), apply this rule to them.

## 5. Forms

- **One form library for every form** that collects or validates input (app, admin, public pages/islands). Client validation runs through a schema validator that mirrors the server's request schema; the server remains authoritative. No second form library, no uncontrolled `FormData` parsing, no hand-rolled validation state. Plain GET navigation forms without validation (search box) may stay HTML.
- **Every form submits through one shared hook/helper** that: runs schema validation; on invalid input shows an error summary at the top and scrolls to and focuses the first invalid field; maps server 422 errors to fields and the summary; blocks re-submission while pending; announces status to screen readers.
- Fields use the kit's field components with label, description, error, `aria-invalid`, `aria-describedby`; labels always above inputs; side-by-side fields only through one shared grid component (single column on mobile).
- Every field: translated label, example placeholder, description when not obvious. Dates use the kit's date picker (never native date inputs for date-only values); date-only values are never zone-converted; instants convert only through the `format` module.
- **Never disable Submit because the form is invalid** (the user must learn what is wrong); disable only while pending or when the user lacks the right. Cross-field/non-schema rules go through the hook's failure path, never a silent return.
- Dialog forms are keyed by entity so they remount with fresh defaults; closing resets form and submit state.
- Full-form edits send back their `version`; a 409 keeps the form open with every typed value and a clear "changed by someone else — reload" message (never a toast that loses work).
- Long forms, error summaries, sticky action bar, unsaved-changes guard: `ux.md` §5.
- **Public forms** carry BOTH protections of `security.md` §9 through one shared wrapper: the ALTCHA widget (self-hosted, status always visible: verifying / verified / failed) and the Amazon-style image captcha with a refresh button and an accessible alternative, plus the honeypot; submit is disabled until ALTCHA is ready (or while pending). Submission goes through one helper that attaches the payloads; on a "renew challenge" error it fetches a new challenge and retries once silently; after any failed submit the captcha image is refreshed and its field cleared.
- **Login form:** ALTCHA always; the image captcha appears **only when the server says `captcha_required`** (response of a failed login, or the status check on page load). The client never counts failures or stores the counter anywhere: after F5 or a new tab it asks the server again, and the requirement persists.

## 6. Error handling — one table, no ad-hoc handling

| Case | UI |
|---|---|
| 422 in a form | inline field messages + summary |
| 422 outside a form (confirm dialog, bulk) | alert inside the dialog |
| 401 | clear cache → login with same-origin return path |
| 403 `forbidden` | toast "You don't have access to this action." |
| 403 limit/feature codes | upgrade dialog stating the limit and the plans that allow it |
| 403 account-gate codes (email unverified, password change required, MFA setup, terms) | navigate to the matching screen, then return |
| 403 step-up required | confirmation dialog, retry once |
| 423 locked / read-only | persistent banner; writes disabled |
| 409 in-flight duplicate | silent bounded retry with the same idempotency key |
| 409 conflict | keep the form, show conflict message |
| 409 domain refusal (guard/safeguard/too-soon/program running) | refusal dialog with the server's reason, countdown from `retry_after_seconds`, an override field only when allowed; never a toast |
| 503 | maintenance view with retry |
| 404 | not-found view (identical for missing, not yours, hidden feature) |
| 429 | toast with "try again in N s" from `Retry-After` |
| 500 / network | toast with generic message + request id; never raw error text |
| CRUD success | short toast (none when the page transition is the confirmation) |

Every mutation used by a form also handles non-422 errors; never two toasts for one failure.

## 7. Structure and style

- One component per file, typed props; framework-idiomatic exports; files/folders kebab-case; components PascalCase; identifiers, enum values, keys, search-param and tab values in English.
- **Reuse first:** look in shared code and the feature before writing a hook/component. Shared code lives in one place; features do not import each other's internals (only shared helpers).
- **Lint gate, no exceptions:** strict type checking, no `any`, no suppression comments, no `console.*`, no unused code, accessibility lint rules on, framework-hook rules on, banned APIs enforced (raw-HTML sinks, physical CSS direction classes, raw palette colours).
- **Design tokens once:** colours, spacing, radius, shadow, z-index, motion, type scale are tokens; components use semantic tokens (never `blue-500` or raw hex); no arbitrary values when a token exists. Logical CSS properties (`margin-inline-start`, `ms-*`) instead of left/right so RTL works.
- **Type:** one scale; body ≥ 14 px; the smallest size only for badges/timestamps; long reading uses a measure (`max-w-prose`) and relaxed leading; weights 400–700.
- Stylesheets are render-blocking `<link>` in `<head>`, never in `<body>`, async or media tricks.
- **Page anatomy:** every page uses the shared page header and the layout of its type (list, detail/form, settings with tabs, dashboard); no ad-hoc title sizes; dialog animation and scroll defined once in the kit.
- **Accessibility (WCAG 2.2 AA):** labels on every control; visible focus; full keyboard use; dialogs trap and return focus; statuses use icon + text, never colour alone; contrast AA in every theme; icon-only buttons have an accessible name (tooltip never the only name; tooltips also open on focus/tap); semantic HTML before ARIA; validation errors linked and announced; loading regions `aria-busy`; live regions for async status; `prefers-reduced-motion` respected.
- **Usable without help:** plain language; every page says what it is for and what to do next; errors next to the field with input kept; empty states offer the next action; destructive or paid actions confirm the consequence; success is always confirmed.
- **Formatting:** dates, times, numbers and currencies go through one `format` module (`Intl`) using the tenant/user zone and the active locale; relative times are anchored to the server clock (HTTP `Date` header), never `Date.now()` alone.
- Tables: server pagination, truncated cells with full-value tooltip, virtualization over ~100 rows.

## 8. Responsive design (non-negotiable)

- Every screen, component, dialog, table, chart, form, public page and email works from **360 px** to ≥ 1440 px, portrait and landscape, both themes, RTL where supported.
- **Mobile-first**: base styles for small screens; breakpoint tokens add layout; no arbitrary pixel breakpoints.
- **Each component is responsive by itself** (no fixed pixel widths/heights; content wraps or truncates with a tooltip; works in a narrow sidebar/dialog/card). Not merged until checked at 360 px and ≥ 1280 px.
- No horizontal page scroll at 360 px; reflow and 200 % zoom stay usable (WCAG 1.4.10); never fix overflow with a page-level `overflow-x: auto`.
- **Touch first:** targets ≥ 48×48 px (primary command buttons ≥ 56 px tall, full width on mobile); nothing hover-only.
- Navigation: sidebar on desktop; bottom navigation + drawer on mobile, the primary task one tap away.
- Tables: stacked cards or horizontal scroll inside their own container with a sticky first column; never a desktop table crammed into a phone.
- Dialogs: full-screen sheets on mobile, centred modals on desktop; forms single-column on mobile; the submit stays visible above the on-screen keyboard.
- Charts resize with their container, keep readable labels at 360 px and have a data-table fallback. Images `srcset/sizes` with `width`/`height`. Installed PWAs respect safe-area insets.
- Low-end phones on slow networks are the performance reference.

## 9. Spacing, layering, consistency

- Spacing comes from the tokens: gap on flex/grid parents, `min-width: 0` on flex/grid children holding text.
- z-index only from tokens (`sticky`, `header`, `overlay`, `toast`); a sticky header never covers an anchor target; popovers/dialogs are portalled kit primitives; nothing overlaps a control or text.
- **Consistency gate:** a UI commit states which existing pattern/component it reuses (or adds) and lists the three viewports, both themes and RTL checked. A duplicated pattern (two headers, two card recipes, two empty states) fails review.

## 10. Known traps

- Utility-CSS content scanning silently matches nothing when sources are misconfigured: run a class-coverage check after changing sources.
- Nested disclosure/group selectors must be named; an unnamed group selector matches ANY ancestor. Test open and closed.
- Responsive display utilities go on a wrapper, never onto a component whose base sets `display` (CSS order wins).
- Layered CSS: rules in a component layer never beat utilities on the same element; overrides go unlayered/utility layer.
- Class mergers may keep both `sm:py-6` and `py-0`: a component with responsive base spacing drops it when the caller passes the same property; QA at ≥ 640 px.
- Never put screen-reader-only text inside a truncating container (it escapes the clip and widens the page).
- A `role="combobox"` button gets no accessible name from content: the trigger carries the field id so the label names it. Loading rows are a plain region, not an `aria-hidden` list widget.
- Dialog libraries return focus through their trigger: a dialog opened from state/URL needs an explicit return-focus helper; keyboard-test every dialog (Enter → Escape → active element).
- An island/lazy widget that mounts on interaction pre-renders real focusable content, never a bare pulsing box.

## 11. Build and delivery

- Assets are built in CI or locally and shipped as artifacts; the server needs no build toolchain. Hashed filenames, long cache; previous release's assets stay reachable for one release; reload once on chunk-load error.
- Dev tools (devtools panels, mock servers) never reach the production bundle.
- Third-party scripts only where `security.md` §5 allows.
