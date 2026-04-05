---
name: tdd
description: >
  Drive feature development using Test-Driven Development: write a failing test first,
  implement the minimum code to pass it, then refactor. Use when the user wants to
  build something using TDD, or when implementing a feature where correctness is critical.
  Triggers: "TDD", "test driven", "write tests first", "red green refactor",
  "failing test first", "implement with TDD".
source: adapted from mattpocock/skills
version: "1.0.0"
---

# TDD Skill

## The TDD cycle

**Red → Green → Refactor**. Never skip a step. Never write implementation before
the test exists and fails.

---

## Step 1 — Understand what to build
Before writing any test:
- Read the spec, PRD, or issue
- Identify the public interface (function signature, API endpoint, component props)
- List the behaviours to test (happy path + 3 edge cases minimum)

## Step 2 — Write a failing test (Red)

```typescript
// Write the test BEFORE the implementation exists
// The test should fail because the function doesn't exist yet

import { describe, it, expect } from "vitest";
import { calculateDiscount } from "./pricing";  // doesn't exist yet — good

describe("calculateDiscount", () => {
  it("applies 10% to orders over £100", () => {
    expect(calculateDiscount(150, "REGULAR")).toBe(135);
  });
});
```

Run it and confirm it fails:
```bash
npx vitest run src/pricing.test.ts 2>/dev/null
# Expected: FAIL — function not found
```

## Step 3 — Write minimum code to pass (Green)

Write the **simplest possible code** that makes the test pass. Do not over-engineer.

```typescript
// src/pricing.ts
export function calculateDiscount(total: number, tier: string): number {
  if (total > 100) return total * 0.9;
  return total;
}
```

Run and confirm it passes:
```bash
npx vitest run src/pricing.test.ts
# Expected: PASS
```

## Step 4 — Add the next failing test

Add one more test case. Run. See it fail. Fix. Repeat.

```typescript
it("does not discount orders under £100", () => {
  expect(calculateDiscount(80, "REGULAR")).toBe(80);
});

it("applies 20% for VIP tier over £100", () => {
  expect(calculateDiscount(150, "VIP")).toBe(120);
});

it("throws on negative total", () => {
  expect(() => calculateDiscount(-10, "REGULAR")).toThrow("Total must be positive");
});
```

## Step 5 — Refactor (with tests green)

Only refactor when all tests pass. Never change behaviour during refactor.

```bash
# Tests must stay green after every refactor step
npx vitest run 2>/dev/null
```

## Step 6 — Commit after each Red→Green→Refactor cycle

```bash
git add -p
git commit -m "feat: calculateDiscount applies tier-based discounts

- 10% for REGULAR tier over £100
- 20% for VIP tier over £100
- throws on negative total"
```

---

## TDD rules
1. Never write production code without a failing test
2. Write only enough code to pass the current test
3. Don't add functionality that no test requires
4. Each commit = one complete Red→Green→Refactor cycle
5. Test the interface, not the implementation

## Python equivalent
```python
# test_pricing.py — write first
import pytest
from pricing import calculate_discount

def test_applies_10_percent_over_100():
    assert calculate_discount(150, "REGULAR") == 135.0

def test_no_discount_under_100():
    assert calculate_discount(80, "REGULAR") == 80.0
```

```bash
pytest test_pricing.py -v 2>/dev/null
# Should FAIL first — then implement
```
