---
name: scaffold-exercises
description: >
  Create exercise directory structures with sections, problems, solutions, and
  explainers for teaching programming concepts. Use when building a course,
  workshop, or tutorial that requires learners to solve problems with provided
  solutions.
  Triggers: "scaffold exercises", "create exercises", "build a course structure",
  "workshop exercises", "tutorial problems", "exercise directory", "create problems
  and solutions", "teaching materials".
source: adapted from mattpocock/skills
version: "1.0.0"
---

# Scaffold Exercises Skill

## Exercise structure

```
exercises/
├── 01-section-name/
│   ├── 01-problem-name/
│   │   ├── problem.ts          # the starting point — has TODOs
│   │   ├── problem.test.ts     # tests that FAIL until solved
│   │   ├── solution.ts         # the completed solution
│   │   └── explainer.md        # why the solution works
│   ├── 02-next-problem/
│   │   └── ...
│   └── README.md               # section overview
└── README.md                   # course overview
```

---

## Step 1 — Get the course spec from the user

Ask:
- What is the subject / technology being taught?
- Who is the audience? (beginner / intermediate / advanced)
- How many sections and exercises per section?
- What is the progression? (what concepts build on what)

---

## Step 2 — Plan the learning path

Design exercises so each one builds on the last. Avoid introducing two new
concepts in one exercise.

```
Section 01: Basics
  01: simple task (one concept)
  02: slightly harder (same concept, new context)
  03: combined (first + second concept together)

Section 02: Next concept
  01: new concept in isolation
  ...
```

---

## Step 3 — Scaffold the directories

```bash
# Get variables from the user
COURSE_ROOT="exercises"
SECTIONS=("01-typescript-basics" "02-generics" "03-utility-types")

for section in "${SECTIONS[@]}"; do
  mkdir -p "$COURSE_ROOT/$section"
  cat > "$COURSE_ROOT/$section/README.md" << EOF
# ${section#*-}

## What you'll learn
[Section objectives]

## Exercises
[List each exercise with a one-line description]
EOF
done

# Create exercise template
create_exercise() {
  local dir="$1"
  local name="$2"
  mkdir -p "$dir"

  # Problem file — where learners work
  cat > "$dir/problem.ts" << EOF
// Exercise: $name
// TODO: Your implementation here

export {}
EOF

  # Tests that drive the exercise
  cat > "$dir/problem.test.ts" << EOF
import { describe, it, expect } from "vitest"
// TODO: import the function you implement

describe("$name", () => {
  it("should [describe expected behaviour]", () => {
    // TODO: add test assertion
    expect(true).toBe(true) // replace this
  })
})
EOF

  # Solution file
  cat > "$dir/solution.ts" << EOF
// Solution: $name
// [Explanation of approach]

export {}
EOF

  # Explainer
  cat > "$dir/explainer.md" << EOF
# $name — Explainer

## The problem
[What the exercise asks]

## The solution
[Walk through the solution step by step]

## Key concept
[The one thing the learner should take away]

## Common mistakes
[What learners typically get wrong]

## Further reading
- [Link to docs]
EOF
}

# Example: create first exercise
create_exercise "$COURSE_ROOT/01-typescript-basics/01-type-annotations" "Type Annotations"
```

---

## Step 4 — Write the actual exercises

For each exercise, create:

**problem.ts** — clear TODO comments, no solution code:
```typescript
// Exercise: Create a function that sums an array of numbers
// It should handle empty arrays by returning 0

// TODO: Implement sumArray
export function sumArray(numbers: number[]): number {
  throw new Error("Not implemented")
}
```

**problem.test.ts** — tests that are RED until the exercise is solved:
```typescript
import { describe, it, expect } from "vitest"
import { sumArray } from "./problem"

describe("sumArray", () => {
  it("sums a list of numbers", () => {
    expect(sumArray([1, 2, 3])).toBe(6)
  })
  it("returns 0 for empty array", () => {
    expect(sumArray([])).toBe(0)
  })
  it("handles negative numbers", () => {
    expect(sumArray([-1, 1])).toBe(0)
  })
})
```

**solution.ts** — fully worked solution:
```typescript
export function sumArray(numbers: number[]): number {
  return numbers.reduce((sum, n) => sum + n, 0)
}
```

---

## Step 5 — Root README

```bash
cat > "$COURSE_ROOT/README.md" << 'EOF'
# [Course Name]

## Getting started
```bash
npm install
npm run test:watch  # runs tests for all exercises in watch mode
```

## How to use these exercises
1. Open `01-section/01-exercise/problem.ts`
2. Read the comments — they explain what to implement
3. Run the tests: `npx vitest run 01-section/01-exercise`
4. Make the tests pass
5. Check `solution.ts` if you get stuck
6. Read `explainer.md` to understand the concepts

## Course structure
- **Section 01** — [Topic]: [What you'll learn]
- **Section 02** — [Topic]: [What you'll learn]
EOF
```

---

## Step 6 — Add vitest config

```bash
cat > vitest.config.ts << 'EOF'
import { defineConfig } from "vitest/config"

export default defineConfig({
  test: {
    include: ["exercises/**/problem.test.ts"],
    exclude: ["**/solution*"],
  },
})
EOF
```
