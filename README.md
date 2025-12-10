# AIAgents

Agent prompt definitions for Claude persona adoption. Store agent expertise here, invoke from any Claude project.

## Quick Start

**Ask the frontend team** — the orchestrator that analyzes your task and chains the right agents:
> "Ask frontend team to build a positions table with sorting and live updates"

Or invoke a specific agent:
> "Fetch `frontend/ui-component` from AIAgents and adopt that persona"

## Aliases

| Alias | Agent | Purpose |
|-------|-------|---------|
| `ask frontend team` | `system/orchestrator` | Analyzes tasks, selects & chains agents |

## Structure

```
AIAgents/
├── system/
│   └── orchestrator.md      — Frontend Team: meta-agent that chains others
├── frontend/
│   ├── ui-component.md      — SolidJS + Kobalte + Tailwind
│   ├── api-backend.md       — Node.js + Fastify
│   ├── build-tooling.md     — Vite, config, scripts
│   └── data-integration.md  — API calls, state, routing
├── backend/                  (future)
├── debug/                    (future)
├── data/                     (future)
└── platform/                 (future)
```

## No API Required

These agents run within your Claude Max subscription. Claude reads the prompt and adopts the persona—no separate API costs.

## Adding Agents

1. Create `category/agent-name.md`
2. Define: Identity, Stack, Patterns, Process, Output format
3. Commit to main
4. (Optional) Add alias to this README
