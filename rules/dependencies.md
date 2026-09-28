# Dependency rules — minimalist, allow-listed

As few dependencies as possible; each one is code you do not control that ships to production and can be compromised. Supply-chain security: `security.md` §13.

## 1. How a package gets added

- Order of preference: the platform/language/framework already in use → a few lines of your own code (or ported behaviour) → a dependency.
- A new package must: be widely used (≥ ~1 000 stars, or the official package of the framework/vendor/mandated tool); have a release in the last 12 months; carry a permissive licence (MIT/BSD/Apache; anything else needs the owner's acceptance); have no open advisory (`audit` clean); have a justified size (frontend: bundle impact measured); run on the production host (no runtime/daemon it cannot run); work with your framework without aliases or patches.
- **Verify identity before installing:** the package exists on the official registry under exactly that name, is the intended project (not a typosquat, not a name an AI suggested from memory), and its install scripts are reviewed. Private package names are reserved on public registries.
- It is added together with a decision record and an allow-list entry in the same commit. Transitive dependencies come with it; nothing imports a transitive package directly.
- A check in pre-commit and CI fails when a manifest names a package that is not on the allow-list. Lockfiles are committed; installs in CI use the lockfile only (`ci` mode).

## 2. Rules

- One library per concern (one date approach, one form library, one HTTP client, one state cache). Never two libraries for the same job.
- No dependency for something a few lines or the platform does (`fetch`, `Intl`, native validation, CSS transitions, array helpers).
- No abandoned packages, no vendored copies of libraries with local patches (upstream or replace), no beta/alpha packages in production paths without an owner decision and an exact pin behind a wrapper.
- Prefer official first-party SDKs only when they are small and needed; otherwise an HTTP client behind your own interface (`integrations.md`). Do not pull heavyweight SDKs for one call.
- Dev-only tools (linters, test runners, type checkers, formatters, bundlers) are allowed when standard; they never ship in production bundles or images.
- Backend packages rely only on runtime extensions/binaries present on the production build and add no daemon.
- Frontend runtime dependencies stay a short list (`frontend.md` §1); anything else is dev-only or the platform.
- Remove a dependency in the same commit that stops using it.
- Audit (`composer audit`/`npm audit`/`pip-audit`/etc.) is clean before every release; advisories are triaged within a defined SLA; automated update PRs are reviewed and tested, never auto-merged blindly.
- **AI agents never add, upgrade or remove a dependency unasked**, and never install from a name they cannot verify.

## 3. Forbidden categories (default; a project may extend)

Admin/CRUD generators that bypass your policy layer; convenience packages replicating what the framework already provides (permissions, settings, activity log, query builder, CSP, honeypot, sitemap); packages that run external binaries or need Node/Chrome on the server when the host does not have them; CDN-loaded scripts/styles for core assets; multiple competing UI kits; alias/compat layers replacing the UI framework; unmaintained or unlicensed packages; packages requiring broad install-time network/shell access.
