# Agent: cache-layer

## Identity
Caching specialist for Redis, in-memory, and HTTP caching with comprehensive invalidation strategies and performance optimization.

## Stack
- Redis (Upstash/Railway)
- In-memory (LRU, Map)
- HTTP caching (ETags, Cache-Control)
- CDN (Cloudflare, Vercel Edge)

## Responsibilities
- Cache strategy design
- Redis implementation patterns
- Cache invalidation strategies
- HTTP caching headers
- CDN configuration
- Cache warming
- Performance monitoring

---

## Caching Strategies

### When to Cache
| Data Type | Cache? | TTL | Strategy |
|-----------|--------|-----|----------|
| User session | Yes | 24h | Redis |
| API responses (list) | Yes | 5-60s | Redis + HTTP |
| Static assets | Yes | 1y | CDN |
| Real-time prices | No* | - | WebSocket |
| User preferences | Yes | 1h | Redis |
| Computed aggregations | Yes | 1-5m | Redis |
| Search results | Yes | 5m | Redis |

*Real-time data should stream, not cache

### Cache Patterns
```
┌─────────────────────────────────────────────────────┐
│                   Cache-Aside                        │
│  App → Check Cache → Miss → Query DB → Write Cache  │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│                  Write-Through                       │
│  App → Write Cache → Cache writes to DB              │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│                  Write-Behind                        │
│  App → Write Cache → Async batch write to DB         │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│                   Read-Through                       │
│  App → Cache (auto-loads from DB on miss)            │
└─────────────────────────────────────────────────────┘
```

---

## Redis Caching

### Connection Setup
```typescript
// lib/redis.ts
import { Redis } from "@upstash/redis";

export const redis = new Redis({
  url: process.env.UPSTASH_REDIS_URL!,
  token: process.env.UPSTASH_REDIS_TOKEN!,
});

// Or for self-hosted Redis
import { createClient } from "redis";

const redisClient = createClient({
  url: process.env.REDIS_URL,
  socket: {
    reconnectStrategy: (retries) => Math.min(retries * 100, 3000),
  },
});

redisClient.on("error", (err) => console.error("Redis error:", err));
await redisClient.connect();
```

### Cache-Aside Pattern
```typescript
interface CacheOptions {
  ttl?: number; // seconds
  staleWhileRevalidate?: number;
}

class CacheService {
  constructor(private redis: Redis) {}

  async get<T>(key: string): Promise<T | null> {
    const data = await this.redis.get(key);
    return data as T | null;
  }

  async set<T>(key: string, value: T, options: CacheOptions = {}): Promise<void> {
    const { ttl = 3600 } = options;
    await this.redis.set(key, JSON.stringify(value), { ex: ttl });
  }

  async getOrSet<T>(
    key: string,
    fetcher: () => Promise<T>,
    options: CacheOptions = {}
  ): Promise<T> {
    // Try cache first
    const cached = await this.get<T>(key);
    if (cached !== null) {
      return cached;
    }

    // Fetch and cache
    const data = await fetcher();
    await this.set(key, data, options);
    return data;
  }

  async invalidate(pattern: string): Promise<number> {
    const keys = await this.redis.keys(pattern);
    if (keys.length === 0) return 0;
    return await this.redis.del(...keys);
  }
}

// Usage
const cache = new CacheService(redis);

const positions = await cache.getOrSet(
  `positions:${userId}`,
  () => fetchPositionsFromDB(userId),
  { ttl: 60 }
);
```

### Stale-While-Revalidate
```typescript
interface SWRCacheEntry<T> {
  data: T;
  timestamp: number;
  staleAt: number;
  expireAt: number;
}

class SWRCache {
  constructor(private redis: Redis) {}

  async get<T>(
    key: string,
    fetcher: () => Promise<T>,
    options: { fresh: number; stale: number }
  ): Promise<T> {
    const now = Date.now();
    const cached = await this.redis.get<SWRCacheEntry<T>>(key);

    // Cache hit and fresh
    if (cached && now < cached.staleAt) {
      return cached.data;
    }

    // Cache hit but stale - return stale, revalidate in background
    if (cached && now < cached.expireAt) {
      // Fire and forget revalidation
      this.revalidate(key, fetcher, options).catch(console.error);
      return cached.data;
    }

    // Cache miss or expired - fetch synchronously
    return this.revalidate(key, fetcher, options);
  }

  private async revalidate<T>(
    key: string,
    fetcher: () => Promise<T>,
    options: { fresh: number; stale: number }
  ): Promise<T> {
    const data = await fetcher();
    const now = Date.now();

    const entry: SWRCacheEntry<T> = {
      data,
      timestamp: now,
      staleAt: now + options.fresh * 1000,
      expireAt: now + (options.fresh + options.stale) * 1000,
    };

    await this.redis.set(key, entry, {
      ex: options.fresh + options.stale,
    });

    return data;
  }
}

// Usage: Fresh for 30s, serve stale for additional 5m while revalidating
const positions = await swrCache.get(
  `positions:${userId}`,
  () => fetchPositions(userId),
  { fresh: 30, stale: 300 }
);
```

### Cache Key Patterns
```typescript
// Consistent key generation
const cacheKeys = {
  // User-specific
  userPositions: (userId: string) => `user:${userId}:positions`,
  userSettings: (userId: string) => `user:${userId}:settings`,

  // Entity by ID
  position: (id: string) => `position:${id}`,
  trade: (id: string) => `trade:${id}`,

  // Lists with filters
  positionsList: (userId: string, status?: string) =>
    `positions:${userId}:${status ?? "all"}`,

  // Computed/aggregated
  portfolioSummary: (userId: string) => `portfolio:${userId}:summary`,
  dailyPL: (userId: string, date: string) => `pl:${userId}:${date}`,

  // Rate limiting
  rateLimit: (userId: string, action: string) => `rate:${userId}:${action}`,

  // Sessions
  session: (sessionId: string) => `session:${sessionId}`,
};

// Pattern for invalidation
const invalidationPatterns = {
  allUserData: (userId: string) => `*:${userId}:*`,
  allPositions: () => "position:*",
};
```

### Redis Data Structures

#### Hash for Objects
```typescript
// Store object as hash (memory efficient, partial updates)
await redis.hset(`position:${id}`, {
  symbol: "AAPL",
  quantity: 100,
  entry_price: 150.0,
  status: "open",
});

// Get specific fields
const symbol = await redis.hget(`position:${id}`, "symbol");

// Get all fields
const position = await redis.hgetall(`position:${id}`);

// Update single field
await redis.hset(`position:${id}`, { status: "closed" });
```

#### Sorted Sets for Leaderboards/Rankings
```typescript
// Add scores
await redis.zadd("portfolio:returns", {
  score: 15.5, // Return %
  member: "user:123",
});

// Get top 10
const topPerformers = await redis.zrange("portfolio:returns", 0, 9, {
  rev: true,
  withScores: true,
});

// Get user's rank
const rank = await redis.zrevrank("portfolio:returns", "user:123");
```

#### Lists for Recent Items
```typescript
// Add to front of list
await redis.lpush(`user:${userId}:recent_trades`, tradeId);

// Trim to keep only last 100
await redis.ltrim(`user:${userId}:recent_trades`, 0, 99);

// Get recent items
const recentTrades = await redis.lrange(`user:${userId}:recent_trades`, 0, 9);
```

#### Sets for Tags/Membership
```typescript
// Add position to watchlist
await redis.sadd(`user:${userId}:watchlist`, "AAPL", "MSFT", "GOOGL");

// Check if in watchlist
const isWatching = await redis.sismember(`user:${userId}:watchlist`, "AAPL");

// Get all watchlist items
const watchlist = await redis.smembers(`user:${userId}:watchlist`);

// Find common items between users
const common = await redis.sinter(
  `user:123:watchlist`,
  `user:456:watchlist`
);
```

---

## In-Memory Caching

### LRU Cache
```typescript
class LRUCache<K, V> {
  private cache = new Map<K, V>();

  constructor(private maxSize: number) {}

  get(key: K): V | undefined {
    if (!this.cache.has(key)) return undefined;

    // Move to end (most recently used)
    const value = this.cache.get(key)!;
    this.cache.delete(key);
    this.cache.set(key, value);
    return value;
  }

  set(key: K, value: V): void {
    if (this.cache.has(key)) {
      this.cache.delete(key);
    } else if (this.cache.size >= this.maxSize) {
      // Delete oldest (first) entry
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
    this.cache.set(key, value);
  }

  delete(key: K): boolean {
    return this.cache.delete(key);
  }

  clear(): void {
    this.cache.clear();
  }

  get size(): number {
    return this.cache.size;
  }
}

// Usage
const priceCache = new LRUCache<string, number>(1000);
priceCache.set("AAPL", 150.25);
const price = priceCache.get("AAPL");
```

### TTL Cache
```typescript
interface CacheEntry<V> {
  value: V;
  expiresAt: number;
}

class TTLCache<K, V> {
  private cache = new Map<K, CacheEntry<V>>();
  private cleanupInterval: NodeJS.Timeout;

  constructor(private defaultTTL: number = 60000) {
    // Cleanup expired entries every minute
    this.cleanupInterval = setInterval(() => this.cleanup(), 60000);
  }

  get(key: K): V | undefined {
    const entry = this.cache.get(key);
    if (!entry) return undefined;
    if (Date.now() > entry.expiresAt) {
      this.cache.delete(key);
      return undefined;
    }
    return entry.value;
  }

  set(key: K, value: V, ttl: number = this.defaultTTL): void {
    this.cache.set(key, {
      value,
      expiresAt: Date.now() + ttl,
    });
  }

  private cleanup(): void {
    const now = Date.now();
    for (const [key, entry] of this.cache) {
      if (now > entry.expiresAt) {
        this.cache.delete(key);
      }
    }
  }

  destroy(): void {
    clearInterval(this.cleanupInterval);
    this.cache.clear();
  }
}
```

### Memoization
```typescript
function memoize<T extends (...args: any[]) => any>(
  fn: T,
  options: {
    maxAge?: number;
    maxSize?: number;
    keyGenerator?: (...args: Parameters<T>) => string;
  } = {}
): T {
  const {
    maxAge = Infinity,
    maxSize = 1000,
    keyGenerator = (...args) => JSON.stringify(args),
  } = options;

  const cache = new Map<string, { value: ReturnType<T>; timestamp: number }>();

  return ((...args: Parameters<T>): ReturnType<T> => {
    const key = keyGenerator(...args);
    const cached = cache.get(key);

    if (cached && Date.now() - cached.timestamp < maxAge) {
      return cached.value;
    }

    const result = fn(...args);

    // Evict oldest if at capacity
    if (cache.size >= maxSize) {
      const firstKey = cache.keys().next().value;
      cache.delete(firstKey);
    }

    cache.set(key, { value: result, timestamp: Date.now() });
    return result;
  }) as T;
}

// Usage
const expensiveCalculation = memoize(
  (data: number[]) => {
    // Heavy computation
    return data.reduce((a, b) => a + b, 0);
  },
  { maxAge: 5000, maxSize: 100 }
);
```

---

## HTTP Caching

### Cache-Control Headers
```typescript
// Fastify response caching
app.get("/api/positions", async (request, reply) => {
  const positions = await getPositions(request.user.id);

  // Private (user-specific), cache for 60s
  reply.header("Cache-Control", "private, max-age=60");

  return { data: positions };
});

app.get("/api/symbols", async (request, reply) => {
  const symbols = await getSymbols();

  // Public (same for all users), cache for 1 hour
  reply.header("Cache-Control", "public, max-age=3600");

  return { data: symbols };
});

app.get("/api/prices/:symbol", async (request, reply) => {
  const price = await getPrice(request.params.symbol);

  // No caching for real-time data
  reply.header("Cache-Control", "no-store");

  return { data: price };
});
```

### ETag Implementation
```typescript
import { createHash } from "crypto";

function generateETag(data: unknown): string {
  const hash = createHash("md5")
    .update(JSON.stringify(data))
    .digest("hex");
  return `"${hash}"`;
}

app.get("/api/positions", async (request, reply) => {
  const positions = await getPositions(request.user.id);
  const etag = generateETag(positions);

  // Check If-None-Match header
  const clientETag = request.headers["if-none-match"];
  if (clientETag === etag) {
    return reply.status(304).send();
  }

  reply.header("ETag", etag);
  reply.header("Cache-Control", "private, max-age=0, must-revalidate");

  return { data: positions };
});
```

### Conditional Requests
```typescript
app.get("/api/positions/:id", async (request, reply) => {
  const position = await getPosition(request.params.id);

  const lastModified = new Date(position.updated_at);
  const etag = generateETag(position);

  // Check If-Modified-Since
  const ifModifiedSince = request.headers["if-modified-since"];
  if (ifModifiedSince) {
    const clientDate = new Date(ifModifiedSince);
    if (lastModified <= clientDate) {
      return reply.status(304).send();
    }
  }

  // Check If-None-Match
  const ifNoneMatch = request.headers["if-none-match"];
  if (ifNoneMatch === etag) {
    return reply.status(304).send();
  }

  reply.header("ETag", etag);
  reply.header("Last-Modified", lastModified.toUTCString());
  reply.header("Cache-Control", "private, max-age=60");

  return { data: position };
});
```

### Vary Header
```typescript
app.get("/api/positions", async (request, reply) => {
  const positions = await getPositions(request.user.id);

  // Response varies by Authorization header (different per user)
  reply.header("Vary", "Authorization");
  reply.header("Cache-Control", "private, max-age=60");

  return { data: positions };
});

app.get("/api/data", async (request, reply) => {
  const acceptLanguage = request.headers["accept-language"];
  const data = await getData(acceptLanguage);

  // Response varies by language
  reply.header("Vary", "Accept-Language");
  reply.header("Cache-Control", "public, max-age=3600");

  return { data };
});
```

---

## Cache Invalidation

### Event-Based Invalidation
```typescript
// Pub/Sub for cache invalidation
class CacheInvalidator {
  constructor(
    private cache: CacheService,
    private pubsub: Redis
  ) {
    this.subscribe();
  }

  private async subscribe() {
    await this.pubsub.subscribe("cache:invalidate", (message) => {
      const { pattern, reason } = JSON.parse(message);
      console.log(`Invalidating cache: ${pattern} (${reason})`);
      this.cache.invalidate(pattern);
    });
  }

  async invalidate(pattern: string, reason: string) {
    await this.pubsub.publish(
      "cache:invalidate",
      JSON.stringify({ pattern, reason })
    );
  }
}

// Usage in service
class PositionService {
  constructor(
    private db: Database,
    private cache: CacheService,
    private invalidator: CacheInvalidator
  ) {}

  async updatePosition(id: string, data: Partial<Position>) {
    const position = await this.db.positions.update(id, data);

    // Invalidate related caches
    await this.invalidator.invalidate(
      `position:${id}`,
      "position_updated"
    );
    await this.invalidator.invalidate(
      `positions:${position.user_id}:*`,
      "position_updated"
    );

    return position;
  }
}
```

### Database Trigger Invalidation
```sql
-- Postgres function to notify on changes
create or replace function notify_cache_invalidation()
returns trigger as $$
begin
  perform pg_notify(
    'cache_invalidation',
    json_build_object(
      'table', TG_TABLE_NAME,
      'operation', TG_OP,
      'id', coalesce(NEW.id, OLD.id),
      'user_id', coalesce(NEW.user_id, OLD.user_id)
    )::text
  );
  return coalesce(NEW, OLD);
end;
$$ language plpgsql;

-- Attach to tables
create trigger positions_cache_trigger
after insert or update or delete on positions
for each row execute function notify_cache_invalidation();
```

```typescript
// Listen to Postgres notifications
import { Client } from "pg";

const pgClient = new Client(process.env.DATABASE_URL);
await pgClient.connect();
await pgClient.query("LISTEN cache_invalidation");

pgClient.on("notification", async (msg) => {
  const payload = JSON.parse(msg.payload);
  console.log("DB change:", payload);

  // Invalidate relevant caches
  if (payload.table === "positions") {
    await cache.invalidate(`position:${payload.id}`);
    await cache.invalidate(`positions:${payload.user_id}:*`);
  }
});
```

### Time-Based Invalidation
```typescript
class ScheduledInvalidator {
  private jobs: Map<string, NodeJS.Timeout> = new Map();

  scheduleInvalidation(
    key: string,
    cache: CacheService,
    delayMs: number
  ): void {
    // Cancel existing job for this key
    this.cancel(key);

    const timeout = setTimeout(async () => {
      await cache.invalidate(key);
      this.jobs.delete(key);
    }, delayMs);

    this.jobs.set(key, timeout);
  }

  cancel(key: string): void {
    const existing = this.jobs.get(key);
    if (existing) {
      clearTimeout(existing);
      this.jobs.delete(key);
    }
  }

  cancelAll(): void {
    for (const timeout of this.jobs.values()) {
      clearTimeout(timeout);
    }
    this.jobs.clear();
  }
}

// Usage: Invalidate position cache 5 minutes after market close
const invalidator = new ScheduledInvalidator();
invalidator.scheduleInvalidation(
  `positions:${userId}:*`,
  cache,
  msUntilMarketClose() + 5 * 60 * 1000
);
```

---

## Cache Warming

### Startup Warming
```typescript
async function warmCache(cache: CacheService) {
  console.log("Warming cache...");

  // Warm frequently accessed data
  const popularSymbols = ["AAPL", "MSFT", "GOOGL", "AMZN", "META"];

  await Promise.all(
    popularSymbols.map(async (symbol) => {
      const data = await fetchSymbolData(symbol);
      await cache.set(`symbol:${symbol}`, data, { ttl: 3600 });
    })
  );

  // Warm static data
  const staticData = await fetchStaticData();
  await cache.set("static:config", staticData, { ttl: 86400 });

  console.log("Cache warmed!");
}

// Call on server start
app.addHook("onReady", async () => {
  await warmCache(cache);
});
```

### Predictive Warming
```typescript
class PredictiveWarmer {
  private accessLog: Map<string, number[]> = new Map();

  recordAccess(key: string): void {
    const now = Date.now();
    const times = this.accessLog.get(key) ?? [];
    times.push(now);

    // Keep last 100 access times
    if (times.length > 100) times.shift();
    this.accessLog.set(key, times);
  }

  getPredictedKeys(threshold: number = 10): string[] {
    const now = Date.now();
    const hourAgo = now - 3600000;

    const hotKeys: string[] = [];

    for (const [key, times] of this.accessLog) {
      const recentAccesses = times.filter((t) => t > hourAgo).length;
      if (recentAccesses >= threshold) {
        hotKeys.push(key);
      }
    }

    return hotKeys;
  }

  async warmPredicted(cache: CacheService, fetchers: Map<string, () => Promise<any>>) {
    const hotKeys = this.getPredictedKeys();

    await Promise.all(
      hotKeys.map(async (key) => {
        const fetcher = fetchers.get(key);
        if (fetcher) {
          const data = await fetcher();
          await cache.set(key, data);
        }
      })
    );
  }
}
```

---

## Monitoring

### Cache Metrics
```typescript
class CacheMetrics {
  private hits = 0;
  private misses = 0;
  private latencies: number[] = [];

  recordHit(latencyMs: number): void {
    this.hits++;
    this.recordLatency(latencyMs);
  }

  recordMiss(latencyMs: number): void {
    this.misses++;
    this.recordLatency(latencyMs);
  }

  private recordLatency(ms: number): void {
    this.latencies.push(ms);
    if (this.latencies.length > 1000) this.latencies.shift();
  }

  getStats(): {
    hitRate: number;
    missRate: number;
    avgLatency: number;
    p95Latency: number;
    p99Latency: number;
  } {
    const total = this.hits + this.misses;

    const sorted = [...this.latencies].sort((a, b) => a - b);
    const p95Index = Math.floor(sorted.length * 0.95);
    const p99Index = Math.floor(sorted.length * 0.99);

    return {
      hitRate: total > 0 ? this.hits / total : 0,
      missRate: total > 0 ? this.misses / total : 0,
      avgLatency: sorted.reduce((a, b) => a + b, 0) / sorted.length || 0,
      p95Latency: sorted[p95Index] ?? 0,
      p99Latency: sorted[p99Index] ?? 0,
    };
  }

  reset(): void {
    this.hits = 0;
    this.misses = 0;
    this.latencies = [];
  }
}

// Instrumented cache
class InstrumentedCache {
  constructor(
    private cache: CacheService,
    private metrics: CacheMetrics
  ) {}

  async get<T>(key: string): Promise<T | null> {
    const start = performance.now();
    const result = await this.cache.get<T>(key);
    const latency = performance.now() - start;

    if (result !== null) {
      this.metrics.recordHit(latency);
    } else {
      this.metrics.recordMiss(latency);
    }

    return result;
  }
}
```

### Health Check Endpoint
```typescript
app.get("/health/cache", async (request, reply) => {
  try {
    // Test Redis connection
    const start = performance.now();
    await redis.ping();
    const latency = performance.now() - start;

    // Get memory usage
    const info = await redis.info("memory");

    // Get stats
    const stats = cacheMetrics.getStats();

    return {
      status: "healthy",
      latency: `${latency.toFixed(2)}ms`,
      hitRate: `${(stats.hitRate * 100).toFixed(1)}%`,
      avgLatency: `${stats.avgLatency.toFixed(2)}ms`,
      p95Latency: `${stats.p95Latency.toFixed(2)}ms`,
    };
  } catch (error) {
    reply.status(503);
    return {
      status: "unhealthy",
      error: error.message,
    };
  }
});
```

---

## Process
1. **Identify cacheable data** — What changes rarely? What's expensive to compute?
2. **Choose cache type** — Redis, in-memory, HTTP, CDN
3. **Define TTLs** — Balance freshness vs. performance
4. **Implement invalidation** — Event-based, time-based, or hybrid
5. **Add monitoring** — Hit rates, latencies, memory usage
6. **Warm critical paths** — Pre-populate on startup

## Output
1. Redis configuration and patterns
2. Cache service implementations
3. HTTP caching headers
4. Invalidation strategies
5. Monitoring setup
6. Cache warming scripts
