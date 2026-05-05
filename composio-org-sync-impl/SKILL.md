---
name: composio-org-sync-impl
description: >-
  Implement the org-level Composio connection unification & sync plan
  (planning/engine/org-level-composio-sync.md, PR #891). Covers the entity
  switch (userId → orgId), the sync-write afterExecute hook, lazy reconcile
  as drift detector, webhook dispatch for expired events, Terraform
  subscription update, and the tenancy/rollout gotchas the review caught.
  Use when implementing any §4.1–4.6 section of that plan, touching
  composio-service.ts / tool-resolver.ts / webhooks/composio.ts /
  managed-app-repository.ts, or landing the org-level Composio entity flip.
---

# Composio Org-Level Sync — Implementation Skill

Source of truth: `planning/engine/org-level-composio-sync.md` (merged as PR #891).
This skill distils the review-cycle gotchas so implementation PRs don't re-open settled decisions.

**If you're unsure about ANY design choice, read the plan first — it has references back to the specific review comment that drove the decision.**

---

## Mental Model (do not deviate)

Three-layer sync strategy, ordered by primacy:

1. **Sync-write `afterExecute` hook** (§4.1) — **primary** persistence for chat-created connections. Closes the authz window.
2. **Lazy reconcile in `listIntegrations`** (§4.2) — **drift detector / safety net**. Not the primary path. It exists to repair missed sync-writes + external deletions + status drift.
3. **Webhook `connected_account.expired`** (§4.3) — real-time expiry status sync. Calls `reconcileForOrg(orgId)`, does **not** do a mini-reconcile inline.

Both creation paths (`/integrations` and chat) write `managed_apps` synchronously at connection time. `managed_apps` is the **tenancy guard** — treat its correctness as an authz requirement, not a UX nicety.

---

## Composio Hard Constraints (verified, do not question)

- **One webhook subscription per project.** All event types arrive at a single `/webhooks/composio` URL; dispatch on `body.type`.
- **Only two event types exist**: `composio.trigger.message` and `composio.connected_account.expired`. **No `created` / `active` / `deleted` events.** That's why sync-write is required — webhooks cannot fill the gap for new connections.
- `expired` payload's `metadata.org_id` is Composio-internal, not our `orgId`. Must call `connectedAccounts.get(data.id)` and read `.user_id` (= our `orgId`).
- API keys do **not** get the `expired` webhook (OAuth only). API-key status drift is handled by lazy reconcile only.
- API version: `/api/v3.1/webhook_subscriptions`. Pass `version: "V3"` explicitly.
- **Keep `verifyComposio` (our handwritten HMAC).** Do NOT switch to the SDK's `triggers.verifyWebhook()` in the implementation PR — signing-string compat is unverified (§7 Q4 follow-up).

---

## §4.1 — Entity Unification & Sync-Write Hook

### Code touch points

| File | Change |
|---|---|
| `apps/api/src/routes/sessions.ts` L151–176 | **No RBAC guard change** — route is already `['org_owner', 'org_admin', 'org_analyst']`. The visibility flip is in the Composio query filter. |
| `apps/api/src/services/composio-service.ts` L329–341 | Rename `listUserConnections(userId)` → `listOrgConnections(orgId)`; change filter from `userIds: [userId]` to `userIds: [orgId]`. |
| `apps/workloads/src/agent-runner.ts` L843–860 | `orgId` is **already** passed positionally. Only `deps` changes: drop `userId: run.createdBy ?? undefined`, add `interactiveComposioSession: run.createdBy != null`. |
| `apps/workloads/src/tool-resolver.ts` L443–460, L585–600 | `ResolveToolsDeps`: remove `userId?`, add `interactiveComposioSession?: boolean`. `resolveComposioSession(userId, ...)` → `resolveComposioSession(composioEntityId, ...)`. Guard `if (deps.userId)` → `if (deps.interactiveComposioSession)`. |

### Gotcha #1: `deps.userId` is double-duty — do NOT collapse naïvely

`tool-resolver.ts` uses `deps.userId` as BOTH (a) the Composio entity id AND (b) an implicit "is this an interactive run?" flag.

Entity switch makes (a) obsolete. But (b) must be preserved because `workload_runs.created_by` is **nullable by design** for four system-triggered entry points:

| Entry point | createdBy | Interactive? |
|---|---|---|
| `POST /workload-runs` (UI chat) | `req.auth?.userId` | yes |
| `POST /webhooks/composio` (trigger.message) | NULL | **no** |
| `POST /vapi/agents/:id/server-url` | NULL | **no** |
| `weekly-report-service` (scheduled) | NULL | **no** |
| `post-call-workload-service` (post-call hook) | NULL | **no** |

A naïve `resolveComposioSession(orgId, ...)` unconditionally would silently create Composio meta-sessions for all four. Solution: split into `interactiveComposioSession` flag, set at `agent-runner.ts` call site via `run.createdBy != null`.

Evidence trail:
- `apps/migrations-lambda/migrations/20260306000000_add_created_by_to_workload_runs.ts` — column is nullable.
- `apps/api/src/resources/workload-run.resource.ts` L104–105 — only API path that sets `createdBy`.
- `apps/api/src/repositories/workload-run-repository.ts` L38 — omits column when null.

### Gotcha #2: RBAC rationale must be explicit in PR description

The route guard does not change; the Composio **filter** changes. State this explicitly or reviewers will flag it as a tenancy loosening. Rationale to include:

> Previously `/chat/connections` filtered by `userIds: [userId]`. Now that entity = orgId, connections are org-owned data by definition — any org member with `/chat` access should see them. Route-level RBAC is unchanged.

### Sync-write afterExecute implementation

Inside `tool-resolver.ts`'s `buildBeforeExecuteModifier` (already wraps Composio tool calls), extend with an `afterExecute` branch:

- `tool.slug === 'COMPOSIO_MANAGE_CONNECTIONS'` → no-op (account not yet active)
- `tool.slug === 'COMPOSIO_WAIT_FOR_CONNECTIONS'` succeeds → read `connected_account_id` from response → `composio.connectedAccounts.get(ca.id)` → `managedAppRepo.upsertByExternalId(db, { orgId, provider: 'composio', externalId: ca.authConfig.id, ... })`
- Failure **logs, does not throw** to the agent. Lazy reconcile is the safety net.

Emit `composio.sync_write` structured log with `{ orgId, authConfigId, connectedAccountId, outcome: 'inserted' | 'resurrected' | 'updated' | 'noop' | 'failed' }`.

---

## §4.2 — Lazy Reconcile (Drift Detector)

### Code touch points

- `apps/api/src/services/composio-provisioning-service.ts` `listIntegrations` L320–365
- New repo method: `apps/api/src/repositories/managed-app-repository.ts` `upsertByExternalId(db, payload)`

### N+1 fix — call out explicitly in PR description

Replace the per-auth-config `listConnectedAccountsByAuthConfig` loop with a **single** `connectedAccounts.list({ userIds: [orgId] })`. This is a real perf win; mention it in the PR description so reviewers don't think it's smuggled in.

### Reconcile three-way merge

Iterate over the union of `managed_apps` rows and Composio connections:

- Both present → `upsertByExternalId` (no-op when equal)
- `managed_apps` present, Composio missing → mark `DELETED_EXTERNALLY` (existing behaviour)
- Composio present, `managed_apps` missing → `authConfigs.get(authConfigId)` for toolkit/name → `upsertByExternalId`

**Failures of `authConfigs.get` (e.g. 429) MUST NOT block the list response** — log a warning, continue, try again next time. Emit `composio.reconcile` structured log with counts.

### `upsertByExternalId` — mandatory semantics

The uniqueness index is **partial**: `(org_id, provider, external_id) WHERE deleted_at IS NULL`. Generated automatically by `createTenantTable` / `addTenantUniqueConstraint` (see `apps/migrations-lambda/src/helpers/tenant-table.ts` L182–185). Verify this is the actual state before implementing — the review required pushing back on a claim it was unconditional.

Four decision branches, implement all four:

| State | Action |
|---|---|
| Live row exists, payload == stored | **no-op** — no write, no `updated_at` bump |
| Live row exists, payload ≠ stored | UPDATE the live row in place |
| No live row, soft-deleted row exists | **Resurrect** — `UPDATE ... SET deleted_at = NULL, status = <new>, metadata = <new>`. Do NOT INSERT (would collide on partial index restoration + loses history) |
| Neither exists | INSERT with `uuidv7()` |

Kysely's `onConflict` chain does not accept `WHERE` predicates for partial indexes → implementation is "SELECT → decide → INSERT/UPDATE" in a single transaction.

`managed_apps` has **no `created_at` / `updated_at` columns** (staging verified). Payload keys: `id`, `org_id`, `provider`, `app_type`, `external_id`, `name`, `status`, `metadata`, `deleted_at`.

Must be callable from three sites: webhook handler, reconcile, sync-write hook. Idempotent by design.

---

## §4.3 — Webhook Handler Extension

### Code touch point

`apps/api/src/routes/webhooks/composio.ts`

### Must-do list (review drove each of these)

1. **Remove `webhookAuth` plugin from the Composio route.** It hard-codes `externalIdPath: 'metadata.trigger_id'` which is absent from `expired` payloads. Scoped to this route only — other providers unaffected. **Also update `planning/engine/triggers.md`** to match; failing to do so blocks the PR (cross-doc conflict).
2. **Manual signature verify** with `verifyComposio`. Keep it — do NOT adopt SDK `verifyWebhook()`.
3. **Cheap pre-check:** if `body.metadata?.project_id` is present and doesn't match the configured project id, return **HTTP 200** and drop with no SDK calls. Cross-env / cross-tenant short-circuit.
4. **Dispatch on `body.type`:**

#### `composio.trigger.message`
Existing logic (lookup `integration_org_mappings` by `metadata.trigger_id`, route to agent). **Unmapped `trigger_id` MUST return HTTP 200 immediately** (ack + drop, do NOT 4xx). Preserves current `webhookAuth` behaviour; prevents Composio retry storms. Log a warning.

#### `composio.connected_account.expired`

```text
1. composio.connectedAccounts.get(body.data.id)
   - 5xx / network / 429 → respond HTTP 503 (Composio retries)
   - 404                 → respond HTTP 200 + drop (dead account, no-op)
   - other 4xx           → respond HTTP 200 + warn

2. Validate get().user_id resolves to a known orgs.id
   - If NOT → HTTP 200 + warn "legacy userId-entity connection"
     (Composio binds entity at creation time — legacy chat connections
     keep returning the old userId forever. Writing managed_apps with
     org_id = userId would corrupt tenancy or trigger FK/RLS retry storms.
     Orphan cleanup script §8 handles these out-of-band.)

3. reconcileForOrg(orgId)
   (the same helper extracted for §4.2 — no inline mini-reconcile)

4. EXPIRED status picked up from Composio's own status via
   reconcileForOrg's "both present → upsert status" branch.
   If body.data.status_reason present, forward it → metadata.status_reason.
```

#### Unknown `type`
`reply.code(200)` and drop silently. Don't 4xx unknown types.

### Tests that MUST cover this handler (§6)

- trigger.message happy path no regression
- `expired` valid signature → `managed_apps` flips to EXPIRED
- Replay `expired` → idempotent
- Unknown `type` → 200 drop
- Invalid signature → 401
- `get()` 404 → 200 drop
- `get()` 5xx/429 → 503
- **`get().user_id` not a known `orgs.id` → 200 drop + warn, no managed_apps write** (legacy entity guard)
- `metadata.project_id` mismatch → 200 drop, no SDK call

---

## §4.4 — Terraform Subscription Update

### Must-do list

- **PATCH, don't POST.** §2.1 "one per project" is a hard constraint. Creating a new subscription fails; we must update the existing one.
- All three endpoints pinned to **v3.1**:
  - `GET /api/v3.1/webhook_subscriptions` (list)
  - `POST /api/v3.1/webhook_subscriptions` (create, only when none exists)
  - `PATCH /api/v3.1/webhook_subscriptions/{id}` (update)
- Flow: `GET` → empty? `POST` + write secret to SSM. Present? `PATCH` + **skip SSM write** (secret preserved per docs).
- `enabled_events`: `["composio.trigger.message", "composio.connected_account.expired"]`
- **Explicitly pass `version: "V3"`** (don't rely on defaults).
- Add a **hash of `enabled_events`** to `triggers_replace` in `terraform/modules/composio/main.tf` so event list changes actually re-apply.

Files: `terraform/modules/composio/scripts/create-subscription.sh`, `terraform/modules/composio/main.tf`.

---

## §4.5 — Frontend Invalidation

The stream-end hook **already exists** in `apps/web/src/components/chat/ChatPage.tsx` L80–102:

```typescript
useEffect(() => {
  if (!isStreaming) refreshConnections();
}, [refreshConnections, isStreaming]);
```

Single required change: inside `refreshConnections`, also invalidate the `/integrations` cache:

```typescript
queryClient.invalidateQueries({ queryKey: ['managed-integrations'] });
```

Obtain `queryClient` via `useQueryClient()`. Keep both trigger points (`!isStreaming` effect and `window focus`).

---

## §4.6 — Tenancy & Authorization Boundaries

Explicit rules to encode (don't invent your own):

| Action | Who | Where enforced |
|---|---|---|
| List connections | Any org member | `/chat/connections`, `/composio/auth-configs` |
| Connect new integration | Any org member | Route RBAC (unchanged) |
| **Delete / disconnect** | **Org admin only** | **NEW** route guard on `DELETE /composio/auth-configs/:id` |
| View `createdByUserId` | Any org member (after immediate follow-up lands) | metadata read |

- No "personal integration" mode — all Composio connections are org-owned.
- **`orgId` is NEVER accepted from client input** on any new or modified endpoint — always server-resolved from JWT.
- User offboarding does NOT revoke their Composio connections (they're org-owned). Document this so offboarding code doesn't tear them down.

---

## §5 — Rollout Order (MANDATORY)

```text
0. Pre-deploy sanity check: enumerate orphan userId-entity accounts in
   Composio dashboard for current chat users. Communicate — they must
   reconnect after step 3.
1. Deploy webhook handler code.
   (Handler ready but subscription doesn't yet send expired events.)
2. Terraform apply: subscription enabled_events += connected_account.expired.
3. Deploy entity switch + sync-write hook + lazy reconcile + frontend
   invalidate. Chat-created connections now bind to orgId.
```

**This is forward-only.** Rollback = users reconnect. Webhook code and Terraform are independently reversible via re-deploy.

Staging: trigger an expired event via Composio dashboard, confirm `managed_apps` flips to EXPIRED, run cross-user visibility integration test.

---

## §9 — Observability (ship alongside the code, not later)

Structured events — minimum set:

| Event | Fields | Alert on |
|---|---|---|
| `composio.sync_write` | `{ orgId, authConfigId, connectedAccountId, outcome }` | Sustained non-zero `outcome: 'failed'` rate |
| `composio.reconcile` | `{ orgId, composioCount, managedAppsCount, filledIn, deletedExternally, syncFailed }` | `filledIn > 0` steady state → sync-write drift |
| `composio.webhook` | `{ type, orgId?, connectedAccountId?, outcome, httpStatus }` | Sustained HTTP 503 rate |

Also: counter for `connectedAccounts.get` calls (answers §7 Q3). Staging synthetic probe: two users in the same org list `/composio/auth-configs` → results must match.

OTel: new code paths (afterExecute hook, `reconcileForOrg`, webhook expired branch) must propagate span context and include `orgId` as a span attribute.

---

## Review-Cycle Gotchas (checked by reviewers — check yourself first)

Before opening the implementation PR, verify each of these is in the diff:

1. **Cross-doc sync.** Any change to how `/webhooks/composio` works? Update `planning/engine/triggers.md` in the same PR (or block on cross-doc conflict). Any change to migration assumptions? Update `planning/engine/composio-multitenancy.md` §3 Scope 3.
2. **Verify claims against code.** Quote the file + line that proves any "current behaviour" assertion. The review pushed back on unverified claims twice in this PR.
3. **No-op bullets.** Before writing "change X from A to B", confirm A is actually the current state. Two plan bullets turned out to be already-true no-ops.
4. **Legacy entity guard.** `get().user_id` → must resolve to known `orgs.id` before writing. Composio binds entity at creation time forever.
5. **Partial index upsert.** Handle all four branches — no-op, update, **resurrect**, insert. The resurrect case is the one that gets missed.
6. **RBAC rationale.** If relaxing a guard or filter, state current state + new state + why the delta is safe.
7. **Forward-only flag.** Entity switch is forward-only — say it in the PR description so reviewers don't expect a rollback plan.
8. **Observability ships with the code.** Not a follow-up.
9. **Rollout `step 0`** exists **before** the deploy steps. Orphan enumeration is pre-flight.

---

## Acceptance-Criteria Self-Check (§10)

Before PR ready-for-review, the diff must show:

- [ ] `listOrgConnections(orgId)` replaces `listUserConnections(userId)`; filter is `userIds: [orgId]`; route RBAC unchanged.
- [ ] `composio.create(orgId, ...)`; `ResolveToolsDeps.userId` removed; `interactiveComposioSession: boolean` added; `resolveComposioSession` only invoked when flag is true.
- [ ] `afterExecute` hook writes `managed_apps` on `COMPOSIO_WAIT_FOR_CONNECTIONS` success.
- [ ] `agent-runner.ts` passes `interactiveComposioSession: run.createdBy != null`.
- [ ] `upsertByExternalId`: resurrect, no-op, update, insert — all four branches tested.
- [ ] `listIntegrations`: single `connectedAccounts.list` (N+1 removed); reconcile non-blocking on upstream failure.
- [ ] `/webhooks/composio`: removes `webhookAuth`; validates `metadata.project_id`; validates `get().user_id` resolves to `orgs.id`; 503 on transient / 200 on 404.
- [ ] `DELETE /composio/auth-configs/:id` guarded to org-admin.
- [ ] Terraform: PATCH flow, v3.1, `version: "V3"`, `enabled_events` hash in `triggers_replace`.
- [ ] `ChatPage` invalidates `['managed-integrations']` on stream end + window focus.
- [ ] `composio.sync_write` / `composio.reconcile` / `composio.webhook` structured logs in staging.
- [ ] Tests: cross-user visibility, type dispatch, expired idempotency, reconcile fill-in, reconcile graceful degradation, soft-delete resurrection, legacy-userId rejection, project_id mismatch drop.
- [ ] Rollout follows §5; staging verified.

---

## Key Files to Read Before Writing

- `planning/engine/org-level-composio-sync.md` — source of truth, read first.
- `planning/engine/composio-multitenancy.md` — Approach B rationale; `managed_apps` as tenancy guard (§4.1–4.3).
- `planning/engine/triggers.md` — superseded §Composio section; contrast with the new handler.
- `apps/api/src/services/composio-service.ts` — current `listUserConnections` + `create` shape.
- `apps/workloads/src/tool-resolver.ts` — `buildBeforeExecuteModifier` (extension point for afterExecute).
- `apps/api/src/routes/webhooks/composio.ts` — current handler using `webhookAuth`.
- `apps/migrations-lambda/src/helpers/tenant-table.ts` L182–185 — confirms partial unique index.
- `apps/api/src/repositories/workload-run-repository.ts` L38 — confirms `createdBy` null-by-design.
