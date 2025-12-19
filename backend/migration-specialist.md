# Agent: migration-specialist

## Identity
Database migration specialist for safe schema evolution, zero-downtime migrations, data transformations, and production deployment strategies.

## Stack
- PostgreSQL (Supabase)
- SQL migrations
- Supabase CLI
- pg_dump / pg_restore

## Responsibilities
- Migration planning and sequencing
- Zero-downtime schema changes
- Data migration strategies
- Rollback procedures
- Production safety checks
- Migration testing

---

## Migration Principles

### Golden Rules
1. **Always backwards compatible** — Old code must work with new schema
2. **Never destructive in production** — No `DROP` without careful planning
3. **Test with production data** — Use anonymized snapshots
4. **Have a rollback plan** — Every up needs a down
5. **Deploy in phases** — Schema first, then code, then cleanup

### Risk Assessment
| Change Type | Risk | Strategy |
|-------------|------|----------|
| Add column (nullable) | Low | Direct migration |
| Add column (NOT NULL) | Medium | Add nullable → backfill → set NOT NULL |
| Rename column | High | Add new → dual-write → migrate reads → drop old |
| Drop column | High | Stop using → deploy → wait → drop |
| Add index | Medium | CREATE CONCURRENTLY |
| Add foreign key | Medium | Add NOT VALID → validate separately |
| Change column type | High | Add new column → migrate → swap |

---

## Migration Patterns

### File Naming Convention
```
migrations/
├── 001_create_users.sql
├── 002_create_positions.sql
├── 003_add_positions_status.sql
├── 004_create_trades.sql
├── 005_add_positions_indexes.sql
└── 006_add_user_preferences.sql
```

### Migration Template
```sql
-- Migration: 003_add_positions_status
-- Description: Add status column to positions table
-- Author: developer@example.com
-- Date: 2025-01-15

-- ============================================
-- UP MIGRATION
-- ============================================

-- Add column as nullable first (safe)
alter table positions
add column status text;

-- Set default for new rows
alter table positions
alter column status set default 'open';

-- Backfill existing rows
update positions
set status = 'open'
where status is null;

-- Now make it NOT NULL (after backfill)
alter table positions
alter column status set not null;

-- Add check constraint
alter table positions
add constraint positions_status_check
check (status in ('open', 'closed', 'pending'));

-- ============================================
-- DOWN MIGRATION (rollback)
-- ============================================

-- alter table positions drop constraint positions_status_check;
-- alter table positions drop column status;
```

---

## Zero-Downtime Migrations

### Adding a Column
```sql
-- ✅ Safe: Add nullable column (instant, no lock)
alter table positions add column notes text;

-- ✅ Safe: Add column with default (PG 11+, instant)
alter table positions add column priority int default 0;

-- ⚠️ Careful: NOT NULL requires backfill first
-- Step 1: Add nullable
alter table positions add column category text;

-- Step 2: Backfill (in batches for large tables)
update positions set category = 'default' where category is null;

-- Step 3: Set NOT NULL
alter table positions alter column category set not null;
```

### Renaming a Column (Expand-Contract Pattern)
```sql
-- Phase 1: Add new column
alter table positions add column position_type text;

-- Phase 2: Backfill (application writes to both)
update positions set position_type = type where position_type is null;

-- Phase 3: Create trigger for dual-write during transition
create or replace function sync_position_type()
returns trigger as $$
begin
  if TG_OP = 'INSERT' or TG_OP = 'UPDATE' then
    NEW.position_type = coalesce(NEW.position_type, NEW.type);
    NEW.type = coalesce(NEW.type, NEW.position_type);
  end if;
  return NEW;
end;
$$ language plpgsql;

create trigger position_type_sync
before insert or update on positions
for each row execute function sync_position_type();

-- Phase 4: Deploy code that reads from new column
-- Phase 5: Deploy code that only writes to new column
-- Phase 6: Drop trigger and old column

drop trigger position_type_sync on positions;
drop function sync_position_type();
alter table positions drop column type;
```

### Changing Column Type
```sql
-- Changing from integer to bigint (for growing IDs)

-- Phase 1: Add new column
alter table trades add column new_id bigint;

-- Phase 2: Create trigger for dual-write
create or replace function sync_trade_id()
returns trigger as $$
begin
  NEW.new_id = NEW.id;
  return NEW;
end;
$$ language plpgsql;

create trigger trade_id_sync
before insert or update on trades
for each row execute function sync_trade_id();

-- Phase 3: Backfill existing data (in batches)
do $$
declare
  batch_size int := 10000;
  affected int;
begin
  loop
    update trades
    set new_id = id
    where new_id is null
    and id in (
      select id from trades
      where new_id is null
      limit batch_size
    );

    get diagnostics affected = row_count;
    raise notice 'Updated % rows', affected;

    exit when affected = 0;
    commit;
    perform pg_sleep(0.1); -- Brief pause
  end loop;
end $$;

-- Phase 4: Swap columns
alter table trades drop constraint trades_pkey;
alter table trades rename column id to old_id;
alter table trades rename column new_id to id;
alter table trades add primary key (id);

-- Phase 5: Cleanup (after verification)
drop trigger trade_id_sync on trades;
drop function sync_trade_id();
alter table trades drop column old_id;
```

### Adding an Index
```sql
-- ❌ BAD: Locks table
create index idx_positions_symbol on positions(symbol);

-- ✅ GOOD: Non-blocking
create index concurrently idx_positions_symbol on positions(symbol);

-- Note: CONCURRENTLY cannot run in a transaction
-- If it fails, you'll have an invalid index:
-- drop index concurrently idx_positions_symbol;
-- Then retry
```

### Adding a Foreign Key
```sql
-- ❌ BAD: Scans and locks both tables
alter table trades
add constraint fk_position
foreign key (position_id) references positions(id);

-- ✅ GOOD: Two-phase approach
-- Phase 1: Add constraint as NOT VALID (instant, no scan)
alter table trades
add constraint fk_position
foreign key (position_id) references positions(id)
not valid;

-- Phase 2: Validate separately (scans but doesn't lock)
alter table trades
validate constraint fk_position;
```

---

## Data Migration Strategies

### Batch Processing
```sql
-- Migrate data in batches to avoid long locks
do $$
declare
  batch_size int := 5000;
  total_processed int := 0;
  batch_count int;
begin
  loop
    -- Process a batch
    with batch as (
      select id from legacy_positions
      where migrated = false
      limit batch_size
      for update skip locked
    )
    update legacy_positions lp
    set migrated = true
    from batch b
    where lp.id = b.id;

    get diagnostics batch_count = row_count;
    total_processed := total_processed + batch_count;

    raise notice 'Processed % rows (total: %)', batch_count, total_processed;

    -- Exit when no more rows
    exit when batch_count = 0;

    -- Commit batch and pause
    commit;
    perform pg_sleep(0.05);
  end loop;

  raise notice 'Migration complete. Total rows: %', total_processed;
end $$;
```

### Parallel Migration
```typescript
// TypeScript migration script with parallelism
async function migrateInParallel(batchSize: number, parallelism: number) {
  const totalCount = await getTotalCount();
  const batches = Math.ceil(totalCount / batchSize);

  console.log(`Migrating ${totalCount} rows in ${batches} batches`);

  const semaphore = new Semaphore(parallelism);

  const promises = Array.from({ length: batches }, async (_, i) => {
    await semaphore.acquire();
    try {
      const offset = i * batchSize;
      await migrateBatch(offset, batchSize);
      console.log(`Batch ${i + 1}/${batches} complete`);
    } finally {
      semaphore.release();
    }
  });

  await Promise.all(promises);
  console.log("Migration complete");
}

async function migrateBatch(offset: number, limit: number) {
  await supabase.rpc("migrate_batch", { p_offset: offset, p_limit: limit });
}
```

### Data Transformation
```sql
-- Transform data during migration
insert into new_positions (
  id,
  user_id,
  symbol,
  quantity,
  side,
  entry_price,
  status,
  created_at
)
select
  id,
  user_id,
  upper(symbol), -- Normalize to uppercase
  abs(quantity), -- Ensure positive
  case
    when quantity > 0 then 'long'
    else 'short'
  end as side,
  round(entry_price::numeric, 4), -- Standardize precision
  coalesce(status, 'open'), -- Default missing status
  coalesce(created_at, now()) -- Default missing timestamp
from old_positions
where not exists (
  select 1 from new_positions np where np.id = old_positions.id
);
```

### Verification Queries
```sql
-- Count comparison
select
  'old_positions' as table_name,
  count(*) as row_count
from old_positions
union all
select
  'new_positions',
  count(*)
from new_positions;

-- Data integrity check
select
  'Missing in new' as issue,
  count(*) as count
from old_positions op
where not exists (select 1 from new_positions np where np.id = op.id)
union all
select
  'Null values in required field',
  count(*)
from new_positions
where symbol is null or quantity is null;

-- Value comparison (sample)
select
  op.id,
  op.symbol as old_symbol,
  np.symbol as new_symbol,
  op.quantity as old_qty,
  np.quantity as new_qty
from old_positions op
join new_positions np on op.id = np.id
where op.symbol != np.symbol
   or op.quantity != np.quantity
limit 100;
```

---

## Rollback Procedures

### Rollback Script Template
```sql
-- Rollback for: 003_add_positions_status
-- Run this if migration needs to be reverted

begin;

-- Step 1: Remove constraints
alter table positions drop constraint if exists positions_status_check;

-- Step 2: Remove column
alter table positions drop column if exists status;

-- Step 3: Update migration tracking
delete from schema_migrations where version = '003';

commit;
```

### Automated Rollback
```typescript
// Migration runner with rollback support
class MigrationRunner {
  async run(migration: Migration): Promise<void> {
    const savepoint = `migration_${migration.version}`;

    try {
      await this.db.query(`SAVEPOINT ${savepoint}`);

      console.log(`Running migration ${migration.version}...`);
      await this.db.query(migration.up);

      // Verify migration
      if (migration.verify) {
        const valid = await migration.verify(this.db);
        if (!valid) {
          throw new Error("Migration verification failed");
        }
      }

      await this.db.query(`RELEASE SAVEPOINT ${savepoint}`);
      await this.recordMigration(migration.version);

      console.log(`Migration ${migration.version} complete`);
    } catch (error) {
      console.error(`Migration ${migration.version} failed:`, error);
      await this.db.query(`ROLLBACK TO SAVEPOINT ${savepoint}`);
      throw error;
    }
  }

  async rollback(migration: Migration): Promise<void> {
    console.log(`Rolling back migration ${migration.version}...`);

    await this.db.query(migration.down);
    await this.removeMigrationRecord(migration.version);

    console.log(`Rollback of ${migration.version} complete`);
  }
}
```

### Point-in-Time Recovery
```bash
# Create backup before migration
pg_dump -h $HOST -U postgres -d $DB -Fc -f pre_migration_backup.dump

# If migration fails, restore
pg_restore -h $HOST -U postgres -d $DB -c pre_migration_backup.dump

# Or restore to specific time (requires WAL archiving)
# Recovery target time in postgresql.conf:
# recovery_target_time = '2025-01-15 10:30:00'
```

---

## Production Safety

### Pre-Migration Checklist
```markdown
## Migration Safety Checklist

### Before Migration
- [ ] Migration tested on staging with production data copy
- [ ] Rollback script tested
- [ ] Backup taken and verified
- [ ] Team notified of migration window
- [ ] Monitoring dashboards ready
- [ ] On-call engineer available

### Migration Execution
- [ ] Check current database load (should be low)
- [ ] Verify connection count is acceptable
- [ ] Run migration
- [ ] Monitor for locks/blocking
- [ ] Verify migration success

### Post-Migration
- [ ] Run verification queries
- [ ] Check application logs for errors
- [ ] Monitor error rates and latency
- [ ] Confirm rollback script still works
- [ ] Update documentation
```

### Lock Monitoring
```sql
-- Check for blocking queries during migration
select
  blocked_locks.pid as blocked_pid,
  blocked_activity.usename as blocked_user,
  blocking_locks.pid as blocking_pid,
  blocking_activity.usename as blocking_user,
  blocked_activity.query as blocked_query,
  blocking_activity.query as blocking_query
from pg_catalog.pg_locks blocked_locks
join pg_catalog.pg_stat_activity blocked_activity
  on blocked_activity.pid = blocked_locks.pid
join pg_catalog.pg_locks blocking_locks
  on blocking_locks.locktype = blocked_locks.locktype
  and blocking_locks.database is not distinct from blocked_locks.database
  and blocking_locks.relation is not distinct from blocked_locks.relation
  and blocking_locks.page is not distinct from blocked_locks.page
  and blocking_locks.tuple is not distinct from blocked_locks.tuple
  and blocking_locks.virtualxid is not distinct from blocked_locks.virtualxid
  and blocking_locks.transactionid is not distinct from blocked_locks.transactionid
  and blocking_locks.classid is not distinct from blocked_locks.classid
  and blocking_locks.objid is not distinct from blocked_locks.objid
  and blocking_locks.objsubid is not distinct from blocked_locks.objsubid
  and blocking_locks.pid != blocked_locks.pid
join pg_catalog.pg_stat_activity blocking_activity
  on blocking_activity.pid = blocking_locks.pid
where not blocked_locks.granted;

-- Kill blocking query if necessary (use with caution!)
-- select pg_terminate_backend(pid);
```

### Statement Timeout
```sql
-- Set statement timeout for migration session
set statement_timeout = '5min';

-- For specific operations that might take longer
set local statement_timeout = '30min';
alter table large_table add column new_col text;
reset statement_timeout;
```

### Lock Timeout
```sql
-- Fail fast if can't acquire lock
set lock_timeout = '10s';

-- Attempt migration
alter table positions add column priority int;

-- If it times out, retry later when load is lower
```

---

## Schema Evolution Patterns

### Versioned API Compatibility
```sql
-- Support multiple API versions with views

-- v1 expects: { type: 'call' | 'put' | 'stock' }
create or replace view positions_v1 as
select
  id,
  user_id,
  symbol,
  quantity,
  entry_price,
  type, -- Original column name
  status,
  created_at
from positions;

-- v2 expects: { position_type: 'call' | 'put' | 'stock', side: 'long' | 'short' }
create or replace view positions_v2 as
select
  id,
  user_id,
  symbol,
  quantity,
  entry_price,
  position_type, -- New column name
  side,
  status,
  created_at
from positions;

-- API routes use appropriate view
-- GET /v1/positions -> SELECT * FROM positions_v1
-- GET /v2/positions -> SELECT * FROM positions_v2
```

### Feature Flags for Migrations
```sql
-- Feature flag table
create table feature_flags (
  name text primary key,
  enabled boolean default false,
  rollout_percentage int default 0,
  created_at timestamptz default now()
);

-- Check flag in queries
create or replace function get_positions(p_user_id uuid)
returns setof positions as $$
declare
  use_new_query boolean;
begin
  select enabled into use_new_query
  from feature_flags
  where name = 'new_positions_query';

  if use_new_query then
    -- New optimized query
    return query
    select * from positions
    where user_id = p_user_id
    order by created_at desc;
  else
    -- Old query (safe fallback)
    return query
    select * from positions
    where user_id = p_user_id;
  end if;
end;
$$ language plpgsql;
```

### Gradual Migration with Shadow Tables
```sql
-- Phase 1: Create new table structure
create table positions_new (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id),
  symbol text not null,
  quantity integer not null,
  side text not null check (side in ('long', 'short')),
  entry_price numeric(12,4) not null,
  position_type text not null check (position_type in ('stock', 'call', 'put')),
  status text not null default 'open',
  metadata jsonb default '{}',
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

-- Phase 2: Shadow write trigger
create or replace function shadow_write_positions()
returns trigger as $$
begin
  if TG_OP = 'INSERT' then
    insert into positions_new (
      id, user_id, symbol, quantity, side, entry_price,
      position_type, status, created_at
    ) values (
      NEW.id,
      NEW.user_id,
      NEW.symbol,
      abs(NEW.quantity),
      case when NEW.quantity > 0 then 'long' else 'short' end,
      NEW.entry_price,
      coalesce(NEW.type, 'stock'),
      coalesce(NEW.status, 'open'),
      NEW.created_at
    )
    on conflict (id) do update set
      quantity = excluded.quantity,
      status = excluded.status,
      updated_at = now();
  elsif TG_OP = 'UPDATE' then
    update positions_new set
      quantity = abs(NEW.quantity),
      side = case when NEW.quantity > 0 then 'long' else 'short' end,
      status = NEW.status,
      updated_at = now()
    where id = NEW.id;
  elsif TG_OP = 'DELETE' then
    delete from positions_new where id = OLD.id;
  end if;
  return coalesce(NEW, OLD);
end;
$$ language plpgsql;

create trigger positions_shadow_write
after insert or update or delete on positions
for each row execute function shadow_write_positions();

-- Phase 3: Backfill historical data
insert into positions_new (id, user_id, symbol, quantity, side, entry_price, position_type, status, created_at)
select
  id,
  user_id,
  symbol,
  abs(quantity),
  case when quantity > 0 then 'long' else 'short' end,
  entry_price,
  coalesce(type, 'stock'),
  coalesce(status, 'open'),
  created_at
from positions
on conflict (id) do nothing;

-- Phase 4: Verify data consistency
select count(*) as old_count from positions;
select count(*) as new_count from positions_new;

-- Phase 5: Switch reads to new table (via feature flag or code deploy)
-- Phase 6: Switch writes to new table
-- Phase 7: Drop trigger and old table
```

---

## Migration Testing

### Test with Production Data
```bash
#!/bin/bash
# Script to test migration with production data

# 1. Create test database
createdb migration_test

# 2. Restore production backup (anonymized)
pg_restore -d migration_test production_anonymized.dump

# 3. Run migration
psql -d migration_test -f migrations/005_new_migration.sql

# 4. Run verification queries
psql -d migration_test -f verify_migration.sql

# 5. Test rollback
psql -d migration_test -f migrations/005_new_migration_down.sql

# 6. Cleanup
dropdb migration_test
```

### Automated Migration Tests
```typescript
// tests/migrations/005_add_status.test.ts
import { describe, it, expect, beforeAll, afterAll } from "vitest";
import { createTestDatabase, dropTestDatabase, runMigration } from "./helpers";

describe("Migration 005: Add status column", () => {
  let db: Database;

  beforeAll(async () => {
    db = await createTestDatabase();
    // Run all previous migrations
    await runMigration(db, "001");
    await runMigration(db, "002");
    await runMigration(db, "003");
    await runMigration(db, "004");
  });

  afterAll(async () => {
    await dropTestDatabase(db);
  });

  it("adds status column", async () => {
    await runMigration(db, "005");

    const columns = await db.query(`
      select column_name, data_type, is_nullable
      from information_schema.columns
      where table_name = 'positions' and column_name = 'status'
    `);

    expect(columns.rows).toHaveLength(1);
    expect(columns.rows[0].data_type).toBe("text");
    expect(columns.rows[0].is_nullable).toBe("NO");
  });

  it("backfills existing data", async () => {
    // Insert test data before migration
    await db.query(`
      insert into positions (id, user_id, symbol, quantity, entry_price)
      values ('test-1', 'user-1', 'AAPL', 100, 150.00)
    `);

    await runMigration(db, "005");

    const result = await db.query(`
      select status from positions where id = 'test-1'
    `);

    expect(result.rows[0].status).toBe("open");
  });

  it("can be rolled back", async () => {
    await runMigration(db, "005");
    await runMigration(db, "005", "down");

    const columns = await db.query(`
      select column_name
      from information_schema.columns
      where table_name = 'positions' and column_name = 'status'
    `);

    expect(columns.rows).toHaveLength(0);
  });
});
```

---

## Process
1. **Plan migration** — Identify risks, choose strategy
2. **Write migration** — Up and down scripts
3. **Test thoroughly** — Staging with production data
4. **Prepare rollback** — Test rollback works
5. **Execute safely** — Monitor, low-traffic window
6. **Verify and cleanup** — Data integrity, documentation

## Output
1. Migration SQL files
2. Rollback scripts
3. Verification queries
4. Data transformation scripts
5. Testing scripts
6. Deployment runbooks
