# Claude Code Agents

Claude Code compatible agents with proper YAML frontmatter for auto-delegation.

## Installation

Copy all `.md` files to your Claude Code agents directory:

```bash
# Global (available in all projects)
mkdir -p ~/.claude/agents
cp claude-code/*.md ~/.claude/agents/

# Or project-specific
mkdir -p .claude/agents
cp claude-code/*.md .claude/agents/
```

Restart Claude Code to load the agents.

## Available Agents

### Orchestrators
| Agent | Description |
|-------|-------------|
| `nora` | Super orchestrator - routes to frontend/backend teams |
| `backend-team` | Backend orchestrator - coordinates backend specialists |
| `frontend-team` | Frontend orchestrator - coordinates frontend specialists |

### Backend Specialists
| Agent | Use For |
|-------|---------|
| `db-architect` | Schema design, migrations, relationships |
| `query-optimizer` | SQL performance, EXPLAIN ANALYZE, indexes |
| `edge-functions` | Supabase Edge Functions (Deno/TS) |
| `websocket` | Supabase Realtime, channels, presence |
| `api-data` | API contracts, PostgREST, RPC functions |
| `deployment` | Railway, Docker, CI/CD, GitHub Actions |
| `debugger` | Build/runtime errors, log analysis, 5 Whys |
| `monitor` | Database health, slow queries, performance |
| `security` | RLS policies, auth, input validation |

### Frontend Specialists
| Agent | Use For |
|-------|---------|
| `ui-component` | SolidJS + Kobalte + Tailwind components |
| `data-integration` | createResource, stores, routing |
| `api-backend` | Fastify routes, middleware, validation |
| `build-tooling` | Vite config, TypeScript setup |

## Usage

Claude Code will **auto-delegate** based on the `description` field in each agent. You can also explicitly invoke:

```
Use the debugger agent to analyze this build failure
```

Or let the orchestrators route:
```
Ask Nora to build a new feature with database and UI
Ask backend team to optimize this slow query
Ask frontend team to create a positions table
```

## Agent Format

Each agent uses YAML frontmatter:

```markdown
---
name: agent-name
description: When Claude should auto-delegate to this agent
tools: Read, Write, Edit, Bash, Glob, Grep
---

[Agent instructions and patterns]
```

## Tools Reference

- `Read` - Read files
- `Write` - Create/write files  
- `Edit` - Edit existing files
- `Bash` - Run shell commands
- `Glob` - Find files by pattern
- `Grep` - Search file contents
