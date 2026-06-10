---
name: backend-db-conventions
description: >-
  Backend database conventions for Aetheron Connect V2. Covers migration patterns,
  column type choices, and schema design rules. Use when writing migrations, adding
  new tables or columns, choosing column types, or reviewing database schema changes.
---

# Backend DB Conventions

Quick reference for database schema decisions when writing migrations or reviewing schema changes.

## PG ENUM over CHECK Constraints

For status/type columns, always use Postgres `CREATE TYPE ... AS ENUM`, never `CHECK` constraints or plain `text`.

```sql
-- BAD — CHECK constraint: kysely-codegen produces `string`, no type safety
ALTER TABLE reports ADD COLUMN status TEXT NOT NULL CHECK (status IN ('draft', 'published'));

-- BAD — plain text: no constraint at all, kysely-codegen produces `string`
ALTER TABLE reports ADD COLUMN status TEXT NOT NULL;

-- GOOD — PG ENUM: kysely-codegen produces 'draft' | 'published'
CREATE TYPE report_status AS ENUM ('draft', 'published');
ALTER TABLE reports ADD COLUMN status report_status NOT NULL;
```

**Why:** `kysely-codegen` generates proper TypeScript union types from Postgres enums (e.g., `'draft' | 'published'`), whereas `text` or `text + CHECK` columns only produce `string`.

**Naming:** `{table_singular}_{column}` (e.g., `workload_run_status`, `report_status`).

### Migration pattern

```typescript
// up migration
await sql`CREATE TYPE report_status AS ENUM ('draft', 'published')`.execute(db);
await db.schema.createTable('reports')
  .addColumn('status', sql`report_status`, (col) => col.notNull())
  // ... other columns
  .execute();

// down migration — drop table BEFORE dropping type
await db.schema.dropTable('reports').execute();
await sql`DROP TYPE report_status`.execute(db);
```

### Adding values to an existing enum

`ALTER TYPE ... ADD VALUE` cannot run inside a transaction. Use a separate migration file:

```typescript
// 20260416000000_add_report_status_archived.ts
export async function up(db: Kysely<unknown>): Promise<void> {
  await sql`ALTER TYPE report_status ADD VALUE 'archived'`.execute(db);
}

// ADD VALUE is not reversible in Postgres — down is a no-op or recreate
```

## Schema field & upsert patterns

### Integer columns for sizes / counts / byte lengths

A field representing a size, count, or byte length is a non-negative whole number. Use `Type.Integer({ minimum: 0 })`, never `Type.Number()` — `Type.Number()` accepts `1.5` and negatives, and the generated OpenAPI emits `number` instead of `integer`.

```ts
// BAD — accepts 1.5 / -3, OpenAPI says number
fileSizeBytes: Type.Number();

// GOOD
fileSizeBytes: Type.Integer({ minimum: 0 });
```

### Tenant `ON CONFLICT` must match the partial unique index

`createTenantTable` builds unique indexes as **partial** (`... WHERE deleted_at IS NULL`) so soft-deleted rows don't block re-inserts. An `onConflict` upsert must reproduce that predicate, or Postgres can't find a matching arbiter and throws at runtime — `there is no unique or exclusion constraint matching the ON CONFLICT specification`. Plan/typecheck pass; the insert fails in prod.

```ts
// BAD — no index matches (the real index is partial)
.onConflict((oc) => oc.columns(['orgId', 'externalId']).doUpdateSet({ /* … */ }))

// GOOD — mirror the partial index predicate
.onConflict((oc) =>
  oc.columns(['orgId', 'externalId']).where('deletedAt', 'is', null).doUpdateSet({ /* … */ }),
)
```

## JSONB Column Handling

When reading JSONB columns via Kysely, the value is a **parsed JavaScript object** (or `null`), NOT a string. `kysely-codegen` types these as `Json | null` where `Json = JsonValue = JsonArray | JsonObject | JsonPrimitive`.

### Common pitfall: `typeof` guards on JSONB data

```typescript
// BAD — typeof is always 'object' for non-null JSONB, this guard skips the merge
function mergeMetadata(existing: unknown, incoming: string | null): string | null {
  if (!existing || typeof existing !== 'string') return incoming; // ← bug: DB objects skip merge
  return JSON.stringify({ ...JSON.parse(existing), ...JSON.parse(incoming!) });
}

// GOOD — handle both string (from JSON.stringify) and object (from DB) inputs
function mergeMetadata(existing: unknown, incoming: string | null): string | null {
  if (!existing) return incoming;
  if (!incoming) return typeof existing === 'string' ? existing : JSON.stringify(existing);
  try {
    const base: unknown = typeof existing === 'string' ? JSON.parse(existing) : existing;
    const overlay: unknown = JSON.parse(incoming);
    if (typeof base === 'object' && base !== null && typeof overlay === 'object' && overlay !== null) {
      return JSON.stringify({ ...base, ...overlay });
    }
    return incoming;
  } catch {
    return incoming;
  }
}
```

### No `as` type assertions

This codebase uses `@typescript-eslint/consistent-type-assertions: never`. Use runtime type narrowing instead:

```typescript
// BAD — ESLint rejects `as`
const obj = JSON.parse(str) as Record<string, unknown>;

// GOOD — runtime narrowing
const parsed: unknown = JSON.parse(str);
if (typeof parsed === 'object' && parsed !== null) {
  // TypeScript narrows to `object`, spread is valid
  return JSON.stringify({ ...parsed, ...overlay });
}
```

### Tombstone resurrection must merge metadata

When resurrecting a soft-deleted row, always **merge** metadata (spread existing + incoming), never overwrite. Critical fields like `slackAppId` in metadata are needed for cleanup flows. Overwriting creates unrecoverable orphans.

### Filtering metadata keeps all JSON primitive types

When whitelisting/copying metadata fields, keep every JSON primitive — `string`, `number`, `boolean` (and `null` if it carries meaning) — not just strings. A `typeof v === 'string'` filter silently drops numeric counters and boolean flags.

```ts
// BAD — drops numbers and booleans
Object.fromEntries(Object.entries(meta).filter(([, v]) => typeof v === 'string'));

// GOOD — keep all JSON primitives
Object.fromEntries(
  Object.entries(meta).filter(
    ([, v]) => v === null || ['string', 'number', 'boolean'].includes(typeof v),
  ),
);
```

## API mappers & runtime patterns

Backend code that doesn't strictly write to the DB but lives in the same `apps/api` / `apps/workloads` boundary.

### Strip null JSONB fields in API response mappers

Frontend code commonly does truthy checks like `if (config.smsConfig)` to render a section conditionally. When the backend column is JSONB and the row's value is genuinely `null` (feature disabled), the mapper must keep `null` distinct from `{}` on the wire — but `fast-json-stringify` (Fastify's default serializer) emits `{}` for any object schema even when the runtime value is `null`, depending on schema shape.

**Anti-pattern (PR #1116):**
```ts
// hotel-config-mapper.ts
export function mapHotelConfig(row: HotelRow): HotelConfigDto {
  return {
    id: row.id,
    name: row.name,
    smsConfig: row.smsConfig,            // null on disabled rows
    bookingConfig: row.bookingConfig,    // null on disabled rows
  };
}
// Wire output: { ..., smsConfig: {}, bookingConfig: {} } — UI thinks both are enabled.
```

**Fix — strip null fields explicitly before serialization:**
```ts
export function mapHotelConfig(row: HotelRow): HotelConfigDto {
  const dto: HotelConfigDto = { id: row.id, name: row.name };
  if (row.smsConfig !== null) dto.smsConfig = row.smsConfig;
  if (row.bookingConfig !== null) dto.bookingConfig = row.bookingConfig;
  return dto;
}
```

**Or — use a TypeBox schema with explicit `nullable: true`** so fast-json-stringify preserves `null`:
```ts
const HotelConfigDto = Type.Object({
  id: Type.String(),
  name: Type.String(),
  smsConfig: Type.Union([SmsConfig, Type.Null()]), // tells the serializer "real null is valid"
});
```

**Apply when:** any API response mapper returning a nullable JSONB / object column. The fast-json-stringify quirk shows up as silent UI bugs that look like backend-side feature flags being wrong.

**Test it:** unit-test the mapper with a `null` field and assert the serialized JSON either omits the key or contains literal `null` — never `{}`. A round-trip `JSON.stringify(mapHotelConfig(row))` test catches this regardless of serializer config.

### Multi-turn agent runs: capture structured tags during the stream, not from `result.output`

In agent-runner / chat-orchestration code, structured payloads (e.g. `<chat_title>...</chat_title>`, `<reasoning_tag>`) may appear in any assistant turn. The final `result.output` is whatever the last turn emitted — typically a tool call or a wrap-up summary, not the original tagged content.

**Anti-pattern (PR #1071):**
```ts
const result = await runAgent(job);
const chatTitle = extractChatTitle(result.output); // null when title was set in turn 1 but turn 3 was a tool call
```

**Fix — walk every assistant message during/after the stream:**
```ts
let streamedChatTitle: string | null = null;

for (const message of sdkMessages) {
  if (message.role !== 'assistant') continue;
  const content = typeof message.content === 'string' ? message.content : '';
  if (content.includes('<chat_title>')) {
    // "last seen wins" — if multiple turns emit a title, the most recent one is the agent's
    // final intent. This is a deliberate choice; document it so future debuggers know.
    streamedChatTitle = extractChatTitle(content);
  }
}

const chatTitle = streamedChatTitle ?? extractChatTitle(result.output);
```

**Apply when:** any code that pulls structured signals (titles, citations, debug flags, telemetry tags) out of agent / streaming output. `result.output` alone is wrong for any multi-turn scenario.

**Test it:** fixture with three turns where the title appears in turn 1, a tool call in turn 2, and a summary in turn 3. Assert the title is captured. Add a "last seen wins" fixture where turns 1 and 3 both emit titles — assert turn 3's wins.

### System-row lookups must be fully qualified, not slug-only

When resolving a shared "system" row (system agent, stable template, platform default) through a system-rows query helper, filter on the full identity — `isSystem = true` **and** `channel = 'stable'` **and** the system org — not just a human-editable `slug` / `name`. A slug-only match can collide with a tenant row that reuses the slug, or pick up a non-stable draft.

**Anti-pattern (PR #1460 / #1557):**
```ts
// slug alone — a tenant row or a draft channel can shadow the system row
selectFromWithSystemRows('agents').where('slug', '=', slug);
```

**Fix — pin the full system identity:**
```ts
selectFromWithSystemRows('agents')
  .where('isSystem', '=', true)
  .where('channel', '=', 'stable')
  .where('slug', '=', slug);
```

**Apply when:** any lookup of platform-owned shared rows. The slug is operator-editable, so it is never a sufficient key on its own.

## Key References

- `planning/db-conventions.md` — full DB conventions (source of truth)
- `AGENTS.md` — always-applied rules including DB & Multi-Tenancy section

## Provenance

- PG ENUM rules derived from `AGENTS.md` and `planning/db-conventions.md`.
- JSONB handling, `as`-assertion ban, and metadata merge rules added after PR #919 (`feat/composio-org-level-sync`), where `typeof existing !== 'string'` silently skipped metadata merge for Kysely-sourced JSONB objects, losing `slackAppId` during tombstone resurrection (Apr 2026).
- "Strip null JSONB fields in API response mappers" added after PR #1116 (`fix/hotel-sms-toggle-state-source-of-truth`), where the FE rendered the SMS section as enabled because the API mapper let fast-json-stringify serialize `null` JSONB fields as `{}` (May 2026).
- "Multi-turn agent runs: capture per turn" added after PR #1071 (`fix(workloads): persist chat title from any assistant turn (VP-199)`), where `extractChatTitle(result.output)` returned `null` whenever the title appeared in an early turn and a later turn was a tool call (May 2026).
- Schema field & upsert patterns ("Integer columns", "Tenant `ON CONFLICT` must match the partial unique index"), "Filtering metadata keeps all JSON primitive types", and "System-row lookups must be fully qualified" added after PRs #1460 + #1557 (`cube-phase1 capability reports`): `Type.Number()` was used for byte/size fields; an `onConflict` upsert omitted the `WHERE deleted_at IS NULL` predicate that `createTenantTable`'s partial unique index carries (runtime "no unique constraint matching"); a metadata filter dropped non-string primitives; and system agent/template lookups matched on slug alone instead of the full `isSystem` + `channel = 'stable'` + system-org identity (Jun 2026).
