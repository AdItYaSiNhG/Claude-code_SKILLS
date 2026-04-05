---
name: testing
description: >
  Use this skill for all testing and QA tasks: unit test generation, integration test
  scaffolding, mock & stub generation, test coverage enforcement, flaky test detection,
  E2E tests (Playwright/Cypress), load & stress tests (k6/Locust), contract testing (Pact),
  snapshot testing, visual regression, test data factories, mutation testing, chaos
  engineering scripts, and test report summarisation.
  Triggers: "write tests", "add unit tests", "integration test", "E2E", "test coverage",
  "flaky tests", "load test", "contract test", "mutation test", "test report".
compatibility: "Claude Code (claude.ai, Claude Desktop) — requires bash/filesystem access"
version: "1.0.0"
---

# Testing & QA Skill

## Why this skill exists

Tests written inconsistently provide false confidence. This skill enforces
**structured, layered test generation** across the testing pyramid: unit →
integration → E2E → performance → chaos. Each layer has its own conventions,
tooling, and failure modes.

Always: read the source before generating tests. Match the project's existing
test framework, file naming, and assertion style.

---

## 0. Test framework detection

```bash
# Detect existing test setup
cat package.json 2>/dev/null \
  | python3 -c "
import json, sys
d = json.load(sys.stdin)
devdeps = d.get('devDependencies', {})
scripts = d.get('scripts', {})
frameworks = [k for k in devdeps if k in
  ('jest', 'vitest', 'mocha', 'jasmine', '@playwright/test',
   'cypress', 'supertest', '@testing-library/react')]
print('Frameworks:', frameworks)
print('Test scripts:', {k:v for k,v in scripts.items() if 'test' in k})
"

# Python
cat pyproject.toml 2>/dev/null | grep -E "pytest|unittest|nose|hypothesis"
cat pytest.ini setup.cfg 2>/dev/null | head -20

# Find existing test files to match style
find . -name "*.test.ts" -o -name "*.spec.ts" \
  -o -name "*_test.py" -o -name "test_*.py" \
  -o -name "*_test.go" \
  | grep -v node_modules | head -10 \
  | xargs head -30 2>/dev/null
```

---

## 1. Unit test generation

### JavaScript / TypeScript (Jest / Vitest)

**Read the source file first:**
```bash
cat src/users/userService.ts  # or the file to test
```

**Template for a service class:**
```typescript
// src/users/userService.test.ts
import { describe, it, expect, beforeEach, vi } from "vitest"; // or jest
import { UserService } from "./userService";
import { UserRepository } from "./userRepository";

// Mock the dependency — never touch the real DB in unit tests
vi.mock("./userRepository");

describe("UserService", () => {
  let service: UserService;
  let mockRepo: jest.Mocked<UserRepository>;

  beforeEach(() => {
    vi.clearAllMocks();
    mockRepo = new UserRepository() as jest.Mocked<UserRepository>;
    service = new UserService(mockRepo);
  });

  describe("getUserById", () => {
    it("returns user when found", async () => {
      // Arrange
      const user = { id: "1", name: "Alice", email: "alice@example.com" };
      mockRepo.findById.mockResolvedValueOnce(user);

      // Act
      const result = await service.getUserById("1");

      // Assert
      expect(result).toEqual(user);
      expect(mockRepo.findById).toHaveBeenCalledWith("1");
      expect(mockRepo.findById).toHaveBeenCalledTimes(1);
    });

    it("throws NotFoundError when user does not exist", async () => {
      mockRepo.findById.mockResolvedValueOnce(null);
      await expect(service.getUserById("999")).rejects.toThrow("User not found");
    });

    it("propagates repository errors", async () => {
      mockRepo.findById.mockRejectedValueOnce(new Error("DB connection failed"));
      await expect(service.getUserById("1")).rejects.toThrow("DB connection failed");
    });
  });
});
```

**Coverage targets:**
- Happy path (expected input → expected output)
- Edge cases: null/undefined, empty string, 0, negative numbers, max values
- Error paths: dependency throws, validation fails, not found
- Boundary conditions: at the limit, just over the limit

### Python (pytest)

```python
# tests/test_user_service.py
import pytest
from unittest.mock import AsyncMock, MagicMock, patch
from app.users.service import UserService
from app.users.schemas import UserCreate


@pytest.fixture
def mock_repo():
    repo = MagicMock()
    repo.find_by_id = AsyncMock()
    repo.save = AsyncMock()
    return repo


@pytest.fixture
def service(mock_repo):
    return UserService(repo=mock_repo)


class TestGetUserById:
    async def test_returns_user_when_found(self, service, mock_repo):
        user = {"id": "1", "name": "Alice"}
        mock_repo.find_by_id.return_value = user

        result = await service.get_user_by_id("1")

        assert result == user
        mock_repo.find_by_id.assert_called_once_with("1")

    async def test_raises_not_found_when_missing(self, service, mock_repo):
        mock_repo.find_by_id.return_value = None
        with pytest.raises(ValueError, match="User not found"):
            await service.get_user_by_id("999")

    @pytest.mark.parametrize("user_id", ["", None, "   "])
    async def test_raises_on_invalid_id(self, service, user_id):
        with pytest.raises(ValueError):
            await service.get_user_by_id(user_id)
```

### Go
```go
// internal/users/service_test.go
package users_test

import (
    "context"
    "errors"
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/mock"
)

type MockRepository struct{ mock.Mock }
func (m *MockRepository) FindByID(ctx context.Context, id string) (*User, error) {
    args := m.Called(ctx, id)
    if args.Get(0) == nil { return nil, args.Error(1) }
    return args.Get(0).(*User), args.Error(1)
}

func TestGetUserByID_ReturnsUser(t *testing.T) {
    repo := new(MockRepository)
    svc := NewUserService(repo)
    expected := &User{ID: "1", Name: "Alice"}
    repo.On("FindByID", mock.Anything, "1").Return(expected, nil)

    result, err := svc.GetUserByID(context.Background(), "1")

    assert.NoError(t, err)
    assert.Equal(t, expected, result)
    repo.AssertExpectations(t)
}

func TestGetUserByID_NotFound(t *testing.T) {
    repo := new(MockRepository)
    svc := NewUserService(repo)
    repo.On("FindByID", mock.Anything, "99").Return(nil, ErrNotFound)

    _, err := svc.GetUserByID(context.Background(), "99")
    assert.True(t, errors.Is(err, ErrNotFound))
}
```

---

## 2. Integration test scaffolding

Integration tests use real (or near-real) dependencies:

```typescript
// src/users/userRepository.integration.test.ts
import { describe, it, expect, beforeAll, afterAll, beforeEach } from "vitest";
import { db } from "../db/client";           // real DB connection
import { UserRepository } from "./userRepository";
import { runMigrations, clearTable } from "../db/testHelpers";

describe("UserRepository (integration)", () => {
  let repo: UserRepository;

  beforeAll(async () => {
    await runMigrations();                   // run against test DB
  });

  beforeEach(async () => {
    await clearTable("users");               // clean state per test
    repo = new UserRepository(db);
  });

  afterAll(async () => {
    await db.destroy();
  });

  it("saves and retrieves a user", async () => {
    const created = await repo.save({ name: "Bob", email: "bob@test.com" });
    expect(created.id).toBeDefined();

    const found = await repo.findById(created.id);
    expect(found).toMatchObject({ name: "Bob", email: "bob@test.com" });
  });
});
```

**Test DB setup helper:**
```bash
# Confirm TEST_DATABASE_URL is set (never run against prod)
echo "Test DB: $TEST_DATABASE_URL"
# Should point to a separate DB, e.g. postgres://localhost:5432/myapp_test
```

---

## 3. Mock & stub generation

```bash
# Auto-generate mocks from TypeScript interfaces (ts-mockito / jest)
# Read interfaces first
grep -n "export interface\|export type\|export abstract class" src/**/*.ts \
  | grep -v node_modules | head -20
```

**Mock factory pattern (reusable across tests):**
```typescript
// tests/factories/userFactory.ts
import { faker } from "@faker-js/faker";
import type { User } from "../../src/users/types";

export const createUser = (overrides: Partial<User> = {}): User => ({
  id: faker.string.uuid(),
  name: faker.person.fullName(),
  email: faker.internet.email(),
  createdAt: faker.date.past(),
  role: "user",
  ...overrides,
});

export const createAdmin = (overrides: Partial<User> = {}) =>
  createUser({ role: "admin", ...overrides });
```

---

## 4. Test coverage enforcement

```bash
# JavaScript / TypeScript (Jest)
npx jest --coverage --coverageThreshold='{"global":{"lines":80,"branches":70}}' 2>/dev/null

# Show uncovered lines
npx jest --coverage --coverageReporters=text 2>/dev/null \
  | grep -E "Uncovered|^\s+[0-9].*\|.*\|" | head -40

# Python
pytest --cov=app --cov-report=term-missing --cov-fail-under=80 2>/dev/null

# Go
go test ./... -coverprofile=coverage.out 2>/dev/null \
  && go tool cover -func=coverage.out \
  | awk '$3 < "80.0%" && $1 != "total:" {print $0}' | head -30

# Find files with 0% coverage
go tool cover -func=coverage.out | grep "0.0%" | head -20
```

**Coverage thresholds (enterprise minimum):**
| Layer | Lines | Branches |
|-------|-------|---------|
| Core business logic | 90% | 85% |
| Services | 80% | 70% |
| Controllers/handlers | 70% | 60% |
| Utilities | 85% | 75% |
| Infrastructure (DB, HTTP) | 50% (integration tested) | — |

---

## 5. Flaky test detection

```bash
# Run tests N times and detect non-deterministic failures
for i in $(seq 1 5); do
  echo "Run $i:"
  npx jest --forceExit 2>/dev/null | tail -5
done

# Jest repeat-each plugin
npx jest --repeat-each=3 2>/dev/null | grep -E "FAIL|PASS|✓|✗" | head -30

# Find tests with timing dependencies
grep -rn "setTimeout\|setInterval\|sleep\|delay\|Date\.now\|new Date()" \
  --include="*.test.ts" --include="*.spec.ts" . \
  | grep -v "node_modules" | head -20

# Find tests that share state (likely cause of flakiness)
grep -rn "let\s\+[a-z]" --include="*.test.ts" . \
  | grep -v "const\|beforeEach\|afterEach" | grep -v node_modules | head -20
```

**Common flakiness causes:**
- Shared mutable state between tests (use `beforeEach` to reset)
- `setTimeout`/timer-based assertions (use `vi.useFakeTimers()`)
- Network requests without mocking
- Tests relying on execution order
- Non-deterministic `Date.now()` comparisons

---

## 6. E2E tests (Playwright)

```typescript
// tests/e2e/login.spec.ts
import { test, expect } from "@playwright/test";

test.describe("Login flow", () => {
  test.beforeEach(async ({ page }) => {
    await page.goto("/login");
  });

  test("successful login redirects to dashboard", async ({ page }) => {
    await page.getByLabel("Email").fill("user@test.com");
    await page.getByLabel("Password").fill("password123");
    await page.getByRole("button", { name: "Sign in" }).click();

    await expect(page).toHaveURL("/dashboard");
    await expect(page.getByRole("heading", { name: "Dashboard" })).toBeVisible();
  });

  test("invalid credentials shows error", async ({ page }) => {
    await page.getByLabel("Email").fill("wrong@test.com");
    await page.getByLabel("Password").fill("wrongpass");
    await page.getByRole("button", { name: "Sign in" }).click();

    await expect(page.getByRole("alert")).toContainText("Invalid credentials");
    await expect(page).toHaveURL("/login");
  });

  test("empty form shows validation errors", async ({ page }) => {
    await page.getByRole("button", { name: "Sign in" }).click();
    await expect(page.getByText("Email is required")).toBeVisible();
    await expect(page.getByText("Password is required")).toBeVisible();
  });
});
```

```bash
# Run E2E tests
npx playwright test 2>/dev/null

# Run with UI for debugging
npx playwright test --ui 2>/dev/null

# Generate test from recording
npx playwright codegen http://localhost:3000 2>/dev/null
```

---

## 7. Load & stress tests (k6)

```javascript
// tests/load/api-load-test.js
import http from "k6/http";
import { check, sleep } from "k6";
import { Rate, Trend } from "k6/metrics";

const errorRate = new Rate("errors");
const responseTime = new Trend("response_time_ms");

export const options = {
  stages: [
    { duration: "30s", target: 10 },    // ramp up
    { duration: "1m",  target: 50 },    // sustained load
    { duration: "30s", target: 100 },   // stress
    { duration: "30s", target: 0 },     // ramp down
  ],
  thresholds: {
    http_req_duration: ["p(95)<500"],   // 95th percentile < 500ms
    errors:            ["rate<0.01"],   // < 1% error rate
  },
};

export default function () {
  const res = http.get(`${__ENV.BASE_URL}/api/users`, {
    headers: { Authorization: `Bearer ${__ENV.API_TOKEN}` },
  });

  const ok = check(res, {
    "status is 200":       (r) => r.status === 200,
    "response time < 500": (r) => r.timings.duration < 500,
    "has data array":      (r) => JSON.parse(r.body).data !== undefined,
  });

  errorRate.add(!ok);
  responseTime.add(res.timings.duration);
  sleep(1);
}
```

```bash
# Run k6 load test
k6 run --env BASE_URL=http://localhost:3000 \
        --env API_TOKEN=your-token \
        tests/load/api-load-test.js 2>/dev/null
```

---

## 8. Contract testing (Pact)

```typescript
// tests/contract/userApi.consumer.test.ts (consumer side)
import { PactV3, MatchersV3 } from "@pact-foundation/pact";
import { getUserById } from "../../src/clients/userApiClient";

const { like, string } = MatchersV3;

describe("User API Consumer", () => {
  const provider = new PactV3({
    consumer: "frontend-app",
    provider: "user-service",
    dir: "./pacts",
  });

  it("returns a user by ID", async () => {
    await provider
      .given("user with ID 1 exists")
      .uponReceiving("a request for user 1")
      .withRequest({ method: "GET", path: "/users/1" })
      .willRespondWith({
        status: 200,
        body: {
          id: like("1"),
          name: like("Alice"),
          email: string("alice@example.com"),
        },
      })
      .executeTest(async (mockServer) => {
        const user = await getUserById("1", mockServer.url);
        expect(user.name).toBeTruthy();
      });
  });
});
```

---

## 9. Snapshot testing

```typescript
// When to use: UI components, API response shapes, serialised output
import { render } from "@testing-library/react";
import { UserCard } from "./UserCard";

it("renders user card correctly", () => {
  const { asFragment } = render(
    <UserCard user={{ id: "1", name: "Alice", role: "admin" }} />
  );
  expect(asFragment()).toMatchSnapshot();
});
```

```bash
# Update snapshots after intentional change
npx jest --updateSnapshot 2>/dev/null

# Review snapshot changes in CI
npx jest --ci 2>/dev/null   # fails if snapshots are outdated
```

---

## 10. Mutation testing

Validates that your tests actually catch bugs:

```bash
# JavaScript / TypeScript — Stryker
npx stryker run 2>/dev/null | tail -30

# Python — mutmut
mutmut run --paths-to-mutate src/ 2>/dev/null
mutmut results 2>/dev/null | head -30

# Interpret: mutation score = % of mutations caught
# Target: > 75%. Score < 50% means tests are not verifying logic.
```

---

## 11. Test data factories

```typescript
// tests/factories/index.ts — centralise all test data creation
import { faker } from "@faker-js/faker";

export const factories = {
  user: (overrides = {}) => ({
    id:        faker.string.uuid(),
    name:      faker.person.fullName(),
    email:     faker.internet.email(),
    createdAt: faker.date.past().toISOString(),
    role:      "user" as const,
    ...overrides,
  }),

  order: (overrides = {}) => ({
    id:        faker.string.uuid(),
    userId:    faker.string.uuid(),
    total:     faker.number.float({ min: 1, max: 1000, fractionDigits: 2 }),
    status:    "pending" as const,
    items:     [],
    ...overrides,
  }),

  // Build a full graph
  orderWithUser: () => {
    const user = factories.user();
    return { user, order: factories.order({ userId: user.id }) };
  },
};
```

---

## 12. Test report summarisation

```bash
# Jest — JSON report
npx jest --json --outputFile=/tmp/jest-results.json 2>/dev/null
cat /tmp/jest-results.json | python3 -c "
import json, sys
d = json.load(sys.stdin)
print(f\"Tests:   {d['numTotalTests']} total\")
print(f\"Passed:  {d['numPassedTests']}\")
print(f\"Failed:  {d['numFailedTests']}\")
print(f\"Skipped: {d['numPendingTests']}\")
print(f\"Time:    {d['testResults'][0]['perfStats']['runtime'] if d['testResults'] else 0}ms\")
if d['numFailedTests']:
    print('\nFailed tests:')
    for suite in d['testResults']:
        for t in suite.get('testResults', []):
            if t['status'] == 'failed':
                print(f\"  ✗ {suite['testFilePath'].split('/')[-1]} > {t['title']}\")
                for msg in t['failureMessages'][:1]:
                    print(f\"    {msg[:120]}\")
"

# pytest — JUnit XML
pytest --junitxml=/tmp/pytest-results.xml 2>/dev/null
python3 - <<'EOF'
import xml.etree.ElementTree as ET
tree = ET.parse("/tmp/pytest-results.xml")
root = tree.getroot()
suite = root.find("testsuite") or root
attrs = suite.attrib
print(f"Tests: {attrs.get('tests','?')}  Errors: {attrs.get('errors','?')}  Failures: {attrs.get('failures','?')}  Skipped: {attrs.get('skipped','?')}")
for tc in suite.findall(".//testcase"):
    failure = tc.find("failure")
    if failure is not None:
        print(f"  ✗ {tc.get('classname','')}.{tc.get('name','')}")
        print(f"    {(failure.text or '')[:120]}")
EOF
```

---

## Quick-reference: test file naming conventions

| Language | Unit | Integration | E2E |
|----------|------|------------|-----|
| TypeScript | `*.test.ts` / `*.spec.ts` | `*.integration.test.ts` | `tests/e2e/*.spec.ts` |
| Python | `test_*.py` | `test_*_integration.py` | `tests/e2e/test_*.py` |
| Go | `*_test.go` (same package) | `*_integration_test.go` | — |
| Java | `*Test.java` | `*IT.java` | — |
