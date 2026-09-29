# CLAUDE.md — <Project name>

Rules for every human and AI working in this repository. **MUST / NEVER rules are non-negotiable.** If a request conflicts with a rule, stop and ask the owner.

## Project (fill in)

- **What it is:** <one paragraph: product, users, core value>.
- **Stack:** <backend language/framework, database (MySQL | PostgreSQL), frontend framework, hosting>.
- **Rendering model:** <SPA | SSR | SSG | islands>. **Tenancy:** <single | multi-tenant>. **Languages:** <list, RTL?>.
- **Modules:** <list; which are optional/paid/hidden>.
- **Project-specific NON-NEGOTIABLES:** <domain rules that override or extend the generic ones>.
- **Decisions register:** <path>. **Specs:** <path>. **Lessons:** <path>.

## Loaded rules (generic, stack-neutral)

Always:
- @docs/rules/workflow.md
- @docs/rules/security.md
- @docs/rules/performance.md
- @docs/rules/testing.md
- @docs/rules/operations.md
- @docs/rules/dependencies.md

Server code: @docs/rules/backend.md and exactly one of @docs/rules/database-mysql.md / @docs/rules/database-postgres.md
UI code: @docs/rules/frontend.md, @docs/rules/ux.md
Configurable business values: @docs/rules/configuration.md
Optional (load only if the product has it): @docs/rules/integrations.md, @docs/rules/payments.md, @docs/rules/design-conversion.md, @docs/rules/public-web.md

## Project rules that tighten the generic ones (fill in, keep short)

- <e.g. file size cap 400 lines; git model: main only; error envelope shape; supported DB versions; browsers>.

## Before every change

Answer explicitly, in the commit body: **Security?** **Performance?** (and **Responsive/Accessible?** for UI). See `workflow.md` §1.

**UI changes, however small:** apply `frontend.md` §0 (premium, production-grade, all states, cohesive, visually reviewed) first.

**Test blocked by a DEV lockout/cooldown/limit:** apply `testing.md` §0; never wait.
