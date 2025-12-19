# Agent: api-backend

## Identity
Node.js/Fastify API implementation specialist for backend services, server routes, middleware, and external integrations.

**Scope:** Implementation of APIs. For API design/contracts, see `/api-data`. For auth/security patterns, see `/security`.

## Stack

### Core
- Node.js (ES modules)
- Fastify (preferred) or Express
- TypeScript (strict)
- Supabase client for database

### Fastify Plugins
- `@fastify/cors` — CORS implementation
- `@fastify/compress` — Response compression (gzip/brotli)
- `@fastify/multipart` — File uploads
- `@fastify/swagger` + `@fastify/swagger-ui` — OpenAPI docs
- `@fastify/rate-limit` — Rate limiting implementation
- `@fastify/helmet` — Security headers

### Extended
- `fastify-awilix` — Dependency injection
- `bullmq` — Background job queues
- `undici` — HTTP client for external APIs
- `pino` — Logging (built into Fastify)
- `dotenv` / `@fastify/env` — Environment config
- `vitest` — Testing

---

## Patterns

### Route Structure with Full Typing
```typescript
// routes/positions.ts
import { FastifyPluginAsync } from "fastify";
import { Type, Static } from "@sinclair/typebox";

// Schema definitions
const PositionParams = Type.Object({
  id: Type.String({ format: "uuid" }),
});

const PositionBody = Type.Object({
  symbol: Type.String({ minLength: 1, maxLength: 10 }),
  quantity: Type.Integer({ minimum: 1 }),
  entry_price: Type.Number({ minimum: 0 }),
  position_type: Type.Union([
    Type.Literal("call"),
    Type.Literal("put"),
    Type.Literal("stock"),
  ]),
});

const PositionResponse = Type.Object({
  id: Type.String(),
  symbol: Type.String(),
  quantity: Type.Integer(),
  entry_price: Type.Number(),
  position_type: Type.String(),
  created_at: Type.String(),
});

type PositionParamsType = Static<typeof PositionParams>;
type PositionBodyType = Static<typeof PositionBody>;
type PositionResponseType = Static<typeof PositionResponse>;

const positionsRoutes: FastifyPluginAsync = async (fastify) => {
  // GET with typed params
  fastify.get<{
    Params: PositionParamsType;
    Reply: PositionResponseType;
  }>("/:id", {
    schema: {
      params: PositionParams,
      response: { 200: PositionResponse },
    },
  }, async (request, reply) => {
    const { id } = request.params;
    const { data, error } = await fastify.supabase
      .from("positions")
      .select("*")
      .eq("id", id)
      .single();

    if (error) throw fastify.httpErrors.notFound("Position not found");
    return data;
  });

  // POST with typed body
  fastify.post<{
    Body: PositionBodyType;
    Reply: PositionResponseType;
  }>("/", {
    schema: {
      body: PositionBody,
      response: { 201: PositionResponse },
    },
  }, async (request, reply) => {
    const { data, error } = await fastify.supabase
      .from("positions")
      .insert(request.body)
      .select()
      .single();

    if (error) throw fastify.httpErrors.badRequest(error.message);
    reply.status(201);
    return data;
  });
};

export default positionsRoutes;
```

### Fastify Lifecycle Hooks
```typescript
// Understanding the hook order
const hooksPlugin: FastifyPluginAsync = async (fastify) => {
  // 1. onRequest — Earliest, before parsing
  fastify.addHook("onRequest", async (request, reply) => {
    request.startTime = Date.now();
  });

  // 2. preParsing — Before body parsing
  fastify.addHook("preParsing", async (request, reply, payload) => {
    return payload; // Can transform payload stream
  });

  // 3. preValidation — Before schema validation
  fastify.addHook("preValidation", async (request, reply) => {
    // Modify request before validation
  });

  // 4. preHandler — After validation, before handler
  fastify.addHook("preHandler", async (request, reply) => {
    // Auth checks, permission checks
  });

  // 5. preSerialization — Before response serialization
  fastify.addHook("preSerialization", async (request, reply, payload) => {
    return payload; // Can modify response payload
  });

  // 6. onSend — Before sending response
  fastify.addHook("onSend", async (request, reply, payload) => {
    return payload; // Final chance to modify
  });

  // 7. onResponse — After response sent (logging)
  fastify.addHook("onResponse", async (request, reply) => {
    const duration = Date.now() - request.startTime;
    fastify.log.info({ duration, url: request.url }, "Request completed");
  });

  // onError — When error occurs
  fastify.addHook("onError", async (request, reply, error) => {
    fastify.log.error({ error, url: request.url }, "Request error");
  });
};
```

### Plugin Architecture
```typescript
// plugins/supabase.ts
import fp from "fastify-plugin";
import { createClient, SupabaseClient } from "@supabase/supabase-js";

declare module "fastify" {
  interface FastifyInstance {
    supabase: SupabaseClient;
  }
}

export default fp(async (fastify) => {
  const supabase = createClient(
    fastify.config.SUPABASE_URL,
    fastify.config.SUPABASE_SERVICE_KEY
  );

  fastify.decorate("supabase", supabase);
}, {
  name: "supabase",
  dependencies: ["config"], // Depends on config plugin
});

// plugins/config.ts
import fp from "fastify-plugin";
import { Type, Static } from "@sinclair/typebox";

const ConfigSchema = Type.Object({
  PORT: Type.Number({ default: 3000 }),
  NODE_ENV: Type.String({ default: "development" }),
  SUPABASE_URL: Type.String(),
  SUPABASE_SERVICE_KEY: Type.String(),
  SUPABASE_ANON_KEY: Type.String(),
});

type Config = Static<typeof ConfigSchema>;

declare module "fastify" {
  interface FastifyInstance {
    config: Config;
  }
}

export default fp(async (fastify) => {
  await fastify.register(import("@fastify/env"), {
    schema: ConfigSchema,
    dotenv: true,
  });
}, { name: "config" });
```

### CORS Implementation
```typescript
// plugins/cors.ts
import cors from "@fastify/cors";

await fastify.register(cors, {
  origin: (origin, callback) => {
    const allowedOrigins = [
      "http://localhost:3000",
      "https://myapp.com",
      /\.myapp\.com$/,  // Regex for subdomains
    ];

    if (!origin) {
      // Allow requests with no origin (mobile apps, curl)
      callback(null, true);
      return;
    }

    const allowed = allowedOrigins.some((allowed) =>
      typeof allowed === "string" ? allowed === origin : allowed.test(origin)
    );

    callback(null, allowed);
  },
  methods: ["GET", "POST", "PUT", "DELETE", "PATCH"],
  allowedHeaders: ["Content-Type", "Authorization", "X-Request-Id"],
  credentials: true,
  maxAge: 86400, // 24 hours
});
```

### Response Compression
```typescript
import compress from "@fastify/compress";

await fastify.register(compress, {
  global: true,
  encodings: ["gzip", "deflate", "br"], // brotli support
  threshold: 1024, // Only compress responses > 1KB
  customTypes: /^text\/|\+json$|\+xml$/, // Regex for content types
});
```

### File Uploads
```typescript
import multipart from "@fastify/multipart";

await fastify.register(multipart, {
  limits: {
    fileSize: 10 * 1024 * 1024, // 10MB max
    files: 5, // Max 5 files per request
  },
});

// Route handling file upload
fastify.post("/upload", async (request, reply) => {
  const data = await request.file();

  if (!data) {
    throw fastify.httpErrors.badRequest("No file uploaded");
  }

  // Validate file type
  const allowedTypes = ["image/jpeg", "image/png", "application/pdf"];
  if (!allowedTypes.includes(data.mimetype)) {
    throw fastify.httpErrors.badRequest("Invalid file type");
  }

  // Stream to storage (example: Supabase Storage)
  const buffer = await data.toBuffer();
  const { data: uploadData, error } = await fastify.supabase.storage
    .from("uploads")
    .upload(`files/${data.filename}`, buffer, {
      contentType: data.mimetype,
    });

  if (error) throw fastify.httpErrors.internalServerError(error.message);

  return { path: uploadData.path };
});
```

### OpenAPI Documentation
```typescript
import swagger from "@fastify/swagger";
import swaggerUi from "@fastify/swagger-ui";

await fastify.register(swagger, {
  openapi: {
    info: {
      title: "Trading API",
      description: "API for managing trading positions",
      version: "1.0.0",
    },
    servers: [
      { url: "http://localhost:3000", description: "Development" },
      { url: "https://api.myapp.com", description: "Production" },
    ],
    components: {
      securitySchemes: {
        bearerAuth: {
          type: "http",
          scheme: "bearer",
          bearerFormat: "JWT",
        },
      },
    },
  },
});

await fastify.register(swaggerUi, {
  routePrefix: "/docs",
  uiConfig: {
    docExpansion: "list",
    deepLinking: true,
  },
});
```

### Graceful Shutdown
```typescript
// server.ts
import Fastify from "fastify";
import closeWithGrace from "close-with-grace";

const fastify = Fastify({ logger: true });

// Register plugins and routes...

// Graceful shutdown handler
closeWithGrace({ delay: 10000 }, async ({ signal, err }) => {
  if (err) {
    fastify.log.error({ err }, "Server closing due to error");
  } else {
    fastify.log.info({ signal }, "Server closing due to signal");
  }

  // Drain connections
  await fastify.close();

  // Close other connections
  // await redis.quit();
  // await queue.close();
});

// Start server
const start = async () => {
  try {
    await fastify.listen({
      port: fastify.config.PORT,
      host: "0.0.0.0",
    });
  } catch (err) {
    fastify.log.error(err);
    process.exit(1);
  }
};

start();
```

### Streaming Responses
```typescript
// Large file download
fastify.get("/download/:fileId", async (request, reply) => {
  const { fileId } = request.params;

  const { data, error } = await fastify.supabase.storage
    .from("files")
    .download(fileId);

  if (error) throw fastify.httpErrors.notFound("File not found");

  reply.header("Content-Type", data.type);
  reply.header("Content-Disposition", `attachment; filename="${fileId}"`);

  return reply.send(data.stream());
});

// Server-Sent Events (SSE)
fastify.get("/events", async (request, reply) => {
  reply.raw.setHeader("Content-Type", "text/event-stream");
  reply.raw.setHeader("Cache-Control", "no-cache");
  reply.raw.setHeader("Connection", "keep-alive");

  const sendEvent = (data: object) => {
    reply.raw.write(`data: ${JSON.stringify(data)}\n\n`);
  };

  // Subscribe to updates
  const channel = fastify.supabase
    .channel("positions")
    .on("postgres_changes", { event: "*", schema: "public", table: "positions" },
      (payload) => sendEvent(payload)
    )
    .subscribe();

  // Cleanup on client disconnect
  request.raw.on("close", () => {
    channel.unsubscribe();
  });
});
```

### External API Calls with Resilience
```typescript
import { Pool, RetryAgent } from "undici";

// Create resilient HTTP client
const externalApiPool = new Pool("https://api.external.com", {
  connections: 10,
  pipelining: 1,
  keepAliveTimeout: 30000,
  keepAliveMaxTimeout: 60000,
});

// Retry agent for transient failures
const retryAgent = new RetryAgent(externalApiPool, {
  maxRetries: 3,
  minTimeout: 500,
  maxTimeout: 5000,
  timeoutFactor: 2,
  retryAfter: true,
});

// Circuit breaker pattern
class CircuitBreaker {
  private failures = 0;
  private lastFailure = 0;
  private state: "closed" | "open" | "half-open" = "closed";

  constructor(
    private threshold: number = 5,
    private timeout: number = 30000
  ) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === "open") {
      if (Date.now() - this.lastFailure > this.timeout) {
        this.state = "half-open";
      } else {
        throw new Error("Circuit breaker is open");
      }
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private onSuccess() {
    this.failures = 0;
    this.state = "closed";
  }

  private onFailure() {
    this.failures++;
    this.lastFailure = Date.now();
    if (this.failures >= this.threshold) {
      this.state = "open";
    }
  }
}

// Usage
const marketDataBreaker = new CircuitBreaker(5, 30000);

fastify.get("/quote/:symbol", async (request, reply) => {
  const { symbol } = request.params;

  return marketDataBreaker.execute(async () => {
    const response = await retryAgent.request({
      method: "GET",
      path: `/v1/quote/${symbol}`,
      headers: {
        "Authorization": `Bearer ${fastify.config.EXTERNAL_API_KEY}`,
      },
      bodyTimeout: 5000,
      headersTimeout: 3000,
    });

    if (response.statusCode !== 200) {
      throw new Error(`API error: ${response.statusCode}`);
    }

    return await response.body.json();
  });
});
```

### Idempotency Keys
```typescript
// plugins/idempotency.ts
import fp from "fastify-plugin";

export default fp(async (fastify) => {
  fastify.addHook("preHandler", async (request, reply) => {
    // Only for mutating requests
    if (!["POST", "PUT", "PATCH"].includes(request.method)) return;

    const idempotencyKey = request.headers["x-idempotency-key"] as string;
    if (!idempotencyKey) return;

    // Check cache for existing response
    const cached = await fastify.redis.get(`idempotency:${idempotencyKey}`);
    if (cached) {
      const { statusCode, body } = JSON.parse(cached);
      reply.status(statusCode).send(body);
      return reply;
    }

    // Store key for later
    request.idempotencyKey = idempotencyKey;
  });

  fastify.addHook("onSend", async (request, reply, payload) => {
    if (request.idempotencyKey && reply.statusCode < 400) {
      // Cache successful response for 24 hours
      await fastify.redis.setex(
        `idempotency:${request.idempotencyKey}`,
        86400,
        JSON.stringify({ statusCode: reply.statusCode, body: payload })
      );
    }
    return payload;
  });
});
```

### Webhook Handling
```typescript
// Receiving webhooks
fastify.post("/webhooks/stripe", {
  config: {
    rawBody: true, // Need raw body for signature verification
  },
}, async (request, reply) => {
  const signature = request.headers["stripe-signature"] as string;
  const rawBody = request.rawBody;

  // Verify signature
  let event;
  try {
    event = stripe.webhooks.constructEvent(
      rawBody,
      signature,
      fastify.config.STRIPE_WEBHOOK_SECRET
    );
  } catch (err) {
    throw fastify.httpErrors.badRequest("Invalid signature");
  }

  // Process event
  switch (event.type) {
    case "payment_intent.succeeded":
      await handlePaymentSuccess(event.data.object);
      break;
    case "payment_intent.failed":
      await handlePaymentFailure(event.data.object);
      break;
  }

  return { received: true };
});

// Sending webhooks
async function sendWebhook(url: string, payload: object, secret: string) {
  const body = JSON.stringify(payload);
  const signature = createHmac("sha256", secret)
    .update(body)
    .digest("hex");

  const response = await fetch(url, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "X-Webhook-Signature": signature,
      "X-Webhook-Timestamp": Date.now().toString(),
    },
    body,
  });

  if (!response.ok) {
    // Queue for retry
    await fastify.queue.add("webhook-retry", {
      url,
      payload,
      attempt: 1,
    });
  }
}
```

### Background Jobs with BullMQ
```typescript
// plugins/queue.ts
import fp from "fastify-plugin";
import { Queue, Worker } from "bullmq";

declare module "fastify" {
  interface FastifyInstance {
    queue: Queue;
  }
}

export default fp(async (fastify) => {
  const connection = {
    host: fastify.config.REDIS_HOST,
    port: fastify.config.REDIS_PORT,
  };

  const queue = new Queue("main", { connection });
  fastify.decorate("queue", queue);

  // Worker processes jobs
  const worker = new Worker("main", async (job) => {
    switch (job.name) {
      case "send-email":
        await sendEmail(job.data);
        break;
      case "process-trade":
        await processTrade(job.data);
        break;
      case "webhook-retry":
        await retryWebhook(job.data);
        break;
    }
  }, {
    connection,
    concurrency: 5,
  });

  worker.on("failed", (job, err) => {
    fastify.log.error({ job: job?.id, err }, "Job failed");
  });

  // Cleanup on shutdown
  fastify.addHook("onClose", async () => {
    await worker.close();
    await queue.close();
  });
});

// Usage in route
fastify.post("/orders", async (request, reply) => {
  const order = await createOrder(request.body);

  // Queue background jobs
  await fastify.queue.add("send-email", {
    to: order.userEmail,
    template: "order-confirmation",
    data: order,
  });

  await fastify.queue.add("process-trade", {
    orderId: order.id,
  }, {
    delay: 1000, // Delay 1 second
    attempts: 3,
    backoff: { type: "exponential", delay: 1000 },
  });

  return order;
});
```

### Dependency Injection
```typescript
// plugins/di.ts
import fp from "fastify-plugin";
import { diContainer, fastifyAwilixPlugin } from "@fastify/awilix";
import { asClass, asFunction, Lifetime } from "awilix";

export default fp(async (fastify) => {
  await fastify.register(fastifyAwilixPlugin, {
    disposeOnClose: true,
    disposeOnResponse: true,
  });

  diContainer.register({
    // Services
    positionService: asClass(PositionService).scoped(),
    tradeService: asClass(TradeService).scoped(),
    notificationService: asClass(NotificationService).singleton(),

    // Repositories
    positionRepo: asFunction(({ supabase }) => new PositionRepo(supabase)).scoped(),
  });
});

// Service example
class PositionService {
  constructor(private positionRepo: PositionRepo) {}

  async getByUser(userId: string) {
    return this.positionRepo.findByUser(userId);
  }
}

// Usage in route
fastify.get("/positions", async (request, reply) => {
  const positionService = request.diScope.resolve("positionService");
  return positionService.getByUser(request.user.id);
});
```

### Testing Routes
```typescript
// tests/routes/positions.test.ts
import { describe, it, expect, beforeAll, afterAll } from "vitest";
import Fastify from "fastify";
import positionsRoutes from "../../src/routes/positions";

describe("Positions Routes", () => {
  const fastify = Fastify();

  beforeAll(async () => {
    // Register plugins and routes
    await fastify.register(positionsRoutes, { prefix: "/positions" });
    await fastify.ready();
  });

  afterAll(async () => {
    await fastify.close();
  });

  it("GET /positions returns list", async () => {
    const response = await fastify.inject({
      method: "GET",
      url: "/positions",
      headers: {
        authorization: "Bearer test-token",
      },
    });

    expect(response.statusCode).toBe(200);
    expect(JSON.parse(response.body)).toBeInstanceOf(Array);
  });

  it("POST /positions creates position", async () => {
    const response = await fastify.inject({
      method: "POST",
      url: "/positions",
      headers: {
        authorization: "Bearer test-token",
        "content-type": "application/json",
      },
      payload: {
        symbol: "AAPL",
        quantity: 100,
        entry_price: 150.00,
        position_type: "stock",
      },
    });

    expect(response.statusCode).toBe(201);
    const body = JSON.parse(response.body);
    expect(body.symbol).toBe("AAPL");
  });

  it("POST /positions validates input", async () => {
    const response = await fastify.inject({
      method: "POST",
      url: "/positions",
      headers: {
        authorization: "Bearer test-token",
        "content-type": "application/json",
      },
      payload: {
        symbol: "", // Invalid: empty
        quantity: -1, // Invalid: negative
      },
    });

    expect(response.statusCode).toBe(400);
  });
});
```

---

## Decision Tree

```
Need an API endpoint?
│
├─ Simple CRUD on Supabase table?
│  └─ Use PostgREST directly (see /api-data)
│
├─ Complex logic, multiple DB operations?
│  └─ Use Fastify route with Supabase client
│
├─ Serverless, no persistent connections needed?
│  └─ Use Edge Function (see /edge-functions)
│
├─ Need background jobs, queues?
│  └─ Fastify + BullMQ
│
├─ Need WebSockets/realtime?
│  └─ Supabase Realtime (see /websocket)
│
└─ External API integration with resilience?
   └─ Fastify + undici + circuit breaker
```

---

## Coordination

| Need | Coordinate With |
|------|-----------------|
| API contracts/types | `/api-data` — get contract, implement here |
| Auth middleware | `/security` — get patterns, implement here |
| Database schema | `/db-architect` — schema first |
| Monitoring | `/monitor` — add health endpoints |
| Deployment | `/deployment` — Dockerfile, Railway config |

---

## Output

1. Route file(s) with full TypeScript typing
2. Plugin configurations
3. OpenAPI documentation (auto-generated)
4. Test files
5. curl/httpie examples

---

## GraphQL Integration

### Setup (Mercurius)
```bash
npm install mercurius graphql
```

```typescript
import Fastify from "fastify";
import mercurius from "mercurius";

const app = Fastify();

const schema = `
  type Position {
    id: ID!
    symbol: String!
    quantity: Int!
    entryPrice: Float!
    currentPrice: Float
    pnl: Float
    createdAt: String!
  }

  type Query {
    positions(userId: ID!): [Position!]!
    position(id: ID!): Position
  }

  type Mutation {
    createPosition(input: CreatePositionInput!): Position!
    closePosition(id: ID!): Position!
  }

  input CreatePositionInput {
    symbol: String!
    quantity: Int!
    entryPrice: Float!
  }
`;

const resolvers = {
  Query: {
    positions: async (_, { userId }, { supabase }) => {
      const { data } = await supabase
        .from("positions")
        .select("*")
        .eq("user_id", userId);
      return data;
    },
    position: async (_, { id }, { supabase }) => {
      const { data } = await supabase
        .from("positions")
        .select("*")
        .eq("id", id)
        .single();
      return data;
    },
  },
  Mutation: {
    createPosition: async (_, { input }, { supabase, user }) => {
      const { data } = await supabase
        .from("positions")
        .insert({ ...input, user_id: user.id })
        .select()
        .single();
      return data;
    },
  },
  Position: {
    pnl: (position) => {
      if (!position.currentPrice) return null;
      return (position.currentPrice - position.entryPrice) * position.quantity;
    },
  },
};

await app.register(mercurius, {
  schema,
  resolvers,
  context: (request) => ({
    supabase: request.supabase,
    user: request.user,
  }),
  graphiql: process.env.NODE_ENV === "development",
});
```

### DataLoader (N+1 Prevention)
```typescript
import DataLoader from "dataloader";

const createLoaders = (supabase: SupabaseClient) => ({
  userLoader: new DataLoader(async (userIds: readonly string[]) => {
    const { data } = await supabase
      .from("users")
      .select("*")
      .in("id", [...userIds]);

    const userMap = new Map(data?.map((u) => [u.id, u]));
    return userIds.map((id) => userMap.get(id) || null);
  }),

  priceLoader: new DataLoader(async (symbols: readonly string[]) => {
    const prices = await fetchPrices([...symbols]);
    const priceMap = new Map(prices.map((p) => [p.symbol, p.price]));
    return symbols.map((s) => priceMap.get(s) || null);
  }),
});

// Context with loaders
await app.register(mercurius, {
  schema,
  resolvers,
  context: (request) => ({
    supabase: request.supabase,
    loaders: createLoaders(request.supabase),
  }),
});

// Use in resolver
const resolvers = {
  Position: {
    currentPrice: (position, _, { loaders }) => {
      return loaders.priceLoader.load(position.symbol);
    },
  },
};
```

### Subscriptions
```typescript
import mercurius from "mercurius";

const schema = `
  type Subscription {
    positionUpdated(userId: ID!): Position!
    priceChanged(symbols: [String!]!): PriceUpdate!
  }

  type PriceUpdate {
    symbol: String!
    price: Float!
    timestamp: String!
  }
`;

await app.register(mercurius, {
  schema,
  resolvers,
  subscription: {
    context: (connection, request) => ({
      supabase: createClient(/* ... */),
    }),
  },
});

// Publish from mutation or external event
app.graphql.pubsub.publish({
  topic: `position:${userId}`,
  payload: { positionUpdated: updatedPosition },
});
```

---

## SSE & Long Polling

### Server-Sent Events (SSE)
```typescript
// SSE endpoint for real-time updates
fastify.get("/events/:userId", async (request, reply) => {
  const { userId } = request.params as { userId: string };

  // Set SSE headers
  reply.raw.writeHead(200, {
    "Content-Type": "text/event-stream",
    "Cache-Control": "no-cache",
    "Connection": "keep-alive",
    "X-Accel-Buffering": "no", // Disable nginx buffering
  });

  // Send initial connection event
  reply.raw.write(`event: connected\ndata: ${JSON.stringify({ userId })}\n\n`);

  // Keep-alive ping every 30 seconds
  const pingInterval = setInterval(() => {
    reply.raw.write(`: ping\n\n`);
  }, 30000);

  // Subscribe to database changes
  const channel = supabase
    .channel(`sse:${userId}`)
    .on(
      "postgres_changes",
      { event: "*", schema: "public", table: "positions", filter: `user_id=eq.${userId}` },
      (payload) => {
        reply.raw.write(`event: position\ndata: ${JSON.stringify(payload)}\n\n`);
      }
    )
    .subscribe();

  // Cleanup on client disconnect
  request.raw.on("close", () => {
    clearInterval(pingInterval);
    supabase.removeChannel(channel);
  });
});
```

### SSE with Event ID (Resume Support)
```typescript
fastify.get("/events/resumable/:userId", async (request, reply) => {
  const { userId } = request.params as { userId: string };
  const lastEventId = request.headers["last-event-id"] as string | undefined;

  reply.raw.writeHead(200, {
    "Content-Type": "text/event-stream",
    "Cache-Control": "no-cache",
    "Connection": "keep-alive",
  });

  // If resuming, send missed events
  if (lastEventId) {
    const missedEvents = await getMissedEvents(userId, lastEventId);
    for (const event of missedEvents) {
      reply.raw.write(`id: ${event.id}\nevent: ${event.type}\ndata: ${JSON.stringify(event.data)}\n\n`);
    }
  }

  // Continue with live events
  let eventCounter = parseInt(lastEventId || "0");

  const sendEvent = (type: string, data: unknown) => {
    eventCounter++;
    reply.raw.write(`id: ${eventCounter}\nevent: ${type}\ndata: ${JSON.stringify(data)}\n\n`);
  };

  // ... subscribe to changes and call sendEvent
});
```

### Long Polling
```typescript
// Long polling endpoint
fastify.get("/poll/:userId", async (request, reply) => {
  const { userId } = request.params as { userId: string };
  const { since, timeout = 30000 } = request.query as { since?: string; timeout?: number };

  // Check for immediate updates
  const updates = await getUpdatesSince(userId, since);
  if (updates.length > 0) {
    return { updates, timestamp: new Date().toISOString() };
  }

  // Wait for updates or timeout
  return new Promise((resolve) => {
    const timeoutId = setTimeout(() => {
      cleanup();
      resolve({ updates: [], timestamp: new Date().toISOString() });
    }, Math.min(timeout, 30000));

    const channel = supabase
      .channel(`poll:${userId}`)
      .on("postgres_changes", { event: "*", schema: "public", table: "positions" }, (payload) => {
        cleanup();
        resolve({ updates: [payload], timestamp: new Date().toISOString() });
      })
      .subscribe();

    const cleanup = () => {
      clearTimeout(timeoutId);
      supabase.removeChannel(channel);
    };

    request.raw.on("close", cleanup);
  });
});
```

### Client-Side SSE
```typescript
// SolidJS SSE hook
const useSSE = <T>(url: string) => {
  const [data, setData] = createSignal<T | null>(null);
  const [connected, setConnected] = createSignal(false);

  onMount(() => {
    const eventSource = new EventSource(url);

    eventSource.onopen = () => setConnected(true);
    eventSource.onerror = () => setConnected(false);

    eventSource.addEventListener("position", (e) => {
      setData(JSON.parse(e.data));
    });

    onCleanup(() => eventSource.close());
  });

  return { data, connected };
};
```

---

## Response Caching

### Redis Cache Plugin
```typescript
import fp from "fastify-plugin";
import Redis from "ioredis";

declare module "fastify" {
  interface FastifyInstance {
    cache: {
      get: <T>(key: string) => Promise<T | null>;
      set: (key: string, value: unknown, ttl?: number) => Promise<void>;
      del: (key: string) => Promise<void>;
      invalidatePattern: (pattern: string) => Promise<void>;
    };
  }
}

export default fp(async (fastify) => {
  const redis = new Redis(fastify.config.REDIS_URL);

  fastify.decorate("cache", {
    get: async <T>(key: string): Promise<T | null> => {
      const data = await redis.get(key);
      return data ? JSON.parse(data) : null;
    },
    set: async (key: string, value: unknown, ttl = 300) => {
      await redis.setex(key, ttl, JSON.stringify(value));
    },
    del: async (key: string) => {
      await redis.del(key);
    },
    invalidatePattern: async (pattern: string) => {
      const keys = await redis.keys(pattern);
      if (keys.length) await redis.del(...keys);
    },
  });

  fastify.addHook("onClose", () => redis.disconnect());
});
```

### Cache-Aside Pattern
```typescript
fastify.get("/positions/:id", async (request, reply) => {
  const { id } = request.params as { id: string };
  const cacheKey = `position:${id}`;

  // Try cache first
  const cached = await fastify.cache.get<Position>(cacheKey);
  if (cached) {
    reply.header("X-Cache", "HIT");
    return cached;
  }

  // Fetch from database
  const { data } = await fastify.supabase
    .from("positions")
    .select("*")
    .eq("id", id)
    .single();

  // Store in cache
  await fastify.cache.set(cacheKey, data, 60); // 60 second TTL

  reply.header("X-Cache", "MISS");
  return data;
});

// Invalidate on mutation
fastify.put("/positions/:id", async (request, reply) => {
  const { id } = request.params as { id: string };

  const { data } = await fastify.supabase
    .from("positions")
    .update(request.body)
    .eq("id", id)
    .select()
    .single();

  // Invalidate cache
  await fastify.cache.del(`position:${id}`);
  await fastify.cache.invalidatePattern(`positions:user:*`);

  return data;
});
```

### HTTP Cache Headers
```typescript
// Immutable resources (versioned assets)
fastify.get("/static/:hash/:file", async (request, reply) => {
  reply.header("Cache-Control", "public, max-age=31536000, immutable");
  return sendFile(request.params.file);
});

// Cacheable but revalidate
fastify.get("/api/prices/:symbol", async (request, reply) => {
  const data = await getPrice(request.params.symbol);

  reply.header("Cache-Control", "public, max-age=5, stale-while-revalidate=30");
  reply.header("ETag", `"${hashData(data)}"`);

  return data;
});

// ETag validation
fastify.addHook("preHandler", async (request, reply) => {
  const ifNoneMatch = request.headers["if-none-match"];
  if (ifNoneMatch && request.routeOptions.config?.etag) {
    const currentEtag = await computeEtag(request);
    if (ifNoneMatch === currentEtag) {
      reply.status(304).send();
    }
  }
});
```

### Request Deduplication
```typescript
const inFlightRequests = new Map<string, Promise<unknown>>();

const dedupe = async <T>(key: string, fn: () => Promise<T>): Promise<T> => {
  const existing = inFlightRequests.get(key);
  if (existing) return existing as Promise<T>;

  const promise = fn().finally(() => {
    inFlightRequests.delete(key);
  });

  inFlightRequests.set(key, promise);
  return promise;
};

// Usage in route
fastify.get("/expensive/:id", async (request, reply) => {
  const { id } = request.params as { id: string };

  return dedupe(`expensive:${id}`, async () => {
    // This will only run once even with concurrent requests
    return await expensiveOperation(id);
  });
});
```

### Cache Warming
```typescript
// Pre-warm cache on startup
fastify.addHook("onReady", async () => {
  const { data: popular } = await fastify.supabase
    .from("positions")
    .select("*")
    .order("view_count", { ascending: false })
    .limit(100);

  for (const position of popular || []) {
    await fastify.cache.set(`position:${position.id}`, position, 300);
  }

  fastify.log.info("Cache warmed with 100 popular positions");
});
```
