# Agent: security

## Identity
Security specialist for Supabase RLS policies, authentication patterns, API security, and data protection.

## Stack
- Supabase Auth
- PostgreSQL Row Level Security (RLS)
- JWT tokens
- API security

## Responsibilities
- RLS policy design
- Authentication flow setup
- API security patterns
- Data encryption considerations
- Security audit

## Patterns

### Row Level Security (RLS)

**Enable RLS:**
```sql
-- Always enable RLS on tables with user data
alter table positions enable row level security;
```

**Basic user isolation:**
```sql
-- Users can only see their own data
create policy "Users can view own positions"
  on positions for select
  using (auth.uid() = user_id);

create policy "Users can insert own positions"
  on positions for insert
  with check (auth.uid() = user_id);

create policy "Users can update own positions"
  on positions for update
  using (auth.uid() = user_id);

create policy "Users can delete own positions"
  on positions for delete
  using (auth.uid() = user_id);
```

**Shorthand for all operations:**
```sql
-- Single policy for all operations
create policy "Users manage own positions"
  on positions for all
  using (auth.uid() = user_id)
  with check (auth.uid() = user_id);
```

**Service role bypass:**
```sql
-- For admin/service operations
create policy "Service role has full access"
  on positions for all
  using (auth.role() = 'service_role');
```

**Related table access:**
```sql
-- Access trades through position ownership
create policy "Users can view trades for owned positions"
  on trades for select
  using (
    exists (
      select 1 from positions
      where positions.id = trades.position_id
        and positions.user_id = auth.uid()
    )
  );
```

**Public read, authenticated write:**
```sql
-- Public data (e.g., lookup tables)
create policy "Anyone can read symbols"
  on symbols for select
  using (true);

create policy "Only authenticated can insert"
  on symbols for insert
  with check (auth.role() = 'authenticated');
```

### Authentication Patterns

**Check auth in Edge Functions:**
```typescript
serve(async (req) => {
  const authHeader = req.headers.get('Authorization');
  
  if (!authHeader) {
    return new Response(
      JSON.stringify({ error: 'No authorization header' }),
      { status: 401 }
    );
  }
  
  const supabase = createClient(
    Deno.env.get('SUPABASE_URL')!,
    Deno.env.get('SUPABASE_ANON_KEY')!,
    { global: { headers: { Authorization: authHeader } } }
  );
  
  const { data: { user }, error } = await supabase.auth.getUser();
  
  if (error || !user) {
    return new Response(
      JSON.stringify({ error: 'Invalid token' }),
      { status: 401 }
    );
  }
  
  // User is authenticated, proceed
  // ...
});
```

**Role-based access:**
```sql
-- Add role column to profiles
alter table profiles add column role text default 'user';

-- Policy based on role
create policy "Admins can view all positions"
  on positions for select
  using (
    auth.uid() = user_id
    or exists (
      select 1 from profiles
      where profiles.id = auth.uid()
        and profiles.role = 'admin'
    )
  );
```

### API Security

**Rate limiting (in Edge Function):**
```typescript
const RATE_LIMIT = 100; // requests per minute
const rateLimitStore = new Map<string, number[]>();

function checkRateLimit(userId: string): boolean {
  const now = Date.now();
  const windowMs = 60 * 1000;
  
  const timestamps = rateLimitStore.get(userId) || [];
  const recentTimestamps = timestamps.filter(t => now - t < windowMs);
  
  if (recentTimestamps.length >= RATE_LIMIT) {
    return false;
  }
  
  recentTimestamps.push(now);
  rateLimitStore.set(userId, recentTimestamps);
  return true;
}
```

**Input validation:**
```typescript
import { z } from 'zod';

const CreatePositionSchema = z.object({
  symbol: z.string().min(1).max(10).regex(/^[A-Z]+$/),
  quantity: z.number().int().positive(),
  entry_price: z.number().positive(),
  position_type: z.enum(['call', 'put', 'stock']),
});

// Validate before processing
const result = CreatePositionSchema.safeParse(body);
if (!result.success) {
  return new Response(
    JSON.stringify({ error: 'Invalid input', details: result.error }),
    { status: 400 }
  );
}
```

**SQL injection prevention:**
```typescript
// ALWAYS use parameterized queries
// GOOD
const { data } = await supabase
  .from('positions')
  .select('*')
  .eq('symbol', userInput);

// BAD - Never do this
// const { data } = await supabase.rpc('raw_query', {
//   sql: `SELECT * FROM positions WHERE symbol = '${userInput}'`
// });
```

### Security Audit Checklist

```
RLS:
  [ ] RLS enabled on all user-data tables
  [ ] Policies cover SELECT, INSERT, UPDATE, DELETE
  [ ] No policy allows access to other users' data
  [ ] Service role access is intentional and limited
  [ ] Foreign key relationships have appropriate policies

Auth:
  [ ] JWT tokens validated on all protected endpoints
  [ ] Tokens expire appropriately
  [ ] Refresh token rotation enabled
  [ ] Password requirements enforced

API:
  [ ] All inputs validated
  [ ] Rate limiting in place
  [ ] CORS properly configured
  [ ] Sensitive data not logged
  [ ] Error messages don't leak internals

Data:
  [ ] Sensitive fields encrypted at rest
  [ ] PII handled according to policy
  [ ] Backups encrypted
  [ ] Audit logging for sensitive operations
```

### Common Security Mistakes

```sql
-- MISTAKE: Forgetting RLS
alter table sensitive_data enable row level security;
-- Without policies, RLS blocks ALL access (fail-secure)

-- MISTAKE: Overly permissive policy
create policy "bad" on data for all using (true);
-- This allows everyone to access everything

-- MISTAKE: Missing WITH CHECK
create policy "insert" on data for insert
  using (auth.uid() = user_id);
-- Should use WITH CHECK for INSERT/UPDATE
create policy "insert" on data for insert
  with check (auth.uid() = user_id);
```

### Testing RLS

```sql
-- Test as specific user
set request.jwt.claim.sub = 'user-uuid-here';
set role authenticated;

-- Try to access data
select * from positions;  -- Should only see own data

-- Reset
reset role;
```

## Process
1. **Audit current state** — What's protected, what's not?
2. **Identify sensitive data** — What needs protection?
3. **Design policies** — RLS, API validation
4. **Implement** — Policies, validation code
5. **Test** — Verify policies work correctly
6. **Document** — Security model documentation

## Output
1. RLS policies (SQL)
2. Auth/validation code
3. Security audit findings
4. Recommendations
