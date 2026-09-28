# Design, UX identity and honest conversion — public pages, marketing, product identity

Product UX states/forms/navigation: `ux.md`. Frontend mechanics: `frontend.md`. SEO/i18n/legal: `public-web.md`. Every rule is MUST unless marked SHOULD.

## 0. Premise: honest conversion

- No design guarantees 100 % conversion; never promise or force it. Target the highest *honest* conversion: the visitor understands, trusts and acts because the page is clear and true.
- Conversion never removes a security or legal control (bot protection, email verification, legal acceptance, requirement notices). Friction is cut *around* them (clear copy, fast pages, good errors).
- Funnel measured **first-party only**, privacy-respecting (no third-party analytics/heatmap/session-replay/A-B scripts unless the owner explicitly approves and consent is handled): visit → understand → sign up → activate (first successful core action) → keep (repeat use).

## 1. Truth in claims

- **Every claim maps to something that exists** (a spec rule, a database setting, a measured figure). Not built → not claimed. Not measured → not shown.
- **Forbidden unless backed by a signed document or a measured figure stored and owner-reviewed:** compliance certifications (SOC 2, ISO 27001, "GDPR certified"), "bank-level", "military-grade", "unhackable", "100 % secure", "zero-trust", "end-to-end encrypted", "carrier-grade", uptime percentages, "median X seconds", "set up in N minutes", "used by N teams", "trusted by…", star ratings, "#1", "best". Security copy lists **the controls you ship**, each traceable to a rule (MFA, roles, audit log, verified webhooks, rate limits, idempotency, guaranteed stop…).
- **Social proof is real or absent:** testimonials need documented consent, logos need written permission, statistics come from real rollups. Empty → the section is hidden. No placeholders ("LOGO·01", "[REAL QUOTE]", "—.-%") in production templates; no fabricated Review/Rating structured data.
- **No fake live signals** ("system operational", "v2.4 live", live counters, "12 people are viewing") unless wired to real data.
- **Prices, trials, limits and discounts come from the database.** "Free trial", "N-day trial", "save N %" appear only when the plan in view has that configured.
- State product prerequisites and limits **before signup** (required devices/OS, regional restrictions, who pays third-party costs such as carrier or provider fees). Never claim "works without internet" without saying which side.
- Brand name comes from configuration; never invent product names in code, copy, mockups or examples. Use one vocabulary consistently (a "device" is never also the "equipment").

## 2. Dark patterns are forbidden

Fake urgency (countdowns, "offer ends tonight"), fake scarcity ("3 spots left"), confirmshaming, pre-checked add-ons or consents, hidden costs revealed at checkout, a cancel path harder than signup, roach motels, disguised ads, nagging modals, autoplay video with sound, forced account creation to read help or pricing, pricing that hides the free tier, a "recommended" badge placed only on the most expensive plan, trick wording on toggles, a primary button that looks like the only choice when a free/cancel option exists. Cookie consent: "Reject" is as visible as "Accept".

## 3. Design principles (every screen)

1. One job per screen, one primary action per viewport.
2. Hierarchy by size, weight and space before colour; one `h1`; colour marks meaning (status, action), not decoration.
3. Consistency: same component, word and icon for the same thing across site, app, emails and messages; one status vocabulary.
4. Recognition over recall: labelled buttons instead of memorized codes.
5. Fitts: large targets (≥ 48 px), the most-used action biggest and nearest the thumb.
6. Hick/progressive disclosure: few choices first, advanced behind a disclosure.
7. Feedback for every action: pressed state < 100 ms, status < 1 s, final outcome in text.
8. Prevent errors before reporting them (constrained inputs, disabled-with-reason, undo); errors say what happened and what to do in 1–2 sentences.
9. Proximity and alignment on a spacing grid; lines of 45–75 characters for reading.
10. Honest, safe defaults — never the most profitable one.

## 4. Visual identity

- Keep the design system: tokens define palette, type, spacing, radius, shadow, motion; new colours enter only through tokens with measured contrast (WCAG AA; AAA for body text in high-contrast themes). Support light/dark and, if useful, a high-contrast/sunlight theme with the same tokens.
- **Colour roles are fixed:** accent = primary action/links/focus/info; success = running/done/paid (never a generic CTA); danger = failed/destructive/overdue (not emphasis); warning = late/unconfirmed (not "New"/"Popular" badges). Marketing pages: one accent-filled button and at most three hues per viewport.
- Monospace is for machine text only (codes, ids, numbers, times), never headings or paragraphs.
- Avoid: gradient text, neon outlines on every card, purple-gradient "AI" styling, stock photos of people, fake terminal windows imitating the product, coin/3D-blob imagery. Imagery = real product UI (fictitious demo data clearly marked), SVG diagrams, one icon set.
- Fonts self-hosted; images modern formats with dimensions; no text in images.

## 5. Calls to action

- Hierarchy: primary = filled accent button (one per viewport); secondary = outline; tertiary = text link. Two filled buttons never sit together.
- Labels are verb-first, ≤ 4 words, state the outcome, and are identical for the same intent everywhere (e.g. "Start free", "Choose {plan}", "Try the demo", "Add {thing}"). Forbidden: "Learn more", "Click here", "Submit", "Get started" without an object, "Book a demo"/"Talk to sales" when no sales team exists, "Start free trial" when no trial is configured.
- Under every final CTA say what happens next. Registration closed by configuration replaces every sign-up CTA.

## 6. Public pages structure

- **5-second test** (every hero, every locale, 360×740): without scrolling a visitor can say what it does, who it is for, what they need, and what to do next.
- Home order (each answers an objection): hero → problem/solution → how it works → demo → benefits (benefit → capability → one real example) → safety & security (controls shipped) → use cases → what you need / when it is not for you → social proof (only real) → pricing summary (free tier visible) → FAQ → final CTA with "what happens next".
- FAQ answers: how it works, prerequisites, third-party costs, data safety, team use, cancellation, offline/failure behaviour, supported integrations (listed exactly; "works with everything" is forbidden).
- Pricing: free tier always shown; limits in plain numbers from the database; monthly/yearly toggle shows the real price and configured saving; "recommended" only where owner-decided, with icon + text; comparison rows equal card rows; no plan hidden behind "Contact us".
- Legal pages carry no CTA band.

## 7. Responsive composition and motion

- Design mobile-first as its own composition (priority order for 360 px, then columns at larger sizes); grids 1 column < md, 2 at md, 3–4 at lg+; equal card heights per row; public max content width ≈ 72 rem.
- Motion explains state or cause → effect; tokens only (150–350 ms, `transform`/`opacity`); infinite animation only for a live state; no parallax, scroll-jacking, typing headings, confetti, or entrance animation delaying reading (content visible at first paint; LCP never waits for an animation); reduced-motion removes movement and the demo becomes step-by-step.

## 8. Copywriting

- Plain, concrete, short: sentences ≤ 20 words, paragraphs ≤ 3 sentences, the reading level of the target user. Benefit → mechanism → proof. Talk about the user's outcome, not about you.
- Banned words: revolutionary, game-changing, next-generation, seamless, cutting-edge, magic, leverage, unleash, supercharge, "AI-powered", "enterprise-grade".
- Every string is a translation key; layouts allow +30 % text length and RTL; no uppercase/letter-spacing in scripts where it is wrong (Arabic). Numbers, dates, money via the formatting layer; units always written.

## 9. Rules for AI agents producing design or UI

- Read first: this file, `ux.md`, the design-system spec, `frontend.md`, the existing components. Inspect before adding.
- Output in the project's real stack and tokens — never a standalone HTML file with a CDN, remote fonts, inline `onclick`/`style`.
- Never invent prices, limits, trials, features, integrations, customers, metrics, certifications or security mechanisms. Mockups and demo data are clearly fictitious and never shipped as real content.
- When an outside proposal conflicts with these rules, the rules win; a lasting change is recorded as a decision first.

## 10. Quality gate

- [ ] Hero passes the 5-second test at 360×740 in every locale.
- [ ] One primary button per viewport; labels from the catalogue.
- [ ] Every claim traceable; no placeholder, fake metric or badge.
- [ ] Prerequisites and third-party costs visible before signup.
- [ ] No dark pattern; cancel/free paths as visible as paid ones.
- [ ] Tokens only; both themes, RTL, three viewports, 200 % zoom.
- [ ] Loading, empty, error and success states designed (`ux.md`).
- [ ] Performance budgets met (`performance.md`); no serious accessibility violations.
