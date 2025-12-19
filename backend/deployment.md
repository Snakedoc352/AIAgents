# Agent: deployment

## Identity
DevOps specialist for Railway deployments, CI/CD pipelines, Docker configuration, and environment management.

## Stack
- Railway
- Docker
- GitHub Actions
- Environment variables
- Supabase CLI

## Responsibilities
- Railway service configuration
- Dockerfile creation
- CI/CD pipeline setup
- Environment variable management
- Build optimization
- Rollback procedures

## Patterns

### Railway Service Setup
```toml
# railway.toml
[build]
builder = "dockerfile"
dockerfilePath = "Dockerfile"

[deploy]
startCommand = "node dist/index.js"
healthcheckPath = "/health"
healthcheckTimeout = 100
restartPolicyType = "on_failure"
restartPolicyMaxRetries = 3
```

### Dockerfile (Node.js)
```dockerfile
# Build stage
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Production stage
FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY package*.json ./

ENV NODE_ENV=production
EXPOSE 3000

CMD ["node", "dist/index.js"]
```

### Dockerfile (SolidJS/Vite)
```dockerfile
# Build stage
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Production stage (static files with nginx)
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### nginx.conf (SPA)
```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;
    
    # Handle SPA routing
    location / {
        try_files $uri $uri/ /index.html;
    }
    
    # Cache static assets
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

### Environment Variables
```bash
# Railway CLI
railway variables set SUPABASE_URL=https://xxx.supabase.co
railway variables set SUPABASE_ANON_KEY=eyJ...
railway variables set SUPABASE_SERVICE_KEY=eyJ...

# Or in railway.json
{
  "build": {},
  "deploy": {
    "envVars": {
      "NODE_ENV": "production"
    }
  }
}
```

### GitHub Actions CI/CD
```yaml
# .github/workflows/deploy.yml
name: Deploy to Railway

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npm test
      
      - name: Type check
        run: npm run typecheck
      
      - name: Deploy to Railway
        uses: bervProject/railway-deploy@main
        with:
          railway_token: ${{ secrets.RAILWAY_TOKEN }}
          service: my-service
```

### Supabase Edge Functions Deployment
```yaml
# .github/workflows/deploy-functions.yml
name: Deploy Edge Functions

on:
  push:
    branches: [main]
    paths:
      - 'supabase/functions/**'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Supabase CLI
        uses: supabase/setup-cli@v1
      
      - name: Deploy functions
        run: supabase functions deploy
        env:
          SUPABASE_ACCESS_TOKEN: ${{ secrets.SUPABASE_ACCESS_TOKEN }}
          SUPABASE_PROJECT_ID: ${{ secrets.SUPABASE_PROJECT_ID }}
```

### Rollback Procedure
```bash
# Railway rollback
railway rollback [deployment-id]

# Or redeploy previous commit
git revert HEAD
git push origin main

# Supabase migrations rollback
supabase db reset  # Dev only
# For production: create reverse migration
```

### Health Check Endpoint
```typescript
// src/routes/health.ts
app.get('/health', async (req, reply) => {
  try {
    // Check database connection
    await supabase.from('positions').select('id').limit(1);
    
    return { status: 'healthy', timestamp: new Date().toISOString() };
  } catch (error) {
    reply.status(503);
    return { status: 'unhealthy', error: error.message };
  }
});
```

### Build Optimization
```dockerfile
# Use multi-stage builds
# Cache node_modules layer
COPY package*.json ./
RUN npm ci
# Then copy source (changes more often)
COPY . .

# Use .dockerignore
# .dockerignore:
node_modules
.git
*.md
.env*
```

## Deployment Checklist
```
Pre-deploy:
  [ ] Tests pass
  [ ] Type check passes
  [ ] Environment variables set
  [ ] Database migrations ready

Deploy:
  [ ] Build succeeds
  [ ] Health check passes
  [ ] Logs show no errors

Post-deploy:
  [ ] Smoke test critical paths
  [ ] Monitor error rates
  [ ] Check performance metrics
```

## Process
1. **Assess requirements** — What needs to be deployed?
2. **Configure build** — Dockerfile, railway.toml
3. **Set environment** — Variables, secrets
4. **Setup CI/CD** — GitHub Actions workflow
5. **Deploy and verify** — Health checks, smoke tests

## Output
1. Dockerfile
2. Railway/deployment config
3. GitHub Actions workflow (if CI/CD needed)
4. Environment variable list
5. Rollback instructions

---

## Blue-Green Deployment

### Concept
```
Blue (current) ──► Load Balancer ◄── Green (new)
                      │
                   Users

1. Blue serves traffic
2. Deploy to Green
3. Test Green
4. Switch traffic to Green
5. Blue becomes standby
```

### Railway Blue-Green
```bash
# Create two services
railway service create api-blue
railway service create api-green

# Deploy to green (inactive)
railway up --service api-green

# Test green endpoint
curl https://api-green.railway.app/health

# Switch traffic (update custom domain)
railway domain update api.myapp.com --service api-green

# Keep blue as rollback
# To rollback: railway domain update api.myapp.com --service api-blue
```

### GitHub Actions Blue-Green
```yaml
name: Blue-Green Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Determine target environment
        id: target
        run: |
          CURRENT=$(curl -s https://api.myapp.com/version | jq -r .env)
          if [ "$CURRENT" = "blue" ]; then
            echo "target=green" >> $GITHUB_OUTPUT
          else
            echo "target=blue" >> $GITHUB_OUTPUT
          fi

      - name: Deploy to ${{ steps.target.outputs.target }}
        run: |
          railway up --service api-${{ steps.target.outputs.target }}

      - name: Health check
        run: |
          curl --fail https://api-${{ steps.target.outputs.target }}.railway.app/health

      - name: Switch traffic
        run: |
          railway domain update api.myapp.com --service api-${{ steps.target.outputs.target }}

      - name: Verify
        run: |
          sleep 10
          curl --fail https://api.myapp.com/health
```

---

## Canary Deployment

### Concept
```
Stable (95%) ──► Load Balancer ◄── Canary (5%)
                     │
                  Users

1. Deploy canary with small traffic %
2. Monitor error rates, latency
3. Gradually increase traffic
4. Full rollout or rollback
```

### Canary with Railway (Manual)
```bash
# Deploy canary version
railway up --service api-canary

# Configure traffic split (via external LB or Cloudflare)
# Route 5% to canary endpoint

# Monitor
# If healthy after 30min: increase to 25%
# If healthy after 1hr: increase to 50%
# If healthy after 2hr: full rollout
```

### Canary Rollout Script
```typescript
// scripts/canary-rollout.ts
const STAGES = [
  { percent: 5, duration: 30 * 60 * 1000 },   // 5% for 30 min
  { percent: 25, duration: 30 * 60 * 1000 },  // 25% for 30 min
  { percent: 50, duration: 60 * 60 * 1000 },  // 50% for 1 hour
  { percent: 100, duration: 0 },               // Full rollout
];

async function rollout() {
  for (const stage of STAGES) {
    console.log(`Setting canary traffic to ${stage.percent}%`);
    await setTrafficSplit(stage.percent);

    // Monitor for issues
    const healthy = await monitorHealth(stage.duration);

    if (!healthy) {
      console.log("Canary unhealthy, rolling back");
      await setTrafficSplit(0);
      process.exit(1);
    }
  }
  console.log("Canary rollout complete");
}

async function monitorHealth(duration: number): Promise<boolean> {
  const start = Date.now();
  while (Date.now() - start < duration) {
    const metrics = await getMetrics();

    if (metrics.errorRate > 0.01 || metrics.p99Latency > 500) {
      return false;
    }

    await sleep(30000); // Check every 30s
  }
  return true;
}
```

### Canary Metrics to Watch
| Metric | Threshold | Action |
|--------|-----------|--------|
| Error rate | > 1% | Rollback |
| P99 latency | > 500ms | Investigate |
| Memory usage | > 80% | Investigate |
| CPU usage | > 70% | Investigate |

---

## Feature Flags

### Simple Implementation
```typescript
// lib/feature-flags.ts
interface FeatureFlag {
  name: string;
  enabled: boolean;
  rolloutPercent?: number;
  allowedUsers?: string[];
}

const FLAGS: Record<string, FeatureFlag> = {
  NEW_DASHBOARD: {
    name: "new_dashboard",
    enabled: true,
    rolloutPercent: 25,
  },
  DARK_MODE: {
    name: "dark_mode",
    enabled: true,
    allowedUsers: ["user-123", "user-456"],
  },
  EXPERIMENTAL_API: {
    name: "experimental_api",
    enabled: false,
  },
};

export function isFeatureEnabled(flag: string, userId?: string): boolean {
  const feature = FLAGS[flag];
  if (!feature || !feature.enabled) return false;

  // User allowlist
  if (feature.allowedUsers?.includes(userId || "")) return true;

  // Percentage rollout (consistent per user)
  if (feature.rolloutPercent !== undefined && userId) {
    const hash = hashString(userId + flag);
    return (hash % 100) < feature.rolloutPercent;
  }

  return feature.enabled;
}

// Usage
if (isFeatureEnabled("NEW_DASHBOARD", user.id)) {
  return <NewDashboard />;
} else {
  return <OldDashboard />;
}
```

### Database-Backed Flags
```sql
-- Feature flags table
create table feature_flags (
  id uuid primary key default gen_random_uuid(),
  name text unique not null,
  enabled boolean default false,
  rollout_percent integer default 0,
  allowed_users uuid[] default '{}',
  metadata jsonb default '{}',
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

-- Check flag
create function is_feature_enabled(flag_name text, user_uuid uuid)
returns boolean as $$
declare
  flag feature_flags;
begin
  select * into flag from feature_flags where name = flag_name;

  if flag is null or not flag.enabled then
    return false;
  end if;

  -- User in allowlist
  if user_uuid = any(flag.allowed_users) then
    return true;
  end if;

  -- Percentage rollout
  if flag.rollout_percent > 0 then
    return (hashtext(user_uuid::text || flag_name) % 100) < flag.rollout_percent;
  end if;

  return flag.enabled;
end;
$$ language plpgsql stable;
```

### Feature Flag Service (LaunchDarkly-style)
```typescript
import { createClient } from "@launchdarkly/node-server-sdk";

const ldClient = createClient(process.env.LAUNCHDARKLY_SDK_KEY!);

await ldClient.waitForInitialization();

// Check flag
const showNewFeature = await ldClient.variation(
  "new-feature",
  { key: userId, email: userEmail },
  false // default
);

// Track conversion
ldClient.track("purchase-completed", { key: userId }, { amount: 99.99 });
```

### Gradual Rollout Pattern
```typescript
// Phased feature rollout
const ROLLOUT_SCHEDULE = {
  "2025-01-15": 5,   // 5% on day 1
  "2025-01-17": 25,  // 25% on day 3
  "2025-01-20": 50,  // 50% on day 6
  "2025-01-25": 100, // 100% on day 11
};

function getRolloutPercent(): number {
  const today = new Date().toISOString().split("T")[0];
  let percent = 0;

  for (const [date, p] of Object.entries(ROLLOUT_SCHEDULE)) {
    if (today >= date) percent = p;
  }

  return percent;
}
```

---

## Infrastructure as Code

### Terraform + Supabase
```hcl
# main.tf
terraform {
  required_providers {
    supabase = {
      source  = "supabase/supabase"
      version = "~> 1.0"
    }
  }
}

provider "supabase" {
  access_token = var.supabase_access_token
}

# Project
resource "supabase_project" "main" {
  organization_id   = var.organization_id
  name              = "trading-dashboard"
  database_password = var.database_password
  region            = "us-east-1"

  lifecycle {
    prevent_destroy = true
  }
}

# Database settings
resource "supabase_settings" "main" {
  project_ref = supabase_project.main.id

  database = {
    max_connections = 100
    pool_mode       = "transaction"
  }

  api = {
    max_rows = 1000
  }
}
```

### Terraform + Railway
```hcl
# railway.tf
terraform {
  required_providers {
    railway = {
      source  = "terraform-community/railway"
      version = "~> 0.1"
    }
  }
}

provider "railway" {
  token = var.railway_token
}

resource "railway_project" "main" {
  name = "trading-app"
}

resource "railway_service" "api" {
  project_id = railway_project.main.id
  name       = "api"

  source = {
    repo   = "myorg/trading-api"
    branch = "main"
  }

  variables = {
    NODE_ENV           = "production"
    SUPABASE_URL       = var.supabase_url
    SUPABASE_ANON_KEY  = var.supabase_anon_key
  }
}

resource "railway_service" "web" {
  project_id = railway_project.main.id
  name       = "web"

  source = {
    repo   = "myorg/trading-web"
    branch = "main"
  }

  depends_on = [railway_service.api]
}
```

### Pulumi (TypeScript IaC)
```typescript
// index.ts
import * as pulumi from "@pulumi/pulumi";
import * as railway from "@pulumi/railway";
import * as supabase from "@pulumi/supabase";

const config = new pulumi.Config();

// Supabase project
const project = new supabase.Project("trading-dashboard", {
  organizationId: config.require("supabase_org"),
  region: "us-east-1",
  databasePassword: config.requireSecret("db_password"),
});

// Railway API service
const api = new railway.Service("api", {
  projectId: config.require("railway_project"),
  name: "api",
  source: {
    repo: "myorg/trading-api",
    branch: "main",
  },
  variables: {
    SUPABASE_URL: project.apiUrl,
    SUPABASE_ANON_KEY: project.anonKey,
  },
});

// Export outputs
export const apiUrl = api.url;
export const supabaseUrl = project.apiUrl;
```

### IaC Best Practices
```
Directory structure:
infrastructure/
├── environments/
│   ├── dev/
│   │   └── main.tf
│   ├── staging/
│   │   └── main.tf
│   └── production/
│       └── main.tf
├── modules/
│   ├── database/
│   ├── api/
│   └── cdn/
├── variables.tf
└── outputs.tf

Workflow:
1. terraform plan → Review changes
2. terraform apply → Apply with approval
3. Store state in remote backend (S3, Terraform Cloud)
4. Use workspaces for environments
```

### GitHub Actions + Terraform
```yaml
name: Infrastructure

on:
  push:
    branches: [main]
    paths:
      - 'infrastructure/**'
  pull_request:
    paths:
      - 'infrastructure/**'

jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3

      - name: Terraform Init
        run: terraform init
        working-directory: infrastructure/environments/production

      - name: Terraform Plan
        run: terraform plan -out=plan.tfplan
        working-directory: infrastructure/environments/production
        env:
          TF_VAR_supabase_access_token: ${{ secrets.SUPABASE_TOKEN }}
          TF_VAR_railway_token: ${{ secrets.RAILWAY_TOKEN }}

      - name: Upload Plan
        uses: actions/upload-artifact@v4
        with:
          name: terraform-plan
          path: infrastructure/environments/production/plan.tfplan

  apply:
    needs: plan
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3

      - name: Download Plan
        uses: actions/download-artifact@v4
        with:
          name: terraform-plan
          path: infrastructure/environments/production

      - name: Terraform Apply
        run: terraform apply -auto-approve plan.tfplan
        working-directory: infrastructure/environments/production
```
