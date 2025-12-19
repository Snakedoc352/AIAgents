# Agent: test-automation

## Identity
Testing specialist for unit, integration, E2E, and API tests with comprehensive coverage strategies and CI integration.

## Stack
- Vitest (unit/integration)
- Playwright (E2E)
- Testing Library (component)
- MSW (API mocking)
- Supertest (API testing)
- GitHub Actions (CI)

## Responsibilities
- Test strategy design
- Unit test patterns
- Integration test setup
- E2E test automation
- API contract testing
- Mock/stub management
- Coverage tracking
- CI pipeline integration

---

## Test Strategy

### Test Pyramid
```
                 /\
                /  \
               / E2E \        <- Few, slow, high confidence
              /______\
             /        \
            / Integration \   <- Some, medium speed
           /______________\
          /                \
         /      Unit        \ <- Many, fast, focused
        /____________________\
```

### Coverage Targets
| Layer | Target | Focus |
|-------|--------|-------|
| Unit | 80%+ | Business logic, utilities, pure functions |
| Integration | 60%+ | API routes, database queries, services |
| E2E | Critical paths | Auth flow, core user journeys |

### Test File Organization
```
src/
├── components/
│   ├── Button.tsx
│   └── Button.test.tsx      # Co-located unit tests
├── hooks/
│   ├── usePositions.ts
│   └── usePositions.test.ts
├── services/
│   └── api.ts
tests/
├── integration/
│   ├── api/
│   │   └── positions.test.ts
│   └── db/
│       └── queries.test.ts
├── e2e/
│   ├── auth.spec.ts
│   └── trading.spec.ts
└── fixtures/
    ├── positions.ts
    └── users.ts
```

---

## Unit Testing (Vitest)

### Configuration
```typescript
// vitest.config.ts
import { defineConfig } from "vitest/config";
import solidPlugin from "vite-plugin-solid";

export default defineConfig({
  plugins: [solidPlugin()],
  test: {
    environment: "jsdom",
    globals: true,
    setupFiles: ["./tests/setup.ts"],
    include: ["src/**/*.test.{ts,tsx}", "tests/unit/**/*.test.ts"],
    coverage: {
      provider: "v8",
      reporter: ["text", "html", "lcov"],
      exclude: [
        "node_modules/",
        "tests/",
        "**/*.d.ts",
        "**/*.config.*",
      ],
      thresholds: {
        lines: 80,
        functions: 80,
        branches: 75,
        statements: 80,
      },
    },
  },
});
```

### Test Setup
```typescript
// tests/setup.ts
import "@testing-library/jest-dom";
import { vi } from "vitest";

// Mock browser APIs
Object.defineProperty(window, "matchMedia", {
  writable: true,
  value: vi.fn().mockImplementation((query) => ({
    matches: false,
    media: query,
    onchange: null,
    addListener: vi.fn(),
    removeListener: vi.fn(),
    addEventListener: vi.fn(),
    removeEventListener: vi.fn(),
    dispatchEvent: vi.fn(),
  })),
});

// Mock localStorage
const localStorageMock = {
  getItem: vi.fn(),
  setItem: vi.fn(),
  removeItem: vi.fn(),
  clear: vi.fn(),
};
Object.defineProperty(window, "localStorage", { value: localStorageMock });

// Reset mocks between tests
beforeEach(() => {
  vi.clearAllMocks();
});
```

### Pure Function Tests
```typescript
import { describe, it, expect } from "vitest";
import { calculatePL, formatCurrency, validateSymbol } from "./utils";

describe("calculatePL", () => {
  it("calculates profit for long position", () => {
    const result = calculatePL({
      entryPrice: 100,
      currentPrice: 110,
      quantity: 10,
      side: "long",
    });

    expect(result.unrealizedPL).toBe(100);
    expect(result.unrealizedPLPercent).toBe(10);
  });

  it("calculates loss for long position", () => {
    const result = calculatePL({
      entryPrice: 100,
      currentPrice: 90,
      quantity: 10,
      side: "long",
    });

    expect(result.unrealizedPL).toBe(-100);
    expect(result.unrealizedPLPercent).toBe(-10);
  });

  it("handles short positions", () => {
    const result = calculatePL({
      entryPrice: 100,
      currentPrice: 90,
      quantity: 10,
      side: "short",
    });

    expect(result.unrealizedPL).toBe(100); // Profit when price drops
  });

  it("handles zero quantity", () => {
    const result = calculatePL({
      entryPrice: 100,
      currentPrice: 110,
      quantity: 0,
      side: "long",
    });

    expect(result.unrealizedPL).toBe(0);
  });
});

describe("formatCurrency", () => {
  it.each([
    [1234.56, "$1,234.56"],
    [-1234.56, "-$1,234.56"],
    [0, "$0.00"],
    [1000000, "$1,000,000.00"],
  ])("formats %d as %s", (input, expected) => {
    expect(formatCurrency(input)).toBe(expected);
  });
});

describe("validateSymbol", () => {
  it.each([
    ["AAPL", true],
    ["MSFT", true],
    ["BRK.A", true],
    ["", false],
    ["aapl", false], // lowercase
    ["A".repeat(11), false], // too long
    ["123", false], // numbers only
  ])("validates %s as %s", (symbol, expected) => {
    expect(validateSymbol(symbol)).toBe(expected);
  });
});
```

### Async Function Tests
```typescript
import { describe, it, expect, vi } from "vitest";
import { fetchPositions, createPosition } from "./api";

// Mock the supabase client
vi.mock("./supabase", () => ({
  supabase: {
    from: vi.fn(() => ({
      select: vi.fn(() => ({
        eq: vi.fn(() => Promise.resolve({ data: mockPositions, error: null })),
      })),
      insert: vi.fn(() => ({
        select: vi.fn(() => ({
          single: vi.fn(() => Promise.resolve({ data: mockPosition, error: null })),
        })),
      })),
    })),
  },
}));

describe("fetchPositions", () => {
  it("returns positions for user", async () => {
    const positions = await fetchPositions("user-123");

    expect(positions).toHaveLength(2);
    expect(positions[0]).toHaveProperty("symbol");
  });

  it("throws on error", async () => {
    vi.mocked(supabase.from).mockReturnValueOnce({
      select: () => ({
        eq: () => Promise.resolve({ data: null, error: new Error("DB error") }),
      }),
    });

    await expect(fetchPositions("user-123")).rejects.toThrow("DB error");
  });
});
```

---

## Component Testing (SolidJS)

### Basic Component Test
```typescript
import { render, screen, fireEvent } from "@solidjs/testing-library";
import { describe, it, expect, vi } from "vitest";
import { Button } from "./Button";

describe("Button", () => {
  it("renders children", () => {
    render(() => <Button>Click me</Button>);
    expect(screen.getByRole("button")).toHaveTextContent("Click me");
  });

  it("calls onClick when clicked", async () => {
    const onClick = vi.fn();
    render(() => <Button onClick={onClick}>Click</Button>);

    await fireEvent.click(screen.getByRole("button"));

    expect(onClick).toHaveBeenCalledOnce();
  });

  it("shows loading state", () => {
    render(() => <Button loading>Submit</Button>);

    expect(screen.getByRole("button")).toBeDisabled();
    expect(screen.getByTestId("spinner")).toBeInTheDocument();
  });

  it("applies variant styles", () => {
    render(() => <Button variant="danger">Delete</Button>);

    expect(screen.getByRole("button")).toHaveClass("bg-red-500");
  });
});
```

### Component with State
```typescript
import { render, screen, fireEvent, waitFor } from "@solidjs/testing-library";
import { describe, it, expect } from "vitest";
import { Counter } from "./Counter";

describe("Counter", () => {
  it("increments on click", async () => {
    render(() => <Counter initial={0} />);

    expect(screen.getByText("Count: 0")).toBeInTheDocument();

    await fireEvent.click(screen.getByRole("button", { name: "Increment" }));

    expect(screen.getByText("Count: 1")).toBeInTheDocument();
  });

  it("respects max value", async () => {
    render(() => <Counter initial={9} max={10} />);

    await fireEvent.click(screen.getByRole("button", { name: "Increment" }));

    expect(screen.getByText("Count: 10")).toBeInTheDocument();
    expect(screen.getByRole("button", { name: "Increment" })).toBeDisabled();
  });
});
```

### Component with Context
```typescript
import { render, screen } from "@solidjs/testing-library";
import { describe, it, expect } from "vitest";
import { ThemeProvider, useTheme } from "./ThemeContext";
import { ThemedButton } from "./ThemedButton";

const renderWithProviders = (ui: () => JSX.Element) => {
  return render(() => (
    <ThemeProvider>
      {ui()}
    </ThemeProvider>
  ));
};

describe("ThemedButton", () => {
  it("applies theme colors", () => {
    renderWithProviders(() => <ThemedButton>Click</ThemedButton>);

    expect(screen.getByRole("button")).toHaveClass("bg-blue-500");
  });
});
```

### Hook Testing
```typescript
import { renderHook, waitFor } from "@solidjs/testing-library";
import { describe, it, expect, vi } from "vitest";
import { usePositions } from "./usePositions";

// Wrapper with providers
const wrapper = (props: { children: JSX.Element }) => (
  <QueryProvider>
    {props.children}
  </QueryProvider>
);

describe("usePositions", () => {
  it("fetches positions", async () => {
    const { result } = renderHook(() => usePositions(), { wrapper });

    expect(result.loading).toBe(true);

    await waitFor(() => {
      expect(result.loading).toBe(false);
    });

    expect(result.positions()).toHaveLength(2);
  });

  it("handles errors", async () => {
    vi.mocked(fetchPositions).mockRejectedValueOnce(new Error("Network error"));

    const { result } = renderHook(() => usePositions(), { wrapper });

    await waitFor(() => {
      expect(result.error()).toBeTruthy();
    });

    expect(result.error()?.message).toBe("Network error");
  });
});
```

---

## API Testing (Supertest)

### Fastify Route Tests
```typescript
import { describe, it, expect, beforeAll, afterAll } from "vitest";
import { build } from "../src/app";
import { FastifyInstance } from "fastify";

describe("Positions API", () => {
  let app: FastifyInstance;

  beforeAll(async () => {
    app = await build({ logger: false });
    await app.ready();
  });

  afterAll(async () => {
    await app.close();
  });

  describe("GET /api/positions", () => {
    it("returns 401 without auth", async () => {
      const response = await app.inject({
        method: "GET",
        url: "/api/positions",
      });

      expect(response.statusCode).toBe(401);
    });

    it("returns positions for authenticated user", async () => {
      const response = await app.inject({
        method: "GET",
        url: "/api/positions",
        headers: {
          authorization: `Bearer ${testToken}`,
        },
      });

      expect(response.statusCode).toBe(200);
      const body = JSON.parse(response.body);
      expect(body).toHaveProperty("data");
      expect(Array.isArray(body.data)).toBe(true);
    });

    it("filters by status", async () => {
      const response = await app.inject({
        method: "GET",
        url: "/api/positions?status=open",
        headers: { authorization: `Bearer ${testToken}` },
      });

      const body = JSON.parse(response.body);
      expect(body.data.every((p: any) => p.status === "open")).toBe(true);
    });
  });

  describe("POST /api/positions", () => {
    it("creates a position", async () => {
      const response = await app.inject({
        method: "POST",
        url: "/api/positions",
        headers: {
          authorization: `Bearer ${testToken}`,
          "content-type": "application/json",
        },
        payload: {
          symbol: "AAPL",
          quantity: 100,
          entry_price: 150.0,
          position_type: "stock",
        },
      });

      expect(response.statusCode).toBe(201);
      const body = JSON.parse(response.body);
      expect(body.data.symbol).toBe("AAPL");
      expect(body.data.id).toBeDefined();
    });

    it("validates required fields", async () => {
      const response = await app.inject({
        method: "POST",
        url: "/api/positions",
        headers: {
          authorization: `Bearer ${testToken}`,
          "content-type": "application/json",
        },
        payload: {
          symbol: "AAPL",
          // missing quantity, entry_price
        },
      });

      expect(response.statusCode).toBe(400);
      const body = JSON.parse(response.body);
      expect(body.error).toContain("quantity");
    });

    it("rejects invalid symbol", async () => {
      const response = await app.inject({
        method: "POST",
        url: "/api/positions",
        headers: {
          authorization: `Bearer ${testToken}`,
          "content-type": "application/json",
        },
        payload: {
          symbol: "invalid symbol!",
          quantity: 100,
          entry_price: 150.0,
          position_type: "stock",
        },
      });

      expect(response.statusCode).toBe(400);
    });
  });
});
```

### API Contract Testing
```typescript
import { describe, it, expect } from "vitest";
import Ajv from "ajv";
import { positionSchema, errorSchema } from "../schemas";

const ajv = new Ajv();

describe("API Contract: Positions", () => {
  const validatePosition = ajv.compile(positionSchema);
  const validateError = ajv.compile(errorSchema);

  it("GET /positions response matches schema", async () => {
    const response = await fetch("/api/positions", {
      headers: { authorization: `Bearer ${token}` },
    });
    const data = await response.json();

    data.data.forEach((position: any) => {
      const valid = validatePosition(position);
      if (!valid) console.log(validatePosition.errors);
      expect(valid).toBe(true);
    });
  });

  it("error response matches schema", async () => {
    const response = await fetch("/api/positions/invalid-id", {
      headers: { authorization: `Bearer ${token}` },
    });
    const data = await response.json();

    expect(validateError(data)).toBe(true);
  });
});
```

---

## E2E Testing (Playwright)

### Configuration
```typescript
// playwright.config.ts
import { defineConfig, devices } from "@playwright/test";

export default defineConfig({
  testDir: "./tests/e2e",
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: [
    ["html", { open: "never" }],
    ["junit", { outputFile: "results/e2e-results.xml" }],
  ],
  use: {
    baseURL: process.env.BASE_URL ?? "http://localhost:3000",
    trace: "on-first-retry",
    screenshot: "only-on-failure",
  },
  projects: [
    {
      name: "chromium",
      use: { ...devices["Desktop Chrome"] },
    },
    {
      name: "firefox",
      use: { ...devices["Desktop Firefox"] },
    },
    {
      name: "mobile",
      use: { ...devices["iPhone 13"] },
    },
  ],
  webServer: {
    command: "npm run dev",
    url: "http://localhost:3000",
    reuseExistingServer: !process.env.CI,
  },
});
```

### Auth Flow Test
```typescript
import { test, expect } from "@playwright/test";

test.describe("Authentication", () => {
  test("user can sign up", async ({ page }) => {
    await page.goto("/signup");

    await page.fill('[name="email"]', "newuser@example.com");
    await page.fill('[name="password"]', "SecurePass123!");
    await page.fill('[name="confirmPassword"]', "SecurePass123!");

    await page.click('button[type="submit"]');

    // Should redirect to dashboard
    await expect(page).toHaveURL("/dashboard");
    await expect(page.locator("text=Welcome")).toBeVisible();
  });

  test("user can log in", async ({ page }) => {
    await page.goto("/login");

    await page.fill('[name="email"]', "test@example.com");
    await page.fill('[name="password"]', "TestPass123!");

    await page.click('button[type="submit"]');

    await expect(page).toHaveURL("/dashboard");
  });

  test("shows error on invalid credentials", async ({ page }) => {
    await page.goto("/login");

    await page.fill('[name="email"]', "test@example.com");
    await page.fill('[name="password"]', "wrongpassword");

    await page.click('button[type="submit"]');

    await expect(page.locator('[role="alert"]')).toContainText("Invalid credentials");
    await expect(page).toHaveURL("/login");
  });

  test("user can log out", async ({ page }) => {
    // Login first
    await page.goto("/login");
    await page.fill('[name="email"]', "test@example.com");
    await page.fill('[name="password"]', "TestPass123!");
    await page.click('button[type="submit"]');
    await expect(page).toHaveURL("/dashboard");

    // Logout
    await page.click('[data-testid="user-menu"]');
    await page.click('text=Log out');

    await expect(page).toHaveURL("/login");
  });
});
```

### Trading Flow Test
```typescript
import { test, expect } from "@playwright/test";
import { login } from "./helpers";

test.describe("Trading", () => {
  test.beforeEach(async ({ page }) => {
    await login(page, "trader@example.com", "TestPass123!");
  });

  test("user can create a position", async ({ page }) => {
    await page.goto("/positions");

    // Open create modal
    await page.click('button:has-text("Add Position")');

    // Fill form
    await page.fill('[name="symbol"]', "AAPL");
    await page.fill('[name="quantity"]', "100");
    await page.fill('[name="entry_price"]', "150.00");
    await page.selectOption('[name="position_type"]', "stock");

    // Submit
    await page.click('button:has-text("Create")');

    // Verify position appears in list
    await expect(page.locator('text=AAPL')).toBeVisible();
    await expect(page.locator('text=100 shares')).toBeVisible();
  });

  test("user can close a position", async ({ page }) => {
    await page.goto("/positions");

    // Click on existing position
    await page.click('[data-testid="position-AAPL"]');

    // Click close button
    await page.click('button:has-text("Close Position")');

    // Confirm
    await page.fill('[name="exit_price"]', "155.00");
    await page.click('button:has-text("Confirm Close")');

    // Verify position is closed
    await expect(page.locator('[data-testid="position-AAPL"] [data-status="closed"]')).toBeVisible();
  });

  test("position list updates in real-time", async ({ page }) => {
    await page.goto("/positions");

    const initialCount = await page.locator('[data-testid^="position-"]').count();

    // Simulate real-time update (via API)
    await page.evaluate(async () => {
      await fetch("/api/test/simulate-position-update", { method: "POST" });
    });

    // Wait for real-time update
    await expect(page.locator('[data-testid^="position-"]')).toHaveCount(initialCount + 1, {
      timeout: 5000,
    });
  });
});
```

### Page Object Model
```typescript
// tests/e2e/pages/PositionsPage.ts
import { Page, Locator } from "@playwright/test";

export class PositionsPage {
  readonly page: Page;
  readonly addButton: Locator;
  readonly positionsList: Locator;
  readonly symbolInput: Locator;
  readonly quantityInput: Locator;
  readonly submitButton: Locator;

  constructor(page: Page) {
    this.page = page;
    this.addButton = page.locator('button:has-text("Add Position")');
    this.positionsList = page.locator('[data-testid="positions-list"]');
    this.symbolInput = page.locator('[name="symbol"]');
    this.quantityInput = page.locator('[name="quantity"]');
    this.submitButton = page.locator('button[type="submit"]');
  }

  async goto() {
    await this.page.goto("/positions");
  }

  async createPosition(symbol: string, quantity: number, entryPrice: number) {
    await this.addButton.click();
    await this.symbolInput.fill(symbol);
    await this.quantityInput.fill(String(quantity));
    await this.page.fill('[name="entry_price"]', String(entryPrice));
    await this.submitButton.click();
  }

  async getPositionCount(): Promise<number> {
    return await this.positionsList.locator('[data-testid^="position-"]').count();
  }

  async getPosition(symbol: string): Promise<Locator> {
    return this.page.locator(`[data-testid="position-${symbol}"]`);
  }
}

// Usage in test
test("create position", async ({ page }) => {
  const positionsPage = new PositionsPage(page);
  await positionsPage.goto();
  await positionsPage.createPosition("AAPL", 100, 150);

  const position = await positionsPage.getPosition("AAPL");
  await expect(position).toBeVisible();
});
```

---

## Mocking (MSW)

### Setup
```typescript
// tests/mocks/handlers.ts
import { rest } from "msw";

export const handlers = [
  rest.get("/api/positions", (req, res, ctx) => {
    return res(
      ctx.json({
        data: [
          { id: "1", symbol: "AAPL", quantity: 100, entry_price: 150 },
          { id: "2", symbol: "MSFT", quantity: 50, entry_price: 300 },
        ],
      })
    );
  }),

  rest.post("/api/positions", async (req, res, ctx) => {
    const body = await req.json();
    return res(
      ctx.status(201),
      ctx.json({
        data: { id: "new-id", ...body },
      })
    );
  }),

  rest.delete("/api/positions/:id", (req, res, ctx) => {
    return res(ctx.status(204));
  }),
];

// tests/mocks/server.ts
import { setupServer } from "msw/node";
import { handlers } from "./handlers";

export const server = setupServer(...handlers);

// tests/setup.ts
import { beforeAll, afterAll, afterEach } from "vitest";
import { server } from "./mocks/server";

beforeAll(() => server.listen({ onUnhandledRequest: "error" }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

### Override Handlers in Tests
```typescript
import { rest } from "msw";
import { server } from "../mocks/server";

describe("PositionsList", () => {
  it("shows error state", async () => {
    server.use(
      rest.get("/api/positions", (req, res, ctx) => {
        return res(ctx.status(500), ctx.json({ error: "Server error" }));
      })
    );

    render(() => <PositionsList />);

    await waitFor(() => {
      expect(screen.getByText("Failed to load positions")).toBeInTheDocument();
    });
  });

  it("shows empty state", async () => {
    server.use(
      rest.get("/api/positions", (req, res, ctx) => {
        return res(ctx.json({ data: [] }));
      })
    );

    render(() => <PositionsList />);

    await waitFor(() => {
      expect(screen.getByText("No positions yet")).toBeInTheDocument();
    });
  });
});
```

---

## CI Integration

### GitHub Actions Workflow
```yaml
# .github/workflows/test.yml
name: Tests

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"

      - run: npm ci

      - name: Run unit tests
        run: npm run test:unit -- --coverage

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/lcov.info
          fail_ci_if_error: true

  integration-tests:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: supabase/postgres:15
        env:
          POSTGRES_PASSWORD: postgres
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"

      - run: npm ci

      - name: Run migrations
        run: npm run db:migrate
        env:
          DATABASE_URL: postgres://postgres:postgres@localhost:5432/test

      - name: Run integration tests
        run: npm run test:integration
        env:
          DATABASE_URL: postgres://postgres:postgres@localhost:5432/test

  e2e-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"

      - run: npm ci

      - name: Install Playwright
        run: npx playwright install --with-deps chromium

      - name: Run E2E tests
        run: npm run test:e2e

      - name: Upload test results
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 7
```

### Package.json Scripts
```json
{
  "scripts": {
    "test": "vitest",
    "test:unit": "vitest run --config vitest.config.ts",
    "test:integration": "vitest run --config vitest.integration.config.ts",
    "test:e2e": "playwright test",
    "test:e2e:ui": "playwright test --ui",
    "test:coverage": "vitest run --coverage",
    "test:watch": "vitest --watch"
  }
}
```

---

## Test Fixtures

### Factory Functions
```typescript
// tests/fixtures/factories.ts
import { faker } from "@faker-js/faker";

export const createPosition = (overrides: Partial<Position> = {}): Position => ({
  id: faker.string.uuid(),
  user_id: faker.string.uuid(),
  symbol: faker.helpers.arrayElement(["AAPL", "MSFT", "GOOGL", "AMZN"]),
  quantity: faker.number.int({ min: 1, max: 1000 }),
  entry_price: faker.number.float({ min: 10, max: 500, precision: 0.01 }),
  position_type: faker.helpers.arrayElement(["stock", "call", "put"]),
  status: "open",
  created_at: faker.date.recent().toISOString(),
  updated_at: faker.date.recent().toISOString(),
  ...overrides,
});

export const createUser = (overrides: Partial<User> = {}): User => ({
  id: faker.string.uuid(),
  email: faker.internet.email(),
  name: faker.person.fullName(),
  created_at: faker.date.past().toISOString(),
  ...overrides,
});

export const createTrade = (overrides: Partial<Trade> = {}): Trade => ({
  id: faker.string.uuid(),
  position_id: faker.string.uuid(),
  side: faker.helpers.arrayElement(["buy", "sell"]),
  quantity: faker.number.int({ min: 1, max: 100 }),
  price: faker.number.float({ min: 10, max: 500, precision: 0.01 }),
  executed_at: faker.date.recent().toISOString(),
  ...overrides,
});

// Batch creation
export const createPositions = (count: number, overrides: Partial<Position> = {}): Position[] =>
  Array.from({ length: count }, () => createPosition(overrides));
```

### Database Seeding
```typescript
// tests/fixtures/seed.ts
import { createPosition, createUser } from "./factories";
import { supabase } from "../setup";

export async function seedTestData() {
  // Create test user
  const user = createUser({ email: "test@example.com" });
  await supabase.from("profiles").insert(user);

  // Create positions
  const positions = createPositions(5, { user_id: user.id });
  await supabase.from("positions").insert(positions);

  return { user, positions };
}

export async function cleanupTestData() {
  await supabase.from("positions").delete().neq("id", "");
  await supabase.from("profiles").delete().neq("id", "");
}
```

---

## Process
1. **Define test strategy** — What needs testing at each level?
2. **Set up frameworks** — Vitest, Playwright, MSW
3. **Write fixtures** — Factories, seeders, mocks
4. **Implement tests** — Unit → Integration → E2E
5. **Configure CI** — GitHub Actions pipeline
6. **Monitor coverage** — Track and enforce thresholds

## Output
1. Test configuration files
2. Unit test suites
3. Integration test suites
4. E2E test specs
5. Mock handlers
6. CI workflow
7. Coverage reports
