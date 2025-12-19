# AI Agent Hierarchy & Skills Map

```
                              NORA
                        (Super Orchestrator)
                               │
              ┌────────────────┴────────────────┐
              │                                 │
       /backend-team                     /frontend-team
      (Team Orchestrator)               (Team Orchestrator)
              │                                 │
    ┌─────────┼─────────┐             ┌────────┼────────┐
    │         │         │             │        │        │
 Database   API      Infra        Component  Data    Build
  Layer    Layer     Layer          Layer   Layer   Layer
```

---

## Tier 1: Super Orchestrator

### `/nora`
**Role:** Cross-team routing, task classification, conflict resolution

| Skill Category | Skills |
|----------------|--------|
| **Routing** | Keyword detection, team selection, dependency detection |
| **Classification** | Scope assessment (trivial → epic), complexity scoring |
| **Sequencing** | Sequential vs parallel, phased execution |
| **Conflict Resolution** | Schema vs API, security vs convenience |
| **Context Preservation** | Decision log, handoff context, constraints |
| **Progress Tracking** | Phase status, blockers, completion % |
| **Risk Assessment** | Escalation triggers, rollback recommendations |

---

## Tier 2: Team Orchestrators

### `/backend-team`
**Role:** Backend task decomposition, agent coordination, quality gates

| Skill Category | Skills |
|----------------|--------|
| **Decomposition** | Task breakdown, dependency graph, critical path |
| **Coordination** | Agent sequencing, parallel execution, handoffs |
| **Quality Gates** | Schema review, security audit, performance check |
| **Risk Management** | Blast radius analysis, rollback plans |

### `/frontend-team`
**Role:** Frontend task decomposition, component orchestration

| Skill Category | Skills |
|----------------|--------|
| **Decomposition** | UI breakdown, data requirements, routing |
| **Coordination** | Component → data → build pipeline |
| **Validation** | Type safety, accessibility, responsiveness |

---

## Tier 3: Specialist Agents

### Database Layer

#### `/db-architect`
**Domain:** PostgreSQL schema design, migrations, data modeling

| Skill Category | Skills |
|----------------|--------|
| **Schema Design** | Tables, columns, constraints, defaults |
| **Relationships** | 1:1, 1:N, N:M, polymorphic associations |
| **Normalization** | 1NF → 3NF, denormalization decisions |
| **Migrations** | Up/down scripts, idempotency, rollback |
| **Temporal Data** | Valid time, transaction time, bitemporal |
| **Multi-tenancy** | Schema isolation, RLS patterns |
| **Type Generation** | TypeScript types from schema |

#### `/query-optimizer`
**Domain:** SQL performance, indexing, query analysis

| Skill Category | Skills |
|----------------|--------|
| **Analysis** | EXPLAIN ANALYZE, pg_stat_statements |
| **Indexing** | B-tree, GIN, partial, covering indexes |
| **N+1 Detection** | Pattern recognition, DataLoader |
| **Query Rewriting** | CTEs, window functions, lateral joins |
| **Caching** | Materialized views, cache invalidation |

---

### API Layer

#### `/api-data`
**Domain:** API contract design, REST/RPC specs

| Skill Category | Skills |
|----------------|--------|
| **Contract Design** | OpenAPI specs, JSON Schema, AsyncAPI |
| **Data Modeling** | Nested vs flat, polymorphic responses |
| **Pagination** | Cursor, offset, keyset strategies |
| **Error Handling** | RFC 7807, error codes, retry hints |
| **Versioning** | URL, header, content negotiation |
| **Webhooks** | Payload design, retry policies, signatures |

#### `/api-backend`
**Domain:** Node.js/Fastify API implementation

| Skill Category | Skills |
|----------------|--------|
| **Routing** | Typed routes, params, query, body |
| **Middleware** | Hooks, plugins, decorators |
| **Plugins** | CORS, compression, rate limiting, helmet |
| **File Handling** | Multipart uploads, streaming |
| **Background Jobs** | BullMQ queues, workers, retries |
| **External APIs** | Undici, circuit breaker, retry agent |
| **Idempotency** | Request deduplication, idempotency keys |
| **Webhooks** | Signature verification, retry queues |
| **Testing** | Route injection, mocking |
| **GraphQL** | Mercurius, DataLoader, subscriptions |
| **SSE/Long Polling** | Event streams, resumable connections |
| **Response Caching** | Redis, cache-aside, HTTP headers, ETag |

#### `/edge-functions`
**Domain:** Deno/Supabase serverless functions

| Skill Category | Skills |
|----------------|--------|
| **Deno Runtime** | Permissions, imports, Deno.serve |
| **HTTP Handling** | Request/Response, streaming |
| **Scheduled Jobs** | Cron triggers, pg_cron |
| **Webhooks** | Verification, processing |
| **Third-party APIs** | OAuth, API keys, rate limiting |

---

### Realtime Layer

#### `/websocket`
**Domain:** Supabase Realtime, channels, presence

| Skill Category | Skills |
|----------------|--------|
| **Subscriptions** | Postgres changes, filtered subscriptions |
| **Channels** | Naming, multiplexing, lifecycle |
| **Presence** | User tracking, sync events |
| **Broadcast** | Ephemeral messaging, events |
| **Reconnection** | Exponential backoff, network awareness, visibility |
| **Offline Queue** | Message persistence, replay, batch processing |
| **Connection Health** | Heartbeat, stale detection, latency metrics |
| **Channel Multiplexing** | Shared channels, priority management, lazy subscription |
| **Error Recovery** | Classification, graceful degradation, conflict resolution |

#### `/data-streams`
**Domain:** Real-time data processing, aggregation

| Skill Category | Skills |
|----------------|--------|
| **Stream Processing** | Event handling, windowing |
| **Aggregation** | Rolling calculations, VWAP, OHLC |
| **Market Data** | Quote normalization, book updates |
| **Live Calculations** | P&L, Greeks, portfolio metrics |

---

### Frontend Layer

#### `/ui-component`
**Domain:** SolidJS components, Tailwind CSS, Kobalte

| Skill Category | Skills |
|----------------|--------|
| **SolidJS Patterns** | Signals, effects, Show/For/Switch |
| **Kobalte Primitives** | Headless components, accessibility |
| **Tailwind Styling** | Utilities, responsive, dark mode |
| **Trading UI** | Profit/loss colors, sentiment indicators |
| **Animation & Transitions** | CSS transitions, TransitionGroup, slide/collapse |
| **Form Patterns** | Controlled inputs, validation, multi-step, stores |
| **Loading States** | Spinners, skeletons, progressive loading, Suspense |
| **Drag & Drop** | Sortable lists, handles, overlays, Kanban |
| **Virtualized Lists** | @tanstack/solid-virtual, infinite scroll |
| **Toast/Notifications** | Context, stacking, auto-dismiss, transitions |
| **Modal/Dialog** | Kobalte Dialog, confirmation, sheets, forms |
| **Component Testing** | Vitest, @solidjs/testing-library, user events |
| **Theme/Dark Mode** | Context, system preference, CSS variables |

#### `/data-integration`
**Domain:** State management, data fetching, caching

| Skill Category | Skills |
|----------------|--------|
| **Resources** | createResource, error/loading states |
| **Stores** | createStore, nested updates, derived |
| **Caching** | Request deduplication, TTL, invalidation |
| **Offline Support** | IndexedDB, sync queues |
| **Optimistic Updates** | Rollback on failure |
| **WebSocket Integration** | Real-time sync, reconnection |

#### `/build-tooling`
**Domain:** Vite configuration, bundling, optimization

| Skill Category | Skills |
|----------------|--------|
| **Vite Config** | Plugins, aliases, env variables |
| **Code Splitting** | Lazy routes, dynamic imports |
| **Tree Shaking** | Side effects, pure annotations |
| **Bundle Analysis** | Size optimization, chunk strategy |
| **Asset Optimization** | Images, fonts, compression |
| **Dev Server** | HMR, proxy, HTTPS |

---

### Infrastructure Layer

#### `/deployment`
**Domain:** Railway, Docker, CI/CD, infrastructure

| Skill Category | Skills |
|----------------|--------|
| **Railway** | Services, volumes, networking, domains |
| **Docker** | Multi-stage builds, optimization, caching |
| **GitHub Actions** | Workflows, matrix builds, secrets |
| **Environment** | Variable management, secrets, configs |
| **Zero-Downtime** | Rolling deploys, health checks |
| **Rollback** | Version management, quick recovery |
| **Preview Environments** | PR deployments, ephemeral envs |

#### `/monitor`
**Domain:** Observability, alerting, diagnostics

| Skill Category | Skills |
|----------------|--------|
| **Dashboards** | Metrics visualization, key indicators |
| **Alerting** | Thresholds, escalation, on-call |
| **Logging** | Structured logs, aggregation, search |
| **Tracing** | Distributed tracing, spans, context |
| **Performance** | APM, bottleneck detection |
| **SLA Management** | Uptime tracking, error budgets |

#### `/security`
**Domain:** Authentication, authorization, encryption

| Skill Category | Skills |
|----------------|--------|
| **RLS Policies** | Row-level security, policy design |
| **Authentication** | JWT, OAuth, session management |
| **Authorization** | RBAC, ABAC, permission systems |
| **Encryption** | At-rest, in-transit, key management |
| **Headers** | CORS, CSP, HSTS, security headers |
| **API Security** | Rate limiting, API keys, signatures |
| **Vulnerability** | OWASP top 10, audit, remediation |

#### `/debugger`
**Domain:** Troubleshooting, root cause analysis

| Skill Category | Skills |
|----------------|--------|
| **Build Failures** | Webpack/Vite errors, dependency issues |
| **Runtime Errors** | Stack traces, reproduction, isolation |
| **Performance Issues** | Profiling, bottleneck identification |
| **Network Debugging** | Request/response, timeouts, CORS |
| **Database Issues** | Connection, query errors, locks |
| **Root Cause Analysis** | 5 Whys, timeline, evidence gathering |

---

## Skill Count Summary

| Agent | Core Skills | Advanced Skills | Total |
|-------|-------------|-----------------|-------|
| `/nora` | 4 | 3 | 7 |
| `/backend-team` | 3 | 1 | 4 |
| `/frontend-team` | 3 | 0 | 3 |
| `/db-architect` | 4 | 3 | 7 |
| `/query-optimizer` | 3 | 2 | 5 |
| `/api-data` | 4 | 2 | 6 |
| `/api-backend` | 9 | 3 | 12 |
| `/edge-functions` | 3 | 2 | 5 |
| `/websocket` | 4 | 5 | 9 |
| `/data-streams` | 2 | 2 | 4 |
| `/ui-component` | 4 | 9 | 13 |
| `/data-integration` | 4 | 2 | 6 |
| `/build-tooling` | 4 | 2 | 6 |
| `/deployment` | 4 | 3 | 7 |
| `/monitor` | 4 | 2 | 6 |
| `/security` | 4 | 3 | 7 |
| `/debugger` | 4 | 2 | 6 |

---

## Cross-Domain Coordination

```
┌─────────────────────────────────────────────────────────────────┐
│                         NORA                                     │
│  Routes to teams, preserves context, resolves conflicts         │
└──────────────────────────┬──────────────────────────────────────┘
                           │
         ┌─────────────────┴─────────────────┐
         ▼                                   ▼
┌─────────────────────┐           ┌─────────────────────┐
│   /backend-team     │◄─────────►│   /frontend-team    │
│                     │  Contracts │                     │
│  Orchestrates:      │  & Types   │  Orchestrates:      │
│  - /db-architect    │           │  - /ui-component    │
│  - /api-data        │           │  - /data-integration│
│  - /api-backend     │           │  - /build-tooling   │
│  - /query-optimizer │           │                     │
│  - /edge-functions  │           │                     │
│  - /websocket       │           │                     │
│  - /data-streams    │           │                     │
│  - /deployment      │           │                     │
│  - /debugger        │           │                     │
│  - /monitor         │           │                     │
│  - /security        │           │                     │
└─────────────────────┘           └─────────────────────┘
```

---

## Invocation Examples

```bash
# Super orchestrator
ask nora "Build a real-time portfolio tracker"

# Team orchestrators
ask backend team "Add a new positions table with RLS"
ask frontend team "Create a positions dashboard"

# Direct specialist invocation
/db-architect "Design schema for trade history"
/ui-component "Build a P&L card component"
/websocket "Subscribe to position updates"
/api-backend "Implement positions CRUD"
```
