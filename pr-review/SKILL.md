---
name: pr-review
description: >-
  Perform comprehensive GitHub PR review: analyze changes, check compliance with
  project rules (AGENTS.md, planning docs, design system), cross-reference
  existing reviewer comments, and post inline review comments via gh CLI.
  Use when the user shares a PR URL or asks to review a pull request.
---

# PR Review Workflow

## Overview

End-to-end PR review: understand changes → check compliance → deduplicate against existing comments → post inline comments to GitHub.

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

## Step 2: Analyze the PR

For each changed file, evaluate against:

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

### Cross-Layer & Multi-Context Consistency

- **Fallback chain alignment**: When the same value (e.g., brand, locale, feature flag) is resolved in multiple layers (e.g., JS runtime, Liquid/Handlebars templates, Terraform locals, subject lines), verify all layers use the **same fallback chain** in the same order. Divergent chains cause subtle inconsistencies (e.g., email From says "BookingBoost" but Subject says "Aetheron Connect").
- **Dead fallbacks / phantom data dependencies**: If code adds a fallback to a field (e.g., `user.app_metadata.brand`), verify something in the system **actually writes that field**. A fallback to unwritten data is dead code that gives false confidence. Flag it: "Nothing sets `X` — this fallback is unreachable."
- **Planning doc alignment**: When a PR adds new patterns, check whether existing planning docs describe a **different target direction** for the same area. If `planning/org-branding.md` says "eliminate per-brand conditionals" but the PR adds more, flag the contradiction. The PR should either align with the plan or explicitly update the planning doc to document the exception.

### Security

- **SSRF prevention (backend only)**: Server-side code (`apps/api/`) that accepts a URL and fetches it must validate the hostname against an allowlist and add `redirect: 'manual'` to block redirect-based SSRF. **This rule does NOT apply to client-side code** (`apps/web/`) — browsers handle CORS/same-origin natively. For frontend presigned URL fetches, a hostname allowlist is sufficient; `redirect: 'manual'` on the client actually breaks S3 cross-region 307 redirects.
- **Unbounded async fan-out**: `Promise.all` over user-controlled-length arrays (e.g., S3 object lists, webhook logs) needs a cap. Without `MaxKeys` or a hard limit, a single request can trigger thousands of downstream calls (DoS risk). If the fan-out is CPU-only (e.g., presigned URL generation via local HMAC signing), add a comment documenting why it's safe.

### Error Handling (beyond AGENTS.md basics)

- **Never silently swallow errors**: `catch {}` or `catch { setError(true) }` with no logging is forbidden. At minimum use `console.error`. AGENTS.md: "Do NOT silently swallow (`catch {}`)."
- **Scope catch blocks narrowly**: If a function does fetch + parse, only catch the parse — let transport/API errors propagate to React Query's `onError` or the global error handler. Don't turn a 403/404 into a misleading "parse error."
- **Don't conflate API errors with empty data**: Mapping `404 → []` conflates "resource not found" with "resource exists but has no data". If the API already returns `200 { items: [] }` for legitimate empty results, let 404 propagate as an error so the UI can distinguish failure from emptiness.

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

### Frontend Robustness

- **Browser APIs that can reject** (e.g., `navigator.clipboard.writeText`, `Notification.requestPermission`) must have `.catch()` handlers — otherwise you get unhandled promise rejections.
- **Timer cleanup**: `setTimeout` / `setInterval` in components must be cleared on unmount (return a cleanup function from `useEffect`). Failing to do so triggers React's "state update on unmounted component" warning.
- **Scroll bounds on dynamic content**: When rendering potentially large content (e.g., `JSON.stringify(data, null, 2)` in `<pre>`, long lists), add `max-h-* overflow-y-auto` to prevent blowing out the page layout.
- **No semantic assumptions in formatting**: Don't blindly apply a format to all values of a type (e.g., appending `%` to every number). Check the field's semantic meaning or add a format hint.
- **Unique element IDs**: IDs constructed from data (e.g., `panel-${type}-${size}`) must be guaranteed unique. Prefer using a truly unique field like the object key or database ID.
- **Consistent data fetching patterns**: If one component uses a server action for S3 presigned URL content (to avoid CORS/LocalStack issues), all similar components must use the same pattern. Don't mix client-side `fetch(presignedUrl)` with server-side `fetchPresignedJson()`.
- **URL/path management**: Route paths used in `router.replace()` / `router.push()` should derive from `usePathname()`, not be hardcoded strings. Hardcoded paths in multiple places break silently when routes change.
- **URL state sync**: When resolving/sanitizing a URL query param (e.g., invalid `?tab=` value), update the URL to match the resolved state so the visible UI and the location bar stay in sync. Preserve unrelated search params when updating.

### i18n (enhanced)

Beyond "all text via next-intl":
- **No string manipulation as i18n substitute**: Don't use `.replace(/_/g, ' ')` + CSS `capitalize` to render enum values when translation keys already exist. This bypasses the i18n system entirely.
- **Localise all formatted values**: Duration units, number formats, singular/plural forms — if a function accepts label overrides (e.g., `formatDuration(val, { hour: t('...') })`), use them.
- **ICU plural rules**: Use `{count, plural, one {# item} other {# items}}` for counts. Don't write `"{count} items"` which is grammatically wrong for count=1.

### Schema & API Consistency

- **TypeBox array `maxItems`**: Array schemas in list responses should include `maxItems` to document the actual server-side limit (e.g., `{ maxItems: 200 }` to match `MAX_WEBHOOK_LOGS`).
- **Presigned URL responses**: Include `expiresAt` so clients know when to refresh. Be consistent with existing presigned URL patterns in the codebase.
- **Schema location**: TypeBox schemas should be defined at module level, not inside plugin/handler functions. Follow the existing pattern in the same file.
- **Parameter naming**: Function parameter names must match what they semantically represent. Don't pass `interaction.providerId` as a parameter named `callId`.
- **Response envelope consistency**: List endpoints use `{ items: [...] }` or `{ data: [...] }` per project convention. Don't invent custom wrappers like `{ logs: [...] }`.

### Testing

- **New endpoints need integration tests**: Every new API endpoint must have tests covering: happy path, empty result, 404 (not found), 403 (role guard / non-admin), and tenancy isolation (cross-tenant 404).
- **Don't remove edge-case tests during refactoring**: If the service still handles a case (e.g., empty/null input returning `[]`), the test protecting that behavior must remain.
- **E2E selector robustness**: DaisyUI components often use hidden inputs (e.g., radio tabs). Playwright `click()` times out on zero-dimension elements. Target the visible `<label>`, not the hidden `<input>`. After removing/renaming headings or text, update all E2E assertions that reference them.

### Information Architecture & UX

- **Don't bury key metrics behind tabs**: When refactoring a page into tabs, audit what was previously visible at a glance. KPIs (scores, status, outcome) that users scan across many items must remain above the tab bar — not hidden behind a click. Moving them into a tab is a workflow regression for power users.
- **Tabs must justify their existence**: If a tab only contains 2-3 data points that are already shown elsewhere (e.g., a "Cost" tab showing duration + cost already in the header), fold the unique info (e.g., provider) into an existing section as a `<Badge>` and remove the tab. YAGNI — re-add when there's enough content.
- **Tab naming must match the data**: If the data is webhook payloads, don't call the tab "Logs" (which implies application logs). Use precise names like "Webhooks" or "Webhook Events".

### Server Actions & Data Fetching

- **Don't use `'use server'` for presigned URL fetches**: S3 presigned URLs carry authentication in the URL itself — they are designed for direct client-side access. Routing every fetch through a Next.js server action (client → server → S3 → server → client) doubles latency for zero security benefit. If CORS in dev (e.g., LocalStack) is the only reason, the workaround must be conditional or documented, not applied to production.
- **Consistent presigned URL pattern**: Once you decide on direct-client or server-proxy, apply it uniformly. Mixed patterns confuse future developers.

### Safety Caps & Defensive Coding

- **Generic utility functions need hard safety caps**: If a reusable function like `listObjects(bucket, prefix, maxKeys?)` paginates S3, add a hard upper bound (e.g., `const HARD_LIMIT = 1000`) inside the function itself. Callers can still pass lower values, but forgetting `maxKeys` won't trigger unbounded pagination.
- **No `console.error` in production frontend code**: If the error state is already surfaced to the user via `<Alert>` or similar UI, `console.error` is redundant and leaks implementation details in the browser console. Remove it.

### React Anti-patterns

- **Key collision on data-driven lists**: Using the item value itself as a React key (e.g., `<li key={item}>`) collides when two items are identical (common with LLM-generated content). Always use `key={\`${index}-${item}\`}` or a unique ID.
- **Config/data constants don't belong in hook files**: Static config mappings (e.g., tool call configs, icon maps) should live in a dedicated `types.ts`, `@/lib/` config file, or co-located with the feature. Hooks are for stateful logic. When hooks import from components, it inverts the dependency direction — move shared config to `@/lib/`.
- **DRY: extract shared hook patterns early**: When the same stateful pattern (e.g., clipboard logic with copied state + timer + cleanup) appears in 2+ files in the same PR, extract a hook immediately. It's cheapest at PR time; waiting creates tech debt.
- **Fragile magic strings from LLM output**: Comparing LLM-generated values against exact strings (e.g., `=== 'Unknown'`) is fragile — models may return `'unknown'`, `'N/A'`, `''`, or locale-specific variants. Always normalize with `.trim().toLowerCase()` and check against a set of known null-ish values.

### Accessibility

- **Hover-only controls need keyboard support**: Icon-only buttons visible only on `group-hover` are invisible to keyboard and screen-reader users. Always add `aria-label` for screen readers and `focus-visible:opacity-100` alongside `group-hover:opacity-100` to reveal the control on keyboard focus.

### General Quality
- Redundant / dead code
- Inconsistencies within the PR itself (e.g., using DS `Loading` in one file but raw DaisyUI in another)
- Breaking changes without backward compatibility strategy
- Missing tests for new functionality

## Step 3: Deduplicate

Cross-reference findings against existing GitHub comments:

```
For each issue found:
  → Search existing inline comments for same file + same concern
  → Check author's triage comment for "addressed" / "pre-existing" status
  → If already raised by another reviewer: skip
  → If already fixed in a later commit: skip
  → If genuinely new: add to comment list
```

Present a summary table to the user:

| Issue | Already Covered By | Action |
|-------|--------------------|--------|
| ... | reviewer-name | Skip |
| ... | — | **New → Comment** |

## Step 4: Draft Comments

For each new issue, draft an inline comment following these principles:

### Tone
- Diplomatic, concise, logical
- Frame as observations/suggestions, not demands
- Use "worth considering" / "for consistency" / "not blocking" for minor items
- Include the **rule reference** (e.g., "Per `frontend-architecture.md` §4")
- Offer escape hatch: "happy to address in a follow-up if preferred"

### Format
- **Bold tag** at the start: `**DS component consistency**`, `**DRY**`, `**Inconsistency**`, `**Architecture**`
- Keep to 2-5 sentences
- Include a brief code suggestion when helpful
- No emoji unless the project convention uses them

### Severity Tags (optional, if the team uses them)
- P1: Must fix before merge
- P2: Should fix, may defer with justification
- P3: Suggestion / nice-to-have

## Step 5: Post to GitHub

Post all comments as a single review via `gh api`:

```bash
# 1. Get head commit SHA
COMMIT=$(gh api repos/{owner}/{repo}/pulls/{number} --jq '.head.sha')

# 2. Create review with all inline comments
gh api repos/{owner}/{repo}/pulls/{number}/reviews \
  --method POST \
  --field commit_id="$COMMIT" \
  --field event="COMMENT" \
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

# 3. If review lands as PENDING, submit it:
gh api repos/{owner}/{repo}/pulls/{number}/reviews/{review_id}/events \
  --method POST \
  --field event="COMMENT" \
  --field 'body=Review summary'
```

**Line numbers**: Use the actual file line number (not diff line), with `"side": "RIGHT"` for additions.

**Review body**: Keep it neutral, no personal signatures. Example:
> ## [Topic] Review
> A few observations around [area]. Most are minor but worth aligning before merge.

## Step 6: Verify

After posting, verify comments are visible:

```bash
gh api repos/{owner}/{repo}/pulls/{number}/reviews/{review_id}/comments \
  --jq '.[] | "\(.path):\(.line) — \(.body[:60])..."'
```

## Checklist

Before posting, confirm:
- [ ] AGENTS.md was read and all applicable rules were checked
- [ ] Relevant planning docs were read based on PR scope
- [ ] All findings cross-checked against existing reviewer comments
- [ ] No duplicates with other reviewers
- [ ] Comments reference specific rules/docs (e.g., "Per AGENTS.md", "Per frontend-architecture.md §4")
- [ ] Tone is constructive, not prescriptive
- [ ] Review body has no unintended signatures or names
- [ ] Line numbers are correct (file lines, not diff lines)
