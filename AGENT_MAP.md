# AI Agent Hierarchy & Core Skills

```
                              NORA ─────────────────────┐
                        (Super Orchestrator)            │
                               │                        │
              ┌────────────────┴────────────────┐       │
              │                                 │       │
       /backend-team ───────────────────► /frontend-team
              │                                 │       │
    ┌────┬────┼────┬────┐              ┌───────┼───────┐│
    │    │    │    │    │              │       │       ││
   DB   API  RT  Infra  tic           UI    Data    Build
    │    │    │    │    │              │       │       ││
    └────┴────┴────┴────┴──────────────┴───────┴───────┴┘
                               │
                               ▼
                        /docs-keeper
                    (Documentation Hub)
                    Receives from ALL agents
```

---

## Tier 1: Super Orchestrator

### `/nora`
**Role:** Cross-team routing, task classification, coordination

| Core Skills |
|-------------|
| Task routing & team selection |
| Scope classification (trivial → epic) |
| Dependency detection & sequencing |
| Conflict resolution |
| Context preservation across handoffs |
| Progress tracking |
| Risk assessment & escalation |
| Decision memory & pattern learning |
| Resource estimation & complexity scoring |
| Failure pattern recognition & prevention |

### `/docs-keeper`
**Role:** Central documentation hub, changelog, versioning, ADRs

| Core Skills |
|-------------|
| Document lifecycle (check, create, populate) |
| Project registry (multi-project) |
| Changelog generation (conventional commits) |
| Semantic versioning & release tags |
| ADR management (decisions & reasoning) |
| Rollback point documentation |
| Git integration (tags, releases) |
| Event routing (all agents → docs) |
| Change impact analysis |

---

## Tier 2: Team Orchestrators

### `/backend-team`
| Core Skills |
|-------------|
| Task decomposition & dependency graph |
| Agent sequencing & parallel execution |
| Quality gates (schema, security, perf) |
| Rollback planning |

### `/frontend-team`
| Core Skills |
|-------------|
| UI breakdown & data requirements |
| Component → data → build pipeline |
| Type safety & accessibility validation |

---

## Tier 3: Specialist Agents

### Database Layer

#### `/db-architect`
**Domain:** PostgreSQL schema, migrations, data modeling

| Core Skills |
|-------------|
| Schema design (tables, constraints, defaults) |
| Relationships (1:1, 1:N, N:M, polymorphic) |
| Normalization (1NF → 3NF) |
| Migrations (up/down, idempotent, rollback) |
| Temporal data patterns |
| Multi-tenancy & RLS |
| TypeScript type generation |
| Table partitioning (range, list, hash) |
| Data archival & retention policies |
| Connection pooling (PgBouncer) |
| Backup & restore (pg_dump, PITR) |

#### `/query-optimizer`
**Domain:** SQL performance, indexing

| Core Skills |
|-------------|
| EXPLAIN ANALYZE, pg_stat_statements |
| Indexing (B-tree, GIN, partial, covering) |
| N+1 detection & DataLoader |
| Query rewriting (CTEs, window functions) |
| Materialized views & cache invalidation |

---

### API Layer

#### `/api-data`
**Domain:** API contracts, REST/RPC specs

| Core Skills |
|-------------|
| OpenAPI specs, JSON Schema, AsyncAPI |
| Pagination (cursor, offset, keyset) |
| Error handling (RFC 7807) |
| API versioning strategies |
| Webhook payload design |

#### `/api-backend`
**Domain:** Node.js/Fastify implementation

| Core Skills |
|-------------|
| Typed routes, middleware, plugins |
| CORS, rate limiting, helmet |
| File uploads & streaming |
| Background jobs (BullMQ) |
| External APIs & circuit breaker |
| Idempotency keys |
| GraphQL (Mercurius, DataLoader) |
| SSE & long polling |
| Response caching (Redis, ETag) |

#### `/edge-functions`
**Domain:** Deno/Supabase serverless

| Core Skills |
|-------------|
| Deno runtime & permissions |
| HTTP handling & streaming |
| Scheduled jobs (pg_cron) |
| Webhook verification |
| Third-party API integration |

---

### Realtime Layer

#### `/websocket`
**Domain:** Supabase Realtime, channels, presence

| Core Skills |
|-------------|
| Postgres change subscriptions |
| Channel naming & lifecycle |
| Presence & user tracking |
| Broadcast messaging |
| Reconnection (exponential backoff) |
| Offline queue & replay |
| Connection health monitoring |
| Channel multiplexing |
| Error recovery & graceful degradation |

#### `/data-streams`
**Domain:** Real-time data processing

| Core Skills |
|-------------|
| Event handling & windowing |
| Rolling aggregations (VWAP, OHLC) |
| Market data normalization |
| Live P&L & Greeks calculations |

---

### Frontend Layer

#### `/ui-component`
**Domain:** SolidJS, Tailwind, Kobalte

| Core Skills |
|-------------|
| SolidJS (signals, effects, Show/For/Switch) |
| Kobalte headless components |
| Tailwind utilities & responsive |
| Animation & transitions |
| Form patterns & validation |
| Loading states & skeletons |
| Drag & drop (sortable, Kanban) |
| Virtualized lists (@tanstack/solid-virtual) |
| Toast/notifications |
| Modal/dialog patterns |
| Component testing (Vitest) |
| Theme/dark mode |
| Internationalization (i18n, RTL, formatting) |
| Accessibility (ARIA, focus management, keyboard nav) |
| Print styles & print-friendly layouts |

#### `/data-integration`
**Domain:** State management, data fetching

| Core Skills |
|-------------|
| createResource & error/loading states |
| createStore & nested updates |
| Request deduplication & TTL |
| Offline support (IndexedDB) |
| Optimistic updates & rollback |
| WebSocket integration |
| State persistence (localStorage, sessionStorage, URL) |
| Optimistic concurrency (versioning, conflict resolution) |

#### `/build-tooling`
**Domain:** Vite, bundling, optimization

| Core Skills |
|-------------|
| Vite config, plugins, aliases |
| Code splitting & lazy routes |
| Tree shaking |
| Bundle analysis & optimization |
| Asset optimization |
| HMR & dev server |

---

### Infrastructure Layer

#### `/deployment`
**Domain:** Railway, Docker, CI/CD

| Core Skills |
|-------------|
| Railway services & networking |
| Docker multi-stage builds |
| GitHub Actions workflows |
| Environment & secrets management |
| Zero-downtime deploys |
| Rollback & version management |
| Preview environments |
| Blue-green deployment |
| Canary releases & gradual rollout |
| Feature flags (database-backed, percentage rollout) |
| Infrastructure as Code (Terraform, Pulumi) |

#### `/monitor`
**Domain:** Observability, alerting

| Core Skills |
|-------------|
| Dashboards & metrics |
| Alerting & escalation |
| Structured logging |
| Distributed tracing |
| APM & performance |
| SLA & error budgets |
| Synthetic monitoring & health checks |
| Cost monitoring & budget alerts |
| Anomaly detection (statistical, pattern-based) |
| Session tracking & analytics |

#### `/security`
**Domain:** Auth, authorization, encryption

| Core Skills |
|-------------|
| RLS policy design |
| JWT, OAuth, sessions |
| RBAC/ABAC permissions |
| Encryption (at-rest, in-transit) |
| Security headers (CORS, CSP, HSTS) |
| API security & rate limiting |
| OWASP vulnerability audit |
| Secrets management (Vault, env vars, rotation) |
| Incident response playbooks |
| Compliance patterns (GDPR, SOC2, audit logging) |
| Security audit automation (CodeQL, Semgrep, Snyk) |

#### `/debugger`
**Domain:** Troubleshooting, root cause analysis

| Core Skills |
|-------------|
| Build failure diagnosis |
| Runtime error isolation |
| Performance profiling |
| Network debugging |
| Database issue resolution |
| 5 Whys root cause analysis |
| Memory profiling (heap snapshots, leak detection) |
| CPU profiling (flame graphs, clinic.js) |
| Distributed tracing (OpenTelemetry, Jaeger) |
| Chaos engineering (failure injection, gameday) |

---

## Quick Reference

| Agent | Domain | Skills |
|-------|--------|--------|
| `/nora` | Orchestration | 10 |
| `/docs-keeper` | Documentation hub | 9 |
| `/backend-team` | Backend coordination | 4 |
| `/frontend-team` | Frontend coordination | 3 |
| `/db-architect` | PostgreSQL | 11 |
| `/query-optimizer` | SQL performance | 5 |
| `/api-data` | API contracts | 5 |
| `/api-backend` | Fastify/Node | 9 |
| `/edge-functions` | Deno/Supabase | 5 |
| `/websocket` | Realtime | 9 |
| `/data-streams` | Data processing | 4 |
| `/ui-component` | SolidJS/UI | 15 |
| `/data-integration` | State/fetching | 8 |
| `/build-tooling` | Vite/bundling | 6 |
| `/deployment` | CI/CD/infra | 11 |
| `/monitor` | Observability | 10 |
| `/security` | Auth/security | 11 |
| `/debugger` | Troubleshooting | 10 |
| **Total** | **18 agents** | **145** |

---

## Invocation

```bash
# Orchestrator
ask nora "Build a real-time portfolio tracker"

# Documentation
/docs-keeper init
/docs-keeper changelog
/docs-keeper adr "Use WebSocket for realtime"

# Teams
/backend-team "Add positions table with RLS"
/frontend-team "Create positions dashboard"

# Direct
/db-architect "Design trade history schema"
/ui-component "Build P&L card"
/websocket "Subscribe to position updates"
```
