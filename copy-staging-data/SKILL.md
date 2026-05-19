---
name: copy-staging-data
description: >-
  Copy data from the Aetheron Connect staging database and S3 to the local
  development environment for testing. Use when the user wants to copy staging
  call records, interactions, or any feature data to their local DB for testing,
  says "copy from staging", "pull staging data", "test with staging data",
  provides a staging URL with a record ID, or mentions needing real data locally
  to test a Jira task or feature.
---

# Copy Staging Data to Local

Copy records from staging (Aurora via RDS Data API + S3) into the local dev environment (Postgres on :5433 + LocalStack S3 on :4566) so the user can test against real data.

## Required Inputs

Collect these from the user before starting:

| Input | Example | Notes |
|-------|---------|-------|
| **What to copy** | Staging URL, record ID, Jira task, or feature description | Determines which tables/records to query |
| **AWS credentials** | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN` | Temporary STS creds with staging access |
| **Local user profile** | Auth0 org ID (`org_xxx`), org name | Used to find the local org_id to remap records to |

## Workflow

### Step 1: Identify Target Data

From the user's task/URL/feature description, determine:
- Which **tables** contain the relevant records (check migrations under `apps/migrations-lambda/migrations/`)
- Which **S3 keys** are referenced (columns ending in `_s3_key`)
- Which **related tables** have FK relationships that need satisfying

Common table mappings:

| Feature | Primary Table | S3 Bucket | Related Tables |
|---------|--------------|-----------|----------------|
| Calls / Call logs | `interactions` | `ac-{env}-interactions` | `organisation_details` (timezone) |
| Assistants | `assistants` | — | `agent_configs`, `assistant_tools`, `assistant_post_call_hooks` |
| Reports | `weekly_reports` | — | `report_configs` |
| Agent configs | `agent_configs` | `ac-{env}-agent-workspaces` | `agent_config_tools`, `agent_config_triggers` |

### Step 2: Connect to Staging DB (RDS Data API)

```bash
# Set AWS creds (from user input)
export AWS_ACCESS_KEY_ID="..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_SESSION_TOKEN="..."
export AWS_REGION="ap-southeast-2"

# Resolve cluster metadata
CLUSTER_ARN=$(aws rds describe-db-clusters \
  --db-cluster-identifier "ac-staging-aurora-postgres" \
  --query 'DBClusters[0].DBClusterArn' --output text --region "$AWS_REGION")
DB_NAME=$(aws rds describe-db-clusters \
  --db-cluster-identifier "ac-staging-aurora-postgres" \
  --query 'DBClusters[0].DatabaseName' --output text --region "$AWS_REGION")
SECRET_ARN=$(aws rds describe-db-clusters \
  --db-cluster-identifier "ac-staging-aurora-postgres" \
  --query 'DBClusters[0].MasterUserSecret.SecretArn' --output text --region "$AWS_REGION")
```

Query pattern (inline creds because env vars don't persist between Shell calls):

```bash
AWS_ACCESS_KEY_ID="..." AWS_SECRET_ACCESS_KEY="..." AWS_SESSION_TOKEN="..." \
  AWS_REGION="ap-southeast-2" \
  aws rds-data execute-statement \
  --resource-arn "CLUSTER_ARN_VALUE" \
  --secret-arn "SECRET_ARN_VALUE" \
  --database "DB_NAME_VALUE" \
  --sql "SELECT * FROM table WHERE id = 'xxx'" \
  --format-records-as JSON \
  --region "ap-southeast-2"
```

**IMPORTANT**: Env vars do NOT persist between Shell tool calls. Always inline all AWS env vars on the same command line, or chain commands with `&&` in a single Shell call.

### Step 3: Find the Local Org

```bash
PGPASSWORD=postgres psql -h localhost -p 5433 -U postgres -d aetheron \
  -c "SELECT id, name, auth0_org_id FROM organisations WHERE auth0_org_id = 'org_xxx'"
```

This gives the **local org_id** to use when inserting records.

### Step 4: Query Staging Records

1. Query the primary table for the target record(s)
2. Note the **staging org_id** (will be replaced with local org_id)
3. Check for S3 key columns — these files need downloading
4. Check `organisation_details` for the staging org (timezone, locale, currency)

### Step 5: Download S3 Assets from Staging

Staging bucket naming: `ac-staging-{name}` (e.g., `ac-staging-interactions`)

```bash
AWS_ACCESS_KEY_ID="..." AWS_SECRET_ACCESS_KEY="..." AWS_SESSION_TOKEN="..." \
  AWS_REGION="ap-southeast-2" \
  aws s3 cp s3://ac-staging-{bucket}/{s3_key} /tmp/{filename} --region ap-southeast-2
```

To find the right bucket name:
```bash
aws s3api list-buckets --query 'Buckets[?contains(Name, `staging`)].Name' --output text
```

### Step 6: Insert Records Locally

Key rules when inserting:

- **Remap `org_id`**: Replace staging org_id with local org_id
- **Generated columns**: Skip columns like `duration_seconds` (generated/stored) — they error on INSERT
- **S3 key paths**: Update the env prefix in the key: `recordings/staging/` → `recordings/local/`, and replace the staging org_id with the local org_id in the path
- **Composite PKs**: All tenant tables use `PRIMARY KEY (org_id, id)` — the id can stay the same
- **`ON CONFLICT DO NOTHING`**: Always use this to make the operation idempotent
- **JSONB columns**: Escape single quotes in JSON values, or use dollar-quoted strings `$$...$$`
- **organisation_details**: Create a record for the local org if it doesn't exist (important for timezone-dependent features)

```bash
PGPASSWORD=postgres psql -h localhost -p 5433 -U postgres -d aetheron <<'SQL'
INSERT INTO target_table (id, org_id, ...)
VALUES ('same-id', 'local-org-id', ...)
ON CONFLICT DO NOTHING;
SQL
```

### Step 7: Upload S3 Assets to LocalStack

Local bucket naming: `ac-local-{name}` (e.g., `ac-local-interactions`)

```bash
AWS_ACCESS_KEY_ID=test AWS_SECRET_ACCESS_KEY=test \
  aws --endpoint-url=http://localhost:4566 \
  s3 cp /tmp/{filename} s3://ac-local-{bucket}/{new_s3_key} --region ap-southeast-2
```

### Step 8: Verify

1. Query local DB to confirm records exist with correct org_id
2. List LocalStack S3 to confirm files uploaded
3. Print the local URL the user can visit to test

```bash
# Verify DB
PGPASSWORD=postgres psql -h localhost -p 5433 -U postgres -d aetheron \
  -c "SELECT id, org_id, ... FROM target_table WHERE id = 'xxx'"

# Verify S3
AWS_ACCESS_KEY_ID=test AWS_SECRET_ACCESS_KEY=test \
  aws --endpoint-url=http://localhost:4566 \
  s3 ls s3://ac-local-{bucket}/{path}/ --recursive --region ap-southeast-2
```

## Gotchas

- **RDS Data API env vars**: Must be inlined on every `aws` command — Shell tool calls don't share env.
- **Generated columns**: Check column definitions before INSERT. If a column errors with "cannot insert a non-DEFAULT value into column X", it's generated — remove it from the INSERT.
- **FK constraints**: Check `information_schema.table_constraints` for FKs. Only `org_id → organisations` is common; most cross-table references (like `assistant_id` on `interactions`) are soft references (text, no FK).
- **Enums**: Staging and local must have the same enum values. If migrations are up to date locally, this is fine.
- **S3 bucket discovery**: Local buckets are listed via `aws --endpoint-url=http://localhost:4566 s3 ls`. Staging buckets via `aws s3api list-buckets`.
- **Recording/binary files**: Use `s3 cp` (not `s3api get-object`) for binary files.
- **Local DB credentials**: App user is `app_user`/`app_user`; migrations and direct queries use `postgres`/`postgres` on port `5433`.
