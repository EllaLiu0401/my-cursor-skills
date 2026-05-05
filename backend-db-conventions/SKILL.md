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

## Key References

- `planning/db-conventions.md` — full DB conventions (source of truth)
- `AGENTS.md` — always-applied rules including DB & Multi-Tenancy section

## Provenance

- PG ENUM rules derived from `AGENTS.md` and `planning/db-conventions.md`.
- JSONB handling, `as`-assertion ban, and metadata merge rules added after PR #919 (`feat/composio-org-level-sync`), where `typeof existing !== 'string'` silently skipped metadata merge for Kysely-sourced JSONB objects, losing `slackAppId` during tombstone resurrection (Apr 2026).
