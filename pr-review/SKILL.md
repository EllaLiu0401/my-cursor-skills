---
name: pr-review
description: >-
  Perform comprehensive GitHub PR review: analyze changes, check compliance with
  project rules (AGENTS.md, planning docs, design system), cross-reference
  existing reviewer comments, and post inline review comments via gh CLI.
  Use when the user shares a PR URL, asks to review a pull request, says
  "check this PR", "code review", "review #123", or "look at my changes".
---

# PR Review Workflow

## Overview

End-to-end PR review: understand business context → validate business flow → evaluate approach → check code → deduplicate with ROI filter → post inline comments to GitHub.

## Step 0: Business Context Gate

**Before reading any code**, establish the business context. A PR review without understanding the business problem is guesswork.

1. **Read the PR description + linked Jira/task spec.** Can you answer these three questions?
   - What business problem does this solve? (What was broken, missing, or inefficient?)
   - Who is affected? (End users, operators, internal team, on-call?)
   - What does "done" look like? (Acceptance criteria, expected user experience change)

2. **If the business context is unclear, stop.** Flag as P1 blocker and request clarification before proceeding to code review. A vague or absent PR description is the first red flag — not a minor issue to note at the end.

3. **Extract acceptance criteria** to validate against later. When evaluating the code, every behavioral change should map back to a stated business goal.

This is the highest-value step. A PR that solves the wrong business problem or implements an incorrect business flow needs to be caught here, not after 30 inline comments on code style.

## Step 1: Gather Context

Collect all inputs in parallel:

```
1. gh pr view <number> --json state,title,commits,reviews,comments,updatedAt
2. gh api repos/{owner}/{repo}/pulls/{number}/comments  (existing inline comments)
3. gh pr diff <number>                                   (full diff)
4. Read project rules and planning docs (see below)
```

### Read Project Rules (mandatory, before analyzing any code)

**Always read `AGENTS.md` first** — it is the single source of truth for all non-lintable constraints. It defines:
- Core principles (K.I.S.S., minimalistic, clean, elegant, best practice)
- Documentation hierarchy (`agents.md` > `planning/` > `features/` > `implementation/`)
- Coding standards, development principles, and all mandatory rules
- API architecture, database conventions, frontend rules, i18n, testing

**Then read planning docs based on which areas the PR touches:**

| PR touches... | Read these docs |
|---------------|-----------------|
| API / backend | `planning/api-architecture.md`, `planning/db-conventions.md` |
| Frontend / UI | `planning/frontend-architecture.md`, `docs/DESIGN_SYSTEM.md` |
| Auth / RBAC | `planning/rbac.md`, `planning/auth0-integration.md` |
| Forms | `planning/frontend-forms.md` |
| Testing | `planning/testing.md` |
| CRUD endpoints | `planning/crud-factory.md` |
| Any area | `planning/README.md` (index of all planning docs) |

**For DS compliance checks**: also inspect `node_modules/@aetheronhq/ui/dist/index.d.ts` exports and `node_modules/@aetheronhq/ui/dist/styles.css` to know what components and CSS utility classes the DS provides.

### Key data to extract
- PR title, state, commit list
- All existing reviewer comments (who said what, on which file:line)
- Author's response/triage comment (often marks issues as "pre-existing" or "addressed")

## Step 2: Validate Business Flow

Before diving into code quality, validate that the PR implements the **right business flow**.

**1. Map the PR to the user journey.**
Trace the change through: user action → system behavior → outcome. Does the flow make sense from the user's perspective? Is there a simpler path to the same business goal?

**2. Check for business flow correctness.**
Business flow issues have the highest impact — they can require a complete rewrite. Watch for:
- **Blind spots** — even experienced engineers can miss edge cases in flows they rarely touch
- **Narrow thinking** — the happy path works, but what about cancellations, partial failures, retries, or concurrent users?

**3. When the business flow is wrong, flag it immediately.**
Don't proceed to code-level review if the flow itself is incorrect or significantly inefficient. A perfectly coded implementation of the wrong flow is worse than a rough implementation of the right one. Frame it constructively: "The implementation is clean, but I think the business flow has an issue — here's what the user would actually experience."

When business flow concerns exist, they take priority over everything else.

## Step 3: Analyze the PR

### Foundational Review Principles

These four principles govern every check below. Violating them produces low-quality reviews.

**1. Evidence-Based Review — verify against real code, not assumptions.**
Every claim in a review comment must be backed by evidence from the actual codebase. Before flagging an issue:
- **Read the real type definition / SDK export** — don't assume a field is optional; check the `.d.ts` or source. If the SDK types say `id: string` (required), an empty-string guard is unnecessary.
- **Trace the actual call path** — don't guess what a function does; read it. If `ScopedDb.selectFrom` already applies `deletedAt IS NULL`, flagging it as missing is noise.
- **Check test fixtures and CI** — if you claim "this test will fail", verify against the actual mock shape. If you claim "this throws at runtime", confirm the runtime type doesn't prevent it.
- **When uncertain, say so** — "I couldn't verify whether X is guaranteed here — worth checking" is more valuable than a false-positive P1.

**2. Third-Party Verification — web-search external API docs, don't invent constraints.**
When a review touches external services (Stripe, Composio, Vapi, AWS, Auth0, etc.):
- **Search the actual docs** for constraints before flagging. Example: Stripe rejects `timestamp` older than 35 calendar days AND more than 5 minutes in the future — cite the docs URL.
- **Check SDK behavior** — e.g., does `react-markdown` v10 render raw HTML by default? (No — no `rehype-raw` plugin.) Check, don't guess.
- **Cite the reference** in your comment: "Per Stripe docs (https://docs.stripe.com/api/billing/meter-event/create), timestamp must be within past 35 days."
- **Never fabricate API constraints** — a wrong external constraint in a review wastes the author's time verifying your fiction.

**3. User Perspective — trace the user journey, not just the code path.**
For every behavioral change, ask: "What does the end user / operator / on-call actually experience?"

- **UI/UX — Visual consistency**: Labels, badges, and status indicators must match across all views. If one view shows "ACTIVE" and another shows "active", that's a bug the user sees.
- **UI/UX — Interaction integrity**: Filter and search state must never orphan (dropdown shows a value that doesn't exist in options). Focus must never be stolen unexpectedly (e.g., agent dropdown change shouldn't yank focus to textarea). Pagination must not shift unrelated content (global "Show More" shouldn't reorder groups above the fold).
- **UI/UX — Information preservation**: Search results must retain context (show category chips when group headers are hidden). Grouped views must be stable on expand — clicking "Show More" must not shuffle existing items.
- **UI/UX — Workflow continuity**: Filter/search state should survive page reload (URL-backed via `?search=` / `?category=`). Loading and error states must always be visible — silent failures are UX bugs. URLs should be shareable (if a user copies the URL and sends it to a teammate, the teammate should see the same view).
- **Operator experience**: Log-level choices affect on-call. `debug` level for a revenue-impacting skip means no one notices until monthly revenue dips. Revenue, billing, and security events need `warn` or `info` minimum. Dashboard visibility matters — if a behavioral change causes an alerting spike, call it out in the PR description.
- **Business impact**: Silent data loss (e.g., swallowing billing errors) is a business bug, not just a code bug. Trace the failure mode to the business consequence.

**4. Don't Manufacture Issues — if the code is clean, say so.**
A review that pads findings with noise erodes trust. Before posting a comment, ask: "Would this cause a real problem, or am I nitpicking to fill space?"
- **Don't force every observation into a critique** — if a pattern is unusual but correct, note it as "VERIFIED" rather than inventing a concern.
- **Nitpicks must be labeled as such** — use `[NITPICK]` or P3 so the author knows it's optional.
- **Don't flag theoretical risks with no practical exploit path** — "if someone passes negative infinity here" is not useful unless the input is user-controlled.
- **Positive verification is part of a thorough review** — marking verified-safe patterns (e.g., "SECURITY VERIFIED — tenant isolation holds because ScopedDb always applies orgId filter", "CHAIN VERIFIED — delete flow has layered ownership checks") shows the review was rigorous, not just looking for problems.

---

### Approach-Level Review (before diving into code details)

Before examining individual files, evaluate the PR holistically:

**1. Is the problem clearly defined?**
Read the PR description and linked issue/task. If the PR doesn't articulate what problem it's solving, flag it immediately — code review without a clear problem statement is guesswork. A good PR description answers: what was broken or missing, who it affects, and what "done" looks like. If the description is vague or absent, request clarification before investing in line-by-line review.

**2. Is this the best approach, or just one that works?**
"It works" is necessary but not sufficient. Ask:
- Is this treating the **root cause** or just suppressing a **symptom**? (E.g., adding a null-check where the real fix is ensuring the value is never null upstream.)
- Is there a simpler, more direct solution? (Per AGENTS.md: "Minimalistic — every line of code is a liability.")
- Does this follow the existing architecture, or does it introduce a parallel pattern that will need reconciliation later?
- Would the approach scale if the same problem appears in 5 more places?

**3. What are the trade-offs?**
Every non-trivial change has costs. Evaluate:
- **Complexity cost**: Does this introduce new abstractions, indirection layers, or special cases? Are they justified?
- **Maintenance burden**: Will the next developer understand why this was done this way? Does it create implicit coupling that's easy to break?
- **Dependency growth**: Does it pull in new libraries or tighten coupling to external services?
- **Scope creep**: Does the PR do more than it claims? Unrelated changes should be in a separate PR.

**4. When the approach is fundamentally wrong, say so.**
If the PR is locally correct but structurally misguided — wrong abstraction layer, wrong data model, solving the wrong problem — don't just leave P2 nitpicks and approve. Flag it as P1 with a concrete alternative direction. A clean rewrite of a small PR is cheaper than maintaining a bad abstraction for years. Frame it constructively: "The code itself is clean, but I think the approach has a structural issue — here's what I'd suggest instead."

When approach-level concerns exist, they take priority over code-level findings. A perfectly written implementation of the wrong approach is still the wrong approach.

---

For each changed file, evaluate against the following categories:

### Architecture & Code Rules (from AGENTS.md + planning docs)

Check against all rules read from AGENTS.md. Common violations to watch for:

- Route → Service → Repository layering (routes never import repositories directly)
- TypeBox schemas, typed errors, OpenAPI coverage for all new endpoints
- Database: composite PKs, RLS, UUIDv7, `.limit()` on all list queries, `request.db()` in routes
- No `any`, no `ts-ignore`, no `eslint-disable` — fix the code, don't weaken the rules
- Error handling: no unnecessary try/catch, let errors propagate to boundaries (API global handler, React error boundaries)
- Stateless functional services over classes; `import * as serviceName` for namespacing
- External SDK errors mapped via `normalizeError`, not caught-and-rethrown in services
- Use `ValidationError` / `NotFoundError` / `ForbiddenError` — never raw `throw new Error()` in routes/services
- Terraform: update module README when changing variables/resources/scopes
- No client-side secrets (`NEXT_PUBLIC_*` forbidden)
- **Specialized helper preference** — when a helper family exists (`with-system-db.ts` exports `withSystemDb` + `withBootstrapSystemDb`; `with-scoped-db.ts` exports `withScopedDb` + `withScopedDbForOrg` + ...), the most specific helper wins. A bootstrap path manually constructing a `SystemDbContext` and calling `withSystemDb(ctx, fn)` should be `withBootstrapSystemDb(reason, fn)` — review-time check: `rg "withSystemDb\\(" apps/api/src` and audit each site for whether a more specific helper applies (PR #1059).

### Behavioral Change Flagging

When code changes observable behavior (not just refactoring), explicitly flag it and ask whether it's intentional:
- **Side effects of refactoring**: Extracting a function may change when/where a side effect fires. E.g., wiring `handleResetChat` to a dropdown `onChange` now steals focus on every agent change — was that intended, or should focus only fire from explicit "New Chat" actions?
- **Removed UI elements**: If a label, badge, or status indicator is removed while sibling elements are kept, flag the visual inconsistency. "System message label removed but agent label kept — intentional?"
- **Changed error propagation**: If errors that were previously swallowed now propagate (or vice versa), trace the operational impact. "Stripe errors now throw → on-call will see webhook retry spikes during outages where it previously saw silence."
- **Changed data scope**: Switching from user-scoped to org-scoped queries changes who sees what. Trace the tenancy implications.
- **Variable value-domain changes (second-order side effects)**: When a PR changes a variable's possible values (e.g., from "always non-null" to "possibly null" via a new conditional/dedup path), search for ALL downstream consumers of that variable in the same function. The consumer code may not be in the PR diff but can break silently. E.g., adding dedup that sets `assistantEvent = null` breaks a downstream `touchSession(lastEventAt: assistantEvent.createdAt)` that assumed non-null.
- **Conditional skip / dedup bypassing side effects**: When a PR adds a conditional skip (dedup, early return, guard clause), trace every side effect in the skipped code path. If any are still needed (metadata updates, timestamps, counters, cache invalidation), flag the gap. The primary behavior may work perfectly while metadata drifts silently.
- **Buffer-then-flush vs centralized fan-out**: When a PR persists agent / SDK events via an in-memory buffer that drains after the run completes (`bufferedEvents.push(...)` in `onMessage`, `for (const ev of bufferedEvents) await appendEvent(ev)` after `runAgent` resolves), check `planning/engine/sessions.md` for the centralization mandate. Buffer-then-flush means refresh during a 60s+ tool-using run still sees nothing — exactly the V2-380-class symptom mid-run. Either flag for centralization (write through `onMessage` live) or push back to tighten the AC wording in the PR description to "post-run only" (PR #997).

### Concurrency & Race Conditions

- **Async race conditions**: When an async operation is started without `await` and a dependent action follows immediately (e.g., `loadSession(id); focusComposer()`), the dependent action can fire before the async work completes. On slow networks, users may interact with stale state.
- **Database concurrency**: `SELECT → decide → INSERT/UPDATE` inside a transaction gives atomicity but not isolation. Two concurrent callers on the same unique key (e.g., sync-write and reconcile on `(orgId, provider, externalId)`) can both SELECT "not found" and both INSERT, causing a 23505 unique violation. Handle with: catch 23505 + retry, or `INSERT ... ON CONFLICT`, or `SELECT ... FOR UPDATE`.
- **Status flip-flop across write paths**: When multiple code paths write to the same column (e.g., sync-write writes `'ACTIVE'` uppercase, reconcile writes `best?.status` lowercase from SDK), every write triggers a spurious UPDATE. Normalize with a shared canonicalization function.
- **React concurrent mode**: `requestAnimationFrame` is not guaranteed to fire after React has committed state in concurrent mode. Use `useEffect` keyed on the relevant state instead.
- **Timer/interval leaks**: `setTimeout` / `setInterval` in components must be cleared on unmount. Without cleanup, callbacks fire on unmounted components.
- **React effect trigger vs latest-read separation**: For effects that hydrate persisted state (localStorage/session/URL-backed state), check whether deps represent true business triggers (namespace keys, route params, entity ids) or unstable tool identities (`loadSession`, `resetChat`, translation callbacks). If the effect should react only to the trigger but needs latest callbacks, use `useEffectEvent` rather than listing callback identities that can re-run hydration on ordinary renders.
- **Self-cancel from synchronous state writes**: Trace whether an async action called inside an effect synchronously sets a state value that is also in that effect's deps. Example: `loadSession(storedId)` sets `sessionId` before awaiting history; if `sessionId` is a dep, the effect cleanup can mark the first run `cancelled=true`, causing `.finally()` cleanup (`setHydrating(false)`, readiness updates) to be skipped.
- **Hydration readiness side-effect gaps**: When an effect marks a namespace/state as PENDING, every branch must either mark it READY or intentionally keep it blocked with a user-visible reason. Early returns for stale `sessionId`, selected-session bypasses, missing stored ids, or superseded loads must be traced so persistence watchers do not silently drop new state.
- **Cancelled load cleanup gaps**: If a namespace/key change cancels an in-flight async load, verify the new run resets loading/hydrating state before any early return. Otherwise the old `.finally()` may skip cleanup due to `cancelled=true`, and the new empty namespace may suppress the empty state forever.
- **Framework-level race prevention**: Before flagging a timing race condition, verify whether the framework prevents it internally. E.g., Next.js `router.replace` wraps navigation in `startTransition` and uses an action queue that **discards** superseded pending navigations (`app-router-instance.js` `dispatchAction`). Consecutive rapid `router.replace` calls don't produce intermediate `searchParams` values — the old navigate is discarded before its state is committed. Read the framework source when official docs are silent on internal behavior. Don't flag theoretical race conditions that the framework's architecture prevents.
- **Global state vs error-shape discriminators in catch blocks**: When a catch block uses _global state_ (`request.signal.aborted`, module-level flags, request-scoped booleans) to decide how to handle an error, verify no other error path can reach the same state. E.g., `request.signal.aborted === true` holds for any error that happens after a client disconnect — including genuine upstream failures (DNS, ECONNREFUSED, TLS) that coincided with the disconnect — causing silent swallowing of real bugs. Prefer error-shape discriminators: `err instanceof DOMException && err.name === 'AbortError'`. This matches the codebase's existing pattern in `useChat.ts:605` / `useChat.ts:723` / `query-retry.ts:43`.

### External API Integration

- **Pagination completeness**: Any call to a paginated external API (e.g., `connectedAccounts.list`) must drain all pages via cursor/offset. Returning only the first page silently truncates data. After switching from user-scoped to org-scoped queries, the result set can grow dramatically.
- **Safety caps on pagination**: Cursor-based loops (`do { ... } while (cursor)`) must have a `MAX_PAGES` cap. A stuck cursor (API returns the same value repeatedly) can infinite-loop. Log a warning when the cap is hit so truncation is visible in observability.
- **Time window constraints**: External APIs often reject timestamps outside a window. E.g., Stripe meter events reject timestamps older than 35 calendar days or more than 5 minutes in the future. When deriving timestamps from stored data (e.g., `interaction.endedAt`), the value can be arbitrarily old during replays.
- **Idempotency key stability**: Idempotency keys (e.g., Stripe `eventId`) must be stable across retries. Using an application-generated UUIDv7 (`interaction.id`) that regenerates on transaction rollback breaks idempotency — use a stable external ID (e.g., `providerId`) instead.
- **SDK documented guidance**: When using an SDK hook (e.g., Composio `afterExecute`) for a purpose beyond its documented intent (data transformation), document the deviation and risk mitigations.
- **SDK schema verification — silent parameter stripping**: When an SDK uses Zod `safeParse` for input validation (most modern SDKs do), unknown keys are **silently stripped**. A parameter that looks correct in the call site can be a complete no-op if it's not in the SDK's schema. **Always verify by reading the actual SDK schema** in `node_modules/.pnpm/{package}/dist/*.mjs` — search for the relevant `z.object({...})` definition. E.g., `@composio/core@0.6.4`'s `ConnectedAccountListParamsSchema` includes `orderBy` but NOT `orderDirection`, so `orderDirection: 'desc'` is dead code (PR #919).
- **SDK source over docs**: When SDK docs are incomplete or ambiguous on default sort order, available parameters, or API scope, read the SDK source directly. Check what's in the Zod schema, what fields are mapped to the API request, and what transformations are applied. This is the only reliable way to verify behavior.

### Security

- **SSRF prevention (backend only)**: Server-side code (`apps/api/`) that accepts a URL and fetches it must validate the hostname against an allowlist and add `redirect: 'manual'` to block redirect-based SSRF. **This rule does NOT apply to client-side code** (`apps/web/`).
- **Unbounded async fan-out**: `Promise.all` over user-controlled-length arrays needs a cap or concurrency limit.
- **Ownership chain verification**: When a function gates access (e.g., `verifyAccountOwnership`), trace ALL callers. A broad `catch` that converts all errors to `NotFoundError` can silently pass through transient failures (429, 500), weakening IDOR protection. Only map the specific "not found" error; let others propagate.
- **Error fallbacks weakening security**: If a delete/refresh flow treats `NotFoundError` as "already gone, skip cleanup", then converting transient errors to `NotFoundError` means transient failures silently skip cleanup. Narrow the catch to the exact 404-equivalent.
- **Tenant isolation on new patterns**: When introducing `includeDeleted`, `withTombstones`, or similar query modifiers, verify they don't bypass the `orgId` tenant filter. Read the actual `ScopedDb` implementation to confirm.
- **Default-deny direction of failures**: When a safety cap (e.g., MAX_PAGES) truncates data, the failure direction matters. For ownership verification, truncation should result in `NotFoundError` (deny access), not silent pass-through. For data listing, truncation loses data but doesn't open a security hole — log a warning.

### Operational Readiness

- **Log level appropriateness**: Revenue-impacting skips, security-relevant events, and unexpected upstream data must log at `warn` or `info`, not `debug`. `debug` is invisible at production log levels. The "no Stripe customer" skip is fine at `debug` (expected state); "no billable duration" needs `warn` (unexpected, may indicate upstream schema drift).
- **`logger.warn` doesn't reach Sentry**: The `sentryLogHook` in `packages/sentry/src/log-hook.ts:27` only captures at level >= 50 (`error` / `fatal`). For user-visible content failures (e.g. `.catch(() => log.warn({ err }, 'Failed to persist'))` swallowing assistant prose), the `warn` level means the failure rate is uncomputable from Sentry — silent data loss with no alert. Recommend either bumping to `logger.error` so the hook fires, or calling `Sentry.captureException(err, { tags: { ... } })` explicitly. Validation: `apps/workloads/src/lib/logger.test.ts:76` (PR #997).
- **Structured logging vs console.warn**: `console.warn` doesn't appear in CloudWatch structured log queries or Datadog dashboards. Operationally meaningful events must use the pino logger, not `console.*`.
- **Observability counters**: When a planning doc specifies metrics (e.g., `syncFailed`, `statusDriftDetected`), verify the implementation actually tracks and logs them. Missing counters mean missing SLO signals.
- **Logger error payload structure for triage**: When a catch logs an error, the payload should include enough structured context for triage: e.g. `log.error({ err, blockType: 'text', sessionId, agentId }, '...')` rather than `{ err }` alone. Without `blockType`, four call shapes (text, tool_use, thinking, tool_result) collapse into one Sentry issue and can't be bisected (PR #997).
- **Audit trail completeness**: When `withScopedDb` is called, check whether `userId`, `requestId`, and `reason` are set. Audit trigger columns getting `null` means forensic queries can't trace who triggered the write.
- **Runbook/rollout notes**: When a PR changes error propagation or alerting behavior, the PR description must document the operational impact. E.g., "On-call will see webhook retry spikes during Stripe outages where it previously saw silence."

### Sentry & Replay Configuration

When a PR touches `sentry.client.config.ts` / `sentry.server.config.ts` / `Sentry.captureException` sites / Sentry filters / replay configuration, also apply the patterns in the dedicated **`sentry-observability`** skill. Common review checks:

- **`ignoreErrors` vs `beforeSend`**: `ignoreErrors` drops events at the SDK level — also dropping the on-error replay buffer (replays attach AFTER `ignoreErrors` runs) and any explicit `Sentry.captureException` with diagnostic tags. For noise filtering with canary sampling and tag-aware bypass, `beforeSend` is correct.
- **Tag-aware bypass coverage**: When a `beforeSend` filter bypasses tagged captures (`'errorBoundary' in event.tags`), audit ALL `Sentry.captureException` sites for the bypass tag. Common misses: `useChat.ts` `loadSession` capture, `QueryProvider.tsx` retry-exhausted fallback (PR #1039).
- **Replay coupling documentation**: `beforeSend` returning `null` discards the on-error replay too, even with `replaysOnErrorSampleRate: 1`. Require an inline comment so the next reader doesn't assume replay survives.
- **Engine-agnostic constant naming**: When a regex covers multiple browsers (`Failed to fetch` Chromium + Firefox, `Load failed` Safari), name it `TRANSIENT_FETCH_FAILURE`, not `CHROMIUM_FETCH_FAILURE`.
- **Canary sampling math**: `2 ** 32` reads better than `0x100000000` and avoids the off-by-one of `0xFFFFFFFF` (which maps the max sample to exactly 1.0).
- **Framework-internal class anchoring**: Filters that match `error.name === 'ResponseAborted'` (Next.js) / `BailoutToCSRError` / `FST_ERR_VALIDATION` (Fastify) must comment the verified framework version. No semver stability on internal types — silent no-op on major bumps. See `sentry-observability` Rule 11 (PR #1110).
- **No 100% blackout on lifecycle errors**: For "expected noise" classes (`ResponseAborted`, `Failed to fetch`, `AbortError`), `beforeSend` must keep at least a 1% canary so infra-anomaly volume signals (keep-alive misconfig, ALB idle-timeout drift, CORS regressions) survive. See `sentry-observability` Rule 12.
- **Scrub before sample/branch**: Header / body scrubbing in `beforeSend` runs at the TOP of the function, before any sampling or filter branch. Otherwise the 1% canary leaks Authorization tokens. See `sentry-observability` Rule 13.
- **Runtime symmetry or YAGNI**: Filter additions in client config must have matching server-side filters (or a documented asymmetry) — and vice versa. Don't add edge-runtime filters when no edge routes exist. See `sentry-observability` Rule 14.
- **Pass-through (negative) test fixture**: Every `beforeSend` filter test must include at least one event that survives the filter. Positive-only tests pass when the matcher silently breaks (regex typo, framework rename). See `sentry-observability` Rule 15.

### Global Error Handler Patterns

When a PR touches `QueryProvider.tsx`, any `useActionErrorHandler`-style hook, `MutationCache.onError` / `QueryCache.onError`, or global toast helpers, also apply the patterns in the dedicated **`frontend-error-handling`** skill:

- **Classification ladder**: Global handlers must early-return per error class (transient network → typed `ApiErrorWrapper` → unknown). The unknown branch is the only one that fires `Sentry.captureException` + the generic `internal_error` toast. A flat capture-everything handler makes Sentry volume signals useless. See `frontend-error-handling` §1 (PR #1069).
- **Narrow `isTransientNetworkError`**: Predicate must combine `instanceof TypeError|DOMException` with the `is-network-error` library + explicit DOMException name check. Plain `error instanceof TypeError` silently downgrades real "cannot read properties of null" bugs to network-blip toasts. See §2.
- **Toast cooldown**: Per-category 5 s cooldown (module-level Map keyed on classification, not message) prevents spam during multi-query offline scenarios. See §3.
- **Server flag → client Provider**: Feature flags resolved server-side must reach client components via Provider, not be re-resolved client-side with hardcoded fallbacks. SSR/CSR divergence is invisible in unit tests. See §4 (PR #1072).

### API Response Mappers / Multi-Turn Capture

When a PR touches API response mappers or agent-runner code, also check `backend-db-conventions`:

- **Strip null JSONB before serialization**: `fast-json-stringify` can emit `{}` for runtime `null` JSONB values, which the FE then mis-reads as "feature enabled". Mapper must explicitly skip `null` keys, or the TypeBox schema must declare `Type.Union([..., Type.Null()])` (PR #1116).
- **Multi-turn agent capture per turn**: Code reading structured tags (`<chat_title>`, citations, telemetry) from agent runs must walk every assistant message during the stream — `result.output` is whatever the final turn was, often a tool call. Use a "last seen wins" comment to document the choice (PR #1071).

### Cross-Service Data Consistency

- **Multiple write paths for the same data**: When sync-write, reconcile, and detail-endpoint all derive the same field (e.g., `effectiveStatus`), all paths must use the same normalization. One writing uppercase and another writing lowercase causes UPDATE churn on every cycle.
- **Data carry-forward correctness**: When reading existing rows before upserting, prefer live rows over tombstones. If a tombstone and a live row coexist for the same key, carrying forward the tombstone's metadata overwrites the live row's data.
- **DRY across app boundaries**: Near-verbatim copies of business logic across `apps/api` and `apps/workloads` (e.g., `upsertManagedApp` vs `upsertByExternalId`) are a maintenance hazard. "Must stay in sync" comments are not enforcement. Flag for extraction to a shared package.
- **Enum/status value consistency**: When an external SDK returns lowercase values (e.g., `"active"`) but the codebase uses uppercase (`"ACTIVE"`), normalize at the boundary. Inconsistency causes spurious writes, wrong UI filtering, and status flip-flop.
- **JSONB column type awareness**: Kysely returns JSONB columns as **parsed JS objects** (not strings). Code that checks `typeof metadata === 'string'` will always be false for JSONB data from the DB. This is a common source of silent bugs in merge/comparison logic — e.g., `mergeMetadataJson` using `typeof existing !== 'string'` to guard against non-string input actually skips the merge for every DB-sourced value, losing critical fields like `slackAppId` during tombstone resurrection (PR #919). When reviewing metadata merge/compare logic, verify the code handles both string inputs (from `JSON.stringify`) and object inputs (from DB reads).
- **Metadata preservation on resurrection**: When a tombstone row is resurrected, metadata must be **merged** (spread existing + incoming), not **overwritten**. Fields like `slackAppId` in metadata are critical for cleanup flows (`deleteIntegration` → `parseSlackAppId` → Slack App cleanup). Overwriting loses them, creating orphans that are unrecoverable from the row alone.
- **Strip output markers consistently across paths**: When the same content is emitted on a live path (SSE) and a persisted path (DDB / Postgres), every path that reads or stores the text must run the same scrub function. E.g., SSE strips `<chat_title>...</chat_title>` via `drainTagBuffer` (`stream-publisher.ts:64`), and the final-output write strips via `stripChatTitle(result.output)` — but a new intermediate-text persistence branch that writes `block.text` raw leaks the literal tag into the DB and renders inside the assistant bubble on refresh. Walk every write-site and confirm the same scrub runs (PR #997).
- **Append-then-write retry safety**: When a job appends an event (`appendEvent(...)`) and then writes a downstream row (`withScopedDb(...)`), ask: if the downstream write throws, does SQS retry the whole job? If yes, the `appendEvent` has already run → the next attempt creates a duplicate event. Required mitigation: wrap the downstream write with `.catch((err) => log.warn({ err }, '...'))` so it's idempotent across retries, OR move the append AFTER the write (subject to the inverse risk: write succeeds but event missing). Default is the catch-and-log pattern, matching `persistContentBlock` / `deps.onMessage` (PR #992).
- **Dedup with `.at(-1)` only matches the last buffered item**: When a final-output write is deduped against a buffered list via `bufferedEvents.at(-1)`, an SDK final output that matches an EARLIER buffered event still produces a duplicate write. If the assumption is "last buffered item is the final output", document it inline with a comment naming the SDK guarantee. Otherwise widen the dedup to `bufferedEvents.some(...)` or normalize via a content hash (PR #997).

### Cross-Layer & Multi-Context Consistency

- **Fallback chain alignment**: When the same value is resolved in multiple layers, verify all layers use the same fallback chain in the same order.
- **Dead fallbacks / phantom data dependencies**: If code adds a fallback to a field, verify something in the system actually writes that field. A fallback to unwritten data is dead code.
- **Planning doc alignment**: When a PR adds new patterns, check whether existing planning docs describe a different target direction. Flag contradictions as blocking.

### Planning Doc Consistency (blocking)

- **Cross-doc conflicts**: When a PR changes behavior documented in another planning doc, that doc must be updated in the same PR. E.g., if `composio-multitenancy.md` says "no migration needed" but the PR introduces a migration, the planning doc must be updated. Flag as blocking.
- **Superseded sections**: When a PR replaces a mechanism described in a planning doc (e.g., removing `webhookAuth` plugin), add a superseded callout in the old doc pointing to the new one.
- **Executable rollout ordering**: Deployment steps labeled "MANDATORY order" must be actually executable in that order. A "Pre-deploy sanity check" listed after deploy steps is self-contradictory.
- **Idempotency semantics**: Upsert/reconcile sections should spell out when a write is a no-op vs update. Without this, implementers may write "select → always update" causing unnecessary DB churn.

### Error Handling (beyond AGENTS.md basics)

- **Never silently swallow errors**: `catch {}` or `catch { setError(true) }` with no logging is forbidden.
- **Scope catch blocks narrowly**: Only catch the specific error you can handle. A bare `catch` on `connectedAccounts.get()` that converts 429/500/network errors to `NotFoundError` masks transient failures and weakens security gates.
- **Don't conflate API errors with empty data**: Mapping `404 → []` conflates "not found" with "empty". Let 404 propagate as an error.
- **Abort/cancel detection must use error shape, not signal state**: When swallowing fetch abort errors under the AGENTS.md "expected errors" exception, discriminate via `err instanceof DOMException && err.name === 'AbortError'` — the spec-compliant shape thrown by both browsers and Node undici. Flag any catch block that uses `request.signal.aborted`, `controller.signal.aborted`, or similar global signal state as the discriminator: those return `true` for any error that lands after abort, silently hiding genuine upstream failures (DNS / ECONNREFUSED / TLS) when they coincide with a client disconnect. For route handlers, the swallow path should return `499` (Nginx convention, non-error in Datadog / Sentry dashboards), not `200` or `500`.
- **Pushback on "add `console.warn` for visibility" in expected-error catches**: When a reviewer requests a log on a branch that legitimately swallows expected noise (abort, 404→null, parse-or-fallback), check AGENTS.md before agreeing. AGENTS.md says: "Do NOT log-and-rethrow — the boundary already logs." The entire point of swallowing is reducing noise; adding a warn subverts that. Legitimate escape hatches: LB access logs (they already record 499 / status codes), dedicated metrics counters, or structured-event trace attributes — not a free-text `console.warn` that pollutes CloudWatch.

### Frontend Rules (from AGENTS.md + planning/frontend-architecture.md)
- **DS compliance**: All standard UI elements MUST use `@aetheronhq/ui` components — not raw DaisyUI HTML classes
  - `<input className="input ...">` → `<Input />`
  - `<input type="checkbox" className="checkbox ...">` → `<Checkbox />`
  - `<span className="badge ...">` → `<Badge />`
  - `<div className="loading ...">` → `<Loading />`
  - Hand-rolled empty states → `<EmptyState />`
  - Raw `<h2>` / `<h3>` → consider `<Heading />`
  - Raw `<div>` / `<span>` for text content → use `<Text />`
- Server Components by default; `'use client'` only when truly needed
- `DashboardPage` for list pages, `FormPage` for create/edit pages
- All text via `next-intl` (no hardcoded strings)
- DRY: no duplicated components across files
- **Server Component callback prop antipattern**: When a child is a Server Component, it should resolve `t` / `formatDate` / `format` itself via `getTranslations()` / `getFormatter()` from `next-intl/server`. Threading these through props as callbacks forces a Client Component ancestor and adds bespoke prop types for no benefit. Flag any `t: (key: string) => string` or `formatDate: (d: Date) => string` prop on a Server Component (PR #869).
- **Mixed `<label>` and DS `FormField` in the same form**: Forms must use one wrapper consistently. A composite input (chip list + button + inline input) wrapped in raw `<label htmlFor='...' className='text-field-name ...'>` while sibling rows use `FormField` will drift on every DS token bump. Flag for migration to `<FormField label hint>...</FormField>` (PR #869).
- **Clearable selects shipping `''` to required fields**: DS `SelectField` with a clear affordance returns `''` when cleared. For fields where `''` is invalid (IANA timezone, ISO currency / country, enum), `field.handleChange(v ?? '')` ships invalid empty strings to the API. Recommend `if (v) field.handleChange(v)` plus a non-empty validator (PR #869).
- **Default-prop-cascade footgun in shared layouts**: When a shared layout (e.g. `AppLayout`) receives a feature-flag-fed prop with a permissive default (`vapiVoiceId = ''`), audit every parent mount site. Sub-routes that don't pass the prop silently lands in the fallback branch. Recommend either making the prop required (TS surfaces the gap) or passing the real default at every mount (PR #1072).
- **Cross-component consistency sweep**: When a fix renders a human-readable label in component A, search the codebase for sibling components rendering the same data shape (e.g. raw `voiceId`, raw `userId`, `font-mono` spans of opaque ids). Reviewers will flag a sibling component still showing the raw value as "same symptom one component over" (PR #1072).
- **Layout pattern consistency between paired containers**: Pages with paired states (loading / error / empty / content) must use the same className shape across all branches. A loading `<div className='flex flex-1 min-h-0'>` paired with an error `<div className='flex-1 min-h-0'>` (missing `flex`) creates a visual jolt during state transitions and surfaces immediately on the support banner / sticky header edge case (PR #990).
- **`Intl.DateTimeFormat(undefined, ...)` causes SSR/client locale drift**: `toLocaleTimeString(undefined, ...)` and `new Intl.DateTimeFormat(undefined, ...)` use the runtime default locale — `en-US` on Node, `navigator.language` in the browser. Same call renders different output server-side vs client-side → hydration mismatch. Require explicit locale: `useLocale()` (client) / `getLocale()` (server) / `next-intl`'s `useFormatter()` / `getFormatter()` (PR #869).
- **`createdAt` vs `updatedAt` fallback for "last modified" displays**: New rows have `createdAt` populated but `updatedAt` is often `NULL` until the first edit. A `formatDate(updatedAt)` footer renders blank / "Invalid Date" for fresh entities. Verify the migration sets `updatedAt = createdAt` on insert via trigger, otherwise require fallback `updatedAt ?? createdAt` (PR #869).
- **Client `maxLength` mirroring backend caps**: When the API caps a string field at N chars (e.g. `reportInstructions` 4000, `reportPrompt` 16000), the textarea must set `maxLength={N}`. Without it, the user types the whole essay before learning from a backend 400. Cross-check the backend schema constants (PR #869).
- **Form values + `useState` duplicate state**: Don't track the same field in both `defaultValues` (form state) AND a separate `useState`. The two diverge the moment one writer skips the other (e.g. `recipients` in `ReportConfigFormValues` AND `useState<string[]>([])`). Recommend picking one (PR #869).
- **User-visible "wiring" fixes need a render test**: When a fix threads a new prop through a layout / context provider, adds a hook call to a component, or passes a new options-bag key (`{ defaultVoiceId }`), require a component-level render test asserting the user-visible string. Without it, anyone dropping the wiring would ship green (PR #1072).
- **Date-only string rendering without `timeZone: 'UTC'`**: When a server returns a `'YYYY-MM-DD'` string and the UI renders it via `format.dateTime(new Date(str), opts)` or `toLocaleDateString(str, opts)` without `timeZone: 'UTC'` in `opts`, the date shifts by one day for users west of UTC (the string is parsed as UTC midnight, then rendered in local TZ). Flag every render site of date-only API fields and require either `timeZone: 'UTC'` in the formatter options or splitting the string into y/m/d parts. Same fix usually has 2–3 paired surfaces (list cell + detail subtitle); when fixing one, grep for siblings (PR #1068).
- **`useMemo` of "current time" in long-lived modals**: `useMemo(() => somethingTimeBased(new Date()), [])` snapshots the value at first mount. Long-lived browser tabs that cross midnight still show yesterday's "this week" / "today" defaults. Recompute inside the modal-open `useEffect`, not in a top-level `useMemo` (PR #1068).
- **`useQuery` / `useMutation` defaults under polling and 409 paths**: Every new query/mutation must answer three questions explicitly: (1) `retry` — default 4 retries × 10s poll = 4 captures per failure per viewer, must be `retry: false` for short-poll loops, 409-conflict mutations, and expected-error paths; (2) `staleTime` — null/undefined results from transient failures should `staleTime: (q) => q.state.data === null ? 0 : N` so a flaky network doesn't pin the inline error banner for the full TTL; (3) `meta.suppressGlobalError` — flag for review (see next bullet) (PR #1068).
- **`meta.suppressGlobalError` asymmetry between `QueryCache` and `MutationCache`**: When a PR sets `meta: { suppressGlobalError: true }` on a `useQuery`, verify `QueryProvider`'s `QueryCache.onError` actually checks the meta flag (it's commonly only wired into `MutationCache.onError`). If the bypass is mutation-only, the flag is dead code on the query and the global error handler still fires. Fix is a 4-line mirror of the mutation branch in `QueryCache.onError` (PR #1068).
- **Status-enum branching without a `default` arm**: When a `switch (status)` or chained `if (status === 'literal')` exhaustively matches the four current backend enum values (`pending | processing | completed | failed`), the next-quarter addition (`cancelled`, `expired`, …) renders blank. Require a `default` arm with a placeholder fallback. Test it with `'cancelled' as unknown as Status` (PR #1068).
- **`if (data)` truthy check vs `typeof data === 'string'`**: Empty-string content (markdown for a week with no calls, optional notes) is a legitimate completed state. A truthy check funnels `data === ''` into the error/empty branch and the user sees "load failed" for healthy data. Require `typeof data === 'string'` discrimination when the union is `string | null` and the empty string is a valid value (PR #1068).
- **`listX({ limit: 100 })` for a single-name lookup**: When a detail / viewer page fetches a list only to translate one foreign-key id into a name, the cap is silently lossy past 100 entries (the lookup falls back to the bare UUID) and wastes a 100-row fetch on every render. Replace with `getX({ id: theOneId })`. Common offenders: `listAssistants`, `listVoices`, `listOrgs` on member detail pages. Don't apply when the page genuinely renders the list — that's a separate paginated-select problem (PR #1068).
- **Mutation error branches must satisfy the three-thing rule**: For every error path in a mutation, verify all three: (1) **user feedback** (toast / banner / inline message), (2) **Sentry signal** with `errorBoundary` tag (especially under `meta.suppressGlobalError`), (3) **recovery path** (cache invalidation for 409 because server state changed, polling restart, retry button re-enable). PR #1068 had the same hook miss each gap separately — 409 fired toast but didn't `invalidateQueries`, leaving the viewer stuck on `failed` Alert + Retry forever; generic-error `throw` was silently swallowed by `suppressGlobalError`; poll failures were fully silent because `retry: false + suppressGlobalError` together drop the Sentry signal too. Each branch needs all three (PR #1068).
- **Shared mutation hook test coverage**: When a PR extracts a shared hook from two callers (e.g. `useGenerateWeeklyReport` factored out of `GenerateReportModal` + `WeeklyReportViewer`), grep the test files for `renderHook(() => theNewHook(`. Both callers almost always `vi.mock` the hook entirely so the hook's own toast / invalidate / conflict-vs-generic logic has zero coverage. Require a dedicated `renderHook` test covering happy path, the conflict / domain-error branch (with `vi.spyOn(queryClient, 'invalidateQueries')` to assert the soft-success path), generic error (no invalidate, no callback, error propagates), and the no-callback default (PR #1068).
- **`router.refresh()` redundant alongside `invalidateQueries`**: `router.refresh()` re-runs the parent server component (with all its `await` data fetches) AND `invalidateQueries` re-fetches the React Query data — double work. Decide based on data ownership: server-fetched (`page.tsx` `await listX(...)`) → `router.refresh`; React Query (`useX` hook) → `invalidateQueries`. Flag any `onSuccess` that calls both (PR #1068).

- **Legacy URL redirects on page deletion/consolidation**: When a PR deletes standalone page routes and consolidates them (e.g., `/team` + `/organisation` → `/settings?tab=team` + `/settings?tab=organisation`), verify that `next.config.ts` has `redirects()` entries for the old paths. Missing redirects break bookmarks, email links, and browser history. Use `permanent: false` (302) for fresh consolidations. Flag as P1 if no redirects exist (PR #1208).
- **Navigation consolidation test completeness**: When nav items are consolidated (e.g., two sidebar entries → one), the nav layout test must assert both the new item (link with correct `href`) AND the removal of old items via negative assertions (`queryByText('OldItem').not.toBeInTheDocument()`). Using `getAllByText` without href validation can match unrelated UI text. Flag if negative assertions are missing (PR #1208).
- **E2e page object heading and tab-navigation alignment**: When a PR changes a page heading (e.g., "Team" → "Settings") or adds tabbed navigation, the e2e page object's `expectLoaded()` must match the current heading, and tab-specific navigation methods (e.g., `gotoTeamTab()`) must exist for e2e flows that target a specific tab. Without this, e2e flows silently land on the wrong tab and fail downstream assertions. Check all callers of `goto()` that depend on a specific tab's content being visible (PR #1208).

### Frontend Performance & State

- **Conditional rendering of inactive content**: Tab contents passed as ReactNode props mount and compute even when hidden. Conditionally render only the active tab's children to avoid unnecessary work as data grows.
- **Suspense boundaries for `useSearchParams()`**: In Next.js App Router, calling `useSearchParams()` outside a `<Suspense>` boundary opts the entire route out of static rendering. Either wrap the component in `<Suspense>` or lift search-params logic into a parent that does.
- **URL state preservation**: Filter and search state stored in React state is lost on reload and not shareable. Sync `?search=`, `?category=`, `?tab=` into the URL so the page is bookmarkable. Use the same `useSearchParams()` pattern for all.
- **Cross-filter state orphaning**: When filter options are derived from already-filtered data (e.g., category options from search-filtered items), the selected filter value can orphan when the options recompute without it. Derive filter options from the full unfiltered catalog, or reset the selection when it falls out of options.

### Frontend Robustness

- **Browser APIs that can reject** (e.g., `navigator.clipboard.writeText`, `Notification.requestPermission`) must have `.catch()` handlers.
- **Scroll bounds on dynamic content**: Large content needs `max-h-* overflow-y-auto`.
- **No semantic assumptions in formatting**: Don't blindly apply a format to all values of a type.
- **Unique element IDs**: IDs constructed from data must be guaranteed unique.
- **Consistent data fetching patterns**: Don't mix client-side `fetch(presignedUrl)` with server-side `fetchPresignedJson()`.
- **URL/path management**: Route paths should derive from `usePathname()`, not be hardcoded strings.
- **URL state sync**: When resolving a URL query param, update the URL to match the resolved state.

### Information Architecture & UX

- **Don't bury key metrics behind tabs**: KPIs that users scan across many items must remain above the tab bar.
- **Tabs must justify their existence**: If a tab only contains 2-3 data points shown elsewhere, fold into an existing section and remove the tab.
- **Tab naming must match the data**: "Logs" implies application logs; if the data is webhook payloads, call it "Webhooks".
- **Search results must retain context**: When grouped views flatten into search results, show category context (chips, badges) so users can orient.
- **Pagination must be stable**: "Show More" must not reorder or shuffle items already visible. Prefer per-group pagination over global budget.

### i18n (enhanced)

- **No string manipulation as i18n substitute**: Don't use `.replace(/_/g, ' ')` + CSS `capitalize` when translation keys exist.
- **Localise all formatted values**: Duration units, number formats, singular/plural forms.
- **ICU plural rules**: Use `{count, plural, one {# item} other {# items}}` for counts.
- **Dead locale keys**: Every key in the translation file must have a call site. Keys added without references are dead code per AGENTS.md.
- **Duplicate JSON keys**: Two objects with the same key in a JSON locale file silently overwrite — the second wins. Check for duplicate top-level keys.

### Schema & API Consistency

- **TypeBox array `maxItems`**: Array schemas in list responses should include `maxItems` to document the actual server-side limit.
- **Presigned URL responses**: Include `expiresAt` so clients know when to refresh.
- **Schema location**: TypeBox schemas at module level, not inside plugin/handler functions.
- **Parameter naming**: Function parameter names must match what they semantically represent.
- **Response envelope consistency**: List endpoints use `{ items: [...] }` or `{ data: [...] }` per convention.
- **`required` + nullable vs `Optional` for nullable DB columns**: When promoting a nullable DB column to the response schema, default to `required` + `T | null` (e.g. `Type.Union([Type.String(), Type.Null()])` in the response object plus the field name in `required`). Meaning: the key is **always present** in the JSON, the value can be `null`. `Optional(...)` means the key may be **absent entirely**, which breaks client type safety unless they handle `key in obj` checks. Reference: `title`, `sourceKey`, `lastEventAt` in `SessionDto`. Flag any `Optional` on a nullable DB column that has no semantic "key may be absent" justification (PR #992).
- **`?? null` redundancy**: `kysely-codegen` types nullable columns as `T | null` already, so `row.foo ?? null` in a mapper is dead code — flag for removal to keep the mapper consistent with surrounding nullable fields (PR #992).

### Testing

- **New endpoints need integration tests**: Every new API endpoint must have tests covering: happy path, empty result, 404, 403, and tenancy isolation (cross-tenant 404).
- **Don't remove edge-case tests during refactoring**: If the service still handles a case, the test protecting it must remain.
- **E2E selector robustness**: Target visible `<label>`, not hidden `<input>`. After renaming text, update all E2E assertions.
- **Tests must lock invariants, not just values**: A test named "passes identical timestamps across retries" should actually call the function twice and assert both get the same value — not just assert a single call's output. The test name must match what's actually exercised.
- **Retry/idempotency regression tests**: When a fix ensures idempotent behavior (e.g., deterministic timestamps for Stripe), add a test that calls the function N times and asserts all invocations produce identical output.
- **Mock alignment with code changes**: When a route renames `listUserConnections` to `listOrgConnections`, tests mocking the old name silently pass (mock is never called, assertion never fires). Test files not in the PR diff but affected by contract changes must be updated.
- **Parameterized tests for multiple code paths**: When a set (e.g., `SYNC_WRITE_TOOL_SLUGS`) contains multiple values, at least one test should use each value. A regression that removes a value from the set should fail a test.
- **Non-exported function testability**: When a private function has non-trivial branching (two-pass parsing, multiple fix rounds during review), consider exporting it for direct testing. Multiple review-round fixes on the same logic are a signal the code is easy to get wrong.
- **Test name vs assertion mismatch**: Read each test name, then verify the mock data and assertions actually exercise that scenario. A test named "writes when differs" that mocks identical values and asserts dedup is a naming bug — it hides the fact that the "differs" branch has zero coverage. Flag it.
- **Side-effect coverage on new branch paths**: When a PR adds a new conditional path (dedup, early return, guard), check if tests assert the side effects of that path — not just the primary return value. E.g., a dedup test should assert that metadata (timestamps, counters) is still correctly updated, not just that the duplicate write was skipped.
- **Mock fidelity**: When production code starts using the return value of a mocked function (e.g., checking `flushed.type` from `appendEvent`), verify the mock returns a shape that includes those fields. A mock returning `{ eventTs, createdAt }` when the code now reads `.type` will silently break the logic.
- **Route handler catch-branch coverage**: When a route handler adds a `try/catch`, tests must exercise BOTH the catch branch (mock fetch to throw the expected error class — e.g., `new DOMException('aborted', 'AbortError')`) AND the re-throw path (mock fetch to throw a non-matching error, assert `await GET(...)` rejects with the same error). The re-throw test is what catches the race-condition regression: without it, a future refactor that widens the catch discriminator silently restores the original bug. Template: `apps/web/src/app/api/csp-report/route.test.ts`.
- **Real-DB integration tests for migration-coupled code with branched WHERE**: Per AGENTS.md "never mock the database". When a function has a multi-branch `WHERE` guard (e.g. `setAutoTitle` writes IF `title IS NULL OR title_source = 'auto'` and skips when `title_source = 'manual'`), every branch needs a Testcontainers-backed test in the appropriate package (`packages/sessions`, `apps/api/test/integration/`, etc.). The most critical branch is the safety branch (the one that PROTECTS user data) — regression there silently overwrites and only surfaces in production support tickets. Mocking the DB in `agent-runner.test.ts` doesn't substitute (PR #992).
- **Upstream-request shape coverage in route-handler tests**: Tests that only assert response status leave the upstream call shape untested. Dropping the `Authorization` header, switching `Accept`, or removing `signal: request.signal` would all keep the suite green. Require: positive `expect.objectContaining({ headers, signal })` and, for conditional spreads (`...(lastEventId ? { 'Last-Event-ID': lastEventId } : {})`), a negative assertion (`expect.not.objectContaining({ 'Last-Event-ID': expect.anything() })`) so a refactor to always-include doesn't silently send `null` upstream (PR #1038).
- **E2e page object drift on page consolidation**: When a PR renames a page or changes its heading, check the e2e page object's `expectLoaded()` assertion. A stale heading regex (e.g., `/team/i` when the page is now "Settings") will fail silently in CI. Also check all e2e test callers: flows that need a specific tab (e.g., team management) must navigate to `?tab=team`, not rely on the default tab (PR #1208).
- **Navigation test positive + negative coverage**: When a PR consolidates nav items, the layout test must use `getAllByRole('link', { name })` with `getAttribute('href')` validation (not `getAllByText`) AND include `queryByText('OldNavItem').not.toBeInTheDocument()` for removed items. A consolidation test that only asserts the new link exists doesn't prove the old items were removed (PR #1208).
- **`vi.unstubAllGlobals()` cleanup**: When a test stubs `global.fetch` (or any other global) via `vi.stubGlobal`, expect an `afterEach(() => vi.unstubAllGlobals())`. Vitest's default `isolate: true` makes omission safe today, but explicit cleanup matches the convention in `proxy.test.ts` and `useChat.test.ts` (PR #1038).
- **Counter-based alternating mocks are fragile**: A `mutateCallIndex++ % 2`-style mock that alternates between two return values silently breaks the moment the component reorders or adds a third hook call. Recommend matching on argument identity (`vi.fn((action) => action === createX ? mockA : mockB)`) so the test stays correct under reorders (PR #869).
- **Case-insensitive validation and `MAX_*` cap coverage**: When the implementation does `.toLowerCase()` dedup or enforces a numeric cap, both invariants need explicit tests. The implementation is easy to delete in a refactor and the existing tests would still pass on the happy path. Flag missing coverage even when the code looks correct today (PR #869).

### Server Actions & Data Fetching

- **Don't use `'use server'` for presigned URL fetches**: Routing through a server action doubles latency for zero security benefit.
- **Consistent presigned URL pattern**: Once you decide on direct-client or server-proxy, apply it uniformly.

### Safety Caps & Defensive Coding

- **Generic utility functions need hard safety caps**: Reusable pagination functions must have internal upper bounds.
- **No `console.error` in production frontend code**: If the error is surfaced to the user via `<Alert>`, `console.error` is redundant.

### React Anti-patterns

- **Key collision on data-driven lists**: Use a stable unique ID for `key` (e.g., `key={item.id}`). Avoid using array index as key — it causes subtle bugs when items are reordered, inserted, or deleted. If no stable ID exists, a composite key from unique item properties is acceptable.
- **Config/data constants don't belong in hook files**: Move shared config to `@/lib/`.
- **DRY: extract shared hook patterns early**: Same stateful pattern in 2+ files → extract a hook.
- **Fragile magic strings from LLM output**: Normalize with `.trim().toLowerCase()` and check against a set of known null-ish values.
- **Hydration regression tests**: When fixing a React hydration/persistence effect, tests should cover both the user-visible persistence outcome and the scheduling contract. Add a test with unstable callback identities across rerenders when the bug came from callback deps, and a namespace/context-switch test when readiness gates can strand writes.

### Edge Cases & Robustness

For each significant code path introduced or modified, systematically check:
- **Empty/null inputs**: What happens when an array is empty, a string is `''`, a field is `undefined`? E.g., `toolCalls?.find(...)` when `toolCalls` is `[]` returns `undefined` — does the caller handle that?
- **Boundary conditions**: Zero items, single item, maximum items. E.g., `connectedAccounts.list` returning exactly one page vs multiple pages vs zero results.
- **Duplicate/repeated values**: Two tool calls with the same function name, two items with the same key, duplicate JSON keys in locale files.
- **Truthiness vs type-check mismatches**: `if (m.result)` drops valid empty strings `""`. Use `typeof m.result === 'string'` when the empty value is meaningful.
- **Mixed-case / normalization**: External APIs returning `"active"` while internal code expects `"ACTIVE"`. Acronyms in title-case utils (`ML` → `Ml`).
- **Stale/orphaned references**: Mocks that reference renamed functions, UI elements referencing deleted translation keys, test assertions referencing removed headings.
- **Whitespace-only content blocks**: A guard like `text.length > 0` admits `"\n"`, `"  "`, `"\n\n"`. Claude routinely emits whitespace-only text blocks between consecutive `tool_use` calls on cached / cold turns; persisting them produces empty bubbles between tool-call rows on refresh (live merges them via `appendToBlock` so it's invisible during streaming). Prefer `text.trim().length > 0`, or — if a downstream scrub function (e.g. `stripChatTitle`) already trims — verify the guard is consistent on both sides of any dedup comparison (PR #997).
- **`??` failing on empty strings**: When a default-value chain (`flag ?? defaultFlag`) reads from a feature-flag SDK or API that returns `string` (not `string | null`), `??` lets `''` through. Trace the read site: if the source returns the raw string verbatim (e.g. `@aetheron/feature-flags` `str()`), recommend `||` or normalize blanks at the boundary (PR #1072).

### Accessibility

- **Hover-only controls need keyboard support**: Add `aria-label` and `focus-visible:opacity-100` alongside `group-hover:opacity-100`.

### Naming Quality

Unclear or misleading names are a red flag — they often indicate deeper design confusion.

- **Self-explanatory names**: Variables, functions, classes, services, events, and queues should communicate their purpose without requiring the reader to trace the implementation. If you need to read the function body to understand what `processItems` does, the name has failed.
- **Generic names**: Flag `handleData`, `processItems`, `doStuff`, `manager`, `helper`, `utils` (when used as a class/module name, not a folder) as P3. These names hide the business intent.
- **Business-domain alignment**: Service and event names should reflect business concepts (`reservationService`, `bookingConfirmedEvent`), not implementation details (`dataProcessor`, `queueHandler`).
- **Cross-PR consistency**: The same concept must use the same name throughout the PR. If one file calls it `assistant` and another calls it `agent` for the same entity, flag the inconsistency.
- **Acronym and abbreviation policy**: Spell out domain terms on first use unless they are universally understood in the project (e.g., `org`, `db`, `id` are fine; `rpt` for "report" is not).

### General Quality
- Redundant / dead code (including dead fields in types that are computed but never read)
- Inconsistencies within the PR itself (e.g., using DS `Loading` in one file but raw DaisyUI in another)
- Breaking changes without backward compatibility strategy
- Missing tests for new functionality
- PR description drift — when implementation changes during review, the PR description must be updated to match the final state
- **Out-of-scope changes**: Flag any file changes unrelated to the PR's stated purpose. Unrelated test refactors, reverted commits from other PRs, or drive-by formatting changes should be in a separate PR. They add review burden and can introduce flaky behavior (e.g., replacing `vi.waitFor` with `setTimeout(0)` in a PR about message persistence).

## Step 4: Deduplicate, Evaluate Pushback & Filter by ROI

Cross-reference findings against existing GitHub comments:

```
For each issue found:
  → Search existing inline comments for same file + same concern
  → Check author's triage comment for "addressed" / "pre-existing" status
  → If already raised by another reviewer: skip
  → If already fixed in a later commit: skip
  → If genuinely new: add to comment list
```

**Evaluate author pushback on existing comments for logical soundness:**
- **Valid pushback**: "Merging these two guards loses diagnostic signal — they represent two different upstream issues" (legitimate: separate log lines for separate root causes).
- **Valid pushback**: "This is a private function; exporting it just for testing changes the module boundary" (debatable but reasoned).
- **Weak pushback**: "This is a follow-up" when the fix is cheap and the bug is real in the current PR (AGENTS.md: "If it needs doing, do it now").
- **Weak pushback**: "Not a real issue — X is always Y" without citing the type definition or SDK source (violates evidence-based principle).

When the author pushes back and cites evidence (SDK types, actual runtime behavior), accept the pushback. When the author pushes back with speculation, ask for the evidence.

### Comment ROI Filter

Before finalizing the comment list, evaluate the **return on investment** of each comment. The goal is delivering business value, not maximizing the number of findings.

For every P2/P3 finding, ask:
- **Modification cost**: How many files, tests, and deployments does this change require?
- **Risk of not changing**: Will this cause a bug, data loss, or user-visible issue? Or is it "not elegant enough"?
- **Proportionality**: Would this comment bump the task from 3 story points to 8? If so, discuss with the author or flag as a separate backlog item rather than blocking the PR.

**Drop or downgrade comments that fail the ROI test:**
- A P3 naming suggestion that requires renaming across 10 files + 3 test files → note as "future improvement" in the review body, not an inline comment
- A theoretical edge case that cannot occur given the current data model → don't post
- A "would be slightly cleaner if..." that adds no measurable safety or readability → don't post

**Never leave a comment just to fill space.** A review with zero comments and an APPROVE is a valid outcome for a clean PR.

Present a summary table to the user:

| Issue | Already Covered By | Action |
|-------|--------------------|--------|
| ... | reviewer-name | Skip |
| ... | — | **New → Comment** |

## Step 5: Draft Comments

Split findings into two streams:
- **Inline comments** — real issues only (P1 / P2 / P3). One comment per concern, anchored to the exact line.
- **Review body** — high-level summary + a consolidated `## Verified` block listing patterns that were checked and found correct.

For each new issue, draft an inline comment following these principles:

### Tone & Psychological Safety

A review comment is never inherently bad — unless it's poorly written. These rules ensure comments are constructive and actionable.

**Language:**
- Use **"we"** instead of **"you"** — "We could improve this by..." not "You should change this to..."
- Frame negatives as improvements: "This method can be optimized by X, Y, Z" not "This method is inefficient"
- Frame as observations/suggestions, not demands
- Use "worth considering" / "for consistency" / "not blocking" for minor items
- Offer escape hatch: "happy to address in a follow-up if preferred"

**Evidence & references:**
- Include the **rule reference** (e.g., "Per `frontend-architecture.md` §4")
- Include the **evidence** (e.g., "Checked `ScopedDb.selectFrom` at L109 — it already applies the filter")
- For external API constraints, **cite the docs URL**

**Actionability:**
- Every comment must propose a solution or outline possible improvements. Never just write "Improve this" or "This is wrong."
- If suggesting a change, include a brief code snippet showing the target state

**Volume management:**
- When finding **8+ issues**, add a note in the review body suggesting a synchronous discussion: "This PR has several structural concerns — a quick sync call might be more efficient than inline comments."
- If the PR has fundamental issues, prefer a direct conversation over 20 inline comments. A call saves time and allows real-time feedback.

### Format
- **Bold tag** at the start: `**DS component consistency**`, `**DRY**`, `**Race condition**`, `**Behavioral change**`, `**Operational impact**`, `**Data consistency**`
- Keep to 2-5 sentences
- Include a brief code suggestion when helpful
- No emoji unless the project convention uses them

### Inline vs. Review-Body Placement

**Only post inline comments for actual issues** — anything tagged P1 / P2 / P3, or any concern that needs the author to act on a specific line. Inline noise erodes review signal; an inline comment is a finger pointing at a problem, not a checkmark.

**Consolidate all positive verifications into the review body**, not inline. Verifications prove the review was thorough but don't require author action, so they belong in a single "Verified" block at the end of the review body where they can be skimmed in one pass. Posting `**VERIFIED**` comments inline floods the file diff with green-light noise that makes the real issues harder to spot.

### Positive Verifications (in review body, not inline)
For important patterns (security, tenancy, concurrency, doc-vs-code claims) that are **correctly implemented**, add a `## Verified` (or `## What's correct`) section to the review body that lists each verified claim with the code reference. Example block to include in the review body:

```
## Verified

- **Security** — org-scoped IDOR protection holds: `ScopedDb` applies orgId filter at `repo.ts:109`, delete checks ownership at `service.ts:142`.
- **Tenancy chain** — layered ownership: route auth → service org check → repo scoped query. All three gates present.
- **Best practice** — UUIDv7 generation is application-side, no DEFAULT in migration.
- **Concurrency** — `SELECT FOR UPDATE` prevents race between status check and update at `service.ts:88`.
- **Doc-vs-code** — STATUS_RANK values match `routes/webhooks/foo.ts:47` exactly; retry timing matches `handler.ts:614+`; etc.
```

This keeps the file diff focused on actionable comments while still demonstrating breadth of review.

### Severity Tags
- P1: Must fix before merge (bugs, security, data loss)
- P2: Should fix, may defer with justification (correctness, consistency)
- P3: Suggestion / nice-to-have (DRY, naming, follow-up improvements)

## Step 6: Determine Review Verdict & Post to GitHub

### Verdict Rules

Choose the review event based on the highest-severity finding:

| Highest severity in findings | `event` value | Meaning |
|------------------------------|---------------|---------|
| **P1** (must fix before merge) | `REQUEST_CHANGES` | Blocks merge until addressed |
| **P2 only** (no P1s) | `COMMENT` | Issues worth fixing but not blocking |
| **P3 only (1-3) or no issues** | `APPROVE` | Ship it (mention P3 suggestions inline) |
| **Many P3s (4+)** | `COMMENT` | Pattern of minor issues suggests a sync discussion |

Include the verdict rationale in the review body summary, e.g.:
- `REQUEST_CHANGES`: "Two P1 blockers must be resolved before merge."
- `COMMENT`: "No blockers — three P2 suggestions worth addressing."
- `APPROVE`: "Clean PR, no issues found. Ship it."
- `APPROVE` with P3s: "Approving — a few minor suggestions inline, none blocking."

### Post all comments as a single review via `gh api`:

```bash
# 1. Get head commit SHA
COMMIT=$(gh api repos/{owner}/{repo}/pulls/{number} --jq '.head.sha')

# 2. Determine event based on findings (set one of: REQUEST_CHANGES, COMMENT, APPROVE)
EVENT="APPROVE"  # default when no issues
# EVENT="COMMENT"  # when P2-only findings exist
# EVENT="REQUEST_CHANGES"  # when any P1 finding exists

# 3. Create review with all inline comments
gh api repos/{owner}/{repo}/pulls/{number}/reviews \
  --method POST \
  --field commit_id="$COMMIT" \
  --field event="$EVENT" \
  --field 'body=Review summary text' \
  --input - <<'PAYLOAD'
{
  "comments": [
    {
      "path": "path/to/file.tsx",
      "line": 42,
      "side": "RIGHT",
      "body": "Comment text here"
    }
  ]
}
PAYLOAD

# 4. If review lands as PENDING, submit it:
gh api repos/{owner}/{repo}/pulls/{number}/reviews/{review_id}/events \
  --method POST \
  --field event="$EVENT" \
  --field 'body=Review summary'
```

**Line numbers**: Use the actual file line number (not diff line), with `"side": "RIGHT"` for additions.

**Special characters in comment bodies**: If comment text contains single quotes, backticks, or the literal string `PAYLOAD`, the HEREDOC approach can break. In that case, write the full JSON payload to a temp file and use `--input /tmp/review-payload.json` instead of piping via HEREDOC.

**Review body**: Keep it neutral, no personal signatures. The opening sentence must state the verdict clearly. Structure:

```
## [Topic] Review

[Verdict line — must match the event type:]
- REQUEST_CHANGES: "Requesting changes — N P1 blocker(s) must be resolved before merge."
- COMMENT: "No blockers — N suggestions/issues worth addressing."
- APPROVE: "Looks good — approving. [Optional: N minor suggestions inline.]"

[1-3 sentence summary of what was checked.]

## Issues

[Optional brief list pointing at the inline comments — or skip if the inline tags speak for themselves.]

## Verified

- **[Category]** — [what was checked] matches [code reference].
- **[Category]** — [what was checked] matches [code reference].
- ...
```

The `## Verified` block is where positive verifications live — never inline.

## Step 7: Verify

After posting, verify comments are visible:

```bash
gh api repos/{owner}/{repo}/pulls/{number}/reviews/{review_id}/comments \
  --jq '.[] | "\(.path):\(.line) — \(.body[:60])..."'
```

## Checklist

Grouped by domain. **Skip sections that don't apply to the PR's scope.**

### Pre-Review (always)
- [ ] **Business context understood**: Can answer what problem this solves, who is affected, and what "done" looks like
- [ ] **Business flow validated**: Changes mapped to user journey; flow is intuitive and simplest path to the business goal
- [ ] **Approach evaluated**: The fix addresses root cause, not just symptoms. No simpler/more direct alternative was overlooked.
- [ ] **Trade-offs assessed**: Complexity cost, maintenance burden, dependency growth, and scope creep evaluated
- [ ] **Structural soundness**: If approach is fundamentally wrong, flagged as P1 with alternative direction
- [ ] AGENTS.md was read and all applicable rules were checked
- [ ] Relevant planning docs were read based on PR scope
- [ ] **No out-of-scope changes**: All modified files relate to the PR's stated purpose
- [ ] All claims are evidence-based (real code/types verified, not assumed)
- [ ] Edge cases systematically checked (empty, null, zero, max, duplicates) for each new code path
- [ ] **Naming quality**: Variables, functions, services are self-explanatory; no generic names; consistent terminology across the PR

### Backend (skip if frontend-only PR)
- [ ] Route → Service → Repository layering (routes never import repositories directly)
- [ ] TypeBox schemas, typed errors, OpenAPI coverage for all new endpoints
- [ ] Database: composite PKs, RLS, UUIDv7, `.limit()` on all list queries, `request.db()` in routes
- [ ] **Specialized helper preference checked**: `withSystemDb` callers audited for whether `withBootstrapSystemDb` (or other more specific helper) applies
- [ ] **Variable value-domain changes traced**: For each variable whose possible values changed, all downstream consumers checked
- [ ] **New branch paths checked for side-effect gaps**: Every new if/else, dedup, or early return traced for bypassed side effects
- [ ] **Catch-block discriminators use error shape, not global state**: `err instanceof DOMException && err.name === 'AbortError'`, not `signal.aborted`
- [ ] **Append-then-write retry safety verified**: Jobs that `appendEvent` then `withScopedDb` have idempotent guards
- [ ] **Buffer-then-flush patterns flagged** for centralization when planning doc mandates centralized SDK fan-out
- [ ] **Output marker stripping consistent** across SSE / persistence paths
- [ ] **`logger.warn` reviewed against Sentry threshold** — user-visible content failures use `logger.error` or explicit `Sentry.captureException`
- [ ] **OpenAPI nullable columns use `required` + `T | null`**, not `Optional`, unless absence is semantically distinct from null
- [ ] **`?? null` redundancy removed** on `kysely-codegen` `T | null` fields
- [ ] **SDK parameters verified against actual Zod schema** in `node_modules`
- [ ] **JSONB column handling verified**: handles both object (DB read) and string (`JSON.stringify`) inputs
- [ ] External API constraints verified via docs (URLs cited where applicable)

### Frontend (skip if backend-only PR)
- [ ] DS compliance: all standard UI elements use `@aetheronhq/ui` components
- [ ] Server Components by default; `'use client'` only when truly needed
- [ ] All text via `next-intl` (no hardcoded strings)
- [ ] **Server Components don't accept `t` / `formatDate` / `format` callbacks as props**
- [ ] **Paired containers (loading / error / empty / content) share identical className shape**
- [ ] **`Intl.DateTimeFormat` calls receive an explicit locale**, never `undefined`
- [ ] **Date-only string rendering** uses `timeZone: 'UTC'` (or splits y/m/d explicitly)
- [ ] **`useMemo` of "current time"** is recomputed in modal-open effects, not snapshot at mount
- [ ] **Every `useQuery` / `useMutation`** has explicit `retry`, `staleTime`, `meta.suppressGlobalError` choices
- [ ] **`meta.suppressGlobalError`** honoured by both `QueryCache.onError` AND `MutationCache.onError`
- [ ] **Status-enum branches** include a `default` arm with placeholder fallback
- [ ] **Empty-string content** discriminated via `typeof === 'string'`, not `if (data)`
- [ ] **Single-record lookups** use `getX({ id })`, not `listX({ limit: 100 })`
- [ ] **Mutation error branches** satisfy three-thing rule: user feedback + Sentry capture + recovery path
- [ ] **Shared mutation hook** has its own `renderHook` test
- [ ] **No `router.refresh()` alongside `invalidateQueries`** for the same data
- [ ] **Legacy URL redirects present** when pages are deleted/consolidated
- [ ] **React hydration/persistence effects checked**: Effect deps are true trigger values, self-canceling state writes are not deps

### Sentry & Observability (skip if no Sentry/logging changes)
- [ ] **Sentry `beforeSend` filter audited** for tag-aware bypass coverage and replay-coupling documentation
- [ ] See dedicated `sentry-observability` skill for full checklist

### Testing (always for PRs with code changes)
- [ ] New endpoints have integration tests (happy path, empty, 404, 403, tenancy isolation)
- [ ] **Test names match assertions**: Each test's name matches what the mock data + assertions actually verify
- [ ] **Real-DB integration tests** present for migration-coupled functions with branched WHERE guards
- [ ] **E2e page objects match current headings**; tab-specific `gotoXTab()` methods exist for tabbed pages
- [ ] **Navigation consolidation tests have negative assertions** for removed items
- [ ] Route handler catch-branch coverage: tests exercise BOTH the catch branch AND the re-throw path

### Infra / Terraform (skip if no infra changes)
- [ ] Module README updated when changing variables/resources/scopes
- [ ] No manual AWS console steps — all resources provisioned by Terraform

### Post-Review (always)
- [ ] **Comment ROI assessed**: Every P2/P3 comment is worth the author's time to fix
- [ ] **Tone is constructive**: Uses "we" not "you"; frames negatives as improvements
- [ ] **Volume check**: If 8+ issues, review body suggests a sync call
- [ ] No manufactured issues — every inline comment addresses a real concern (P1/P2/P3)
- [ ] **Inline comments are issues only** — positive verifications go in the review body's `## Verified` block
- [ ] Verified-safe patterns recorded in `## Verified` block with code references
- [ ] All findings cross-checked against existing reviewer comments; no duplicates
- [ ] Author pushback evaluated for logical soundness
- [ ] Comments reference specific rules/docs (e.g., "Per AGENTS.md", "Per frontend-architecture.md §4")
- [ ] Review body has no unintended signatures or names
- [ ] Line numbers are correct (file lines, not diff lines)

## Provenance

- Base checklist and foundational principles established from PR review cycles across the codebase.
- External API integration, security, operational readiness, and cross-service consistency sections built from patterns observed across PRs #867–#997.
- SDK schema verification and JSONB type awareness rules added after PR #919 (`feat/composio-org-level-sync`), where `orderDirection` was silently stripped by Composio SDK's Zod safeParse, and `mergeMetadataJson` failed on JSONB objects because `typeof existing !== 'string'` skipped the merge for DB-sourced metadata (Apr 2026).
- Framework-level race prevention check added after PR #1017 (`fix/integrations-search-lag`) (Apr 2026).
- React hydration / persistence scheduling checks added after PR #1044 (`fix/agent-config-render-loop`), where unstable chat callback identity caused repeated hydration/SSE reconnects, `sessionId` self-cancelled cleanup, and namespace readiness / hydrating early-return gaps needed review fixes (May 2026).
- Global-state vs error-shape discriminator rule (Concurrency), abort/cancel error-shape rule + `console.warn`-on-expected-errors pushback (Error Handling), and route-handler catch-branch test coverage added after PR #1038 (`fix/connect-web-27-session-stream-abort`). First pass used `request.signal.aborted` to detect client disconnects in an SSE proxy route; reviewer caught that the discriminator would silently 499 genuine upstream failures (ECONNREFUSED / DNS / TLS) that coincided with the disconnect. Fix switched to error-shape detection matching the codebase's existing `useChat.ts` / `query-retry.ts` pattern and added `route.test.ts` with abort→499, non-abort→rethrow, happy-path, and guard coverage (May 2026).
- Inline-vs-review-body placement rule (Step 4 / Step 5) added after PR #1040 (`docs(V2-273): tighten SMS planning doc`) review pass posted 5 inline `**VERIFIED**` comments alongside 3 issue comments. User feedback: positive verifications inline drown the real issues — consolidate them into a single `## Verified` block in the review body so the inline diff stays signal-only (May 2026).
- Specialized helper preference (Architecture & Code Rules) added after PR #1059 (`fix/structured-output-bootstrap-rls`), where the bootstrap path manually constructed a `SystemDbContext` and called `withSystemDb` instead of using the dedicated `withBootstrapSystemDb(reason, fn)` helper that exists for exactly this case (May 2026).
- Buffer-then-flush vs centralized fan-out, output-marker scrub consistency, append-then-write retry safety, `.at(-1)` dedup, whitespace-only content blocks, real-DB tests for branched WHERE, `logger.warn`-to-Sentry threshold, structured logger error payloads, `required` + nullable vs `Optional`, `?? null` redundancy added after PR #997 (`V2-380/fix-intermediate-messages`) and PR #992 (`V2-382/chat-session-title-persistence`). Both PRs landed multi-round reviewer fixes around mid-run vs post-run persistence, dedup edge cases, retry idempotency, and OpenAPI contract shape (Apr–May 2026).
- Frontend Rules expansion (Server Component callback prop antipattern, mixed `<label>` / `FormField`, clearable selects shipping `''`, default-prop-cascade footgun, cross-component consistency sweep, paired-container className shape, `Intl.DateTimeFormat(undefined)` SSR drift, `createdAt`/`updatedAt` fallback, client `maxLength` mirroring backend caps, form-values + `useState` duplicate state, render tests for wiring fixes) added after PR #869 (`V2-329/report-config-form`), PR #990 (`V2-376/agent-config-support-banner-cutoff`), and PR #1072 (`fix/voice-label-default-voice`) — recurring single-line wiring/consistency reviewer feedback (Apr–May 2026).
- Test-coverage expansion (upstream-request shape with negative `expect.not.objectContaining`, `vi.unstubAllGlobals()` cleanup, counter-based mock fragility, case-insensitive validation + `MAX_*` cap coverage) added after PR #1038 (`fix/connect-web-27-session-stream-abort`) and PR #869. The shape-locking rule is what catches refactors that drop `Authorization` / `Last-Event-ID` / `signal` from upstream calls without failing the suite (May 2026).
- Sentry & Replay Configuration section added after PR #1039 (`fix/sentry-ignore-fetch-network-failure`) — `ignoreErrors` was filtering both replay buffers and tagged diagnostic captures. Switched to `beforeSend` with 1% canary sampling + tag-aware bypass; review checklist now requires auditing all `Sentry.captureException` sites for the bypass tag, documenting replay coupling, and using engine-agnostic constant names. Detailed patterns live in the dedicated `sentry-observability` skill (May 2026).
- Frontend Rules expansion — date-only `timeZone: 'UTC'` rendering, `useMemo` of current time in modals, `useQuery`/`useMutation` defaults under polling, `meta.suppressGlobalError` cache asymmetry, status-enum forward-compat default branches, `typeof === 'string'` vs truthy for empty-string content, `getX({ id })` over `listX({ limit: 100 })`, mutation-error three-thing rule, shared-hook `renderHook` coverage, `router.refresh()` redundancy with `invalidateQueries` — added after PR #1068 (`V2-329/pr3-report-history-viewer`), six rounds of multi-reviewer feedback (CodeRabbit, Codex, Judge Codex, human reviewers) clustered around date/timezone drift, React Query defaults under polling × retry × N-viewer load, status-enum forward-compat, single-record API lookup shape, and mutation error completeness (May 2026).
- Frontend Rules — legacy URL redirects on page consolidation, navigation consolidation test positive + negative assertions, e2e page object heading + tab-navigation alignment — added after PR #1208 (`AP-491/settings-tabbed-navigation`). Consolidating `/team` + `/organisation` into a tabbed `/settings` page surfaced three recurring gaps: (1) no `redirects()` in `next.config.ts` for deleted routes → old bookmarks 404; (2) `AppLayout.test.tsx` only asserted the new "Settings" link without negative assertions for removed "Organisation"/"Team" items; (3) e2e `SettingsPage.expectLoaded()` still matched `/team/i` and `goto()` defaulted to the wrong tab, breaking `analyst.setup.ts` and `brand-isolation.e2e.ts` flows (May 2026).
- **Structural refactor** (May 2026): Reordered workflow to prioritize business context over code conventions. Added Step 0 (Business Context Gate), Step 2 (Business Flow Validation), Comment ROI Filter, Naming Quality section, and expanded Tone/Psychological Safety guidance. Restructured flat 60+ item checklist into domain-grouped sections (Pre-Review, Backend, Frontend, Sentry, Testing, Infra, Post-Review) with skip conditions. Fixed React key anti-pattern example, added verdict nuance for many P3s, and added `gh api` special-character note. Motivated by Dorin Baba's PR review principles — business context first, ROI of comments, naming matters, and psychological safety.
