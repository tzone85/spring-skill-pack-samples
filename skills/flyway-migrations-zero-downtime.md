---
name: flyway-migrations-zero-downtime
description: Use when adding a Flyway migration to a service that runs in production with rolling deploys, blue/green, or any setup where two app versions briefly touch the DB at once. Catches breaking schema changes before they break readiness.
---

# flyway-migrations-zero-downtime

## When this fires

- Adding any migration to a production service.
- Renaming or removing a column / table the running app still reads or writes.
- Changing a column type, NOT NULL constraint, or default.
- Adding an index on a large table.
- Any rolling-deploy or canary environment where N-1 and N versions coexist for minutes.

## The rule

A migration is zero-downtime if **and only if** both the old code AND the new code work against the new schema. Plan in phases.

## Safe vs unsafe — quick table

| Change | Safe single-step? | Pattern |
|---|---|---|
| Add nullable column | Yes | Single migration |
| Add NOT NULL column with default | Postgres 11+: yes; older: no | Add nullable → backfill → set NOT NULL → flip default |
| Drop column | No | Stop writing → deploy → stop reading → deploy → drop in next release |
| Rename column | No | Add new column → dual-write → backfill → switch reads → drop old |
| Change column type | No | Add new column → dual-write with cast → backfill → switch reads → drop old |
| Add index on large table | No (locks writes) | `CREATE INDEX CONCURRENTLY` outside Flyway, OR migration with `concurrent=true` (Flyway 9+) |
| Add foreign key | Risky | Add `NOT VALID` → `VALIDATE CONSTRAINT` in second migration |
| Drop NOT NULL | Yes | Single migration |
| Add NOT NULL to existing column | No | Backfill → validate-not-null in two steps |

## Concrete example — renaming `customer.email` to `customer.email_address`

**Migration V12 (with release N):**
```sql
ALTER TABLE customer ADD COLUMN email_address TEXT;
UPDATE customer SET email_address = email WHERE email_address IS NULL;
-- backfill in batches if table is large; see batched-backfill skill
```

**Code in release N:** writes BOTH columns, reads `email` (old).

**Migration V13 (with release N+1):** no schema change. Just an empty marker.

**Code in release N+1:** writes BOTH columns, reads `email_address`. Verify telemetry shows `email_address` reads only.

**Migration V14 (with release N+2):**
```sql
ALTER TABLE customer ALTER COLUMN email_address SET NOT NULL;
ALTER TABLE customer DROP COLUMN email;
```

**Code in release N+2:** writes and reads `email_address` only.

Three releases, never any downtime.

## Flyway conventions to enforce

- Migrations are immutable once merged. New change = new V file. Never edit a checked-in migration.
- Naming: `V{yyyyMMddHHmm}__{ticket}_{snake_case_desc}.sql`. Example: `V202605311430__DIGI_5133_add_email_address.sql`.
- One logical change per migration. Don't bundle "add table + add index + alter another table".
- `BEGIN; ... COMMIT;` wrap explicit only when the statements support it (e.g. `CREATE INDEX CONCURRENTLY` cannot be in a transaction; mark `transactional=false` on those migrations).
- Repeatable migrations (`R__*.sql`) for views and functions; they re-run when checksum changes.
- Bootstrap data goes in a separate folder, gated by Spring profile, not in `V__` files.

## Pre-merge checklist (CI gate)

```yaml
# .github/workflows/flyway-safety.yml — runs on PRs touching db/migration/
- name: detect-renames-or-drops
  run: ./scripts/lint-migration.sh   # greps for DROP COLUMN / RENAME COLUMN, fails build
- name: flyway-validate-on-prod-clone
  run: flyway migrate -dryRun=true   # runs against a snapshot of prod schema
- name: archunit-no-entity-rename
  run: ./gradlew archTest
```

## Anti-patterns to refuse

- `DROP COLUMN` in the same migration that deploys with the code that stopped reading it — old replicas still serve traffic. Reject.
- `ALTER COLUMN TYPE` with implicit cast on a hot table — rewrites the whole table, locks writes. Reject; new-column-and-backfill.
- `CREATE INDEX` (no CONCURRENTLY) on a >10M row table — locks writes for minutes. Reject.
- "I'll just run it manually after hours" — bypasses Flyway, leaves history table in lying state. Reject.
- Editing a deployed migration to fix a typo — production-state divergence on next checksum check. Reject; new migration that fixes.

## Output contract

When invoked on a proposed migration: classify the change against the table; if unsafe, produce the multi-release plan with the exact SQL for each step and the code-change description that pairs with each.

## See also

- `jpa-entity-design`
- `testcontainers-postgres-spring-boot` (test migrations in CI)
- `outbox-pattern` (zero-downtime event delivery)
