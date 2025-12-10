# AIAgents

Agent prompt definitions for Claude persona adoption. Store agent expertise here, invoke from any Claude project.

## Usage

In any Claude conversation:
> "Fetch the `frontend/ui-component` agent from my AIAgents repo and adopt that persona"

Claude reads the prompt via GithubMCP and becomes that specialist.

## Structure

```
AIAgents/
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
