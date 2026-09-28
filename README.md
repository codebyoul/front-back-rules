# front-back-rules

Framework-agnostic engineering rules for **back end, front end and database**, written to be read and obeyed by AI coding agents (Claude Code, Codex, Cursor…) and humans. They cover security, performance, data integrity, UX, testing, workflow and operations for production-grade software.

They are stack-neutral: no rule depends on Laravel, React or any single framework, language or vendor.

## Keywords

- **MUST / MUST NOT / NEVER** — non-negotiable. No exception without a recorded owner decision.
- **SHOULD** — default; deviation needs a recorded reason.
- Tags: **[multi-tenant]** (drop if single-tenant) · **[optional module]** (load only if the product has it) · **[i18n]**.

A project may make any rule **stricter**. It must never weaken security, performance or data-integrity rules.

## Files

| File | Scope | Load when |
|---|---|---|
| [`rules/workflow.md`](rules/workflow.md) | Per-change questions, definition of done, minimalism, truth gate, git, **AI-agent conduct**, QA | always |
| [`rules/security.md`](rules/security.md) | Authn/authz, input/output, web hardening, SSRF, secrets/crypto, uploads, abuse, webhooks, privacy, audit, supply chain, LLM features | always |
| [`rules/performance.md`](rules/performance.md) | Budgets, N+1, indexes, caching, queues, Web Vitals, bundle size, perceived speed | always |
| [`rules/testing.md`](rules/testing.md) | Endpoint test matrix, tenant isolation, concurrency, frontend/e2e, traps | always |
| [`rules/backend.md`](rules/backend.md) | Layering, tenancy, validation, pagination, API contract, integrity, time, jobs, admin | any server code |
| [`rules/database-mysql.md`](rules/database-mysql.md) | MySQL 8+/InnoDB: types, keys, indexes, locking, online DDL, backups, MariaDB differences | project uses MySQL/MariaDB |
| [`rules/database-postgres.md`](rules/database-postgres.md) | PostgreSQL: types, RLS, indexes, locking, safe migrations, vacuum, pooling, backups | project uses PostgreSQL |
| [`rules/frontend.md`](rules/frontend.md) | Data fetching, browser security, forms, errors, style, accessibility, responsive | any UI code |
| [`rules/ux.md`](rules/ux.md) | Loading/empty/error states, feedback, navigation, long forms, layout, dialogs | any UI code |
| [`rules/configuration.md`](rules/configuration.md) | Dynamic-by-default settings, what stays in code | any product with business values |
| [`rules/operations.md`](rules/operations.md) | Build, deploy, readiness, backups, health, key rotation, constrained-host profile | always |
| [`rules/dependencies.md`](rules/dependencies.md) | Minimalist allow-list and vetting of packages | always |
| [`rules/integrations.md`](rules/integrations.md) | Third-party APIs, webhooks, devices, safety-critical commands | [optional module] |
| [`rules/payments.md`](rules/payments.md) | Billing engine, provider rails, webhooks, dunning | [optional module] |
| [`rules/design-conversion.md`](rules/design-conversion.md) | Honest marketing, identity, CTAs, public-page structure | public/marketing pages |
| [`rules/public-web.md`](rules/public-web.md) | i18n/RTL, SEO, legal pages and consent, abuse reports | [i18n], public site, regulated data |

Cross-references are by file and section (`security.md` §7), so keep section numbers stable.

## Adopt in a project

1. Copy `rules/` into the project (e.g. `docs/rules/`) or add this repo as a git submodule.
2. Copy [`CLAUDE.template.md`](CLAUDE.template.md) to the project's `CLAUDE.md` / `AGENTS.md`, fill in the project-specific parts (product, stack, decisions, tighter rules), and import the rule files you need (`@docs/rules/security.md`, …).
3. Keep project-specific facts (product names, vendors, decision ids) in the project, never in these files.
4. Budget context: agent tools warn when the loaded instruction files get large. Load the "always" files first; load optional modules only where they apply.

## Are these rules valid for other frameworks?

Yes. They describe *what must hold*, not *which library provides it*. Map the vocabulary once:

| Rule vocabulary | Laravel / PHP | Django / Python | Rails | Node (Nest/Express) | Spring |
|---|---|---|---|---|---|
| request schema | FormRequest | Serializer / Form | strong params | DTO + class-validator/zod | `@Valid` DTO |
| policy | Policy / Gate | permissions | Pundit | guards | `@PreAuthorize` |
| use-case object | Action / service | service | service object | provider/service | service |
| scoped accessor | scoped relation | filtered manager | `current_account.things` | repository + tenant filter | repository + filter |
| job / queue | queued job | Celery | ActiveJob | BullMQ | `@Async`/queue |

| Rule vocabulary | React | Vue | Svelte | Angular |
|---|---|---|---|---|
| server-state cache | TanStack Query / SWR | TanStack Query / Pinia Colada | TanStack Query | TanStack Query / signals |
| side-effect discipline | `useEffect` | `watch` | `$effect` | `effect` |
| component kit | shadcn/ui, Radix | shadcn-vue, Radix Vue | shadcn-svelte | Angular Material/Spartan |

Rules that are project *choices* rather than principles (client-only rendering, no Node on the server, no Redis, cron-driven jobs) are **not** hard rules here: rendering model is a project decision (`frontend.md` §1) and the constrained-host limits live in an optional profile (`operations.md` §9).

## Precedence and change control

Project entry file → these rules → project decision records → specs → legacy code. Keep every file ≤ 500 lines, one line per rule, no duplicates (cross-reference the home file). A new rule goes into exactly one home file.
