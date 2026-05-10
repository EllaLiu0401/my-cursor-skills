---
name: frontend-error-handling
description: >-
  Global frontend error-handling patterns for Aetheron Connect V2 (apps/web):
  classifying errors before defaulting to unknown, narrowing transient network
  detection (TypeError / DOMException), toast cooldown for simultaneous
  failures, and threading server-resolved feature flags through client
  Providers. Use when:
  (a) editing `QueryProvider.tsx`, any `useActionErrorHandler`-style hook,
  React Query `MutationCache.onError` / `QueryCache.onError` wiring, or
  global toast handlers;
  (b) adding `isTransientNetworkError` / network-error detection logic,
  toast deduplication, or server→client feature-flag context plumbing;
  (c) the user pastes a runtime error stack trace, a "Failed to fetch" /
  "Load failed" / "TypeError" / "Network connection failed" report, or a
  reproduction of unexpected toast spam;
  (d) investigating "why is the user seeing five identical toasts",
  "real bug got swallowed as a generic toast", or "client and server defaults
  diverged" questions.
---

# Frontend Error Handling — Aetheron Connect V2

Lessons codified from review feedback on PR #1069 (`fix/global-error-handler-classification`) and PR #1072 (`fix/voice-label-default-voice`). Apply the relevant rule before declaring a global-handler / network-error / toast change done.

This skill is the home for **handler-shape** rules. Per-call-site Sentry tagging, replay coupling, canary sampling, and `logger.warn` thresholds live in `sentry-observability`.

## Trigger scenarios

Run the relevant check when the change touches any of:

- `apps/web/src/components/QueryProvider.tsx`
- Any `useActionErrorHandler` / `useGlobalErrorHandler` style hook
- React Query `QueryCache` / `MutationCache` `onError` wiring
- Toast helpers shown from a global handler (vs from individual call sites)
- `isTransientNetworkError` / network-error utilities
- Feature-flag Provider plumbing (server → Provider → client `useFeatureFlags`)

## Rule checklist

### 1. Classify before defaulting to `unknown`

Global error handlers must early-return per error class, ordered from most specific to most generic. The **last** branch is the only one that fires `Sentry.captureException` + the generic `internal_error` toast/state.

**Anti-pattern (PR #1069):**
```ts
function useActionErrorHandler() {
  return (error: unknown) => {
    Sentry.captureException(error);
    setError({ code: 'internal_error' });
    toast.error(t('errors.internal'));
  };
}
```

Every transient blip and every typed `ApiErrorWrapper` ends up as `internal_error` in Sentry — the volume signal is useless and on-call can't tell a real bug from a flapping CDN.

**Fix — classify first, then handle:**
```ts
function useActionErrorHandler() {
  return (error: unknown) => {
    if (isTransientNetworkError(error)) {
      Sentry.captureMessage(error.message, 'warning'); // fingerprinted, not full capture
      toast.error(t('errors.networkConnection'), { dedupeKey: 'network' });
      return;
    }
    if (error instanceof ApiErrorWrapper) {
      // mapped error code / field-level validation
      setError({ code: error.code, fieldErrors: error.fieldErrors });
      return;
    }
    // Truly unknown — this is the only branch that should look like a real bug.
    Sentry.captureException(error, { tags: { errorBoundary: 'useActionErrorHandler' } });
    setError({ code: 'internal_error' });
    toast.error(t('errors.internal'));
  };
}
```

**Apply when:** writing or extending any global handler that fans error → user feedback + Sentry. Both `MutationCache.onError` and `QueryCache.onError` need symmetric classification (cross-ref `frontend-code-checks` §35 — `meta.suppressGlobalError` must be honoured by both caches).

**Test it:** for each branch (transient / API / unknown), assert (a) which Sentry method fires, (b) which toast fires, (c) which state mutation runs. A missing branch silently funnels into the wrong arm.

### 2. `isTransientNetworkError` must narrow `TypeError` / `DOMException`

Network-failure detection should match a **specific set** of `TypeError` / `DOMException` patterns, not the full `instanceof TypeError`.

**Anti-pattern (PR #1069):**
```ts
function isTransientNetworkError(error: unknown): boolean {
  return error instanceof TypeError; // matches "cannot read property of null" too
}
```

A real programming bug — `TypeError: Cannot read properties of null (reading 'name')` from a render, or a response-mapping thunk — gets silently downgraded to a "Network connection failed" toast. The user sees an inscrutable retry suggestion; Sentry sees nothing (rule §1 short-circuited the unknown branch).

**Fix — combine the `is-network-error` library with explicit DOMException name check:**
```ts
import { isNetworkError } from 'is-network-error';

function isTransientNetworkError(error: unknown): boolean {
  if (!(error instanceof TypeError || error instanceof DOMException)) {
    return false;
  }
  if (error instanceof DOMException && (error.name === 'AbortError' || error.name === 'TimeoutError')) {
    return true; // see frontend-code-checks §13, §20
  }
  return (
    isNetworkError(error) ||
    error.message?.includes('Failed to fetch') ||
    error.message?.includes('Load failed') // Safari/WebKit, see sentry-observability Rule 4
  );
}
```

**Apply when:** writing any predicate that decides "this error is a transient network blip we can suppress / soft-toast". Reuse the same predicate for the global handler **and** any per-mutation `onError` override — drift between the two reintroduces this bug at one site.

**Test it:** add a fixture for `new TypeError('Cannot read properties of null (reading "x")')` → predicate returns `false`. Without that test, a future "simplify this match" refactor reintroduces the broad `instanceof TypeError`.

### 3. Cooldown to dedupe toast spam from simultaneous failures

When a global handler fires for many concurrent failures (offline drop, CDN blip, Auth0 re-auth), users see the same toast 5–10 times. Add a per-category cooldown.

**Anti-pattern (PR #1069):**
```ts
toast.error(t('errors.networkConnection')); // fires once per failed query
```

**Fix — short cooldown keyed on category:**
```ts
const TOAST_COOLDOWN_MS = 5_000;
const lastShownAt = new Map<string, number>();

function emitToast(category: 'network' | 'auth' | 'internal', message: string) {
  const last = lastShownAt.get(category) ?? 0;
  if (Date.now() - last < TOAST_COOLDOWN_MS) return;
  lastShownAt.set(category, Date.now());
  toast.error(message);
}
```

**Decision rule:**
- Use a **module-level** map (not React state) so the cooldown spans handler invocations and component remounts.
- Cooldown duration: 5 s for noisy categories (network), 0 (no cooldown) for genuinely unique errors (validation field errors carry context; deduping them hides specifics).
- The category key should match the rule §1 classification, not the error message — message-keyed dedup misses Safari's "Load failed" vs Chrome's "Failed to fetch".

**Test it:** drive the handler with two fast-following network errors → `toast.error` is called once. Then advance fake timers past cooldown → next error toasts again.

### 4. Server-resolved feature flags must reach the client via Provider, not re-resolved

For flags read at server boundary (Server Component, layout `await resolveFeatureFlag(...)`), thread the resolved value through a Provider. Do **not** call the resolver again in client components with a hardcoded fallback — server and client will diverge.

**Anti-pattern (PR #1072):**
```tsx
// app/(dashboard)/layout.tsx — server resolves
const vapiVoiceId = await resolveFeatureFlag('vapiVoiceId');
return <AppLayout vapiVoiceId={vapiVoiceId}>{children}</AppLayout>;

// In a client component, somebody re-resolves with hardcoded default:
const vapiVoiceId = useFeatureFlag('vapiVoiceId') ?? '4Hm8...'; // diverges if Flagsmith returns ''
```

The bugs that follow: (a) SSR renders "Aetheron Australian" but client hydration renders the raw id `4Hm8…`; (b) `??` doesn't catch `''` from Flagsmith's `str()` (cross-ref `frontend-code-checks` §14); (c) every new mount site that forgets the prop falls back silently (cross-ref §15).

**Fix — Provider as single source of truth:**
```tsx
// Server component / layout
const vapiVoiceId = await resolveFeatureFlag('vapiVoiceId');
return (
  <FeatureFlagProvider value={{ vapiVoiceId }}>
    <AppLayout>{children}</AppLayout>
  </FeatureFlagProvider>
);

// Client consumer
const { vapiVoiceId } = useFeatureFlags(); // matches server, no fallback drift
const label = getVoiceLabel(id, { defaultVoiceId: vapiVoiceId });
```

**Apply when:** any feature flag drives UI rendering (theme, default voice, locale, copy variant, feature toggle). Especially anything that affects label resolution or default selection — those hide the divergence longest.

**Audit checklist** when adding a new flag-fed prop:
- [ ] `rg "<FeatureFlagProvider"` — every mount path has the provider
- [ ] `rg "resolveFeatureFlag\\('flagName'"` — only resolved at server / route-segment boundary, never inside client components
- [ ] If a layout has a permissive default (`vapiVoiceId = ''`), prefer making the prop required so TS surfaces every forgotten mount (cross-ref §15)

## Pre-submission checklist

```
Frontend error-handling checks:
- [ ] Global handler classifies error → transient → API → unknown (each branch handles capture/toast/state)
- [ ] `isTransientNetworkError` narrows TypeError / DOMException (no raw `instanceof TypeError`)
- [ ] Toast spam deduped by 5s cooldown keyed on classification category
- [ ] `MutationCache.onError` and `QueryCache.onError` both honour the same classification ladder + suppressGlobalError flag
- [ ] Server-resolved feature flags reach client via Provider, never re-resolved client-side
- [ ] Tests cover each branch of the handler + a "real bug TypeError" fixture that must NOT match the network-error predicate
```

## References

- `apps/web/src/components/QueryProvider.tsx` — global handler wiring
- `apps/web/src/lib/errors/is-network-error.ts` (or wherever `isTransientNetworkError` lives)
- `apps/web/src/contexts/FeatureFlagProvider.tsx` — server → client flag plumbing
- Related skills:
  - `frontend-code-checks` §13 (AbortError discrimination), §14 (`??` vs `||`), §15 (default-prop cascade), §20 (TimeoutError), §35 (suppressGlobalError both caches)
  - `sentry-observability` Rule 1 (`beforeSend` over `ignoreErrors`), Rule 3 (tag-aware bypass), Rule 4 (cross-browser fetch failure regex)

## Provenance

- Rules 1–3 added after PR #1069 (`fix/global-error-handler-classification`), where the original `useActionErrorHandler` collapsed transient network blips and real `TypeError` bugs into the same `internal_error` Sentry event + toast. Reviewer-driven fixes split the ladder, narrowed the network predicate to the `is-network-error` lib + explicit DOMException check, and added a 5 s per-category toast cooldown to stop the offline-scenario toast spam (May 2026).
- Rule 4 added after PR #1072 (`fix/voice-label-default-voice`) — the SSR/client divergence angle that didn't fit `frontend-code-checks` §14 (`??` vs `||`) or §15 (default-prop cascade). Server resolved `vapiVoiceId` correctly but client-side re-resolution with a hardcoded fallback caused the original `Aetheron Australian` label to render only on the server (May 2026).

When new global-error-handler / network-detection / toast-coordination lessons appear on future PRs, extend this file rather than fixing the symptom once.
