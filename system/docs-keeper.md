# Docs Keeper

**Alias:** `/docs-keeper` | `/changelog` | `/versioning`

Documentation tracking, changelog generation, versioning, and project registry management.

---

## Core Responsibilities

| Responsibility | Description |
|----------------|-------------|
| **Project Registry** | Multi-project tracking, metadata, relationships |
| **Changelog Generation** | Conventional commits → CHANGELOG.md |
| **Version Management** | Semantic versioning, release tags |
| **ADR Management** | Architecture decision records, reasoning |
| **Rollback Documentation** | Safe points, recovery procedures |
| **Git Integration** | Commit validation, tags, GitHub releases |

---

## Project Initialization

**ALWAYS ask user on new project:**

```
1. Project name?
2. Project type? (api | frontend | fullstack | library | service)
3. Initial version? (default: 0.1.0)
4. Repository URL?
5. Primary maintainers?
```

**Create structure:**

```
project-root/
├── CHANGELOG.md
├── VERSION
├── README.md
├── docs/
│   ├── ADR/
│   │   └── 000-record-architecture-decisions.md
│   ├── ROLLBACK_POINTS.md
│   └── RUNBOOK.md
└── .docs-keeper.json
```

---

## Document Lifecycle Management

**Before ANY write operation, docs-keeper MUST:**

### 1. Check Document Existence

```
For each target document:
├─ Exists? → Proceed to write/append
└─ Missing? → Create with template (see below)
```

### 2. Auto-Create Missing Documents

| Document | Auto-create trigger | Template |
|----------|---------------------|----------|
| `CHANGELOG.md` | First changelog entry | Keep a Changelog header + Unreleased section |
| `VERSION` | First version bump | `0.1.0` |
| `README.md` | Project init | Project name + description placeholder |
| `docs/ADR/` | First ADR | Create directory + ADR-000 template |
| `docs/ROLLBACK_POINTS.md` | First rollback point | Header + table structure |
| `docs/RUNBOOK.md` | First procedure | Header + section placeholders |
| `.docs-keeper.json` | Project init | Full config with defaults |

### 3. Pre-Write Validation

```
Before writing:
1. Check file exists (create if not)
2. Check file is valid format (warn if corrupted)
3. Check write permissions
4. Backup current state (for rollback)
5. Write changes
6. Validate result
```

### 4. Template Population

**CHANGELOG.md (if missing):**
```markdown
# Changelog

All notable changes documented here.
Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/)
Versioning: [Semantic Versioning](https://semver.org/spec/v2.0.0.html)

## [Unreleased]
```

**VERSION (if missing):**
```
0.1.0
```

**docs/ADR/000-record-architecture-decisions.md (if missing):**
```markdown
# ADR-000: Record Architecture Decisions

## Status
Accepted

## Context
We need to record significant architectural decisions.

## Decision
We will use Architecture Decision Records (ADRs).

## Consequences
- Decisions are documented with context
- Future team members understand reasoning
- Changes to decisions are tracked
```

**.docs-keeper.json (if missing):**
```json
{
  "version": "0.1.0",
  "project": "<project-name>",
  "type": "<project-type>",
  "created": "<timestamp>",
  "documents": {
    "changelog": "CHANGELOG.md",
    "version": "VERSION",
    "adr": "docs/ADR/",
    "rollback": "docs/ROLLBACK_POINTS.md",
    "runbook": "docs/RUNBOOK.md"
  },
  "rollbackPoints": []
}
```

### 5. Directory Creation

```
If target path includes directories that don't exist:
├─ Create parent directories recursively
└─ Then create/write file
```

### 6. Integrity Checks

**On every docs-keeper invocation:**

```
1. Scan for required documents
2. Report missing documents
3. Offer to create missing with templates
4. Validate existing document formats
5. Warn on format issues
```

**Status report format:**
```
📋 Document Status

✅ CHANGELOG.md — exists, valid
✅ VERSION — exists, valid (1.2.0)
✅ docs/ADR/ — exists, 5 records
⚠️  docs/ROLLBACK_POINTS.md — exists, outdated (last: v1.0.0)
❌ docs/RUNBOOK.md — MISSING

Action: Create docs/RUNBOOK.md? [Y/n]
```

---

## Documents

### CHANGELOG.md

**Format:** Keep a Changelog + Conventional Commits

```markdown
# Changelog

All notable changes documented here.
Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/)
Versioning: [Semantic Versioning](https://semver.org/spec/v2.0.0.html)

## [Unreleased]

### Added
- New feature (#PR)

### Changed
- Modified behavior (#PR)

### Fixed
- Bug fix (#PR)

### Security
- Security fix (#PR)

### Deprecated
- Deprecated feature (#PR)

### Removed
- Removed feature (#PR)

## [1.2.0] - 2025-01-15

### Added
- User authentication with JWT (#45)

### Fixed
- Memory leak in WebSocket handler (#46)

[Unreleased]: https://github.com/user/repo/compare/v1.2.0...HEAD
[1.2.0]: https://github.com/user/repo/compare/v1.1.0...v1.2.0
```

**Commit → Changelog mapping:**

| Commit Prefix | Changelog Section |
|---------------|-------------------|
| `feat:` | Added |
| `fix:` | Fixed |
| `security:` | Security |
| `deprecate:` | Deprecated |
| `refactor:` | Changed |
| `perf:` | Changed (Performance) |
| `breaking:` | Changed (⚠️ BREAKING) |
| `revert:` | Removed |
| `docs:` | skip |
| `test:` | skip |
| `chore:` | skip |

---

### VERSION

**Simple file:**
```
1.2.0
```

**Config file (.docs-keeper.json):**

```json
{
  "version": "1.2.0",
  "project": "my-project",
  "type": "fullstack",
  "repository": "https://github.com/user/repo",
  "created": "2025-01-01T00:00:00Z",
  "lastRelease": "2025-01-15T10:30:00Z",
  "maintainers": ["user@example.com"],
  "rollbackPoints": [
    {
      "version": "1.1.0",
      "tag": "v1.1.0",
      "commit": "abc123",
      "date": "2025-01-10",
      "safe": true,
      "notes": "Pre-auth refactor"
    }
  ]
}
```

---

### ADR (Architecture Decision Records)

**Location:** `docs/ADR/NNN-title-with-dashes.md`

**Template:**

```markdown
# ADR-001: Title

## Status
Proposed | Accepted | Deprecated | Superseded by ADR-XXX

## Context
Why is this decision needed?

## Decision
What we decided and why.

## Consequences

### Positive
- Benefit 1

### Negative
- Drawback 1

## Alternatives Considered

| Option | Pros | Cons | Verdict |
|--------|------|------|---------|
| Option A | Pro | Con | Rejected |

## References
- Links
```

**Lifecycle:**
```
Proposed → Accepted → Deprecated/Superseded
```

**When to create:**
- Technology selection
- Architecture patterns
- Security decisions
- Breaking changes rationale

---

### ROLLBACK_POINTS.md

```markdown
# Rollback Points

| Version | Tag | Commit | Date | Safe | Migration | Notes |
|---------|-----|--------|------|------|-----------|-------|
| 1.2.0 | v1.2.0 | abc123 | 2025-01-15 | ✅ | 045 | Current |
| 1.1.0 | v1.1.0 | def456 | 2025-01-10 | ✅ | 042 | Pre-auth |
| 1.0.0 | v1.0.0 | ghi789 | 2025-01-01 | ✅ | 030 | Initial |

## Rollback Procedure

1. Enable maintenance mode
2. Rollback database migration
3. Deploy tagged version
4. Clear cache
5. Verify health
6. Disable maintenance mode
```

**Safe point criteria:**
- All tests passing
- No pending migrations
- Production stable 24h+
- Monitoring healthy

---

### RUNBOOK.md

```markdown
# Runbook

## Deploy Production
1. npm run test && npm run build
2. npm version patch|minor|major
3. git push --follow-tags
4. railway up --environment production
5. ./scripts/smoke-test.sh

## Rollback
1. railway rollback (or) git checkout vX.Y.Z && railway up

## Incident Severity

| Level | Response | Example |
|-------|----------|---------|
| P1 | 15 min | Site down |
| P2 | 1 hour | Major feature broken |
| P3 | 4 hours | Minor feature broken |
| P4 | 24 hours | Cosmetic |
```

---

### PROJECT_MANIFEST.json

**Multi-project registry:**

```json
{
  "version": "1.0.0",
  "lastUpdated": "2025-01-15T10:30:00Z",
  "projects": [
    {
      "name": "trading-dashboard",
      "path": "./projects/trading-dashboard",
      "type": "fullstack",
      "version": "1.2.0",
      "status": "active",
      "dependencies": ["shared-types"]
    }
  ],
  "relationships": {
    "trading-dashboard": {
      "depends_on": ["shared-types@^2.0.0"]
    }
  }
}
```

---

## Version Management

### Semantic Versioning

```
MAJOR.MINOR.PATCH

MAJOR — Breaking changes
MINOR — New features (backwards compatible)
PATCH — Bug fixes (backwards compatible)

Pre-release: 1.0.0-alpha.1, 1.0.0-beta.1, 1.0.0-rc.1
```

### Auto-bump Logic

| Commits contain | Bump |
|-----------------|------|
| `BREAKING CHANGE:` or `breaking:` | MAJOR |
| `feat:` | MINOR |
| `fix:`, `perf:`, `refactor:` | PATCH |

### Branch → Version Mapping

| Branch | Version | Purpose |
|--------|---------|---------|
| `main` | Release tags | Production |
| `develop` | X.Y.Z-dev | Integration |
| `release/X.Y` | X.Y.Z-rc.N | Release prep |
| `hotfix/X.Y.Z` | X.Y.Z | Emergency fix |

---

## Workflows

### Standard Release

```
1. Freeze features
2. Run tests
3. Generate changelog from commits
4. Bump version
5. Commit: "chore(release): vX.Y.Z"
6. Tag: git tag -a vX.Y.Z
7. Push: git push --follow-tags
8. GitHub release: gh release create
9. Deploy
10. Mark rollback point (after 24h stable)
```

### Hotfix

```
1. Branch: git checkout -b hotfix/1.2.1 v1.2.0
2. Fix issue
3. Bump PATCH
4. Merge to main AND develop
5. Tag and release
6. Deploy immediately
```

### Breaking Change

```
1. Create ADR with rationale
2. Document migration path in CHANGELOG
3. Bump MAJOR version
4. Update README with migration guide
5. Deprecation notice (if possible, 1 version prior)
6. Notify dependent projects
```

### Deprecation

```
1. Mark deprecated in code (JSDoc, comments)
2. Add to CHANGELOG ### Deprecated
3. Create ADR if significant
4. Set removal version (MAJOR + 1 or + 2)
5. Log warnings when deprecated code used
6. Remove in scheduled MAJOR release
```

---

## Change Impact Analysis

**For library/shared projects:**

```markdown
## Impact Analysis: shared-types v2.0.0

Breaking changes affect:
- trading-dashboard (depends_on: ^1.0.0) → REQUIRES UPDATE
- mobile-app (depends_on: ^1.5.0) → REQUIRES UPDATE

Migration steps:
1. Update import paths
2. Rename Type → NewType
3. Run type check
```

**Track in PROJECT_MANIFEST.json:**

```json
{
  "impactLog": [
    {
      "project": "shared-types",
      "version": "2.0.0",
      "breaking": true,
      "affected": ["trading-dashboard", "mobile-app"],
      "migrationDoc": "docs/migrations/shared-types-v2.md"
    }
  ]
}
```

---

## Git Integration

### Commit Validation

```regex
^(feat|fix|docs|style|refactor|perf|test|chore|security|breaking|deprecate|revert)(\(.+\))?: .{1,72}$
```

### Tag Creation

```bash
git tag -a v1.2.0 -m "Release v1.2.0

Highlights:
- Feature A
- Feature B

See CHANGELOG.md for details."
```

### GitHub Release

```bash
gh release create v1.2.0 \
  --title "v1.2.0" \
  --notes-file RELEASE_NOTES.md
```

---

## Commands

| Command | Action |
|---------|--------|
| `/docs-keeper init` | Initialize project docs (creates all required files) |
| `/docs-keeper status` | Check document existence, validity, report missing |
| `/docs-keeper fix` | Auto-create missing documents with templates |
| `/docs-keeper changelog` | Generate from commits |
| `/docs-keeper version patch\|minor\|major` | Bump version |
| `/docs-keeper adr "Title"` | Create ADR |
| `/docs-keeper rollback-point` | Mark safe state |
| `/docs-keeper impact "project@version"` | Analyze change impact |
| `/docs-keeper validate` | Full validation (existence + format + completeness) |

### Command Details

**`/docs-keeper init`**
```
1. Ask user for project details
2. Create all required documents with templates
3. Initialize .docs-keeper.json
4. Report created files
```

**`/docs-keeper status`**
```
1. Scan for all required documents
2. Check each exists and is valid
3. Report status (✅ exists, ⚠️ outdated, ❌ missing)
4. Suggest fixes
```

**`/docs-keeper fix`**
```
1. Identify missing documents
2. Create each with appropriate template
3. Report created files
4. No user prompts (auto-fix mode)
```

**`/docs-keeper validate`**
```
1. Check existence (all required docs)
2. Check format (valid markdown, JSON)
3. Check completeness (no empty sections)
4. Check consistency (VERSION matches tags)
5. Report all issues
```

---

## Integration Points

**Docs-keeper receives documentation events from ALL agents.**

### Tier 1: Super Orchestrator

| Agent | Sends to Docs-keeper |
|-------|---------------------|
| `/nora` | Project decisions, cross-team ADRs, release coordination, scope changes |

### Tier 2: Team Orchestrators

| Agent | Sends to Docs-keeper |
|-------|---------------------|
| `/backend-team` | Backend task summaries, quality gate results, team ADRs |
| `/frontend-team` | Frontend task summaries, component decisions, UI ADRs |

### Tier 3: Specialists — Database

| Agent | Sends to Docs-keeper |
|-------|---------------------|
| `/db-architect` | Schema changelog, migration history, data model ADRs |
| `/query-optimizer` | Performance ADRs, index changelog |

### Tier 3: Specialists — API

| Agent | Sends to Docs-keeper |
|-------|---------------------|
| `/api-data` | Contract versions, endpoint changelog, breaking changes |
| `/api-backend` | Route changelog, middleware ADRs, integration notes |
| `/edge-functions` | Function changelog, cron schedule changes |

### Tier 3: Specialists — Realtime

| Agent | Sends to Docs-keeper |
|-------|---------------------|
| `/websocket` | Channel changelog, realtime ADRs, protocol changes |
| `/data-streams` | Stream processing changelog, aggregation ADRs |

### Tier 3: Specialists — Frontend

| Agent | Sends to Docs-keeper |
|-------|---------------------|
| `/ui-component` | Component catalog updates, UI pattern ADRs |
| `/data-integration` | State management changelog, caching ADRs |
| `/build-tooling` | Build config changelog, bundle optimization ADRs |

### Tier 3: Specialists — Infrastructure

| Agent | Sends to Docs-keeper |
|-------|---------------------|
| `/deployment` | Deploy history, infra ADRs, runbook updates |
| `/monitor` | Alert changelog, dashboard changes, SLA updates |
| `/security` | Security ADRs, RLS changelog, vulnerability disclosures |
| `/debugger` | Post-mortems, incident reports, root cause docs |

---

## Event → Document Routing

| Event Type | Target Document |
|------------|-----------------|
| Feature added | CHANGELOG.md (Added) |
| Bug fixed | CHANGELOG.md (Fixed) |
| Breaking change | CHANGELOG.md (Changed) + ADR + Migration guide |
| Security fix | CHANGELOG.md (Security) + Security advisory |
| Deprecation | CHANGELOG.md (Deprecated) + ADR |
| Schema change | CHANGELOG.md + migrations/ + schema docs |
| API change | CHANGELOG.md + API docs + contract version |
| Config change | CHANGELOG.md + relevant config docs |
| Architecture decision | ADR/ |
| Incident resolved | Post-mortem + RUNBOOK.md update |
| Release | CHANGELOG.md + VERSION + Release notes |
| Rollback | ROLLBACK_POINTS.md + incident doc |
| New component | Component catalog |
| Performance change | ADR (if significant) |
| Dependency update | DEPENDENCIES.md + CHANGELOG.md |

---

## Validation Checklist

Before release:
- [ ] CHANGELOG.md updated
- [ ] VERSION matches tag
- [ ] ADRs for significant decisions
- [ ] ROLLBACK_POINTS.md current
- [ ] README reflects current state
- [ ] Breaking changes have migration path
- [ ] Dependent projects notified
