# UX rules — states, feedback, navigation, forms, layout

Complements `frontend.md` (mechanics) and `design-conversion.md` (marketing, identity). One line per rule; every rule is MUST unless marked SHOULD.

## 1. Quality bar

- "It works" is not done. Done = secure + correct + performant + accessible + responsive + understandable + consistent + professionally designed.
- Simple where possible, powerful where necessary, always clear. Ask "how would a best-in-class product (Stripe, Linear, Vercel) ship this?" and "will we regret this in a year?"; use established patterns, never blind copies.
- One screen, one job, one primary action (§7). Small focused components; never one wall of markup or one page mixing many forms, tables and buttons: split into sections, tabs, drawers or pages.
- Gate before "done": what happens while loading, empty, failing, invalid, on a slow network, on double click, after permission loss, with 100 000 rows, with very long text, on mobile, in RTL, after refresh, on browser Back, when the request succeeds but the UI does not update?

## 2. Loading states — choose by situation (no blank page, no frozen screen)

| Situation | Pattern |
|---|---|
| Load < ~300 ms | show nothing (delay indicators; a 100 ms flash looks broken) |
| First load, layout known (list, table, card, dashboard, detail) | skeleton mirroring the final geometry (same rows, heights, card shapes; no layout shift) |
| Refetch, filter, sort, page or tab change over shown data | keep data visible (keep-previous-data); subtle indicator only (dim, top progress line, "Updated just now"); never a skeleton over live content |
| Button or small inline action (< ~1 s) | spinner inside the button; compact dropdown content: spinner |
| Measurable long work (upload, import, export, batch) | determinate progress bar with label ("Sending 45 of 120…") |
| Long work, progress unknown | indeterminate bar + step text; a bare spinner never longer than ~2 s |
| Route change (lazy chunk / loader) | keep the shell; route-level pending state with a start delay and a minimum display time so fast routes do not strobe; prefetch on intent |
| Form whose structure is known | render the form; disable until ready; no skeleton |

- Full-page spinner or blocking overlay is forbidden. Never stack two loading indicators for one operation (skeleton + button spinner only when they mean different things).
- Never fake progress. Skeletons/spinners carry `aria-busy` and an accessible label; shimmer/pulse off under reduced motion.
- **Button loading:** immediate feedback; re-submit blocked; stable width (spinner takes the reserved icon slot: `[spinner] Saving`, never a resize to `Loading…`); only that action disabled; state restored on success and failure. The user always knows the click was received.
- A mutation on one row never makes the whole page look loading; no global blocking loader.

## 3. State matrix

Every data-driven component designs: initial loading · data · zero data · background refresh · error · denied/unavailable · mutation pending / success / failure · offline. Never only the happy path; never render `data` while undefined, stale after an error, or as if a failed request had succeeded.

- **Empty:** icon + one sentence saying what is empty (and why, when useful) + one primary next action + help link. Never "No data". No call to action when the user lacks the right. "No results for filters" differs from "nothing yet" and offers "Clear filters".
- **Error:** short, translated, actionable, non-technical; retry when meaningful; request id for unexpected errors; shown next to the failing UI.
- **Offline / dependency offline:** last-synced time + explicit reconnecting state; writes that need the network say so.

## 4. Feedback: toast vs banner vs inline

- Toast: short confirmation or background result, auto-dismiss, never the only channel for an error the user must act on; one toast per operation; group bursts of background events.
- Inline (next to field/row/dialog): validation and errors needing action. Persistent banner: account/state restrictions (locked, read-only, offline, maintenance). Blocking dialog: only decisions.
- An obvious page transition after success (create → detail) needs no extra toast.
- **Optimistic updates:** only for safe, reversible, low-stakes writes (toggle, rename, reorder) with rollback on error; never for commands, payments, refunds, statuses or anything displayed as "confirmed": show pending until the server confirms.
- Reversible action → do it and offer Undo; confirm only when §9 says so.

## 5. Forms UX

- Minimum fields, sensible defaults, optional fields marked, unusual fields explained beside them. Do not split into steps to look sophisticated: a stepper needs real logical stages with visible progress.
- **Long forms** (taller than one screen): titled sections (cards) grouping related fields, progressive disclosure for advanced fields, a sticky action bar (submit + error count), section navigation when > 4 sections. Never dozens of unrelated controls in one dense card.
- **On failed submit:** an error summary at the top ("3 problems with this form"), each item a link that scrolls to and focuses its field (focus goes to the summary when it exists, else the first invalid field, always visible); an inline error under each field with icon + text (never colour alone), cleared as the field is fixed; the error count in the sticky bar; entered values kept; non-field errors shown in the summary; retry stays available.
- Never steal focus during background validation; never reset the whole form; never disable Submit for invalidity.
- Success: immediate confirmation, move to the resulting state, re-submit impossible until done.
- **Unsaved changes:** warn on leaving only when something changed (router blocker + `beforeunload`), say what will be lost, offer to stay. Drafts are not stored in browser storage; a draft feature is a server-side feature with its own spec.

## 6. Navigation and wayfinding

- Every screen answers: where am I, how did I get here, where can I go, how do I return. No dead ends.
- **Breadcrumbs** on every page ≥ 2 levels deep: real hierarchy, clickable ancestors, current page as plain text last; on mobile a back arrow + parent/current title. Not on top-level pages; they do not mirror the sidebar.
- A detail/edit opened from a list has an obvious way back that restores list context (page, sort, filters live in the URL). No redundant Back buttons beside breadcrumbs.
- Browser Back, refresh and deep links always work (state in the URL); no history hacks; dialogs close by X, Esc and backdrop, and return focus.
- Grow by grouping (sections, tabs, hubs), not by adding menu items; labels in the user's words, never database terms; primary workflows separate from configuration/administration.

## 7. Layout, density, hierarchy

- One primary (filled) action per viewport, top of the page or in a sticky bar; secondary actions outline/ghost or in a "⋯" menu; never five equal buttons. Prominence follows importance; secondary information stays secondary.
- **Grid rule:** the main task (form, list, detail) owns the dominant column; help, tips, metadata and summaries take a narrow column or a collapsible panel and stack **below** the main task on mobile. Secondary content never takes more space than the task; no cramped inputs, no tiny columns, no orphan whitespace.
- Avoid walls of text, giant forms, card/badge/icon overload and dense dashboards where everything has equal weight. Progressive disclosure hides secondary detail, never critical information.
- Spacing and type come from tokens only; related items grouped, groups clearly separated; nothing overlaps, clips or wraps badly at 360 px, on wide screens, with long text in other languages, in RTL.
- Mobile is its own composition: priority order, no hover dependence, reachable actions, dialogs that fit, an on-screen keyboard that never hides the field or the submit.

## 8. Lists, tables, search

- Every list is paginated in API and UI from the first commit, whatever today's size; the UI never assumes a collection is small; loading rows, empty and error states per §2–3.
- Search is debounced; active filters are visible as chips with "Clear all"; filters/sort/page persist in the URL; a filter change resets only the page; "no match" says so; the pager shows position and keeps a stable order.
- Rows: important columns first, discoverable row actions, targets ≥ 48 px, truncation with full-value tooltip; on mobile cards or contained scroll, never dropping critical data.

## 9. Destructive actions and dialogs

- Destructive controls look destructive (danger role), are never the default/primary action, and state the consequence and what is kept. Tiers: reversible → act + Undo; significant → confirm dialog; irreversible or takeover-class (delete account, rotate keys, anonymize) → typed confirmation (+ step-up authentication). No confirmation on trivial reversible actions.
- A dialog is one focused decision: title, purpose, obvious primary action, obvious cancel, trapped focus, focus returned, keyboard works, fits mobile. No nested dialogs; a complex workflow is a page or a drawer.

## 10. Async correctness in the UI

- Assume responses arrive out of order: cache keys include every parameter, only the latest result is shown, superseded requests are aborted; rapid repeats are disabled or deduplicated (and never only in the UI: server idempotency, `backend.md` §5–6).
- **Permission lost mid-session** (403/404 or capability change): refetch the capabilities, drop that feature's cached queries, remove now-forbidden controls, never keep privileged data on screen (UX only; the policy is the authority).
- After a successful mutation the UI reflects server truth (invalidate/refetch); a success with no visible change is a defect.
- Consistency: the same action = the same component, label and place everywhere (Save, Cancel, Delete, Retry, Connect…); a pattern used twice becomes a shared component.

## 11. Perceived performance and motion

- Acknowledge every action within 100 ms; skeletons instead of blank areas; keep old content during refresh; prefetch on intent; lazy-load below the fold; communicate slow operations honestly, no fake speed.
- Motion explains state or cause → effect: short (150–350 ms), purposeful, subtle, interruptible, `transform`/`opacity` only; infinite animation only for a live state (spinner, running indicator); never on CTAs, headings or decoration; no parallax or scroll-jacking; `prefers-reduced-motion` removes movement; the UI is understandable without animation.
