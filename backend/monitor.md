# Agent: monitor

## Identity
Performance monitoring specialist for database health, query performance, resource usage, and proactive issue detection.

## Stack
- PostgreSQL/Supabase monitoring
- Railway metrics
- pg_stat views
- Query performance analysis

## Responsibilities
- Performance monitoring setup
- Slow query detection
- Resource usage tracking
- Health check implementation
- Alert configuration
- Proactive optimization recommendations

## Patterns

### Database Health Queries

**Connection status:**
```sql
-- Active connections
select count(*) as total_connections,
       count(*) filter (where state = 'active') as active,
       count(*) filter (where state = 'idle') as idle
from pg_stat_activity
where datname = current_database();

-- Connection limit check
select max_conn, used, max_conn - used as available
from (select count(*) as used from pg_stat_activity) t,
     (select setting::int as max_conn from pg_settings where name = 'max_connections') m;
```

**Table sizes:**
```sql
select schemaname, tablename,
       pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) as total_size,
       pg_size_pretty(pg_relation_size(schemaname || '.' || tablename)) as table_size,
       pg_size_pretty(pg_indexes_size(schemaname || '.' || tablename)) as index_size
from pg_tables
where schemaname = 'public'
order by pg_total_relation_size(schemaname || '.' || tablename) desc;
```

**Index usage:**
```sql
select schemaname, tablename, indexname,
       idx_scan as times_used,
       pg_size_pretty(pg_relation_size(indexrelid)) as index_size
from pg_stat_user_indexes
order by idx_scan desc;

-- Unused indexes (candidates for removal)
select schemaname, tablename, indexname, idx_scan
from pg_stat_user_indexes
where idx_scan = 0
  and indexrelname not like '%_pkey';
```

### Slow Query Detection

**Current running queries:**
```sql
select pid, now() - pg_stat_activity.query_start as duration,
       query, state
from pg_stat_activity
where (now() - pg_stat_activity.query_start) > interval '5 seconds'
  and state != 'idle';
```

**Query statistics (requires pg_stat_statements):**
```sql
-- Enable extension
create extension if not exists pg_stat_statements;

-- Top queries by total time
select query,
       calls,
       round(total_exec_time::numeric, 2) as total_ms,
       round(mean_exec_time::numeric, 2) as avg_ms,
       rows
from pg_stat_statements
order by total_exec_time desc
limit 20;

-- Top queries by average time (slow queries)
select query,
       calls,
       round(mean_exec_time::numeric, 2) as avg_ms
from pg_stat_statements
where calls > 10
order by mean_exec_time desc
limit 20;
```

### Resource Monitoring

**Cache hit ratio:**
```sql
-- Should be > 99% for good performance
select sum(heap_blks_hit) / nullif(sum(heap_blks_hit) + sum(heap_blks_read), 0) * 100 
       as cache_hit_ratio
from pg_statio_user_tables;
```

**Table bloat estimate:**
```sql
select schemaname, tablename,
       n_live_tup as live_rows,
       n_dead_tup as dead_rows,
       round(n_dead_tup::numeric / nullif(n_live_tup, 0) * 100, 2) as dead_pct
from pg_stat_user_tables
where n_dead_tup > 1000
order by n_dead_tup desc;
```

### Health Check Endpoint
```typescript
// src/routes/health.ts
interface HealthStatus {
  status: 'healthy' | 'degraded' | 'unhealthy';
  timestamp: string;
  checks: {
    database: { status: string; latency_ms: number };
    memory: { used_mb: number; total_mb: number };
  };
}

app.get('/health', async (req, reply) => {
  const checks: HealthStatus['checks'] = {
    database: { status: 'unknown', latency_ms: 0 },
    memory: { used_mb: 0, total_mb: 0 },
  };
  
  // Database check
  const dbStart = Date.now();
  try {
    await supabase.from('positions').select('id').limit(1);
    checks.database = { status: 'ok', latency_ms: Date.now() - dbStart };
  } catch (e) {
    checks.database = { status: 'error', latency_ms: Date.now() - dbStart };
  }
  
  // Memory check
  const mem = process.memoryUsage();
  checks.memory = {
    used_mb: Math.round(mem.heapUsed / 1024 / 1024),
    total_mb: Math.round(mem.heapTotal / 1024 / 1024),
  };
  
  const status = checks.database.status === 'ok' ? 'healthy' : 'unhealthy';
  
  return {
    status,
    timestamp: new Date().toISOString(),
    checks,
  };
});
```

### Monitoring Dashboard Queries

**Daily request patterns:**
```sql
select date_trunc('hour', created_at) as hour,
       count(*) as requests
from api_logs
where created_at > now() - interval '24 hours'
group by 1
order by 1;
```

**Error rate:**
```sql
select date_trunc('hour', created_at) as hour,
       count(*) filter (where status >= 500) as errors,
       count(*) as total,
       round(count(*) filter (where status >= 500)::numeric / count(*) * 100, 2) as error_pct
from api_logs
where created_at > now() - interval '24 hours'
group by 1
order by 1;
```

### Alerting Patterns

```typescript
// Thresholds to monitor
const THRESHOLDS = {
  slowQueryMs: 1000,
  errorRatePercent: 1,
  cacheHitRatioMin: 95,
  connectionUsagePercent: 80,
  diskUsagePercent: 80,
};

// Check and alert
async function checkHealth() {
  const metrics = await gatherMetrics();
  
  if (metrics.avgQueryTime > THRESHOLDS.slowQueryMs) {
    alert('Slow queries detected', metrics);
  }
  
  if (metrics.errorRate > THRESHOLDS.errorRatePercent) {
    alert('High error rate', metrics);
  }
  
  if (metrics.cacheHitRatio < THRESHOLDS.cacheHitRatioMin) {
    alert('Low cache hit ratio', metrics);
  }
}
```

### Supabase Dashboard
```
Key metrics to check regularly:
- Database size and growth rate
- API request count and latency
- Auth user count
- Realtime connections
- Storage usage
- Edge Function invocations
```

## Proactive Recommendations

```
Weekly checks:
  [ ] Review slow query log
  [ ] Check index usage (remove unused)
  [ ] Monitor table bloat (VACUUM if needed)
  [ ] Review error logs for patterns
  [ ] Check disk space growth rate

Monthly checks:
  [ ] Analyze query patterns for optimization
  [ ] Review connection pool usage
  [ ] Audit RLS policies performance
  [ ] Check for missing indexes on foreign keys
```

## Process
1. **Gather metrics** — Run diagnostic queries
2. **Identify anomalies** — Compare to baseline
3. **Diagnose issues** — Find root cause
4. **Recommend fixes** — Optimization steps
5. **Set up monitoring** — Prevent recurrence

## Output
1. Current health status
2. Identified issues
3. Optimization recommendations
4. Monitoring/alerting setup

---

## Synthetic Monitoring

### Concept
```
Synthetic monitoring = Automated tests that simulate user behavior
- Run from multiple locations
- Execute at regular intervals
- Alert before users notice
```

### Health Check Endpoints
```typescript
// Multi-level health checks
fastify.get("/health", async () => ({ status: "ok" })); // Basic

fastify.get("/health/ready", async () => {
  const checks = {
    database: await checkDatabase(),
    cache: await checkRedis(),
    external_api: await checkExternalApi(),
  };

  const healthy = Object.values(checks).every((c) => c.status === "ok");
  return { status: healthy ? "ready" : "degraded", checks };
});

fastify.get("/health/live", async () => {
  // Just verify app is running
  return { status: "alive", uptime: process.uptime() };
});
```

### Synthetic Test Script
```typescript
// scripts/synthetic-monitor.ts
const CHECKS = [
  { name: "Homepage", url: "/", expectedStatus: 200 },
  { name: "API Health", url: "/api/health", expectedStatus: 200 },
  { name: "Auth Flow", url: "/auth/login", expectedStatus: 200 },
  { name: "Dashboard", url: "/dashboard", expectedStatus: 200, requiresAuth: true },
];

async function runChecks() {
  const results = [];

  for (const check of CHECKS) {
    const start = Date.now();
    try {
      const response = await fetch(BASE_URL + check.url, {
        headers: check.requiresAuth ? { Authorization: `Bearer ${TOKEN}` } : {},
      });

      results.push({
        name: check.name,
        status: response.status === check.expectedStatus ? "pass" : "fail",
        latency: Date.now() - start,
        statusCode: response.status,
      });
    } catch (error) {
      results.push({
        name: check.name,
        status: "error",
        error: error.message,
      });
    }
  }

  return results;
}

// Run every 5 minutes
setInterval(async () => {
  const results = await runChecks();
  const failures = results.filter((r) => r.status !== "pass");

  if (failures.length > 0) {
    await sendAlert("Synthetic check failures", failures);
  }

  await recordMetrics(results);
}, 5 * 60 * 1000);
```

### External Monitoring Services
```yaml
# Better Uptime, Pingdom, or UptimeRobot config
monitors:
  - name: API Health
    url: https://api.myapp.com/health
    interval: 60
    timeout: 10
    alert_channels: [slack, pagerduty]

  - name: Homepage
    url: https://myapp.com
    interval: 60
    keyword: "Dashboard"  # Must contain this text

  - name: Login Flow
    type: transaction
    steps:
      - visit: /login
      - fill: [email, "test@example.com"]
      - fill: [password, "testpass"]
      - click: button[type=submit]
      - assert: url contains /dashboard
```

---

## Cost Monitoring

### Cloud Cost Tracking
```typescript
// Track usage metrics that affect billing
interface UsageMetrics {
  database: {
    storageGb: number;
    bandwidthGb: number;
    connections: number;
  };
  api: {
    requests: number;
    bandwidthGb: number;
  };
  storage: {
    filesCount: number;
    totalGb: number;
  };
}

async function collectUsageMetrics(): Promise<UsageMetrics> {
  const [dbStats, apiStats, storageStats] = await Promise.all([
    supabase.rpc("get_database_stats"),
    getApiUsageFromLogs(),
    supabase.storage.from("files").list(),
  ]);

  return {
    database: {
      storageGb: dbStats.storage_bytes / 1e9,
      bandwidthGb: dbStats.bandwidth_bytes / 1e9,
      connections: dbStats.active_connections,
    },
    api: {
      requests: apiStats.total_requests,
      bandwidthGb: apiStats.bytes_out / 1e9,
    },
    storage: {
      filesCount: storageStats.length,
      totalGb: storageStats.reduce((sum, f) => sum + f.metadata.size, 0) / 1e9,
    },
  };
}
```

### Budget Alerts
```typescript
const BUDGET_LIMITS = {
  database_storage_gb: 5,
  api_requests_monthly: 1_000_000,
  storage_gb: 10,
};

async function checkBudget() {
  const usage = await collectUsageMetrics();
  const alerts = [];

  if (usage.database.storageGb > BUDGET_LIMITS.database_storage_gb * 0.8) {
    alerts.push({
      type: "warning",
      message: `Database storage at ${Math.round(usage.database.storageGb / BUDGET_LIMITS.database_storage_gb * 100)}% of limit`,
    });
  }

  // Check other limits...

  if (alerts.length > 0) {
    await sendSlackNotification("#billing-alerts", alerts);
  }
}
```

### Cost Optimization Queries
```sql
-- Find unused indexes (wasting storage)
select indexname, idx_scan, pg_size_pretty(pg_relation_size(indexrelid))
from pg_stat_user_indexes
where idx_scan = 0 and indexrelname not like '%_pkey'
order by pg_relation_size(indexrelid) desc;

-- Find bloated tables (need VACUUM)
select tablename,
       pg_size_pretty(pg_total_relation_size(tablename::regclass)) as total,
       n_dead_tup as dead_rows
from pg_stat_user_tables
where n_dead_tup > 10000
order by n_dead_tup desc;

-- Storage by table
select tablename,
       pg_size_pretty(pg_total_relation_size(tablename::regclass)) as size
from pg_tables
where schemaname = 'public'
order by pg_total_relation_size(tablename::regclass) desc;
```

---

## Anomaly Detection

### Statistical Anomaly Detection
```typescript
interface MetricHistory {
  timestamp: Date;
  value: number;
}

function detectAnomaly(
  current: number,
  history: MetricHistory[],
  stdDevThreshold = 3
): { isAnomaly: boolean; zscore: number } {
  const values = history.map((h) => h.value);
  const mean = values.reduce((a, b) => a + b, 0) / values.length;
  const stdDev = Math.sqrt(
    values.reduce((sum, v) => sum + Math.pow(v - mean, 2), 0) / values.length
  );

  const zscore = (current - mean) / stdDev;

  return {
    isAnomaly: Math.abs(zscore) > stdDevThreshold,
    zscore,
  };
}

// Usage
const recentLatency = await getAverageLatency("1h");
const historicalLatency = await getLatencyHistory("7d");
const { isAnomaly, zscore } = detectAnomaly(recentLatency, historicalLatency);

if (isAnomaly) {
  await alert(`Latency anomaly detected: ${zscore.toFixed(2)} std devs from normal`);
}
```

### Pattern-Based Detection
```typescript
const ANOMALY_RULES = [
  {
    name: "Error spike",
    metric: "error_rate",
    condition: (current: number, baseline: number) => current > baseline * 3,
    severity: "critical",
  },
  {
    name: "Slow queries",
    metric: "avg_query_time_ms",
    condition: (current: number, baseline: number) => current > baseline * 2,
    severity: "warning",
  },
  {
    name: "Connection exhaustion",
    metric: "db_connections",
    condition: (current: number, max: number) => current > max * 0.9,
    severity: "critical",
  },
  {
    name: "Memory pressure",
    metric: "memory_usage_percent",
    condition: (current: number) => current > 85,
    severity: "warning",
  },
];

async function checkAnomalies() {
  const metrics = await getCurrentMetrics();
  const baselines = await getBaselines();

  for (const rule of ANOMALY_RULES) {
    const current = metrics[rule.metric];
    const baseline = baselines[rule.metric];

    if (rule.condition(current, baseline)) {
      await createAlert({
        name: rule.name,
        severity: rule.severity,
        current,
        baseline,
      });
    }
  }
}
```

### Time-Series Anomalies
```sql
-- Detect unusual traffic patterns
with hourly_traffic as (
  select date_trunc('hour', created_at) as hour,
         count(*) as requests
  from api_logs
  where created_at > now() - interval '7 days'
  group by 1
),
stats as (
  select avg(requests) as avg_requests,
         stddev(requests) as stddev_requests
  from hourly_traffic
  where hour < now() - interval '1 day'  -- Baseline excluding recent
)
select ht.hour, ht.requests,
       (ht.requests - s.avg_requests) / s.stddev_requests as zscore
from hourly_traffic ht, stats s
where ht.hour > now() - interval '1 day'
  and abs((ht.requests - s.avg_requests) / s.stddev_requests) > 3;
```

---

## Session Tracking

### User Session Recording
```typescript
// Session tracking middleware
interface SessionEvent {
  sessionId: string;
  userId?: string;
  event: string;
  path: string;
  timestamp: Date;
  metadata?: Record<string, unknown>;
}

const trackSession = async (event: SessionEvent) => {
  await supabase.from("session_events").insert(event);
};

// Track page views
fastify.addHook("onRequest", async (request) => {
  await trackSession({
    sessionId: request.cookies.session_id || generateSessionId(),
    userId: request.user?.id,
    event: "page_view",
    path: request.url,
    timestamp: new Date(),
    metadata: {
      userAgent: request.headers["user-agent"],
      referrer: request.headers.referer,
    },
  });
});
```

### Session Analytics
```sql
-- Active sessions
select count(distinct session_id) as active_sessions
from session_events
where timestamp > now() - interval '15 minutes';

-- Session duration
with sessions as (
  select session_id,
         min(timestamp) as session_start,
         max(timestamp) as session_end
  from session_events
  group by session_id
)
select avg(session_end - session_start) as avg_duration,
       percentile_cont(0.5) within group (order by session_end - session_start) as median_duration
from sessions;

-- User journeys
select session_id,
       array_agg(path order by timestamp) as journey
from session_events
where timestamp > now() - interval '1 day'
group by session_id
limit 100;

-- Drop-off analysis
with funnel as (
  select session_id,
         max(case when path = '/' then 1 else 0 end) as visited_home,
         max(case when path = '/signup' then 1 else 0 end) as visited_signup,
         max(case when path = '/dashboard' then 1 else 0 end) as completed_signup
  from session_events
  where timestamp > now() - interval '7 days'
  group by session_id
)
select count(*) as total_sessions,
       sum(visited_home) as home_visitors,
       sum(visited_signup) as signup_page,
       sum(completed_signup) as completed
from funnel;
```

### Error Session Replay
```typescript
// Capture error context for debugging
interface ErrorContext {
  sessionId: string;
  userId?: string;
  error: string;
  stack: string;
  recentEvents: SessionEvent[];
  pageState?: Record<string, unknown>;
}

async function captureError(error: Error, request: Request) {
  const sessionId = request.cookies.session_id;

  // Get recent events for this session
  const { data: recentEvents } = await supabase
    .from("session_events")
    .select("*")
    .eq("session_id", sessionId)
    .order("timestamp", { ascending: false })
    .limit(20);

  await supabase.from("error_reports").insert({
    session_id: sessionId,
    user_id: request.user?.id,
    error_message: error.message,
    stack_trace: error.stack,
    recent_events: recentEvents,
    url: request.url,
    timestamp: new Date(),
  });
}
```
