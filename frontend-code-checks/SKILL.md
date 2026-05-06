---
name: frontend-code-checks
description: Pre-submission checks for Aetheron Connect V2 frontend code in apps/web. Captures recurring reviewer feedback (HTML semantics, img hygiene, silent catch, utility reuse, DS prop conventions, visual state single-source-of-truth, React Query retry/staleTime/suppressGlobalError defaults, timezone-safe date rendering, status-enum forward-compat, mutation error three-thing rule, single-record API lookup over listX(limit:100), shared hook test coverage) so the agent surfaces these issues before the PR is opened. Use whenever writing or editing code under apps/web, adding a useQuery/useMutation, rendering a date-only string, branching on a backend status enum, looking up a single entity by id, extracting a shared hook, or before marking a frontend task complete.
---

# Frontend Code Checks (apps/web)

Lessons codified from review comments on Ella's PRs (PR #867 — EntityCard migration). Apply the relevant rule before declaring a frontend task done. **Don't skip `task typecheck && task lint`** — these rules catch what the compiler can't.

## Trigger scenarios

Run the relevant check when:

- Editing any file under `apps/web/src/**`
- Introducing a new utility in `apps/web/src/lib/**` (consistency sweep)
- Migrating a card / list item to DS `EntityCard`
- Adding a `<button>` inside a `<Link>` / `<a>` wrapper
- Adding an `<img>` whose `src` comes from user-supplied / external data
- Writing any `catch` block

## Rule checklist

### 1. Silent-catch — AGENTS.md §170

**Never** `.catch(() => {})` or `try { ... } catch {}` except in these three cases:
- `JSON.parse` / parse-or-fallback
- Expected 404 → null
- Cleanup / rollback before re-throwing

**Anti-pattern:**
```tsx
Promise.resolve(onAction(id)).catch(() => {});
```

**Fix:**
```tsx
void onAction(id);
```

Use `void` to satisfy `no-floating-promises` without swallowing. Errors should reach React Query `onError` / React error boundary / the global API error handler.

#### 1a. `void` operator is the house style — do NOT "fix" it

`eslint.config.mjs` sets:
- `@typescript-eslint/no-floating-promises: 'error'`
- `no-void: ['error', { allowAsStatement: true }]`

So `void asyncFn()` as an independent statement is **deliberate and required**. External tools (SonarQube `S3735`) will flag these as code smells — they're **false positives** in this repo. Confirm the rule with `rg 'no-floating-promises' eslint.config.mjs` before ever touching one.

- Do **not** refactor `void foo()` → `foo().catch(() => {})` — that violates rule §1 above.
- Do **not** refactor `void foo()` → `(async () => { await foo() })()` — unnecessary and less readable.
- When SonarQube flags `void`, mark as `Won't fix` / `False positive` on Sonar instead of changing code.
- The only acceptable change: if the callback was already `async`, drop the arrow wrapper and `await` directly.

### 1b. No nested ternaries — SonarQube `S3358`

More than one level of ternary nesting is hard to read and easy to misparse. This **is** a real lint concern (unlike §1a).

**Anti-pattern:**
```tsx
const label =
  id === CUSTOM
    ? t('custom')
    : id === OTHER
      ? t('other')
      : (card.categories[0]?.name ?? t('other'));
```

**Fix — extract a local helper (preferred inside `useMemo` / loops):**
```tsx
const labelFor = (id: string, card: Card): string => {
  if (id === CUSTOM) return t('custom');
  if (id === OTHER) return t('other');
  return card.categories[0]?.name ?? t('other');
};
```

**Fix — `if`/`else if` chain when no helper reuse is needed:**
```tsx
let label: string;
if (id === CUSTOM) label = t('custom');
else if (id === OTHER) label = t('other');
else label = card.categories[0]?.name ?? t('other');
```

One-level ternaries (`isX ? a : b`) are fine. The rule kicks in the moment a ternary sits in **either branch** of another ternary.

### 2. HTML semantics — no nested interactive elements

HTML5 forbids `<button>` (or another `<a>`) as a descendant of `<a>`. `e.preventDefault()` masks the click but screen readers still double-announce and keyboard focus lands twice.

**Anti-pattern:**
```tsx
<Link href="/foo">
  <EntityCard footer={<Button onClick={(e) => e.preventDefault()}>Delete</Button>} />
</Link>
```

**Correct approaches:**
- Use DS primitives that expose an `href` prop and render the anchor internally (e.g. a future `EntityCard` with `href` support).
- Or: make **only** the title/content a `<Link>` and render action buttons as siblings outside the anchor.
- Pick one convention and apply it to all cards in the same surface (don't mix `MyIntegrationsGrid` / `AvailableIntegrationsGrid`).

### 3. External `<img>` hygiene

Any `<img>` whose `src` comes from user/admin input (avatar URLs, integration logos, uploaded assets) must set:

```tsx
<img
  src={userSuppliedUrl}
  alt={label}
  loading="lazy"
  referrerPolicy="no-referrer"
  ...
/>
```

- `loading="lazy"` — rows below the fold defer the fetch.
- `referrerPolicy="no-referrer"` — prevents leaking the dashboard path to whatever host serves the asset.

### 4. Utility consistency sweep

When introducing a new helper (e.g. `formatPhoneNumber`, `truncateText`), **grep the whole `apps/web/src/**` tree** for all places that could benefit and migrate them **in the same PR**.

Reviewers will flag it as "visual inconsistency" if one card formats but another still renders raw.

Checklist before opening the PR:
- [ ] `rg "{thing the utility replaces}" apps/web/src` — zero remaining raw uses
- [ ] Add Vitest unit tests for the utility (`apps/web/src/lib/*.test.ts`)

### 5. DS `EntityCard` conventions

When migrating a card to `EntityCard`:

- **Pass `title` as a string** whenever possible. Strings get the built-in `isTitleTruncated` + hover-tooltip behaviour; ReactNode titles bypass it. If you need mono / special styling, apply it via the DS prop system, not by wrapping in a custom `<h3>`.
- **Drop `mt-1` / bespoke spacing on subtitle**. `EntityCard`'s `.entity-card-header-content` already stacks title-row + subtitle. Adding `mt-1` only on some cards creates visual drift across grid pages.
- **Set `interactive={false}`** for read-only cards (no `<Link>` / `<button>` wrapper).
- **Place the card inside a parent with `className="group"`** so the built-in title-hover colour and arrow-indicator activation fire.

### 6. `iconVariant` semantic rule

Per-card choice of `iconVariant` must follow the **semantic rule**, not a coin flip:

| Scenario | `iconVariant` |
|---|---|
| Entity is the primary brand subject | `"brand"` (always) |
| Entity displays an admin-uploaded avatar / logo | `undefined` when image present, `"brand"` when falling back to a glyph |
| Variant reflects runtime state (active / released / disabled) | `isActive ? "brand" : "secondary"` |
| Icon is third-party artwork (integration logo) | `undefined` (no coloured wrapper) |

### 7. Visual state — single source of truth

Don't derive a card's visual state from the **combination** of multiple fields.

**Anti-pattern:**
```tsx
const isFullyActive = isActive && isAssigned;
// shell tone, icon variant, status dot all toggled on isFullyActive
```

**Fix:** pick the canonical field (usually `status`) and drive shell tone / icon variant / `StatusDot` off it. Use the secondary field (e.g. `isAssigned`) only for structural content (which footer action to render), not visual state.

### 8. DRY shared components

If two or more cards render the same status indicator / helper markup, extract to `apps/web/src/components/shared/*` **in the same PR** — don't half-adopt. Specifically:

- `StatusDot` → `@/components/shared/StatusDot`
- Remove any re-export from feature barrels (e.g. `integrations/index.ts`) so there's exactly one import path.

### 9. KISS — reject heavy deps for narrow needs

Before adding an npm dep (especially UI / formatting libs), ask:

- What's the gzipped bundle cost? (`libphonenumber-js` ≈ 130 KB gzipped for what ends up being 2 country formats.)
- Does the happy path pass with a pure 10-line helper + tests?
- Is fallback-to-raw acceptable for the long tail?

If yes / yes / yes → KISS helper wins. Add the dep only when the feature actually needs parse / validate / i18n semantics.

### 10. URL ↔ local state sync patterns

When a search input needs both **instant feedback** (controlled input) and **URL bookmarkability** (`searchParams`):

**Recommended pattern** (used in `IntegrationsPageContent`):
```tsx
const [search, setSearch] = useState(() => searchParams.get('search') ?? '');
const deferredSearch = useDeferredValue(search);

// Sync back from external URL changes (back/forward)
useEffect(() => {
  const urlSearch = searchParams.get('search') ?? '';
  setSearch((prev) => (prev === urlSearch ? prev : urlSearch));
}, [searchParams]);

// Update both local state and URL on input
const handleSearchChange = useCallback((value: string) => {
  setSearch(value);
  updateParams({ search: value || undefined });
}, [updateParams]);
```

Key points:
- `useState` + controlled input for instant feedback; `useDeferredValue` defers expensive rendering
- `useEffect` syncs from `searchParams` on back/forward navigation
- `router.replace` (not `push`) avoids accumulating history entries
- The `prev === urlSearch` guard is sufficient — Next.js's action queue discards superseded navigations, so `searchParams` never shows intermediate values during fast typing (verified in `app-router-instance.js` source)
- Pass `deferredSearch` (not `search`) consistently to all child components that render filtered data
- **Alternative pattern** (Next.js official tutorial): `useDebouncedCallback` + uncontrolled `defaultValue` input. Simpler but adds 300ms delay to input feedback. Prefer the controlled pattern when instant responsiveness matters.

### 11. React effect hydration / persistence scheduling

For effects that hydrate persisted state (chat sessions, URL-backed state, localStorage-backed selections), separate:

- **Trigger values**: route params, namespace keys, entity ids — values that mean the user is now in a different logical place.
- **Latest values read by the effect**: callbacks, translation functions, `sessionId`, refs — values the effect needs to use, but whose identity/value changes should not by itself restart hydration.

Use `useEffectEvent` when the effect should react only to trigger values but must call the latest callbacks.

**Anti-pattern:**
```tsx
useEffect(() => {
  resetChat();
  void loadSession(storedId);
}, [scopedPersistKey, loadSession, resetChat]);
```

If `loadSession` / `resetChat` close over `useTranslations()` or other unstable deps, this re-runs hydration on ordinary renders, causing reset/load loops and repeated SSE reconnects.

**Preferred pattern:**
```tsx
const onHydrate = useEffectEvent((nextPersistKey: string) => {
  resetChat();
  void loadSession(readPersistedSession(nextPersistKey));
});

useEffect(() => {
  if (!scopedPersistKey) return;
  return onHydrate(scopedPersistKey);
}, [scopedPersistKey]);
```

When writing these effects, verify:

- A state value synchronously set by the effect's own async action (for example `loadSession()` setting `sessionId`) is not also a dependency that self-cancels the effect.
- Every `markNamespacePending(...)` path has a matching `markNamespaceReady(...)` path, including early returns and stale-state fall-through branches.
- Changing namespace while a previous async load is in flight cannot leave `hydrating` stuck because the old `.finally()` was skipped by `cancelled=true`.
- Regression tests cover unstable callback identity and namespace/context switches when the bug involves persistence or hydration scheduling.

### 12. Scope discipline

If a reviewer suggests extending the refactor to files outside the current PR's scope (e.g. `ReportConfigCard.tsx` still on `.assistant-card`):

- Acknowledge the point.
- Track as a follow-up ticket, not a last-minute addition.
- State the reason on the PR (avoid scope creep / avoid a divergent half-migration).

### 13. Abort detection in fetch `catch` blocks

When catching errors from `fetch()` (hooks, route handlers, SSE/streaming proxies), discriminate on the **error shape**, not global signal state. This is a concrete case of Rule §1 "swallowing expected errors" — AbortError is the expected noise we want to silence.

**Anti-pattern (race condition — silently hides real failures):**
```ts
try {
  const upstream = await fetch(url, { signal: request.signal });
} catch (err: unknown) {
  if (request.signal.aborted) return new Response(null, { status: 499 });
  throw err;
}
```

`request.signal.aborted` returns `true` for _any_ error that happens after the client disconnected — including genuine upstream failures (DNS, ECONNREFUSED, TLS, timeout). Those get silently swallowed as 499 instead of reaching Sentry.

**Correct pattern (matches `useChat.ts:605`, `useChat.ts:723`, `query-retry.ts:43`):**
```ts
try {
  const upstream = await fetch(url, { signal: request.signal });
} catch (err: unknown) {
  if (err instanceof DOMException && err.name === 'AbortError') {
    return new Response(null, { status: 499 });
  }
  throw err;
}
```

Node.js (undici) and browsers both throw `DOMException` with `name === 'AbortError'` when a fetch is aborted via signal — this is the Web spec-compliant shape. Other errors (TypeError for DNS, generic Error for ECONNREFUSED) still propagate to Sentry.

**For route handlers** (`apps/web/src/app/api/**/route.ts`):

- Return **`499`** (Nginx convention: "Client Closed Request"). AWS ALB surfaces this as `460` in access logs; Datadog / Sentry default dashboards treat 499 as non-error so it doesn't pollute error-rate SLOs.
- Do NOT use `200`, `204`, `408`, or `500` — each is semantically wrong (wasn't successful / wasn't a timeout / wasn't a server error).
- Do NOT add a `console.warn` on this branch just for "visibility". If reviewers request it, push back: the whole point of swallowing expected noise is reducing noise, and LB access logs already show the 499/460 count. Per AGENTS.md: "Do NOT log-and-rethrow — the boundary already logs."

**Test coverage is mandatory.** When adding a try/catch to a route handler, add a `route.test.ts` covering:

1. **Abort path**: `global.fetch` mocked to throw `new DOMException('aborted', 'AbortError')` → response status is 499
2. **Re-throw path** (the race-condition guard): fetch throws `new Error('ECONNREFUSED')` → `await GET(...)` rejects with the same error
3. Happy path + early-exit guards (400 for bad input, 401 for missing token, upstream non-OK passthrough)

Template: [`apps/web/src/app/api/csp-report/route.test.ts`](apps/web/src/app/api/csp-report/route.test.ts). Use `vi.mock('@/lib/api-config')` for auth and `vi.stubGlobal('fetch', ...)` for the upstream.

Without the re-throw test (#2), a future refactor that widens the catch or switches back to `signal.aborted` silently reintroduces the original bug.

### 14. `??` vs `||` for empty-string-default fields

`??` only catches `null` / `undefined`. Empty strings (`''`) slip through and are passed downstream verbatim.

**Anti-pattern (PR #1072):**
```tsx
const vapiVoiceId = flags.aetheronVapiVoiceId ?? defaultFlags.vapiVoiceId;
// If Flagsmith returns '', vapiVoiceId becomes '', and
// getVoiceLabel(id, { defaultVoiceId: '' }) short-circuits the new branch.
```

**Fix — choose by intent:**
```tsx
// (a) treat blank as "use the default":
const vapiVoiceId = flags.aetheronVapiVoiceId || defaultFlags.vapiVoiceId;

// (b) normalize at the read site, then pass through:
const raw = flags.aetheronVapiVoiceId;
const vapiVoiceId = raw === '' ? defaultFlags.vapiVoiceId : raw;
```

Trigger scenarios where this matters:
- Reads from `@aetheron/feature-flags` (`str()` returns the raw string verbatim, including `''`)
- API responses where backend has not promoted the column to `NOT NULL`
- Form submits where a cleared select returns `''` instead of `undefined`

### 15. Default-prop-cascade footgun in shared layouts

When a shared layout component has a permissive default (e.g. `vapiVoiceId = ''`), every parent mount site that doesn't pass the real value silently lands in the fallback branch.

**Anti-pattern (PR #1072):**
```tsx
// AppLayout.tsx
export function AppLayout({ vapiVoiceId = '' }: Props) { ... }

// app/(dashboard)/layout.tsx — passes the real flag
<AppLayout vapiVoiceId={flags.aetheronVapiVoiceId ?? defaultFlags.vapiVoiceId} ... />

// app/demo/layout.tsx — forgets to pass it; renders raw id forever
<AppLayout>{children}</AppLayout>
```

**Fix — pick one:**

(a) Make the prop required so TS surfaces every forgotten call site:

```tsx
type Props = { vapiVoiceId: string; ... };
```

(b) Pass the real default at every mount site (`demo/layout.tsx` etc. should read `defaultFlags.vapiVoiceId` server-side and forward it).

Audit checklist when adding a feature-flag-fed prop to a layout:
- [ ] `rg "<AppLayout"` (or whichever layout) — every match passes the prop
- [ ] If the prop is `string` rather than `string | null`, prefer required over a permissive default

### 16. Cross-component consistency sweep when fixing a UI symptom

When fixing a "renders raw value instead of human-readable label" bug in one component, search the codebase for OTHER components rendering the same data shape. Reviewers will flag a sibling component that still has the symptom.

**Example (PR #1072):** `AssistantDetail.tsx` was fixed to render `getVoiceLabel(voiceId, { defaultVoiceId: vapiVoiceId })`. The reviewer found `VoiceOverrideIndicator.tsx:40` directly below it still rendered the raw `4Hm8…` id in a `font-mono` span.

**Process:**
- `rg` the **data shape** (e.g. `voiceId`, `voiceOverride`, `font-mono.*Id`), not the component name
- For every match, decide: pipe through the same helper, hide when `=== defaultId`, or document why it stays raw
- Land all of them in the **same PR** — a half-migrated symptom is worse than the original because reviewers now have to track two states

### 17. Render test for user-visible "wiring" fixes

Even when the fix is just "pass a prop / resolve a value", add a `*.test.tsx` that mounts the component end-to-end and asserts the user-visible string. Without it, anyone dropping the hook call, the prop, or the options-bag key would ship green.

**Pattern (PR #1072):**
```tsx
it('shows default voice label when assistant uses default voiceId', () => {
  render(
    <FeatureFlagProvider vapiVoiceId='4Hm8iqw2xCZz1wwdml3H'>
      <AssistantDetail assistant={{ ...base, voiceId: '4Hm8iqw2xCZz1wwdml3H' }} />
    </FeatureFlagProvider>,
  );
  expect(screen.getByText('Aetheron Australian')).toBeInTheDocument();
});
```

Apply when the bug fix is any of:
- A new prop being threaded through a layout / context provider
- A hook call (e.g. `useFeatureFlags()`) being added to a component
- A new options-bag key (e.g. `{ defaultVoiceId }`) being passed to a util

### 18. `vi.unstubAllGlobals()` cleanup convention

When a test stubs `global.fetch` (or any other global) with `vi.stubGlobal`, add `afterEach(() => vi.unstubAllGlobals())` to match the convention in `apps/web/src/app/api/auth/proxy/proxy.test.ts:102-104` and `apps/web/src/hooks/useChat.test.ts:114-116`.

```ts
afterEach(() => {
  vi.unstubAllGlobals();
});
```

Vitest's default `isolate: true` makes this safe today, but explicit cleanup is a free guarantee against flakiness if a future config flip removes isolation.

### 19. Lock down the upstream request shape in route-handler tests

Route-handler tests that only assert response status leave the upstream call shape untested. Dropping the `Authorization` header, switching to a wrong `Accept` header, or removing `signal: request.signal` would all keep the suite green.

**Pattern (PR #1038):**
```ts
expect(global.fetch).toHaveBeenCalledWith(
  expect.stringContaining('/stream'),
  expect.objectContaining({
    headers: expect.objectContaining({
      Authorization: 'Bearer test-token',
      Accept: 'text/event-stream',
    }),
    signal: expect.any(AbortSignal),
  }),
);
```

For **conditional headers** (e.g. `Last-Event-ID` only forwarded when present), add a **negative assertion** so a refactor to always-include doesn't silently send `null`:

```ts
it('does not include Last-Event-ID header when absent from inbound request', async () => {
  // ...
  expect(global.fetch).toHaveBeenCalledWith(
    expect.anything(),
    expect.objectContaining({
      headers: expect.not.objectContaining({ 'Last-Event-ID': expect.anything() }),
    }),
  );
});
```

### 20. `AbortSignal.timeout()` throws `TimeoutError`, not `AbortError`

Heads-up for Rule §13: if you ever wrap a `fetch` with `AbortSignal.timeout(ms)` (or compose it via `AbortSignal.any([signal, timeoutSignal])`), the abort discriminator must accept both names:

```ts
if (
  err instanceof DOMException &&
  (err.name === 'AbortError' || err.name === 'TimeoutError')
) {
  return new Response(null, { status: 499 });
}
```

`AbortSignal.timeout()` rejects with `DOMException` `name === 'TimeoutError'`. The bare `AbortError` discriminator silently lets timeouts propagate to Sentry as 500s. Reference pattern: `apps/api/src/lib/proxy.ts:210`.

### 21. Layout pattern consistency between paired containers

When a page has paired states (loading / error / empty / content), all four containers must share the same className shape. Dropping a class on one of them creates a visual jolt during state transitions and shows up immediately on the support banner edge case.

**Anti-pattern (PR #990):**
```tsx
if (isLoading) return <div className='flex flex-1 min-h-0'><Loading /></div>;
if (error)     return <div className='flex-1 min-h-0'><Alert /></div>;
//                                  ^ missing `flex` — content jumps when transitioning
return <div className='flex flex-1 min-h-0'>{children}</div>;
```

**Fix:** extract the shared shell, or copy the className verbatim across all four containers. Diff the className strings character-by-character before opening the PR.

### 22. `Intl.DateTimeFormat(undefined, ...)` causes SSR/client locale drift

`Date.prototype.toLocaleTimeString(undefined, opts)` and `new Intl.DateTimeFormat(undefined, opts)` both fall back to the **runtime default locale**. On Node (SSR) that's typically `en-US` from the container; in the browser it's the user's `navigator.language`. The same `formatHourLabel(13)` call renders `1:00 PM` server-side and `13:00` client-side → hydration mismatch and inconsistent labels across page sections.

**Anti-pattern (PR #869):**
```ts
export function formatHourLabel(hour: number): string {
  const date = new Date(2000, 0, 1, hour);
  return date.toLocaleTimeString(undefined, { hour: 'numeric', hour12: true });
}
```

**Fix — accept an explicit locale param:**
```ts
export function formatHourLabel(hour: number, locale: string): string {
  const date = new Date(2000, 0, 1, hour);
  return date.toLocaleTimeString(locale, { hour: 'numeric', hour12: true });
}

// Client component
const locale = useLocale();
formatHourLabel(13, locale);

// Server component
const locale = await getLocale();
formatHourLabel(13, locale);
```

Reference pattern: `apps/web/src/components/dashboard/ActivityFeed.tsx`. Prefer `next-intl`'s `useFormatter()` / `getFormatter()` when available — they pin the locale automatically.

### 23. Counter-based alternating mocks are fragile

Don't use a module-level counter to alternate between two mock return values. The test silently breaks the moment the component reorders or adds a third hook call.

**Anti-pattern (PR #869):**
```ts
let mutateCallIndex = 0;
vi.mock('@/hooks/useActionMutate', () => ({
  useActionMutate: vi.fn(() => ({
    mutateAsync: mutateCallIndex++ % 2 === 0 ? mockCreate : mockUpdate,
    isPending: false,
  })),
}));
```

**Fix — match on argument identity:**
```ts
vi.mock('@/hooks/useActionMutate', () => ({
  useActionMutate: vi.fn((action: unknown) => ({
    mutateAsync: action === createReportConfig ? mockCreate : mockUpdate,
    isPending: false,
  })),
}));
```

The argument-identity match is stable across hook reorders and surfaces the intent inline.

### 24. Don't mix raw `<label>` with DS `FormField` in the same form

When a form uses DS `FormField` / `field.SelectField` / `field.TextareaField` for most rows, every row must use the same wrapper — including composite inputs (chip list + input + button).

**Anti-pattern (PR #869):**
```tsx
<FormField label={t('form.title')} hint={t('form.titleHint')}>
  <Input ... />
</FormField>

{/* one composite row sneaks in raw <label> */}
<div>
  <label htmlFor='recipients' className='text-field-name text-base-content'>
    {t('form.recipients')}
  </label>
  <Text className='mt-1 text-sm'>{t('form.recipientsHint')}</Text>
  <RecipientsChipInput id='recipients' ... />
</div>
```

**Fix:** wrap the composite in `FormField` too:
```tsx
<FormField label={t('form.recipients')} hint={t('form.recipientsHint')}>
  <RecipientsChipInput ... />
</FormField>
```

This keeps label / hint typography in sync when the DS bumps `--text-field-name` or related tokens.

### 25. Clearable selects + invalid empty values

DS `SelectField` with clearable behaviour returns `''` when the user clears the dropdown. For fields where `''` is not a valid value (IANA timezone, currency code, ISO country, etc.), `field.handleChange(v ?? '')` ships an invalid empty string to the API.

**Anti-pattern (PR #869):**
```tsx
<field.SelectField
  options={timezones}
  onChange={(v) => field.handleChange(v ?? '')} // '' reaches API → 400
/>
```

**Fix — block empty by ignoring the clear:**
```tsx
<field.SelectField
  options={timezones}
  onChange={(v) => { if (v) field.handleChange(v); }}
/>
```

Or remove the clear affordance for required fields. Pair with a `nonEmpty` validator on submit so the backend isn't the safety net.

### 26. Don't thread `t` / `formatDate` callbacks into Server Components

If a child is a Server Component, it can call `getTranslations()` / `getFormatter()` directly via `next-intl/server`. Threading callbacks through props forces the parent (or some ancestor) to be a Client Component just to provide them, cascading `'use client'` unnecessarily and adding bespoke prop types.

**Anti-pattern (PR #869):**
```tsx
type Props = {
  t: (key: string) => string;
  formatDate: (d: Date) => string;
  config: ReportConfigDto;
};

export function ReportConfigDetail({ t, formatDate, config }: Props) { ... }
```

**Fix — make the child `async` and resolve in-place:**
```tsx
import { getTranslations, getFormatter } from 'next-intl/server';

export async function ReportConfigDetail({ config }: { config: ReportConfigDto }) {
  const t = await getTranslations('common.reports');
  const format = await getFormatter();
  // ...
}
```

Now `'use client'` can drop off the parent and the bespoke callback prop types disappear.

### 27. Form values + `useState` is a duplicate source of state

Don't track the same field in **both** `defaultValues` (form state) **and** a separate `useState`. The two diverge the moment one writer skips the other.

**Anti-pattern (PR #869):**
```tsx
type ReportConfigFormValues = {
  // ...
  recipients: string[]; // form-state field, never read
};

const [recipients, setRecipients] = useState<string[]>([]); // local-state field, actually used
const onSubmit = (values: ReportConfigFormValues) => {
  saveConfig({ ...values, recipients }); // form-state value silently overridden
};
```

**Fix — pick one:**

(a) Fully in form state — register the field, use `field.handleChange`, drop the `useState`.

(b) Fully in local state — remove `recipients` from `ReportConfigFormValues` and merge in `onSubmit` only.

Same rule for arrays, draft text, and any "complex" field where a custom UI feels easier than form integration.

### 28. `createdAt` vs `updatedAt` fallback for "last modified" displays

New rows have `createdAt` populated but `updatedAt` is often `NULL` until the first edit (depending on migration / trigger setup). Rendering a "Last updated" footer from `updatedAt` alone shows blank / "Invalid Date" for fresh entities.

**Anti-pattern (PR #869):**
```tsx
<Text className='text-base-content-soft'>
  {t('detail.lastUpdated', { date: formatDate(config.updatedAt) })}
</Text>
```

**Fix — fall back to `createdAt` (or render both with explicit labels):**
```tsx
<Text className='text-base-content-soft'>
  {t('detail.lastUpdated', { date: formatDate(config.updatedAt ?? config.createdAt) })}
</Text>
```

Verify against the actual migration: if the DB sets `updatedAt = createdAt` on insert via trigger, fallback isn't needed — but read the migration before assuming.

### 29. Client `maxLength` must mirror backend caps

If the API caps a string field at `N` chars (e.g. `reportInstructions` at 4000, `reportPrompt` at 16000), set `maxLength={N}` on the input. Without it, the user types the whole essay, hits Save, and learns from a backend 400.

**Pattern (PR #869):**
```tsx
<field.TextareaField
  label={t('form.reportInstructions')}
  maxLength={4000} // matches API cap in apps/api/src/schemas/report-config.ts
  rows={4}
/>
<field.TextareaField
  label={t('form.reportPrompt')}
  maxLength={16000}
  rows={8}
/>
```

When changing a backend cap, `rg "{cap}"` to find the mirroring frontend constant and update both in the same PR.

### 30. Test case-insensitive validation and max caps

When a form does case-insensitive dedup (`.toLowerCase()`) or enforces a `MAX_*` cap, both invariants need explicit tests. The implementation is easy to delete in a refactor and silently regress.

**Pattern (PR #869):**
```ts
it('rejects case-insensitive duplicate recipient', async () => {
  const { user } = renderForm();
  await addRecipient(user, 'a@b.com');
  await addRecipient(user, 'A@B.COM');
  expect(screen.getByText(t('validation.duplicateEmail'))).toBeInTheDocument();
});

it('disables Add button at max recipients cap', () => {
  renderForm({ defaultValues: { recipients: Array(20).fill(0).map((_, i) => `r${i}@example.com`) } });
  expect(screen.getByRole('button', { name: t('form.addRecipient') })).toBeDisabled();
});
```

### 31. `?? null` is a no-op on `T | null` codegen fields

`kysely-codegen` already types nullable columns as `T | null`. `row.titleSource ?? null` is dead code — remove for consistency with the surrounding mapper.

**Anti-pattern (PR #992):**
```ts
return {
  id: row.id,
  title: row.title,                       // typed string | null already
  titleSource: row.titleSource ?? null,   // ← redundant
  sourceKey: row.sourceKey,
};
```

**Fix:**
```ts
return {
  id: row.id,
  title: row.title,
  titleSource: row.titleSource,
  sourceKey: row.sourceKey,
};
```

If you need to coerce `undefined` → `null`, the column type is wrong upstream — fix the schema, not every read site.

### 32. Date-only strings need `timeZone: 'UTC'` when rendered

`new Date('YYYY-MM-DD')` parses as **UTC midnight**, not local. `format.dateTime` / `toLocaleDateString` then renders in the runtime locale's timezone, so users west of UTC see the **previous calendar day**.

**Anti-pattern (PR #1068):**
```tsx
const start = format.dateTime(new Date(report.periodStart), {
  month: 'short',
  day: 'numeric',
}); // user in UTC-5 sees April 12 for periodStart='2026-04-13'
```

**Fix — pin the timezone explicitly:**
```tsx
const start = format.dateTime(new Date(report.periodStart), {
  month: 'short',
  day: 'numeric',
  timeZone: 'UTC',
});
```

Apply everywhere a date-only string from the API is rendered. The fix is the same pair of surfaces in this PR (`WeeklyReportList.formatPeriod` + `WeeklyReportViewer.periodLabel`) — when you see one, grep the codebase for siblings.

**Decision rule:** if the string has no timezone/offset (`'2026-04-13'`, `'2026-04-13T00:00:00'`), pin `timeZone: 'UTC'`. If it has an offset (`'2026-04-13T00:00:00+10:00'`), local rendering is fine.

### 33. `useMemo` of "current time" doesn't refresh on modal reopen

`useMemo(() => somethingTimeBased(), [])` snapshots the value when the component first mounts. Long-lived tabs cross midnight and the modal still shows yesterday's defaults.

**Anti-pattern (PR #1068):**
```tsx
const defaults = useMemo(() => lastCompletedWeek(new Date()), []);
const [periodStart, setPeriodStart] = useState(defaults.start);

useEffect(() => {
  if (!open) return;
  setPeriodStart(defaults.start); // still last week's stale value
}, [open]);
```

**Fix — recompute inside the open-reset effect:**
```tsx
useEffect(() => {
  if (!open) return;
  const fresh = lastCompletedWeek(new Date());
  setPeriodStart(fresh.start);
  setPeriodEnd(fresh.end);
}, [open]);
```

Trigger scenarios: modal/dialog defaults derived from current time, "today" pickers, "this week" filters, recently-viewed shortlists.

### 34. Three-decision checklist for every `useQuery` / `useMutation`

Before declaring a query/mutation done, answer all three questions explicitly. Defaults bite hardest when interval × retry × component count multiplies.

| Decision | Default | When to override |
|---|---|---|
| **`retry`** | 4 retries | `retry: false` for: short-poll intervals (every-10s polls × 4 retries = 5s of flooding per failure), 409-conflict mutations (retry hits the same row → phantom toast race), expected-error paths (`not_found`, `unauthorized`) |
| **`staleTime`** | `0` | For data with TTL bounds (presigned URLs ~1h), set just under the TTL. **Conditional `staleTime` for null fallbacks**: `staleTime: (q) => q.state.data === null ? 0 : 50 * 60_000` — null means transient failure, refetch on next mount instead of pinning the inline error banner. |
| **`meta.suppressGlobalError`** | dead code unless your `QueryProvider` honours it (see §35) | Polling, expected 404s, or any path where the local UI already surfaces the failure |

**Anti-pattern (PR #1068):**
```ts
return useQuery({
  queryKey: ['report-poll', id],
  queryFn: () => getWeeklyReport({ path: { id } }),
  refetchInterval: 10_000, // ← 10s × default retry: 4 = 4 captures per error per viewer
});
```

**Fix:**
```ts
return useQuery({
  queryKey: ['weekly-reports', id],
  queryFn: async () => { /* + Sentry capture inside */ },
  refetchInterval: (q) =>
    q.state.data?.status === 'pending' || q.state.data?.status === 'processing'
      ? 10_000
      : false,
  retry: false,
  meta: { suppressGlobalError: true },
});
```

### 35. `meta.suppressGlobalError` is asymmetric — verify both caches honour it

`QueryClient` exposes two error pipelines: `MutationCache.onError` and `QueryCache.onError`. The `meta.suppressGlobalError` convention is opt-in **per cache** — wiring it on one without the other makes the flag dead code on the other side.

**Anti-pattern (PR #1068, before fix):**
```ts
// QueryProvider.tsx
const queryClient = new QueryClient({
  mutationCache: new MutationCache({
    onError: (error, _vars, _ctx, mutation) => {
      if (mutation.options.meta?.suppressGlobalError) return; // ← only honoured here
      handleQueryError(error);
    },
  }),
  queryCache: new QueryCache({
    onError: (error) => handleQueryError(error), // ← always fires globally
  }),
});

// useReportContent.ts — flag is dead code
return useQuery({ /* ... */, meta: { suppressGlobalError: true } });
```

**Fix — mirror the bypass in `QueryCache.onError`:**
```ts
queryCache: new QueryCache({
  onError: (error, query) => {
    if (query.meta?.['suppressGlobalError']) {
      console.warn('[Query Error - handled locally]', error);
      return;
    }
    handleQueryError(error);
  },
}),
```

**Review-time check:** when you see `meta: { suppressGlobalError: true }` on a `useQuery`, grep `QueryProvider.tsx` for the `QueryCache.onError` branch. If it's missing, the flag is doing nothing.

### 36. Status enum from backend is forward-compat — UI must have a `default` branch

Backend `status` columns grow over time (`pending | processing | completed | failed` → adds `cancelled` next quarter). UI `switch` / multi-`if` chains that exhaustively match the four current literals render **blank** for any future status the UI hasn't been taught about.

**Anti-pattern (PR #1068):**
```tsx
if (status === 'pending') return <Pending />;
if (status === 'processing') return <Processing />;
if (status === 'completed') return <Completed />;
if (status === 'failed') return <Failed />;
// future 'cancelled' renders nothing
```

**Fix — explicit default with a placeholder:**
```tsx
if (status === 'pending') return <Pending />;
if (status === 'processing') return <Processing />;
if (status === 'completed') return <Completed />;
if (status === 'failed') return <Failed />;
return <CardValue empty>{tViewer('noContent')}</CardValue>;
```

**Apply when:** rendering badges, content cards, or icons keyed off a backend enum string. Same rule for `WeeklyReportStatusBadge`-style components — at least pass the raw string through as a fallback label.

**Test it:** cast a fictional status (`'cancelled' as unknown as Status`) into the component and assert the default branch renders.

### 37. `if (data)` truthy check vs `typeof data === 'string'` — empty strings are valid content

For optional content that may legitimately be empty (markdown for a week with no calls, comment for a row with nothing flagged), a truthy check funnels `data === ''` into the error/empty branch and the user sees "load failed" for healthy data.

**Anti-pattern (PR #1068):**
```tsx
if (contentQuery.data) return <MarkdownViewer content={contentQuery.data} />;
return <ContentLoadError />; // empty markdown lands here
```

**Fix — discriminate on type:**
```tsx
if (typeof contentQuery.data === 'string') return <MarkdownViewer content={contentQuery.data} />;
return <ContentLoadError />; // only `null` / `undefined` reach here
```

**Decision table:** when the union is `string | null` and `''` is a valid value, use `typeof === 'string'` (or `data !== null`). When the union is `string | undefined` and the difference doesn't matter, truthy is fine — but document the assumption.

**Trigger scenarios:** markdown / HTML / plain-text content from blob storage, optional descriptions / notes / instructions, transcripts that may be silent.

### 38. `listX(limit:100)` for a single name lookup is the wrong tool — use `getX(id)`

Pages that fetch a list only to translate one foreign-key id into a name pay the cost of the full list AND silently lose the lookup once the org crosses the cap.

**Anti-pattern (PR #1068):**
```tsx
const [config, assistants] = await Promise.all([
  getReportConfig({ path: { id } }),
  listAssistants({ query: { limit: 100 } }),
]);
const assistantMap = new Map((assistants.data?.items ?? []).map((a) => [a.id, a.name]));
const assistantName = assistantMap.get(config.assistantId) ?? config.assistantId; // bare UUID for 101st+
```

**Fix — single-record fetch:**
```tsx
const result = await getReportConfig({ path: { id } });
const config = result.data;
let assistantName: string | null = null;
if (config.assistantId) {
  const assistantResult = await getAssistant({ path: { id: config.assistantId } });
  assistantName = assistantResult.data?.name ?? t('reports.unknownAssistant');
}
```

**Trigger scenarios:** any detail / viewer page that fetches a list to render a single label or breadcrumb. Common offenders:

- `listAssistants({ limit: 100 })` for one `assistantId`
- `listVoices({ limit: 100 })` for one `voiceId`
- `listOrgs(...)` for one `orgId` on a member detail page

**Review-time grep:** `rg "list\w+\(\{ query: \{ limit:" apps/web/src/app` — for each match, ask "is this for a list view, or is it a one-record lookup pretending to be a list?"

**Don't fix in this PR if:** the page genuinely renders the list AND uses one entry from it. Then the cap is a separate problem (paged select, `listAll` helper) — flag as follow-up.

### 39. Mutation success ≠ "toast and done" — every error branch needs the three-thing check

Every error path in a mutation must answer all three questions, not just the first:

1. **Does the user see a clear next step?** (toast / banner / inline message)
2. **Does Sentry have a tagged signal?** (`Sentry.captureException(err, { tags: { errorBoundary: '...' } })` — see `sentry-observability` skill)
3. **Can the user actually move forward?** (cache invalidation, polling restart, focus return, retry button)

PR #1068 had the same mutation hit each gap separately:

| Branch | Gap | Fix |
|---|---|---|
| 409 conflict | Toast fires but viewer stays on `failed` Alert + Retry forever (poll's `refetchInterval` is `false` for `failed`, no path back to `processing`) | Add `invalidateQueries({ queryKey: ['weekly-reports'] })` to the conflict branch — same as `onSuccess` — so the existing row refreshes and poll picks up the new status |
| Generic error | `suppressGlobalError + throw` silently swallows (mutation handler suppresses, viewer doesn't surface) | Replace `throw error` with `toast.error(tErrors('generic'))` |
| Poll failure | `retry: false + suppressGlobalError` together silence Sentry too | Add `Sentry.captureException(err, { tags: { hook: 'useReportPoll', errorBoundary: 'useReportPoll-failure' }, extra: { reportId } })` inside the queryFn before throwing |

**Apply when:** writing any mutation `onError`, any polling queryFn, any `.catch(() => fallback)` in async UI code.

**Smell:** if you can answer "what does the user see?" but you can't answer "what does Sentry see?" or "can the user click out of this state?", you have a partial fix.

### 40. Shared hooks need their own tests — caller mocks aren't enough

When a hook is extracted from two callers (e.g. `useGenerateWeeklyReport` factored out of `GenerateReportModal` and `WeeklyReportViewer`), the existing component tests almost always mock the hook entirely (`vi.mock('@/hooks/useWeeklyReports', ...)`) so the hook's own logic — toast copy, `invalidateQueries`, conflict-vs-generic mapping, `onGenerated` callback — has zero coverage.

**Anti-pattern (PR #1068, before follow-up):**
```ts
// useWeeklyReports.test.ts: tests useReportContent + useReportPoll, but NOT useGenerateWeeklyReport
// WeeklyReportViewer.test.tsx: vi.mocks the whole hook → no real coverage of toast/invalidate
// GenerateReportModal.test.tsx: same
```

**Fix — `renderHook` test for the shared hook:**
```ts
import { renderHook, waitFor } from '@testing-library/react';

it('treats a 409 conflict as a soft success: conflict toast + invalidate + onGenerated', async () => {
  mockGenerateAction.mockResolvedValue({ error: { code: 'conflict', statusCode: 409 } } as Awaited<ReturnType<typeof generateAction>>);
  const onGenerated = vi.fn();
  const queryClient = new QueryClient({ defaultOptions: { mutations: { retry: false } } });
  const invalidateSpy = vi.spyOn(queryClient, 'invalidateQueries');
  const wrapper = ({ children }) => createElement(QueryClientProvider, { client: queryClient }, children);

  const { result } = renderHook(() => useGenerateWeeklyReport({ onGenerated }), { wrapper });
  result.current.mutate(buildBody());

  await waitFor(() => expect(result.current.status).toBe('error'));
  expect(mockToastError).toHaveBeenCalledWith('common.reports.viewer.generateConflict');
  expect(onGenerated).toHaveBeenCalledTimes(1);
  expect(invalidateSpy).toHaveBeenCalledWith({ queryKey: ['weekly-reports'] });
});
```

**Coverage targets for any shared mutation hook:**
- happy path: success toast + invalidate + callback
- the conflict / domain-specific error code: dedicated toast + invalidate + callback (the "soft success" branch)
- generic error: generic toast, **no** invalidate, **no** callback, error propagates
- works when the optional callback isn't provided (don't crash)

**Review-time check:** when a PR extracts a shared hook, grep the test file for `renderHook(() => useTheNewHook(`. Zero matches means the hook is tested only through its callers' mocks — flag the gap.

### 41. `router.refresh()` is redundant after `invalidateQueries`

`router.refresh()` re-runs the parent server component (with all its `await` data fetches) and re-renders the route. Calling it alongside `invalidateQueries({ queryKey: [...] })` for the same data is double-work — pick one based on **who owns the data**.

**Decision rule:**
- Data fetched by a Server Component (`await listAssistants(...)` in `page.tsx`) → `router.refresh()`
- Data fetched by a client React Query hook (`useWeeklyReports`) → `invalidateQueries`
- Both? Keep only the React Query path; the server component prop will be re-read on the next genuine navigation.

**Anti-pattern (PR #1068):**
```tsx
const mutation = useActionMutate(generateWeeklyReport, {
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ['weekly-reports'] }); // already refreshes the list
    router.refresh(); // ← also re-fetches listReportConfigs + listAssistants(100). Wasted RTT.
  },
});
```

**Fix — drop `router.refresh()`** unless the mutation also changed something a server component reads (e.g. modifying a config that the server component fetches). When in doubt, remove it; you can always add it back when a missing-refresh bug appears.

### 42. Auth0 claim namespace must match `AUTH0_CLAIMS`

When mocking `useUser()` in tests, the claim key for roles **must** match the runtime namespace defined in `apps/web/src/lib/auth-constants.ts`. Using a stale or incorrect namespace (e.g. `https://aetheron.io/roles`) causes `getRoleFlags()` to silently return all-false.

**Anti-pattern (PR #1078):**
```ts
const SUPERUSER = { 'https://aetheron.io/roles': ['superuser'] };
// getRoleFlags(SUPERUSER) → { isSuperuser: false } — wrong namespace
```

**Fix — use the canonical namespace:**
```ts
const SUPERUSER = { 'https://connect.aetheron.app/claims/roles': ['superuser'] };
// or, import AUTH0_CLAIMS and use AUTH0_CLAIMS.ROLES as the key
```

Before writing a mock user object, check existing test fixtures (`rg 'connect.aetheron.app/claims/roles' apps/web/src --glob '*.test.*'`) to copy the exact key.

### 43. `mockReturnValueOnce` is fragile with React hooks

`vi.fn().mockReturnValueOnce(...)` is consumed by the first call. React may call hooks multiple times per render (StrictMode double-invocation, SWR revalidation). Once the `Once` mock is consumed, subsequent calls fall through to the default.

**Anti-pattern (PR #1078):**
```ts
mockUseUser.mockReturnValueOnce({ user: SUPERUSER });
render(<MyComponent />);
// StrictMode renders twice → second call returns default { user: null }
```

**Fix — set/reset with `mockReturnValue`:**
```ts
mockUseUser.mockReturnValue({ user: SUPERUSER });
render(<MyComponent />);
mockUseUser.mockReturnValue({ user: null });
```

This guarantees every hook call during the render sees the intended value.

## Before-submission checklist

Copy this at the end of a frontend task:

```
Frontend checks:
- [ ] No `.catch(() => {})` / silent catch (AGENTS.md §170)
- [ ] `void asyncFn()` statements kept as-is (house style, per eslint.config.mjs)
- [ ] No nested ternaries (`a ? b : c ? d : e`) — extract helper or use if/else
- [ ] No `<button>` / `<a>` nested inside another `<a>`
- [ ] External `<img>` has loading="lazy" + referrerPolicy="no-referrer"
- [ ] New utility applied everywhere (grep the codebase)
- [ ] New utility has Vitest coverage
- [ ] EntityCard title passed as string where possible
- [ ] No per-card bespoke subtitle spacing (no stray `mt-1`)
- [ ] iconVariant chosen by semantic rule, not drift
- [ ] Visual state driven by ONE field, not a boolean combination
- [ ] Shared components live in components/shared/, not re-exported from feature barrels
- [ ] URL-synced search uses controlled input + `useDeferredValue`; `deferredSearch` passed consistently to all child components
- [ ] Hydration/persistence effects depend on trigger values only; latest callbacks/session state are read via `useEffectEvent` when needed
- [ ] Fetch `catch` blocks discriminate via `err instanceof DOMException && err.name === 'AbortError'` (NOT `signal.aborted`) — race-safe against upstream failures that coincide with client disconnects
- [ ] Route handlers with try/catch have tests covering both the catch branch AND the re-throw path (template: `apps/web/src/app/api/csp-report/route.test.ts`)
- [ ] `??` only used when `''` is a valid value to pass through; otherwise `||` or normalize blanks at the read site
- [ ] Layout / shared component prop defaults are either `required` (TS-enforced) or every mount site passes the real default
- [ ] After fixing a UI symptom, `rg` the data shape to find sibling components rendering it raw — fix all in the same PR
- [ ] User-visible "wiring" fixes have a render test asserting the user-visible string
- [ ] Tests stubbing globals add `afterEach(() => vi.unstubAllGlobals())`
- [ ] Route-handler tests assert upstream request shape (headers, signal, conditional spreads with negative `expect.not.objectContaining`)
- [ ] Paired containers (loading / error / empty / content) share identical className shape
- [ ] `Intl.DateTimeFormat` / `toLocaleTimeString` receive an explicit `locale` from `useLocale()` (client) or `getLocale()` (server) — never `undefined`
- [ ] Mocks alternating between values match on argument identity, not a `++ % 2` counter
- [ ] Forms use DS `FormField` consistently — composite inputs included, no raw `<label>` + manual classNames
- [ ] Clearable selects guard against `''` for fields that require a non-empty value
- [ ] Server Components call `getTranslations()` / `getFormatter()` directly — `t` / `formatDate` are never threaded through props
- [ ] Single source of state per field — no duplication between form values and `useState`
- [ ] "Last updated" displays fall back to `createdAt` when `updatedAt` may be null on fresh rows
- [ ] Textarea `maxLength` mirrors backend cap
- [ ] Case-insensitive validations and `MAX_*` caps have explicit tests
- [ ] No `?? null` on `kysely-codegen` `T | null` fields
- [ ] Date-only strings (`'YYYY-MM-DD'`) rendered with `timeZone: 'UTC'` (or via `next-intl` `format.dateTime`)
- [ ] `useMemo(() => somethingTimeBased(), [])` recomputed in modal/dialog open-reset effect
- [ ] Every `useQuery` / `useMutation` has explicit choices for `retry`, `staleTime`, and `meta.suppressGlobalError` — defaults verified safe under polling × retry × component count
- [ ] If using `meta.suppressGlobalError`, `QueryProvider`'s `QueryCache.onError` (not just `MutationCache.onError`) honours it
- [ ] Status-enum branching has a `default` arm — backend enum is forward-compat
- [ ] Empty-string-as-valid-content uses `typeof data === 'string'`, not `if (data)`
- [ ] One-name-by-id lookups use `getX({ id })`, not `listX({ limit: 100 })`
- [ ] Every mutation error branch (success / domain-error / generic) answers all three: user feedback, Sentry capture, recovery path
- [ ] Shared mutation hook has its own `renderHook` test — happy / domain-error / generic / no-callback branches
- [ ] No `router.refresh()` alongside `invalidateQueries` for the same data
- [ ] Mock user objects use `https://connect.aetheron.app/claims/roles` (not stale namespaces)
- [ ] `mockReturnValueOnce` not used for hooks that React may call multiple times — use `mockReturnValue` + reset
- [ ] `pnpm turbo run typecheck --filter=@aetheronhq/web` PASS
- [ ] `pnpm turbo run lint --filter=@aetheronhq/web` PASS
- [ ] `pnpm turbo run test --filter=@aetheronhq/web` PASS
```

## Provenance

- Rules 1–9 derived from reviewer feedback on PR #867 (`V2-360/entity-card-migration`).
- Rules §1a (void operator is house style) and §1b (no nested ternaries) added after SonarQube flagged `void handleDelete(...)` and a 2-level ternary on `feat/integrations-page-polish` (Apr 2026).
- Rule §10 (URL ↔ local state sync) added after PR #1017 (`fix/integrations-search-lag`), where `useState` + `useDeferredValue` + `useEffect` sync pattern was validated against Next.js action queue internals (Apr 2026).
- Rule §11 (React effect hydration / persistence scheduling) added after PR #1044 (`fix/agent-config-render-loop`), where unstable `useChat` callback identity re-ran chat hydration, repeatedly reconnected SSE streams, and exposed namespace readiness / hydrating edge cases (May 2026).
- Rule §13 (fetch abort detection + SSE proxy route testing pattern) added after PR #1038 (`fix/connect-web-27-session-stream-abort`), where Reza caught a race in which `request.signal.aborted` as the discriminator could silently 499 a genuine upstream failure (ECONNREFUSED / DNS / TLS) that coincided with a client disconnect. Fix switched to error-shape detection matching the existing `useChat.ts` / `query-retry.ts` pattern and added 6-case `route.test.ts` coverage (May 2026).
- Rules §14–§17 added after PR #1072 (`fix/voice-label-default-voice`), where: (§14) `??` failed to catch a Flagsmith-served empty string and let the raw-id branch reappear; (§15) `AppLayout`'s `vapiVoiceId = ''` default silently disabled the fix on `/demo` because that layout never forwards the prop; (§16) `VoiceOverrideIndicator.tsx:40` rendered the raw id directly under the fixed `AssistantDetail` card — same symptom one component over; (§17) the wiring fix had no render test, so dropping the hook/prop/options-bag would have shipped green (May 2026).
- Rules §18–§20 added after PR #1038 review pass — §18 (`vi.unstubAllGlobals()` cleanup convention from `proxy.test.ts` / `useChat.test.ts`); §19 (lock down upstream request shape with positive + negative header assertions, including `expect.not.objectContaining` for conditional spreads like `'Last-Event-ID'`); §20 (heads-up: `AbortSignal.timeout()` rejects with `TimeoutError`, not `AbortError`, so the §13 discriminator must expand if a future change wraps the fetch with a timeout) (May 2026).
- Rule §21 added after PR #990 (`V2-376/agent-config-support-banner-cutoff`), where the loading container kept `flex flex-1 min-h-0` but the error container dropped to `flex-1 min-h-0` — paired state containers must share the same className shape (May 2026).
- Rules §22–§30 added after PR #869 (`V2-329/report-config-form`), where: (§22) `toLocaleTimeString(undefined, ...)` caused SSR/client locale drift; (§23) `mutateCallIndex++ % 2` mocks broke silently when hook order changed; (§24) raw `<label>` mixed with DS `FormField` left typography drift on token bumps; (§25) DS `SelectField`'s clearable behaviour shipped `''` to a non-nullable IANA timezone column; (§26) Server Components were threading `t` / `formatDate` callbacks instead of calling `getTranslations()` / `getFormatter()` directly; (§27) `recipients` was tracked in both `defaultValues` and `useState`; (§28) "Last updated" footer rendered blank for fresh rows where `updatedAt` was null; (§29) textareas didn't enforce backend's 4000 / 16000 char caps client-side; (§30) case-insensitive duplicate dedup and `MAX_RECIPIENTS` cap had no tests (May 2026).
- Rule §31 added after PR #992 (`V2-382/chat-session-title-persistence`), where `row.titleSource ?? null` was a no-op on a `kysely-codegen` `SessionTitleSource | null` column (May 2026).
- Rules §32–§41 added after PR #1068 (`V2-329/pr3-report-history-viewer`), six rounds of review across CodeRabbit / Codex / Judge Codex / human reviewers. The 10 rules cluster into four concern areas the PR repeatedly tripped on: (a) **Date/time** — §32 (`new Date('YYYY-MM-DD')` parses as UTC midnight, render in local timezone shifts the day for users west of UTC) and §33 (`useMemo(() => fn(new Date()), [])` snapshots the value forever); (b) **React Query defaults under load** — §34 (every query/mutation needs explicit retry/staleTime/suppressGlobalError choices once polling × retry × N viewers is in play), §35 (`meta.suppressGlobalError` was honoured by `MutationCache.onError` but dead code on `QueryCache.onError` until the PR extended it), §41 (`router.refresh()` re-runs the parent server component on top of `invalidateQueries`, doubling RTT); (c) **State machines and content discrimination** — §36 (every status switch needs a `default` arm because backend enums grow), §37 (`if (data)` funnels valid empty strings into the error branch — discriminate via `typeof === 'string'`), §39 (every mutation error branch — success, 409, generic, poll failure — must answer user-feedback / Sentry-capture / recovery-path together; the same hook had each gap separately); (d) **Architecture / API shape** — §38 (`listX({ limit: 100 })` to translate one id to a name silently truncates and uses the wrong tool — use `getX({ id })`), §40 (extracting a shared mutation hook leaves the toast / invalidate / conflict-vs-generic logic untested if both callers `vi.mock` the hook entirely; needs a dedicated `renderHook` test) (May 2026).
- Rules §42–§43 added after PR #1078 (`fix/voice-label-friendly-names` + custom-voice-settings superuser gate): (§42) test mock used stale Auth0 claim namespace `https://aetheron.io/roles` instead of the canonical `https://connect.aetheron.app/claims/roles`, causing `getRoleFlags()` to silently return all-false and the CI test to find nothing to render; (§43) `mockReturnValueOnce` was consumed by React StrictMode's double-invocation, so the superuser mock fell through to the default `{ user: null }` on the second call — switched to `mockReturnValue` + explicit reset (May 2026).

When new recurring reviewer comments appear on future PRs, extend this file rather than fixing the symptom once.
