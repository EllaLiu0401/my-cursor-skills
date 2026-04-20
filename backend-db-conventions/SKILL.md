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

## Key References

- `planning/db-conventions.md` — full DB conventions (source of truth)
- `AGENTS.md` — always-applied rules including DB & Multi-Tenancy section
