# AIAgents

Agent prompt definitions for Claude persona adoption. Store agent expertise here, invoke from any Claude project.

## Quick Start

**Ask Nora** — the super orchestrator that routes to any team:
> "Ask Nora to build a new feature with database schema and UI"

**Ask a specific team:**
> "Ask frontend team to build a positions table with sorting"
> "Ask backend team to optimize my slow query"

**Or invoke a specific agent:**
> "Fetch `backend/query-optimizer` from AIAgents and adopt that persona"

## Aliases

| Alias | Agent | Scope |
|-------|-------|-------|
| `ask Nora` / `ask nora` | `system/orchestrator` | Cross-team, routes to any domain |
| `ask frontend team` | `frontend/_team` | Frontend-specific orchestration |
| `ask backend team` | `backend/_team` | Backend-specific orchestration |

## Structure

```
AIAgents/
├── system/
│   └── orchestrator.md       — Nora: super agent, cross-team routing
│
├── frontend/
│   ├── _team.md              — Frontend team orchestrator
│   ├── ui-component.md       — SolidJS + Kobalte + Tailwind
│   ├── api-backend.md        — Node.js + Fastify routes
│   ├── build-tooling.md      — Vite, config, scripts
│   └── data-integration.md   — State, routing, data fetching
│
├── backend/
│   ├── _team.md              — Backend team orchestrator
│   ├── db-architect.md       — Schema design, migrations
│   ├── query-optimizer.md    — SQL optimization, indexing
│   ├── edge-functions.md     — Supabase Edge Functions
│   ├── websocket.md          — Realtime, channels, presence
│   ├── api-data.md           — API design, data contracts
│   ├── deployment.md         — Railway, CI/CD, Docker
│   ├── debugger.md           — Build/runtime error diagnosis
│   ├── monitor.md            — Performance monitoring, health
│   └── security.md           — RLS, auth, data protection
│
├── data/                      (future)
└── platform/                  (future)
```

## When to Use What

| Need | Command |
|------|---------|
| Full feature (UI + database) | `ask Nora` |
| UI components, styling | `ask frontend team` |
| Database, queries, API | `ask backend team` |
| Specific expertise | Fetch specific agent |

## No API Required

These agents run within your Claude Max subscription. Claude reads the prompt and adopts the persona—no separate API costs.

## Adding Agents

1. Create `category/agent-name.md`
2. Define: Identity, Stack, Patterns, Process, Output format
3. Commit to main
4. (Optional) Add alias to this README
