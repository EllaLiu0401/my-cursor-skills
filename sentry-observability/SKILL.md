---
name: sentry-observability
description: >-
  Sentry SDK + replay + structured-logger patterns for Aetheron Connect V2.
  Covers `beforeSend` vs `ignoreErrors`, canary sampling, tag-aware bypass,
  engine-agnostic regex naming, replay-coupling semantics, the
  `logger.warn` → Sentry capture threshold, and structured error-payload
  conventions. Use when:
  (a) editing `sentry.client.config.ts` / `sentry.server.config.ts` /
  `apps/api/src/sentry.ts`;
  (b) adding or filtering `Sentry.captureException` / `captureMessage` sites,
  configuring replays or sampling rates, or routing structured logger errors
  into Sentry;
  (c) the user pastes a Sentry issue URL — e.g.
  `https://aetheron.sentry.io/issues/<id>/?project=...&query=...`,
  any `*.sentry.io/issues/` link, or a `sentry.io` issue/event/replay link —
  and asks to triage, debug, classify, decide whether to filter, or
  root-cause the error;
  (d) investigating "why didn't this error reach Sentry", "Sentry noise
  spike", "should I suppress this error", or "is this a real bug or expected
  noise" questions;
  (e) reviewing PRs that touch Sentry config, `beforeSend`, replay sampling,
  or any `Sentry.captureException` site.
---

# Sentry & Observability — Aetheron Connect V2

Lessons codified from PR #1039 (`fix/sentry-ignore-fetch-network-failure`) and PR #997 (`V2-380/fix-intermediate-messages`). Apply the relevant rule before declaring a Sentry / observability change done.

## Trigger scenarios

Read this skill when the change touches any of:

- `apps/web/sentry.client.config.ts` / `apps/web/sentry.server.config.ts` / `apps/api/src/sentry.ts`
- `Sentry.captureException(...)` call sites
- Sentry filter logic (`ignoreErrors`, `beforeSend`, `beforeSendTransaction`)
- Replay configuration (`replaysSessionSampleRate`, `replaysOnErrorSampleRate`)
- `logger.warn` / `logger.error` calls in code paths that may need Sentry visibility
- New `metrics.ts` counters next to error-prone code paths

## Rule checklist

### 1. `beforeSend` over `ignoreErrors` for noise filters

`ignoreErrors` runs at the SDK level, **before** replay attachment, and matches on event message only. That has three consequences:

1. **Replay buffer drops with the event** — even with `replaysOnErrorSampleRate: 1`, dropped events lose their on-error replay too. We lose the richest debugging artefact for the noisy class.
2. **Tagged explicit captures get dropped** — `Sentry.captureException(error, { tags: { errorBoundary: 'global' } })` with the same message is filtered identically. Diagnostic canaries vanish along with the noise.
3. **Total blackout, not noise filter** — `ignoreErrors: ['Failed to fetch']` becomes a 100% blackout for that message. Real CORS regressions, S3 bucket-policy slips, and CDN/Auth0 outages all surface as `Failed to fetch` and disappear.

**Use `beforeSend` instead** — it runs after event construction, before replay attachment, and has full access to `event.tags`, `event.extra`, etc. Surviving 1% as a canary preserves both the volume signal and the replay for sampled events.

```ts
const TRANSIENT_FETCH_FAILURE = /^(Failed to fetch|Load failed)$/;

Sentry.init({
  beforeSend(event, hint) {
    const error = hint.originalException;
    const message = typeof error === 'string' ? error : (error as Error)?.message;

    if (message && TRANSIENT_FETCH_FAILURE.test(message)) {
      // Tag-aware bypass — explicit captures with diagnostic context always pass through.
      // NOTE: returning null discards the on-error replay too (Sentry SDK couples them),
      // so for the dropped 99% we lose both the event and the replay even with
      // replaysOnErrorSampleRate: 1.
      const tags = event.tags ?? {};
      if ('errorBoundary' in tags || 'api_request_id' in tags) {
        return event;
      }

      // 1% canary — deterministic on event_id so we don't silently lose the volume signal.
      const hash = parseInt((event.event_id ?? '').slice(0, 8), 16);
      if (Number.isFinite(hash) && hash / (2 ** 32) < 0.01) {
        return event;
      }

      return null;
    }

    return event;
  },
  // ...
});
```

When to keep `ignoreErrors`: pure SDK-internal noise that has no replay value and no diagnostic tag (e.g. `ResizeObserver loop limit exceeded` from third-party libraries). Default to `beforeSend`.

### 2. Canary sampling math

Use a **deterministic** hash of `event.event_id`, not `Math.random()`:

- Deterministic sampling avoids correlated drops during a volume spike (random sampling can drop ALL events of a spike if the RNG is unlucky)
- `event.event_id` is unique per event and uniformly distributed
- `2 ** 32` reads better than `0x100000000` — both produce 4 294 967 296. The math is right (using `2 ** 32` not `0xFFFFFFFF` correctly avoids the max sample mapping to exactly 1.0)

```ts
const SAMPLE_RATE = 0.01;
const hash = parseInt((event.event_id ?? '').slice(0, 8), 16);
const sampled = Number.isFinite(hash) && hash / (2 ** 32) < SAMPLE_RATE;
```

`Number.isFinite(hash)` guards against `event_id` being `undefined` or non-hex (defensive — the SDK always sets it, but the type is `string | undefined`).

### 3. Tag-aware bypass for diagnostic captures

Every `Sentry.captureException(...)` site that exists for a **specific reason** (retry exhausted, error boundary fired, request id known) should set a tag the bypass logic can match:

| Capture site | Tag |
|--------------|-----|
| Global React error boundary (`app/global-error.tsx`) | `errorBoundary: 'global'` |
| React Query retry exhausted (`QueryProvider.tsx`) | `errorBoundary: 'react-query-unknown'` |
| `useChat.loadSession` failure | `errorBoundary: 'useChat-loadSession'` |
| API client wrapping a known request | `api_request_id: <uuid>` |

In the `beforeSend` filter, use **key-presence checks**, not truthy checks:

```ts
// GOOD — key present in tags wins (even if value is '' or false)
if ('errorBoundary' in (event.tags ?? {})) return event;

// BAD — truthy check drops the event for falsy tag values
if (event.tags?.errorBoundary) return event;
```

When **changing a `beforeSend` filter**, audit ALL `Sentry.captureException` sites (`rg "Sentry\\.captureException" apps/web/src`) for the bypass tag. Common misses:

- `apps/web/src/components/QueryProvider.tsx:95` — react-query retry-exhausted fallback
- `apps/web/src/hooks/useChat.ts:907` — `loadSession` failure (use `Sentry.withScope` to tag)

```ts
Sentry.withScope((scope) => {
  scope.setTag('errorBoundary', 'useChat-loadSession');
  scope.setExtras({ sessionId, agentId });
  Sentry.captureException(err);
});
```

### 4. Engine-agnostic constant naming

When a regex matches **multiple browsers**, name the constant by **what** it captures, not by **which engine**:

| Browser / engine | `fetch` failure message |
|------------------|-------------------------|
| Chromium (Chrome, Edge) | `Failed to fetch` |
| Firefox | `Failed to fetch` |
| Safari (WebKit) | `Load failed` |

```ts
// GOOD
const TRANSIENT_FETCH_FAILURE = /^(Failed to fetch|Load failed)$/;

// BAD — name implies engine specificity that doesn't exist
const CHROMIUM_FETCH_NETWORK_FAILURE = /^Failed to fetch$/;
```

Engine-qualified names (`FIREFOX_STREAM_ABORT`) are appropriate when the regex is genuinely engine-specific. Otherwise a future cross-browser variant forces a rename.

### 5. Document replay coupling in code comments

`beforeSend` returning `null` discards both the event AND the on-error replay buffer (Sentry SDK couples them). This is the opposite of `ignoreErrors` users expect when they migrate, and `replaysOnErrorSampleRate: 1` does not save it.

Always include a comment in the `beforeSend` body so the next reader doesn't assume replay survives:

```ts
// NOTE: returning null discards the on-error replay too (Sentry SDK couples them).
// Even with replaysOnErrorSampleRate: 1, dropped events lose their replay buffer.
// The 1% canary sample preserves both event AND replay for sampled events.
```

### 6. `logger.warn` does NOT reach Sentry

`packages/sentry/src/log-hook.ts:27` only captures pino events at level `>= 50` (`error`, `fatal`). `warn` is level 40 and is **invisible to Sentry**. Validation: `apps/workloads/src/lib/logger.test.ts:76`.

This matters when a `.catch(...)` swallows user-visible content (e.g. an assistant text block failing to persist):

```ts
// BAD — failure rate uncomputable from Sentry; user sees text disappear from history
.catch((err) => log.warn({ err }, 'Failed to persist assistant text'));
```

**Fix — pick one based on severity:**

(a) Bump to `error` so the log hook captures it:

```ts
.catch((err) =>
  log.error({ err, blockType: 'text', sessionId, agentId }, 'Failed to persist assistant text'),
);
```

(b) Or call `Sentry.captureException` explicitly and keep `warn` for log volume:

```ts
.catch((err) => {
  Sentry.captureException(err, {
    tags: { errorBoundary: 'agent-runner-persistence', blockType: 'text' },
    extra: { sessionId, agentId },
  });
  log.warn({ err, sessionId }, 'Failed to persist assistant text');
});
```

Use (a) when the error rate is the SLO signal (most cases). Use (b) when log volume would be punishing but Sentry needs each individual event with rich context.

**Pre-existing patterns in the codebase**: agent-runner uses `(a)` for content-block persistence — `apps/workloads/src/agent-runner.ts` `.catch((err) => log.error({ err, blockType }, '...'))`.

### 7. Structured error payload for triage

Logger error calls in catch blocks should include enough structured context to bisect the failure mode in CloudWatch / Datadog without re-deploying:

```ts
// BAD — four call shapes (text, tool_use, thinking, tool_result) all log {err}
// Sentry collapses them into one issue; can't tell which one is failing
.catch((err) => log.error({ err }, 'Failed to persist content block'));

// GOOD — block_type splits the issue, sessionId / agentId enable replay
.catch((err) =>
  log.error(
    { err, blockType: 'text', sessionId: job.sessionId, agentId: job.agentId },
    'Failed to persist content block',
  ),
);
```

Required fields for any error payload:
- `err` (always)
- The discriminator that splits call shapes — `blockType`, `eventType`, `provider`, etc.
- The tenancy / entity ids the on-call needs to find the related row (`orgId`, `sessionId`, `userId`, `interactionId`)

### 8. Metrics counter alongside structured log error

Logs are searchable; metrics are aggregable and alarmable. For SLO-tracked failure modes (revenue-impacting, user-visible, compliance), pair the `log.error` with a metrics counter split by the same dimensions:

```ts
// apps/workloads/src/metrics.ts
export const persistContentBlockFailures = new Counter({
  name: 'persist_content_block_failures_total',
  help: '...',
  labelNames: ['block_type', 'result'] as const,
});

// at the call site
.catch((err) => {
  persistContentBlockFailures.inc({ block_type: 'text', result: 'error' });
  log.error({ err, blockType: 'text', sessionId }, 'Failed to persist content block');
});
```

Set a low-rate alarm in CloudWatch / Datadog once the feature has rolled out (e.g. `> 1% of attempts over 15 minutes`). Without the counter, the failure rate is uncomputable and the on-call has no alarm — only Sentry issue counts, which lag and don't normalize for traffic.

### 9. `Sentry.withScope` over inline `setTags`

When a single capture needs request-specific tags / extras, use `withScope` so the modifications don't leak to subsequent captures on the same SDK instance:

```ts
// GOOD — scope is local to this capture
Sentry.withScope((scope) => {
  scope.setTag('errorBoundary', 'useChat-loadSession');
  scope.setExtras({ sessionId, agentId });
  Sentry.captureException(err);
});

// BAD — Sentry.setTag mutates the global scope; next unrelated capture also gets this tag
Sentry.setTag('errorBoundary', 'useChat-loadSession');
Sentry.captureException(err);
```

### 10. Don't add `console.warn` to expected-error catches "for visibility"

When a reviewer requests a `console.warn` on a branch that legitimately swallows expected noise (abort, 404 → null, parse-or-fallback), check AGENTS.md before agreeing:

> AGENTS.md: "Do NOT log-and-rethrow — the boundary already logs."

The whole point of swallowing is reducing noise. Adding a `console.warn`:
- Pollutes CloudWatch with high-volume noise
- Doesn't aggregate (no metrics)
- Doesn't reach Sentry
- Subverts the AGENTS.md error-handling rule

Legitimate alternatives if visibility is genuinely needed:
- LB access logs (already record 499 / status codes)
- Dedicated metrics counter
- Structured-event trace attributes (OTel)

Push back politely with a reference to AGENTS.md when this comes up.

### 11. Anchor framework-internal class-name filters with the verified version

When `beforeSend` filters on a class name from a framework's internal types — Next.js `ResponseAborted`, `BailoutToCSRError`, `DynamicServerError`; Fastify `FST_ERR_VALIDATION`; etc. — frameworks have **no semver stability** for these names. A major bump can rename or remove them, and the filter silently no-ops without any test failure.

Always anchor the version you verified against in a single-line comment:

```ts
beforeSend(event, hint) {
  const error = hint.originalException;
  // Verified against next@15.4.x — this class is in dist/server/lib/router-utils/response-aborted.js.
  // On Next major bump (>=16), grep dist/ for the class and re-confirm before assuming this still matches.
  if (error instanceof Error && error.name === 'ResponseAborted') {
    return null;
  }
  return event;
}
```

**Apply when:** any `beforeSend` / `ignoreErrors` filter that matches against a framework or library internal class name (not a public exported type). If the type is publicly exported and semver-stable, no anchor needed.

**Test it:** a unit test that asserts the filter still matches the class name from the currently installed version (`expect(new ResponseAborted().name).toBe('ResponseAborted')`). On version bumps the test fails fast instead of silently letting the filter break.

### 12. Don't 100% blackout an "expected lifecycle" error — keep a 1% canary

For errors that are 99% noise but not 100% (`ResponseAborted` from client disconnects, `Failed to fetch` from offline blips, `AbortError` from navigation), do not return `null` from every match in `beforeSend`. Sudden volume spikes on these "harmless" errors **are real signals** of infrastructure problems:

- `ResponseAborted` spike → keep-alive misconfig, ALB idle-timeout < client read-timeout, CDN edge dropping connections
- `Failed to fetch` spike → CORS regression, S3 bucket-policy slip, Auth0 outage, CDN DNS issue
- `AbortError` spike → client SDK regression that aborts on every navigation

A 100% blackout costs nothing on the steady-state day but blinds on-call exactly when the volume signal would have helped them.

**Pattern — reuse the deterministic 1% canary from Rule 1:**
```ts
beforeSend(event, hint) {
  const error = hint.originalException;
  if (error instanceof Error && error.name === 'ResponseAborted') {
    // See Rule 1 for the deterministic event_id-hash sampling math.
    if (sampledForCanary(event, 0.01)) return event;
    return null;
  }
  return event;
}
```

**Decision rule — when 100% blackout IS acceptable:**
- The error has zero diagnostic value even at high volume (e.g. `ResizeObserver loop limit exceeded` from a third-party widget — you can't act on it regardless of rate).
- The error is generated by the SDK itself (e.g. Sentry rate-limit messages).
- Keeping any volume signal would breach a contractual data-residency rule.

For everything else — including everything that looks like "user-side noise we can ignore" — keep the 1% canary. The cost is negligible; the upside is detecting infra anomalies on day zero.

### 13. Sanitization runs **before** any conditional early return in `beforeSend`

The header / body scrubber that strips Authorization tokens, cookies, PII, and request bodies must execute on **every** event before any branch decides whether to drop, sample, or pass through. Otherwise the 1% canary from Rule 12 leaks credentials.

**Anti-pattern:**
```ts
beforeSend(event, hint) {
  if (sampledForCanary(event, 0.01)) return event;     // ← canary leaks unscrubbed event
  if (isExpectedLifecycleError(hint)) return null;
  scrubHeaders(event);                                  // ← only the surviving 99% gets scrubbed
  return event;
}
```

**Fix — scrub first, branch after:**
```ts
beforeSend(event, hint) {
  scrubHeaders(event);
  scrubRequestBody(event);

  if (isExpectedLifecycleError(hint)) {
    if (sampledForCanary(event, 0.01)) return event; // safe — already scrubbed
    return null;
  }
  return event;
}
```

**Apply when:** any `beforeSend` that combines (a) request/response data attachment with (b) sampling or filtering branches. Always scrub at the top.

**Test it:** unit-test `beforeSend` with a fixture event that has `Authorization: Bearer secret` in `event.request.headers`. Drive both the canary-pass and the canary-drop paths; assert the header is scrubbed in **both** outputs (and on the dropped path, assert nothing leaks via `Sentry.getCurrentScope` or breadcrumbs).

### 14. Defense-in-depth must be runtime-symmetric — or YAGNI

If the same error class is filtered at two layers (e.g. `instrumentation.ts` Node-side AND `beforeSend` browser-side; or Sentry server config AND a Fastify error hook), both layers must agree on which classes are dropped. Drift means an error that escapes one layer surfaces at the other unexpectedly.

But the corollary: **don't add the second filter pre-emptively** if no events actually flow through that runtime. Edge-runtime filter dead code "just in case we add an edge route later" is YAGNI noise — verify the runtime is live before duplicating.

**Apply when:** changing filtering rules in any of the Sentry config files. Audit checklist:
- [ ] Same class name filtered consistently in `sentry.client.config.ts` AND `sentry.server.config.ts` if both runtimes can produce it
- [ ] If only one runtime produces the class, document why the other is exempt
- [ ] Don't add an edge-runtime filter unless `apps/web/src/app/**/route.ts` actually has `runtime: 'edge'` exports

### 15. Sentry filter tests must include a pass-through (negative) case

A filter test that only asserts "matched message → null" passes if the matcher silently breaks (regex typo, framework rename, SDK upgrade changing event shape). Always pair the positive test with at least one negative case where the event must survive.

**Pattern (PR #1110):**
```ts
describe('beforeSend', () => {
  it('drops ResponseAborted (with 1% canary)', () => {
    const event = makeEvent({ exception: { name: 'ResponseAborted' } });
    // Force canary-drop branch by stubbing the sampler
    expect(beforeSend(event, { originalException: new ResponseAborted() })).toBeNull();
  });

  // Negative — the matcher must NOT swallow unrelated errors:
  it('passes through a real TypeError', () => {
    const event = makeEvent({ exception: { name: 'TypeError' } });
    const result = beforeSend(event, {
      originalException: new TypeError('Cannot read properties of null'),
    });
    expect(result).not.toBeNull();
    expect(result?.exception?.values?.[0]?.type).toBe('TypeError');
  });

  it('passes through ApiErrorWrapper', () => {
    const event = makeEvent({ exception: { name: 'ApiErrorWrapper' } });
    const result = beforeSend(event, { originalException: new ApiErrorWrapper(...) });
    expect(result).not.toBeNull();
  });
});
```

**Apply when:** writing or extending any `beforeSend` filter. The pass-through cases are the cheap insurance against framework renames and SDK shape changes — they fail fast on bumps instead of silently 100%-blackouting your production error stream.

## Pre-submission Sentry checklist

```
Sentry / observability checks:
- [ ] Use `beforeSend` not `ignoreErrors` for noise filters (preserves replay + tagged captures)
- [ ] Canary 1% sampling uses deterministic hash of event_id, with `2 ** 32` (not `0x100000000`)
- [ ] All `Sentry.captureException` sites with diagnostic context have `errorBoundary` or `api_request_id` tag
- [ ] `beforeSend` bypass uses key-presence checks (`'errorBoundary' in event.tags`), not truthy
- [ ] Cross-browser regex named engine-agnostically (`TRANSIENT_FETCH_FAILURE` not `CHROMIUM_*`)
- [ ] `beforeSend` body has comment documenting replay-coupling (returning `null` drops replay too)
- [ ] User-visible content failures use `logger.error` (level >= 50) or explicit `Sentry.captureException` — never `logger.warn` alone
- [ ] Logger error payloads include the call-shape discriminator (`blockType`, `eventType`) plus tenancy ids
- [ ] SLO-tracked failure modes have a `metrics.ts` counter alongside the log error
- [ ] Inline `setTag` replaced with `Sentry.withScope` so request-local tags don't leak
- [ ] No `console.warn` added to expected-error catches "for visibility" (push back per AGENTS.md)
- [ ] Framework-internal class-name filters are anchored with the verified version in a code comment
- [ ] No 100% blackout on "expected lifecycle" errors — keep the 1% canary unless the error truly has zero diagnostic value at any volume
- [ ] Header / body scrubbing runs at the TOP of `beforeSend`, before any sampling or filter branch
- [ ] Filter changes are runtime-symmetric (client + server) where both runtimes can produce the class — or the asymmetry is documented
- [ ] Every `beforeSend` filter has at least one positive AND one pass-through (negative) test fixture so framework renames / SDK upgrades fail loudly
- [ ] PR description documents the noise-vs-canary tradeoff and the on-call impact (e.g. "expect ~99% drop in 'Failed to fetch' Sentry volume; canary preserves spike detection at 1%")
```

## References

- `apps/web/sentry.client.config.ts` — current `beforeSend` implementation (PR #1039)
- `packages/sentry/src/log-hook.ts:27` — pino → Sentry level threshold (>= 50)
- `apps/workloads/src/lib/logger.test.ts:76` — validation of the threshold
- `apps/workloads/src/agent-runner.ts` — `log.error({ err, blockType }, ...)` pattern
- `apps/web/src/hooks/useChat.ts:907` — `Sentry.captureException` site that needs `errorBoundary` tag
- `apps/web/src/components/QueryProvider.tsx:95` — same

## Provenance

- Rules 1–5 (beforeSend tradeoff, canary sampling math, tag-aware bypass with key-presence checks, engine-agnostic naming, replay coupling documentation) added after PR #1039 (`fix/sentry-ignore-fetch-network-failure`), where the original `ignoreErrors` patch dropped both the on-error replay buffer and explicit diagnostic captures with the same message. Switched to `beforeSend` with deterministic 1% canary, key-presence tag bypass, and replay-coupling comment. Subsequent reviewer comments fixed `0x100000000` → `2 ** 32` and audited `useChat.ts:907` / `QueryProvider.tsx:95` for missing `errorBoundary` tags (May 2026).
- Rules 6–7 (`logger.warn` doesn't reach Sentry, structured error payload for triage) added after PR #997 (`V2-380/fix-intermediate-messages`), where the existing `.catch(() => log.warn({ err }, ...))` pattern for content-block persistence was hiding user-visible text drops from Sentry. The `sentryLogHook` level >= 50 threshold was confirmed against `logger.test.ts:76`. Reviewer also pushed for `blockType` in the payload so the four call shapes (text, tool_use, thinking, tool_result) wouldn't collapse into one Sentry issue (Apr–May 2026).
- Rule 8 (metrics counter alongside log error) added after PR #997 review feedback that the failure rate was uncomputable from Sentry alone without a counter (Apr 2026).
- Rule 9 (`Sentry.withScope` over inline `setTag`) — long-standing Sentry SDK best practice; codified here while documenting the tag-aware bypass in Rule 3.
- Rule 10 (push back on `console.warn` in expected-error catches) added after PR #1038 (`fix/connect-web-27-session-stream-abort`), where the reviewer initially suggested `console.warn` on the 499 branch "for visibility" and the agreed resolution per AGENTS.md was to leave it silent and rely on LB access logs (May 2026).
- Rules 11–15 added after PR #1110 (`fix/sentry-response-aborted-filter`), where the original patch (a) matched `error.name === 'ResponseAborted'` against an unanchored Next.js internal class, (b) returned `null` 100% of the time so the inevitable infra-anomaly volume signal would have been invisible, (c) ran sampling before scrubbing so the 1% canary leaked Authorization headers, (d) added an edge-runtime filter despite the project having no edge routes (dead code), and (e) only tested the positive match path — a regex typo would have silently let everything through. The five rules are the directly-corresponding fixes plus the missing pass-through (negative) test case (May 2026). Description was simultaneously rewritten to add multi-arm trigger logic so pasted Sentry issue URLs (`https://aetheron.sentry.io/issues/<id>/?...`) and abstract "should I suppress this error" questions reliably load this skill in both Claude Code and Cursor.

When new Sentry / observability lessons appear on future PRs, extend this file rather than fixing the symptom once.
