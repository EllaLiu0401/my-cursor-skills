---
name: rbac-role-access
description: >-
  Implement and modify role-based access control for Aetheron Connect V2.
  Covers backend route guards, frontend tab/section visibility, integration
  test patterns, and role hierarchy. Use when changing who can access a
  feature, adding role restrictions, reviewing RBAC, or when the user
  mentions roles, permissions, admin-only, superuser-only, or access control.
---

# RBAC Role Access — Aetheron Connect V2

Quick reference for implementing role-based visibility and access guards.

---

## Role Hierarchy (inclusive flags)

```text
superuser  →  isSuperuser = true,  isOwner = true,   isAdmin = true
org_owner  →  isSuperuser = false, isOwner = true,   isAdmin = true
org_admin  →  isSuperuser = false, isOwner = false,   isAdmin = true
org_analyst → isSuperuser = false, isOwner = false,   isAdmin = false
```

Source: `apps/web/src/lib/auth-constants.ts` — `buildRoleFlags()`

Each flag **includes all levels above it**. Use the *lowest* flag that covers your intent:
- Feature for all admins → `isAdmin`
- Feature for owners only → `isOwner`
- Internal tooling only → `isSuperuser`

---

## Backend: Route-Level Guard

Declare required roles in route config. The RBAC plugin (`apps/api/src/plugins/rbac.ts`) enforces via `preHandler` hook.

```typescript
fastify.get('/resource', {
  config: { roles: ['org_admin'] },
  schema: { ... },
}, handler);
```

### Common role patterns in this codebase

| Access Level | `config.roles` value | When to use |
|---|---|---|
| All authenticated | `['org_analyst']` | Read-only data visible to everyone |
| Admin+ (mutate) | `['org_owner', 'org_admin']` | Standard create/update/delete |
| Owner only | `['org_owner']` | Billing, API keys, destructive actions |
| Superuser only | `['superuser']` | Internal tools, debug endpoints |
| Owner + superuser | `['org_owner', 'superuser']` | Org cleanup, special admin |

### Key rule
`org_analyst` is the **lowest** role. Setting `roles: ['org_analyst']` means "all authenticated users" because the RBAC check uses `userRoles.some(r => requiredRoles.includes(r))` and higher roles (admin, owner, superuser) always pass independently via their own role string in the JWT.

**Exception**: `superuser` is NOT automatically included when you write `roles: ['org_admin']`. Superuser works because `buildRoleFlags` maps it to `isAdmin` on the frontend, but on the backend the RBAC plugin does a **literal array intersection** — superuser passes `org_admin` routes only if the middleware explicitly adds `org_admin` to the user's role array during impersonation. Check `planning/rbac.md` for impersonation flow details.

---

## Frontend: Conditional Visibility

### Server Component (page.tsx)

```typescript
import { getServerRoleFlags } from '@/lib/auth-constants';

const { isAdmin, isSuperuser } = await getServerRoleFlags();
// Pass as props to client components
<MyComponent isAdmin={isAdmin} isSuperuser={isSuperuser} />
```

### Client Component (reading from Auth0 hook)

```typescript
import { useUser } from '@auth0/nextjs-auth0/client';
import { getRoleFlags } from '@/lib/auth-constants';

const { user } = useUser();
const { isAdmin } = getRoleFlags(user);
```

### Tab/Section visibility pattern

For restricting tabs or UI sections by role tier:

```typescript
const ADMIN_TABS = new Set(['messages']);
const SUPERUSER_TABS = new Set(['logs', 'structuredOutputs']);

const isTabVisible = (tab: string) =>
  (!ADMIN_TABS.has(tab) && !SUPERUSER_TABS.has(tab)) ||
  (ADMIN_TABS.has(tab) && isAdmin) ||
  (SUPERUSER_TABS.has(tab) && isSuperuser);

const visibleTabs = ALL_TABS.filter(isTabVisible);
```

### URL param sanitization

When tab state is URL-driven (`?tab=`), sanitize restricted tabs for unauthorized users:

```typescript
const activeTab = !isTabVisible(resolvedTab) ? 'defaultTab' : resolvedTab;
```

---

## Frontend vs Backend Guard Decision

| Scenario | Guard type | Rationale |
|---|---|---|
| Tab has its own API endpoint | Backend `config.roles` + frontend hide | Defense in depth |
| Tab displays data from a shared endpoint (same data, different view) | Frontend-only hide | Backend guard would break other features using the same endpoint |
| Page-level restriction | `redirect()` or `notFound()` in server component | Standard Next.js pattern |

**Rule from `planning/rbac.md`**: "UI must not hide read-restricted data through client-only checks; enforce on the server." Apply backend guards whenever the data source has a dedicated endpoint.

---

## Integration Tests: Role Guard Testing

### Test setup pattern

```typescript
const headersA = {
  owner: dynamicAuthHeader('org_owner', auth0OrgIdA),
  admin: dynamicAuthHeader('org_admin', auth0OrgIdA),
  analyst: dynamicAuthHeader('org_analyst', auth0OrgIdA),
  superuser: dynamicAuthHeader('superuser', auth0OrgIdA),
};
```

### Required test cases for every guarded endpoint

1. **Happy path** — correct role gets 200
2. **403** — insufficient role gets 403 (test the role one level below the guard)
3. **404** — non-existent resource
4. **Tenancy isolation** — different org gets 404 (not 403)

### Example: superuser-only endpoint test

```typescript
it('returns 200 for superuser', async () => {
  const response = await app.inject({
    method: 'GET',
    url: `/endpoint/${resourceId}`,
    headers: headersA.superuser,
  });
  expect(response.statusCode).toBe(200);
});

it('returns 403 for org_admin', async () => {
  const response = await app.inject({
    method: 'GET',
    url: `/endpoint/${resourceId}`,
    headers: headersA.admin,
  });
  expect(response.statusCode).toBe(403);
});
```

---

## Checklist: Changing Role Access

1. **Identify the target access level** using the role hierarchy above
2. **Backend**: Update `config: { roles: [...] }` on the route
3. **Frontend**: Update visibility check (flag prop or conditional render)
4. **Tests**: Update happy-path headers + add 403 test for newly-denied role
5. **OpenAPI**: Update endpoint description if it mentions the access level
6. **PR description**: Include a role access table showing before/after

---

## Key Files

- `planning/rbac.md` — authoritative RBAC design doc
- `apps/web/src/lib/auth-constants.ts` — `RoleFlags`, `getServerRoleFlags()`, `getRoleFlags()`
- `apps/api/src/plugins/rbac.ts` — backend RBAC enforcement plugin
- `apps/api/src/types/fastify.d.ts` — `FastifyContextConfig.roles` type
